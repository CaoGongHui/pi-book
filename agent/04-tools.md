# 第 4 章 · 工具：一次调用的三段管线

第 2 章跟着一次 `prompt()` 走到"执行工具"时，只看了主路：prepare、execute、finalize 三段，各引了一段代码。这一章把工具单独拎出来，回答三个问题：

1. 一个工具在这个包里长什么样，哪些字段给模型看，哪些给程序看？
2. 一次工具调用从模型发出到变成 `toolResult` 消息，中间每一步做了什么，会在哪里失败？
3. 一批调用并发时，事件和消息各按什么顺序出来？

## 两张脸：声明与执行体

先看类型。`AgentTool` 继承自 pi-ai 的 `Tool`：

```typescript
// pi-ai/src/types.ts:754
export interface Tool<TParameters extends TSchema = TSchema> {
	name: string;
	description: string;
	parameters: TParameters;
	constrainedSampling?: false | ConstrainedSamplingConfig;
}
```

```typescript
// src/types.ts:466（删减：JSDoc 只留首句）
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
	/** Human-readable label for UI display. */
	label: string;
	/** Optional compatibility shim for raw tool-call arguments before schema validation. ... */
	prepareArguments?: (args: unknown) => Static<TParameters>;
	/** JSON Schema of `structuredContent` in successful results. ... */
	outputSchema?: TSchema;
	/** Execute the tool call. Throw on failure, or return a result with `isError: true`; ... */
	execute: (
		toolCallId: string,
		params: Static<TParameters>,
		signal?: AbortSignal,
		onUpdate?: AgentToolUpdateCallback<TDetails>,
	) => Promise<AgentToolResult<TDetails>>;
	/** Recovery policy for an effect whose durable intent exists but whose outcome is unknown. */
	replay?: "never" | "safe";
	/** Per-tool execution mode override. ... */
	executionMode?: ToolExecutionMode;
}
```

父类的四个字段是**给模型看的声明**：名字、描述、参数 schema。子类加的六个字段模型一个都看不到：

| 字段 | 谁用 | 作用 |
|---|---|---|
| `label` | UI | 人读的名字，模型看的是 `name` |
| `prepareArguments` | 循环 | 校验前修补模型发来的参数 |
| `outputSchema` | 程序调用方 | 声明 `structuredContent` 的形状 |
| `execute` | 循环 | 真正干活的函数 |
| `replay` | 持久化运行时 | 中断后能不能重跑（本包不读） |
| `executionMode` | 循环 | 这个工具能不能和别人并发 |

第 3 章讲过 `toToolDeclaration`：工具进入消息记录之前，子类字段全被剥掉，只剩父类那四个。所以一个 `AgentTool` 有两张脸，**声明那张进日志、给模型；执行体那张留在内存、给循环**。执行集（`context.tools`）存的是完整的 `AgentTool`，消息记录里存的是剥过皮的 `Tool`。

`replay` 是个特例：这个包的循环从不读它。它是给 pi-durable 那类持久化运行时准备的——进程在工具执行到一半时死掉，重启后不知道副作用发生了没有，`"safe"` 表示可以重跑。pi-durable 自己的工具执行逻辑里有对应的判断（`packages/durable/src/harness/tool.ts:185`：存下的策略和当前工具的策略都是 `safe` 才重跑）。字段留在本包的类型里，是为了让同一个工具定义能在两种运行时之间通用。

## 结果的两个读者

`execute` 返回的结果同样分两半：

```typescript
// src/types.ts:424（删减：JSDoc 只留首句）
export interface AgentToolResult<T = JsonValue | undefined> {
	/** Text or image content returned to the model. */
	content: (TextContent | ImageContent)[];
	/** Arbitrary structured details for logs or UI rendering. */
	details: T;
	/** Machine-readable result matching the tool's `outputSchema`, for programmatic callers. ... */
	structuredContent?: JsonValue;
	/** Usage from the final tool execution itself, if available. Not used for main LLM context accounting. */
	usage?: Usage;
	/** Report a failure without throwing. ... */
	isError?: boolean;
	/** Hint that the agent should stop after the current tool batch. ... */
	terminate?: boolean;
}
```

- **给模型的**：只有 `content`。文字和图片，会原样进 `toolResult` 消息、进下一次请求。
- **给程序的**：`details`（UI 渲染、日志）、`structuredContent`（机器可读、符合 `outputSchema`）、`usage`（工具自己花的 token，比如它内部又调了一次模型）。
- **给循环的**：`isError` 和 `terminate` 两面旗。

为什么要分？因为模型和 UI 要的东西不一样。一个 `edit` 工具，模型只需要"改好了，第 12 行"这样一句话；UI 却想要完整的 diff 来高亮显示。都塞进 `content`，模型的上下文被 diff 撑爆；都塞进 `details`，模型什么都看不见。两个字段，各喂一个读者。

结果最终被包成消息：

```typescript
// src/agent-loop.ts:930
function createToolResultMessage(finalized: FinalizedToolCallOutcome): ToolResultMessage {
	return {
		role: "toolResult",
		toolCallId: finalized.toolCall.id,
		toolName: finalized.toolCall.name,
		// Untyped tools (JS extensions) can return results without content; normalize
		// so the null never enters session history or provider payloads.
		content: finalized.result.content ?? [],
		details: finalized.result.details,
		usage: finalized.result.usage,
		isError: finalized.isError,
		...(finalized.durationMs === undefined ? {} : { durationMs: finalized.durationMs }),
		timestamp: Date.now(),
	};
}
```

注意少了什么：`structuredContent` 和 `terminate` 不进消息。`terminate` 是循环当场用掉的控制信号；`structuredContent` 是给程序调用方的，在 `tool_execution_end` 事件的 `result` 里拿得到，但不进对话记录。`content ?? []` 那行注释交代了一个现实：类型系统管得住 TypeScript 写的工具，管不住 JS 写的扩展，所以在入日志的最后一步再兜一次底。

## 管线第一段：准备

一次调用进管线，先过 `prepareToolCall`。它返回两种结果之一：`prepared`（准备好了，可以执行）或 `immediate`（已经有结论了，不用执行）。第 2 章引过它的几个片段，这里按顺序把五道关卡排开：

```typescript
// src/agent-loop.ts:708（prepareToolCall，删减：beforeToolCall 块与 return 体）
async function prepareToolCall(
	currentContext: AgentContext,
	assistantMessage: AssistantMessage,
	toolCall: AgentToolCall,
	config: ToolCallHooks,
	signal: AbortSignal | undefined,
	tools: readonly AgentTool<any>[] = currentContext.tools ?? [],
): Promise<PreparedToolCall | ImmediateToolCallOutcome> {
	const tool = tools.find((t) => t.name === toolCall.name);
	if (!tool) {
		// ...（关卡一：工具不存在）
	}

	try {
		const preparedToolCall = prepareToolCallArguments(tool, toolCall);
		const validatedArgs = validateToolArguments(tool, preparedToolCall);
		if (config.beforeToolCall) {
			// ...（关卡四：beforeToolCall，内含一次 abort 检查）
		}
		if (signal?.aborted) {
			// ...（关卡五：已中止）
		}
		return {
			kind: "prepared",
			toolCall,
			tool,
			args: validatedArgs,
		};
	} catch (error) {
		return {
			kind: "immediate",
			result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
			isError: true,
		};
	}
}
```

**关卡一：找工具。** 按名字在执行集里找，找不到就是 `Tool X not found`。

**关卡二：修补参数。** 模型不总是按 schema 出参数。`prepareArguments` 是工具作者留的兼容层，在校验之前把常见的歪形状掰正：

```typescript
// src/agent-loop.ts:694
function prepareToolCallArguments(tool: AgentTool<any>, toolCall: AgentToolCall): AgentToolCall {
	if (!tool.prepareArguments) {
		return toolCall;
	}
	const preparedArguments = tool.prepareArguments(toolCall.arguments);
	if (preparedArguments === toolCall.arguments) {
		return toolCall;
	}
	return {
		...toolCall,
		arguments: preparedArguments as Record<string, any>,
	};
}
```

返回值和原参数是同一个对象，就连 `toolCall` 也原样返回，不造新对象。coding-agent 的 `edit` 工具是真实用例，注释写得很具体：

```typescript
// packages/coding-agent/src/core/tools/edit.ts:110（仓库根起，prepareEditArguments 内）
// Some models (Opus 4.6, GLM-5.1) send edits as a JSON string instead of an array.
// Others send a single edit object instead of a one-element edits array.
```

这类修补放在工具上而不是循环里，是因为只有工具作者知道"哪种歪法是可以掰正的"。循环只提供一个调用点。

**关卡三：校验。** `validateToolArguments` 来自 pi-ai，做的事比"对 schema 打勾"多：

```typescript
// pi-ai/src/utils/validation.ts:317（删减：非 typebox schema 的强制转换分支）
export function validateToolArguments(tool: Tool, toolCall: ToolCall): any {
	if (toolCall.argumentsError) {
		throw new Error(`Invalid arguments for tool "${toolCall.name}": ${toolCall.argumentsError}`);
	}
	const args = structuredClone(toolCall.arguments);
	normalizeOptionalNulls(args, tool.parameters as JsonSchemaObject);
	Value.Convert(tool.parameters, args);
	// ...

	if (validator.Check(args)) {
		return args;
	}
	// ...（把每条校验错误拼成一行）
	const errorMessage = `Validation failed for tool "${toolCall.name}":\n${errors}\n\nReceived arguments:\n${JSON.stringify(toolCall.arguments, null, 2)}`;

	throw new Error(errorMessage);
}
```

四件事：模型发来的 JSON 根本解析不了（`argumentsError`），直接报；**深拷贝**一份再动手，不碰消息里的原参数；把可选字段上多余的 `null` 删掉（不少模型习惯用 `null` 表示"没填"）、按 schema 做类型转换（字符串 `"3"` 变数字 `3`）；还不过，就抛一个带完整错误列表和原始参数的异常。这个异常被 `prepareToolCall` 的 `catch` 接住，变成一条错误结果——**错误文本是写给模型看的**，它读到"哪个字段错了、你发的是什么"，下一轮就能自己改。

深拷贝那一步决定了一件事：校验后的 `args` 和消息里的 `toolCall.arguments` 是两个对象。消息记录里留的永远是模型的原话。

**关卡四：`beforeToolCall`。** 宿主的拦截点，第 2 章引过它的完整调用现场。这里补一个细节：钩子拿到的 `args` 是校验后的那份拷贝，**它可以就地改，改完不再校验**。测试把这条行为钉住了：

```typescript
// test/agent-loop.test.ts:528（删减：工具定义与流的构造）
it("should execute mutated beforeToolCall args without revalidation", async () => {
	// ...
	const config: AgentLoopConfig = {
		model: createModel(),
		convertToLlm: identityConverter,
		beforeToolCall: async ({ args }) => {
			const mutableArgs = args as { value: string | number };
			mutableArgs.value = 123;
			return undefined;
		},
	};
	// ...
	expect(executed).toEqual([123]);
});
```

schema 要求 `value` 是字符串，钩子塞了个数字进去，工具照样收到 `123`。这是一个有意的信任边界：**校验防的是模型，不防宿主**。宿主改参数（补默认值、改写路径、注入凭据）是它的权利，责任也在它自己。

**关卡五：中止。** 钩子跑完再看一眼 signal——钩子可能很慢（比如弹窗等用户确认），这期间用户可能已经按了停止。

五道关卡里，四道的失败都变成同一种东西：`kind: "immediate"`、`isError: true` 的结果。没有一道会让 `prepareToolCall` 抛出去。

## 管线第二段：执行

准备好的调用交给 `executePreparedToolCall`。第 2 章看过它的 `try/catch` 骨架，这里看被折叠掉的那部分——进度更新怎么收：

```typescript
// src/agent-loop.ts:821（executePreparedToolCall，删减：catch 分支的返回体）
async function executePreparedToolCall(
	prepared: PreparedToolCall,
	signal: AbortSignal | undefined,
	onUpdate: ToolUpdateSink,
): Promise<ExecutedToolCallOutcome> {
	const updateEvents: Promise<void>[] = [];
	let acceptingUpdates = true;
	const startedAt = performance.now();
	const elapsed = () => Math.round(performance.now() - startedAt);

	try {
		const result = await prepared.tool.execute(
			prepared.toolCall.id,
			prepared.args as never,
			signal,
			(partialResult) => {
				if (!acceptingUpdates) return;
				updateEvents.push(Promise.resolve(onUpdate(partialResult)));
			},
		);
		const durationMs = elapsed();
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return { result, isError: result.isError === true, durationMs };
	} catch (error) {
		const durationMs = elapsed();
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		// ...
	} finally {
		acceptingUpdates = false;
	}
}
```

工具通过第四个参数 `onUpdate` 报进度（"已扫描 300 个文件"），每次调用变成一个 `tool_execution_update` 事件。这里有两个容易被忽略的约束：

- **回调对工具是同步的。** `AgentToolUpdateCallback` 的返回类型是 `void`（`src/types.ts:463`），工具调了就走，不用等事件发完。但事件的 `emit` 是异步的，可能要等订阅者。所以循环把每次 emit 的 Promise 收进 `updateEvents`，工具返回之后统一 `await`——**保证所有 update 事件都在 `tool_execution_end` 之前落地**。
- **结算之后的调用一律丢弃。** `acceptingUpdates` 在工具返回（或抛出）的那一刻关上。一个工具如果把 `onUpdate` 存起来、事后再调（比如定时器没清），这些调用静默失效，不会在 `tool_execution_end` 之后冒出一个迟到的 update。JSDoc 写明了这条："Calls made after the tool promise settles are ignored."（`src/types.ts:460`，这句跨了两行注释）

`durationMs` 只量 `execute()` 本身，在关门之前取值，不含等待 update 事件落地的时间。

## 管线第三段：收尾

`finalizeExecutedToolCall` 给 `afterToolCall` 一次改写结果的机会，第 2 章「执行工具」一节有完整引文和 `structuredContent` 的例外规则，这里不再重复。值得补一句的是它的定位：**`afterToolCall` 是唯一能看到"工具实际返回了什么"、又能在模型看到之前改掉它的地方**。脱敏（把结果里的密钥抹掉）、截断（把十万行输出砍到一千行）、举旗（替工具决定这批之后停下）都在这里做。

三段走完，得到一个 `AgentToolCallOutcome`：

```typescript
// src/types.ts:448
/** Final outcome of a tool call after hooks ran. */
export interface AgentToolCallOutcome {
	toolCall: AgentToolCall;
	result: AgentToolResult<any>;
	isError: boolean;
	/** Milliseconds `execute()` took, measured with a monotonic clock; absent when the tool did not run. */
	durationMs?: number;
}
```

`isError` 在结果外面单独一份，而不是只看 `result.isError`：结果是工具说的，`isError` 是管线的最终判定——抛异常、hook 改判都会落到这里。

## 一批调用：两种顺序

一条 assistant 消息可以带多个工具调用。每个调用的事件序列是固定的：

```text
tool_execution_start
  tool_execution_update × N      （工具报进度时）
tool_execution_end
message_start / message_end      （toolResult 消息）
```

顺序执行时，一个调用的这一串走完，下一个才开始（`executeToolCallsSequential`，`src/agent-loop.ts:530`）。并行执行时（默认），情况复杂一些：

```typescript
// src/agent-loop.ts:596（executeToolCallsParallel 内，删减：被拦下与已中止两个分支的函数体）
for (const toolCall of toolCalls) {
	await emit({
		type: "tool_execution_start",
		toolCallId: toolCall.id,
		toolName: toolCall.name,
		args: toolCall.arguments,
	});

	const preparation = await prepareToolCall(currentContext, assistantMessage, toolCall, config, signal);
	if (preparation.kind === "immediate") {
		// ...（立即发 tool_execution_end，结果直接入列）
		continue;
	}

	finalizedCalls.push(async () => {
		// ...（执行前再查一次 abort）
		const executed = await executePreparedToolCall(preparation, signal, emitToolExecutionUpdate(toolCall, emit));
		const finalized = await finalizeExecutedToolCall(
			currentContext,
			assistantMessage,
			preparation,
			executed,
			config,
			signal,
		);
		await emitToolExecutionEnd(finalized, emit);
		return finalized;
	});
	// ...
}
```

关键在 `finalizedCalls.push(async () => {...})`：推进去的是一个**还没调用的函数**，不是一个已经在跑的 Promise。所以这个 `for` 循环走完时，没有一个工具开始执行——所有调用都只是被准备好了，`beforeToolCall` 按声明顺序一个一个问过了。然后才是并发的那一下：

```typescript
// src/agent-loop.ts:646
const orderedFinalizedCalls = await Promise.all(
	finalizedCalls.map((entry) => (typeof entry === "function" ? entry() : Promise.resolve(entry))),
);
```

`map` 里依次调用每个函数，它们几乎同时开跑；`Promise.all` 的结果数组保持**入列顺序**，与完成顺序无关。于是同一批调用产生两种顺序：

- `tool_execution_start`：全部在准备阶段发出，按声明顺序。
- `tool_execution_end`：每个工具跑完立刻发，按**完成顺序**。被拦下的调用例外，它在准备阶段就发了。
- `toolResult` 消息：等所有工具都结束后，按**声明顺序**逐条发出、入日志。

测试把这条规则写成了断言——第二个工具先完成：

```typescript
// test/agent-loop.test.ts:757（"should emit tool_execution_end in completion order but persist tool results in source order"）
expect(toolExecutionEndIds).toEqual(["tool-2", "tool-1"]);
expect(toolResultIds).toEqual(["tool-1", "tool-2"]);
```

UI 关心"哪个先好"，按事件流画进度；对话记录关心"可重放"，按声明顺序存。provider 对 `toolResult` 的顺序也有要求：它们要和 assistant 消息里的 `toolCall` 一一对应。

## 管线的另一个入口：runToolCall

三段管线不只服务于模型发起的调用。`agent-loop.ts` 导出了一个独立入口：

```typescript
// src/agent-loop.ts:803
/**
 * Run one tool call through the same steps as a model-issued call: argument preparation, schema
 * validation, `beforeToolCall`, execution, and `afterToolCall`. Emits no events and adds no
 * messages. Tools that call other tools use this so the hooks (for example permission checks)
 * apply to those calls too.
 *
 * Never rejects for tool failures: unknown tools, validation errors, blocked calls, and thrown
 * errors come back as `isError: true`.
 */
export async function runToolCall(toolCall: AgentToolCall, options: RunToolCallOptions): Promise<AgentToolCallOutcome> {
	const { assistantMessage, context, signal } = options;
	const preparation = await prepareToolCall(context, assistantMessage, toolCall, options, signal, options.tools);
	if (preparation.kind === "immediate") {
		return { toolCall, result: preparation.result, isError: preparation.isError };
	}
	const executed = await executePreparedToolCall(preparation, signal, options.onUpdate ?? (() => {}));
	return finalizeExecutedToolCall(context, assistantMessage, preparation, executed, options, signal);
}
```

六行就是整条管线，去掉了事件和消息。场景写在注释里：**工具调用工具**。一个跑脚本的工具（coding-agent 的 codemode），脚本里又要读文件、跑 shell——这些嵌套调用不该绕过权限检查。`runToolCall` 让它们走同一套 `prepareArguments`、校验、`beforeToolCall`、`afterToolCall`。`prepareToolCall` 签名里那个默认值为 `currentContext.tools` 的 `tools` 参数，就是为这个入口留的：嵌套调用可以查另一份工具清单。

事件和消息不由它负责，因为嵌套调用怎么呈现是宿主的事。coding-agent 的做法是给事件带上 `parentToolCallId`、把嵌套调用的记录挂到外层调用的结果消息上：

```typescript
// packages/coding-agent/src/core/nested-tool-calls.ts:1（仓库根起）
/**
 * Tool calls that a tool makes while it runs (`ctx.executeTool()`), for example from codemode
 * scripts. The agent loop does not know about them: the session runs each one through the agent's
 * tool pipeline (`runToolCall`) with its own hooks, emits `tool_execution_*` events with
 * `parentToolCallId`, and records the calls and their usage on the model-issued call's tool result
 * message.
 *
 * Nothing here runs until a tool calls `ctx.executeTool()`.
 */
```

"The agent loop does not know about them"——循环不知道它们存在，但它们走的是循环的管线。这是把管线拆成三个独立函数的回报。

## 一章小结

- `AgentTool` 有两张脸：父类 `Tool` 的四个字段是给模型的声明，进日志；其余字段是执行体，留在内存。`replay` 是给持久化运行时的，本包不读。
- 结果分给三个读者：`content` 给模型，`details` / `structuredContent` / `usage` 给程序，`isError` / `terminate` 给循环。`structuredContent` 和 `terminate` 不进消息记录。
- 准备阶段五道关卡：找工具、`prepareArguments` 修补、`validateToolArguments` 深拷贝并校验、`beforeToolCall`、abort 检查。失败都变成 `isError: true` 的结果，错误文本写给模型看。
- 校验防模型，不防宿主：`beforeToolCall` 改过的参数不再校验。
- 执行阶段收集进度事件，保证它们都在 `tool_execution_end` 之前落地；工具结算之后的 `onUpdate` 调用被丢弃。
- 并行模式先顺序准备、再并发执行：`tool_execution_end` 按完成顺序，`toolResult` 消息按声明顺序。
- `runToolCall` 是去掉事件和消息的同一条管线，给"工具调用工具"用。

下一章回到 `Agent` 类本身：状态怎么归约，两条队列怎么工作，一个 run 出错时怎么收场。

## 为什么不去

> **为什么失败一律变成结果，而不是让 `execute` 的异常往上抛？** 因为循环的下游是模型。一个异常往上抛，run 就断了，模型没有机会知道发生了什么；变成一条 `isError: true` 的 `toolResult`，模型下一轮就能看到"文件不存在"或"参数 `path` 必须是字符串"，然后自己换个办法。`runToolCall` 的注释把这个契约写成了一句话："Never rejects for tool failures"。工具作者因此可以放心地抛异常，循环替它兜住；而 `AgentTool.execute` 的 JSDoc 又给了更好的选项——返回 `isError: true` 的结果，同时保留 `details`："Throw on failure, or return a result with `isError: true`; do not only describe the failure in `content`"（`src/types.ts:480`）。最后半句防的是第三种写法：在 `content` 里写"出错了"却不举旗，那样 UI 和宿主都无从分辨成败。

> **为什么并行模式要先把所有调用准备完，而不是准备一个、启动一个？** 准备一个启动一个看起来更快，但 `beforeToolCall` 就得和已经在跑的工具并发——一个权限弹窗还没关，前一个工具已经在删文件了。先全部准备再统一启动，换来一个简单的保证：**任何工具开始执行之前，这一批的所有拦截决定都已经做完**。宿主想做"整批审批"（一次弹窗列出这条消息要做的五件事），也只有在这个顺序下才可能。代价是第一个工具要等最后一个工具的准备阶段，而准备阶段通常只是校验加一次钩子调用，开销很小。
