# 第 3 章 · 消息是轴心：AgentMessage 与 LLM 边界

上一章跟着一次 `prompt()` 把控制流走完了。你可能注意到，一路上被传来传去、被 push、被 emit 的，几乎只有一样东西：消息。这一章把镜头对准它，回答两个问题：

1. 为什么整个循环只说 `AgentMessage` 这一种语言？
2. 这种语言在哪里、经过几道闸门，被翻译成模型听得懂的话？

本章文中的"旧基线""旧版本"指 commit `9e05370b2`（"Mid conversation system messages"）之前的代码——正是这个提交把 system prompt 和工具声明搬进了消息记录。

> 引用写法：`src/...` 相对于 `packages/agent/`；本章新出现的 `pi-ai/src/...` 相对于 `packages/ai/`。

## 一个开放的联合类型

先看定义：

```typescript
// src/types.ts:365
export interface CustomAgentMessages {
	// Empty by default - apps extend via declaration merging
}

/**
 * AgentMessage: Union of LLM messages + custom messages.
 * This abstraction allows apps to add custom message types while maintaining
 * type safety and compatibility with the base LLM messages.
 */
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

两半拼成一个类型：

- **`Message`**：来自 pi-ai，模型能理解的四种角色。
- **`CustomAgentMessages[keyof CustomAgentMessages]`**：一个空接口的"值类型联合"。接口里没有键，`keyof` 是 `never`，这一半也是 `never`——默认情况下，`AgentMessage` 就等于 `Message`。

```typescript
// pi-ai/src/types.ts:644
export type Message = SystemMessage | UserMessage | AssistantMessage | ToolResultMessage;
```

注意是**四种**。旧版本只有 `user`/`assistant`/`toolResult` 三种，`system` 是后来加入的（第 2 章「插叙：消息的四种角色与内容块」点过）；它为什么会出现在消息记录里，是本章后半段的主线。

### 插叙：declaration merging

TypeScript 允许同名 `interface` 在不同位置重复声明，编译器会把它们的成员合并成一个。配合 `declare module`，你可以在自己的代码里往别人包里的接口"追加字段"：

```typescript
// 演示：在应用里扩展消息类型
declare module "@earendil-works/pi-agent-core" {
	interface CustomAgentMessages {
		notification: { role: "notification"; text: string; timestamp: number };
		artifact: { role: "artifact"; path: string; timestamp: number };
	}
}
// 此后，AgentMessage = Message | { role: "notification"; ... } | { role: "artifact"; ... }
```

这里用的是接口而不是联合，因为联合类型不能被合并——`type` 声明一旦写下就是封闭的。接口的键名（`notification`、`artifact`）只是合并用的把手，真正进入联合的是值类型；`role` 字段才是运行时的判别依据。

源码里那段 JSDoc 示例写的是 `declare module "@mariozechner/agent"`（`src/types.ts:357`）——这是包改名前的旧名字，照抄会静默失效：模块名对不上，合并就落在一个不存在的模块上。以 `package.json` 的 `name` 为准。

### 为什么是轴心

开放联合的效果是：应用自定义的消息和模型消息住在**同一个数组**里。这个数组就是 `AgentContext.messages`：

```typescript
// src/types.ts:502
export interface AgentContext {
	/** Transcript visible to the model. */
	messages: AgentMessage[];
	/** Tools available for execution in this run. */
	tools?: AgentTool<any>[];
}
```

循环对这个数组只做三件事：push、替换最后一项（流式时的 partial 消息）、整体交给边界上的闸门。它几乎不按 `role` 分支，只认得两个角色：`assistant`——continue 时拒绝从模型的话之后接着说：

```typescript
// src/agent-loop.ts:139（runAgentLoopContinue 内）
if (context.messages[context.messages.length - 1].role === "assistant") {
	throw new Error("Cannot continue from message role: assistant");
}
```

以及 `system`——本章后半段的 `declareToolChanges`。其余角色，包括你自定义的所有角色，循环一概不看。换句话说，**循环不需要理解你的消息，它只需要搬运它们**。

这带来一个直接的好处：UI 想在对话流里插一条"已切换模型"的提示、一张工件卡片，不需要另起一个平行数组再去和消息对齐时间线。它就是一条消息，事件照发（`message_start` / `message_end`），顺序天然正确。代价也很清楚：这些消息必须在调模型之前被处理掉，否则会被原样送给 provider。处理它们的地方，就是边界上的闸门。

在看闸门之前，先看一种特殊的消息——它既不是用户说的，也不是模型说的，却决定了模型"是谁"。

## 消息记录拥有 system prompt

### 旧的形状：系统提示是"开局条件"

在旧基线上，system prompt 和工具是 `AgentContext` 上与 `messages` 并列的两个字段，每次调模型时拼进请求：

```typescript
// src/types.ts:406（cd20a8d2e 版本）
export interface AgentContext {
	/** System prompt included with the request. */
	systemPrompt: string;
	/** Transcript visible to the model. */
	messages: AgentMessage[];
	/** Tools available for this run. */
	tools?: AgentTool<any>[];
}
```

这个形状简单，但它有一个隐含假设：**system prompt 是对话的开局条件，而不是对话里发生过的事**。你在第 20 轮改了提示词，第 21 轮的请求会带着新提示词出发，但消息记录里没有任何痕迹说明"这里换过"。后果有三：

- 恢复一个会话时，你得另外存下"当时用的是哪个提示词"，否则重放出来的对话前提就变了。
- 在分支间跳转时，每个分支可能有不同的工具集，而消息记录本身回答不了"这个分支此刻有哪些工具"。
- 改提示词等于改了整个请求的前缀，provider 侧的 prompt cache 全部失效。

commit `9e05370b2` 的说明把动机写得很直接："makes system prompt text and tool changes part of the transcript rather than silently rewriting its starting conditions"。

### 新的形状：系统消息是一条增量

现在 `AgentContext` 只剩 `messages` 和 `tools`（上一节引过，`src/types.ts:502`），系统提示住进了消息记录，形状是 pi-ai 的 `SystemMessage`：

```typescript
// pi-ai/src/types.ts:546
export interface SystemMessage {
	role: "system";
	/** Instruction text. On the leading message this is the base prompt; later, additional instructions. */
	content: string | TextContent[];
	/**
	 * Named, ordered prompt sections rendered verbatim after `content`. The leading message
	 * declares them; later messages replace sections by name, and `null` removes one. ...
	 */
	sections?: Record<string, string | null>;
	/** Complete definitions of tools that become available at this point. */
	toolsAdded?: Tool[];
	/** Tools that stop being available at this point. */
	toolsRemoved?: ToolReference[];
	timestamp: number; // Unix timestamp in milliseconds
}
```

关键在于语义分了两种位置：

| 位置 | `content` | `sections` | `toolsAdded` / `toolsRemoved` |
|---|---|---|---|
| 开头那条（leading） | 基础提示词 | 声明全部分节 | 初始工具集 |
| 后续任意一条 | **追加**的指令 | 按名替换，`null` 删除 | 从此处起增减工具 |

后续的系统消息不是"新的 system prompt"，而是一个**补丁**。把消息记录里所有系统消息按顺序叠起来，得到的才是此刻的提示词和工具集。这是一个事件溯源（event sourcing）式的设计：状态不单独存，状态 = 日志的重放。

### 重放：从日志里读出当前状态

重放函数在 pi-ai 里，一共两个核心：

```typescript
// pi-ai/src/utils/transcript.ts:58
export function getCurrentTools(messages: TranscriptMessages): Tool[] {
	const tools = new Map<string, Tool>();
	for (const message of messages) {
		if (!isSystemMessage(message)) continue;
		for (const tool of message.toolsRemoved ?? []) tools.delete(tool.name);
		for (const tool of message.toolsAdded ?? []) tools.set(tool.name, tool);
	}
	return [...tools.values()];
}
```

同一条消息里先删后加——所以"删掉 `read` 再加一个新定义的 `read`"可以写在一条消息里，表达"重定义"。提示词的重放结构相同，只是叠的是文本和分节：

```typescript
// pi-ai/src/utils/transcript.ts:73（getCurrentSystemMessage，删减：返回值组装）
export function getCurrentSystemMessage(messages: TranscriptMessages): SystemMessage | undefined {
	const content: string[] = [];
	const sections = new Map<string, string>();
	let timestamp: number | undefined;
	for (const message of messages) {
		if (!isSystemMessage(message)) continue;
		timestamp ??= message.timestamp;
		const text = contentText(message.content);
		if (text.length > 0) content.push(text);
		for (const [name, value] of Object.entries(message.sections ?? {})) {
			if (value === null) sections.delete(name);
			else sections.set(name, value);
		}
	}
	const tools = getCurrentTools(messages);
	// ...拼成一条 leading 形状的 SystemMessage：content 用 "\n\n" 连接
}
```

注意参数类型是 `TranscriptMessages`，不是 `Message[]`：

```typescript
// pi-ai/src/utils/transcript.ts:36
/**
 * Any message list. The replay helpers only read entries whose role is `"system"`, so
 * agent transcripts that carry custom message roles can be passed without filtering.
 */
export type TranscriptMessages = readonly { role: string }[];
```

这是一个刻意放宽的签名：pi-ai 不认识 `AgentMessage`（它在下游），但只要求"有个 `role` 字段"，就能直接吃下 agent 侧夹着自定义消息的数组。上一节说"循环不理解你的消息"，这里是同一条原则在 pi-ai 里的体现——重放函数只看 `system`，其余一律跳过。

### Agent 这一侧：只读的 systemPrompt

`Agent` 的构造函数把旧式的 `systemPrompt` + `tools` 参数折成开头那条系统消息：

```typescript
// src/agent.ts:82（createMutableAgentState，删减：model 等字段）
function createMutableAgentState(initialState?: AgentInitialState): MutableAgentState {
	let tools = initialState?.tools?.slice() ?? [];
	let messages = initialState?.messages?.slice() ?? [];
	const initialMessage = createInitialSystemMessage(initialState?.systemPrompt, tools.map(toToolDeclaration));
	if (messages[0]?.role !== "system" && initialMessage) messages.unshift(initialMessage);

	return {
		get systemPrompt() {
			return getCurrentSystemPrompt(messages);
		},
		// ...
	};
}
```

两处值得停顿：

- **只在消息记录还没有开头系统消息时才插入。** 恢复一个已有的会话（`messages` 从存储里读出来），开头那条已经在了，构造参数里的 `systemPrompt` 会被忽略——消息记录赢。
- **`systemPrompt` 是 getter，没有 setter。** `AgentState` 上它被声明为 `readonly`，JSDoc 写明了改法："to change the prompt, append a system message with `content` or `sections`"（`src/types.ts:386`）。读它，就是做一次重放。

改提示词的正确姿势，README 给了示例——和用户消息一起作为一批 prompt 发出去：

```typescript
// packages/agent/README.md:304
await agent.prompt([
  { role: "system", content: "", sections: { skills: "<skills>...</skills>" }, timestamp: Date.now() },
  { role: "user", content: "Continue", timestamp: Date.now() },
]);
```

`reset()` 也随之改了含义。清空对话不再是"清空数组"，而是"把日志压缩成一条基线"：

```typescript
// src/agent.ts:355（Agent.reset，删减：其余状态复位）
reset(): void {
	if (this.activeRun) {
		throw new Error("Agent is already processing. Wait for completion before resetting.");
	}

	const baseline = getCurrentSystemMessage(this._state.messages);
	this._state.messages = baseline ? [baseline] : [];
	// ...
}
```

重放的结果正好是一条 leading 形状的系统消息，可以原样当作新日志的第一条。重放函数和日志格式闭合在同一个形状上，这让"压缩"变成了一行代码。

## 两份工具清单

系统消息搬进日志之后，工具出现了两份：

- **`context.tools`**（`AgentTool[]`）：运行时**能执行**的工具，带 `execute` 函数。改它的方式是 `agent.state.tools = [...]`。
- **消息记录里声明的工具**（`Tool[]`）：模型**被告知可以调用**的工具，是重放 `toolsAdded` / `toolsRemoved` 的结果，只有名字、描述和参数 schema。

两份清单必须一致。如果模型被告知有 `bash`，运行时却没有，它的调用会撞上"工具不存在"；如果运行时加了 `search`，消息记录里却从没声明，模型永远不会用它。`AgentState.tools` 的 JSDoc 写明了谁负责对齐：

```typescript
// src/types.ts:394（AgentState.tools 的 JSDoc）
/**
 * Executable tools. Assigning a new array copies the top-level array.
 *
 * Differences from the tools declared in the transcript are announced to the model
 * with a system message before the next request.
 */
```

"announced to the model"——是循环在每次请求前主动补一条系统消息。做这件事的函数是 `declareToolChanges`。

### 执行集是真相，声明是投影

它在循环里被调用两次，位置正是"消息进入日志"的两个入口：run 开头的 prompt 消息，以及每圈开头的 prepared + steering 消息：

```typescript
// src/agent-loop.ts:110（runAgentLoop 内）
const initialMessages = declareToolChanges(context, prompts);
```

```typescript
// src/agent-loop.ts:210（runLoop 内层循环）
// Process prepared and queued messages before the next assistant response.
for (const message of declareToolChanges(currentContext, [...preparedMessages, ...pendingMessages])) {
	await emit({ type: "message_start", message });
	await emit({ type: "message_end", message });
	currentContext.messages.push(message);
	newMessages.push(message);
}
```

签名是"一批待入日志的消息进，一批（可能多一条）消息出"。函数本体：

```typescript
// src/agent-loop.ts:333
function declareToolChanges(context: AgentContext, pendingMessages: AgentMessage[]): AgentMessage[] {
	let systemIndex = -1;
	for (let i = pendingMessages.length - 1; i >= 0; i--) {
		if (pendingMessages[i].role === "system") {
			systemIndex = i;
			break;
		}
	}
	const pending = pendingMessages[systemIndex] as SystemMessage | undefined;
	const baseline = pending
		? pendingMessages.map((message, index) =>
				index === systemIndex ? withToolChanges(pending, NO_CHANGES) : message,
			)
		: pendingMessages;
	const changes = getToolStateChanges(
		getCurrentTools([...context.messages, ...baseline]),
		(context.tools ?? []).map(toToolDeclaration),
	);
	const unchanged = changes.toolsAdded.length === 0 && changes.toolsRemoved.length === 0;

	if (pending) {
		// Keep the caller's message object when it already declares no tool changes.
		if (unchanged && !pending.toolsAdded?.length && !pending.toolsRemoved?.length) return pendingMessages;
		return baseline.map((message, index) => (index === systemIndex ? withToolChanges(pending, changes) : message));
	}
	if (unchanged) return pendingMessages;
	const update = withToolChanges({ role: "system", content: "", timestamp: Date.now() }, changes);
	const insertIndex = pendingMessages.findIndex((message) => message.role !== "system");
	const index = insertIndex === -1 ? pendingMessages.length : insertIndex;
	return [...pendingMessages.slice(0, index), update, ...pendingMessages.slice(index)];
}
```

拆成三步读：

1. **找"待定"的系统消息。** 在这批消息里倒着找最后一条 `system`。找到了，先把它的工具字段清空，得到 `baseline`——调用方在这条消息里写的 `toolsAdded` / `toolsRemoved` **只被当作意图，不被当作事实**。
2. **算差。** "已提交的日志 + baseline"重放出来的声明集，对比 `context.tools` 剥成声明后的执行集。`getToolStateChanges` 把差异算成增删两张表。
3. **落笔。** 有待定系统消息，就把差写进它（一批里不会冒出两条系统消息）；没有，就新造一条空 `content` 的系统消息，插在第一条非系统消息之前——让工具变化先于用户的话生效。

第 1 步是这个函数最反直觉的地方。函数头注释把规则说透了："its tool fields are treated as intent and replaced with the delta between the committed transcript and the executable set, so replay always yields exactly `context.tools`"（`src/agent-loop.ts:328`）。不变量只有一条：**重放日志得到的工具集，永远等于执行集**。调用方手写的声明对不上，以执行集为准。测试把这条规则钉住了：

```typescript
// test/agent.test.ts:258（"rewrites pending tool declarations to match the executable set"，删减）
// The pending message claims to add `second` and remove `first`, but the executable
// set still has `first` and lacks `second`: the executable set wins.
await agent.prompt([
	{
		role: "system",
		content: "",
		sections: { note: "<note>x</note>" },
		toolsAdded: [toToolDeclaration(createTool("second"))],
		toolsRemoved: [{ name: "first" }],
		timestamp: 1,
	},
	{ role: "user", content: "hi", timestamp: 2 },
]);

expect(agent.state.messages[1]).toEqual({
	role: "system",
	content: "",
	sections: { note: "<note>x</note>" },
	timestamp: 1,
});
```

`sections` 原样保留，工具字段被擦掉——因为执行集里本来就只有 `first`，与已提交的声明无差。

### 声明的"剥皮"

两边比较之前，执行集要先过一遍 `toToolDeclaration`：

```typescript
// pi-ai/src/utils/transcript.ts:122
/** Strip executable and display-only fields from a tool before transcript comparison or persistence. */
export function toToolDeclaration(tool: Tool): Tool {
	return {
		name: tool.name,
		description: tool.description,
		parameters: JSON.parse(JSON.stringify(tool.parameters)) as Tool["parameters"],
		...(tool.constrainedSampling === undefined ? {} : { constrainedSampling: tool.constrainedSampling }),
	};
}
```

`AgentTool` 上的 `execute`、`label`、`prepareArguments`、`executionMode` 全被剥掉：日志要能持久化、要能被 provider 读，函数和 UI 字段都不该进去。`parameters` 走一次 JSON 往返，顺手抹掉 typebox 塞在 schema 上的 symbol 键——这样两份声明可以直接比较序列化字符串，不用引入深比较依赖（`declarationsEqual`，`pi-ai/src/utils/transcript.ts:140` 的注释："This avoids a deep-equal dependency in a browser-safe package"）。

`getToolStateChanges` 用的正是这个比较，并且把"定义变了"表达成"先删再加"：

```typescript
// pi-ai/src/utils/transcript.ts:149
/** Compare two complete tool states. A changed definition is a removal followed by an addition. */
export function getToolStateChanges(previous: readonly Tool[], current: readonly Tool[]): ToolStateChanges {
```

和上一节 `getCurrentTools` 的"同一条消息里先删后加"严丝合缝。

### 一次切换工具的完整轨迹

测试 `"declares tool loadout changes to the model before the next request"` 记录了每次请求里模型看到的系统消息（`+` 为增，`-` 为删）：

```typescript
// test/agent.test.ts:188（删减：streamFn 实现）
await agent.prompt("one");
agent.state.tools = [second];
await agent.prompt("two");
await agent.prompt("three");

expect(requests).toEqual([
	["+first", "-"],
	["+first", "-", "+second", "-first"],
	["+first", "-", "+second", "-first"],
]);
```

读法：

- 第一次请求，只有构造时折出来的开头那条，声明 `first`。
- 赋值 `agent.state.tools` 本身什么都不发——它只改执行集。下一次 `prompt("two")` 入日志时，`declareToolChanges` 发现差异，补了一条 `+second -first`。
- 第三次请求没有任何新系统消息：差为零，函数原样返回（`if (unchanged) return pendingMessages`）。日志前缀保持不变，这正是 prompt cache 能命中的前提。

工具切换不再"静默改写开局条件"，而是作为一个带时间戳的事件留在日志里——模型看得到，存储存得下，分支跳转后重放得出来。

## 边界上的闸门

日志里现在混着四种模型角色和任意多种自定义角色，系统消息还可能散落在中间。把它变成一次 provider 请求，要过几道闸门。第 2 章「调模型：闸门与回流」点过 agent 侧的三道；这里把它们展开，再加上 provider 侧的第四道。全部在 `streamAssistantResponse` 的开头：

```typescript
// src/agent-loop.ts:388（streamAssistantResponse 内）
// Apply context transform if configured (AgentMessage[] → AgentMessage[])
let messages = context.messages;
if (config.transformContext) {
	messages = await config.transformContext(messages, signal);
}

// Convert to LLM-compatible messages (AgentMessage[] → Message[])
const llmMessages = await config.convertToLlm(messages);

const llmContext = normalizeContext({ messages: llmMessages });
```

每道闸门换一次类型，类型就是闸门的边界：

| 闸门 | 类型变化 | 谁提供 | 职责 |
|---|---|---|---|
| `transformContext` | `AgentMessage[]` → `AgentMessage[]` | 调用方，可选 | 剪枝、注入外部上下文 |
| `convertToLlm` | `AgentMessage[]` → `Message[]` | 调用方，必需 | 翻译或丢弃自定义角色 |
| `normalizeContext` | `Message[]` → `TranscriptContext` | pi-ai，固定 | 盖章：这是一份合规的日志 |
| `resolveTranscript` | `TranscriptContext` → `TranscriptContext` | provider 内部 | 按模型能力摊平系统消息 |

### 第一、二道：只改请求，不改日志

注意 `messages` 是个局部变量。`transformContext` 返回的新数组只流向 `convertToLlm`，`context.messages` 一条都没少——紧接着流式回来的 partial 消息，push 的是 `context.messages`，不是 `messages`。所以：

- 你在 `transformContext` 里剪掉的旧消息，**只是这次请求看不到**，日志和 `agent_end` 里的 `messages` 照旧完整。想真正改写日志（比如压缩后替换历史），要走 `prepareNextTurn` / `prepareRequest` 返回新的 `context`（第 2 章岔路一节）。
- 两道闸门都签了"不许抛"的契约（`src/types.ts:203`、`src/types.ts:231`："Contract: must not throw or reject"）。`convertToLlm` 的 JSDoc 把后果也写了："Throwing interrupts the low-level agent loop without producing a normal event sequence"——订阅者等不到一个正常收尾的事件序列。

`convertToLlm` 在新基线上多了一个容易踩的坑：**它必须放行 `system`**。`Agent` 的默认实现已经改过：

```typescript
// src/agent.ts:38
function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
	return messages.filter(
		(message) =>
			message.role === "system" ||
			message.role === "user" ||
			message.role === "assistant" ||
			message.role === "toolResult",
	);
}
```

如果你按旧习惯自己写了一个只认 `user` / `assistant` / `toolResult` 的过滤器，升级之后系统消息会被静默丢掉——模型既收不到提示词，也收不到工具声明。类型系统帮不了你：返回 `Message[]` 的子集永远是合法的 `Message[]`。

### 第三道：一个只有 pi-ai 能盖的章

`normalizeContext` 在这里做的实际工作几乎为零：

```typescript
// pi-ai/src/utils/transcript.ts:30
export function normalizeContext(context: Context): TranscriptContext {
	const initialMessage = createInitialSystemMessage(context.systemPrompt, context.tools);
	const messages = initialMessage ? [initialMessage, ...context.messages] : context.messages;
	return { messages } as TranscriptContext;
}
```

循环传进来的 `Context` 没有 `systemPrompt` 也没有 `tools`，`createInitialSystemMessage` 返回 `undefined`，消息原样通过。这道闸门的价值不在运行时，在类型：

```typescript
// pi-ai/src/types.ts:777
declare const transcriptContextBrand: unique symbol;

/**
 * Normalized request context passed to providers and API implementations. The
 * prompt and tool declarations are carried by the transcript's system messages.
 * Only `normalizeContext()` produces this type, so a raw `Context` cannot reach
 * provider code by accident.
 */
export type TranscriptContext = {
	messages: Message[];
	readonly [transcriptContextBrand]: true;
};
```

### 插叙：branded type

TypeScript 是结构类型系统：两个形状相同的类型可以互相赋值。想让"形状一样但来路不同"的值不能互换，惯用手法是加一个只存在于类型层的"品牌"字段——用 `declare const` 声明一个 `unique symbol`，它没有运行时值，外部代码也拿不到它，于是除了做 `as` 断言的那个函数，没人造得出带这个键的对象。代价是零：编译后品牌字段完全消失。

`StreamFn` 的参数类型正是 `TranscriptContext`（`src/types.ts:33`）。结果是：一个旧式的 `{ systemPrompt, messages, tools }` 对象，**在编译期就无法直接喂给任何 provider**。如果有人绕过循环直接调 provider，又忘了把 `systemPrompt` 折进日志，编译器会拦下他。`normalizeContext` 是唯一的入口，也是唯一会处理旧式字段的地方——兼容逻辑被收拢在一个函数里，provider 里不用再处理两种输入。

### 第四道：按模型能力摊平

不是所有模型都接受对话中间的系统消息。provider 收到 `TranscriptContext` 后第一件事是按模型的兼容标记决定要不要摊平：

```typescript
// pi-ai/src/utils/transcript.ts:104
/**
 * Rebuild the transcript for APIs without mid-conversation system messages: the replayed
 * system message leads, and every later system message is dropped.
 */
export function collapseSystemMessages(context: TranscriptContext): TranscriptContext {
	const head = getCurrentSystemMessage(context.messages);
	const messages = context.messages.filter((message) => message.role !== "system");
	return { messages: head ? [head, ...messages] : messages } as TranscriptContext;
}

/** Keep later system messages in place when the model accepts them; otherwise collapse them. */
export function resolveTranscript(
	context: TranscriptContext,
	supportsMidConvoSystemMessages: boolean | undefined,
): TranscriptContext {
	return supportsMidConvoSystemMessages ? context : collapseSystemMessages(context);
}
```

两条路：

- **支持中途系统消息**：日志原样发出，每条后续系统消息在原位置出现。前缀不变，cache 不破。provider 用 `renderSystemMessageUpdate`（`pi-ai/src/utils/text.ts:28`）把分节变更渲染成 `Updated system prompt section "skills": ...` 这样的文字，让模型能把补丁和开头的提示词对上号。
- **不支持**：重放出当前状态，作为唯一一条系统消息放在开头，其余系统消息删除。语义等价，但前缀变了。

这道闸门在 provider 里，不在本包里——本包把日志交出去就不管了。但它解释了为什么日志要存"增量"而不是"当前快照"：**增量可以无损地摊平成快照，快照却推不回增量**。日志选择信息量更大的那种形状，把有损的那一步推迟到最后一刻、推给最了解模型能力的那一层。

## 一章小结

- `AgentMessage` = pi-ai 的四种 `Message` + 应用通过 declaration merging 追加的自定义消息。循环只搬运，不理解，只认 `assistant` 和 `system` 两个角色。
- 新基线上，system prompt 和工具声明住进日志：开头那条系统消息是基线，后续系统消息是补丁（追加 `content`、按名替换 `sections`、增删工具），当前状态 = 重放。
- `Agent.state.systemPrompt` 只读，改提示词就是追加一条系统消息；`reset()` 把日志压成一条重放出来的基线。
- 工具有两份：执行集（`context.tools`）是真相，日志里的声明是投影。`declareToolChanges` 在消息入日志前补齐差异，保证重放结果永远等于执行集。
- 边界上四道闸门：`transformContext`、`convertToLlm` 只改请求不改日志；`normalizeContext` 用 branded type 盖章；provider 里的 `resolveTranscript` 按模型能力决定是否摊平。

下一章进入工具本身：`AgentTool` 的形状，以及一次工具调用从参数校验到结果落地的三段管线。

## 为什么不去

> **为什么不保留 `AgentContext.systemPrompt`，只在它变化时往日志里记一笔？** 那会让同一份状态有两个真相来源：字段说一个，日志说另一个，任何一次漏记都会让两者分叉，而且没有一方能判定谁对。commit `9e05370b2` 选的是单一来源——日志是唯一的状态，字段降级成重放的结果（`get systemPrompt()`）。提交说明里列的三个收益——"record when instructions changed or tools became available, restore that state after resuming or navigating branches, and preserve cached prompt prefixes"——都只在"状态就是日志"时才能同时成立。

> **为什么调用方手写的 `toolsAdded` 要被覆盖，而不是信任它？** 因为只有执行集是可验证的：工具真的在 `context.tools` 里，`execute` 真的能跑。手写声明可能过时、可能拼错名字、可能是从别的分支拷来的。让声明服从执行集，不变量"重放 = 执行集"就由循环自己保证，不依赖每个调用方的自觉。调用方真正能控制的，是系统消息的 `content` 和 `sections`——那是纯粹的指令文本，没有可验证的"真相"可以对照。

> **为什么 `getCurrentTools` 这些重放函数放在 pi-ai，而不是本包？** 因为 provider 也要用：第四道闸门 `collapseSystemMessages` 摊平时要重放，Anthropic 等 provider 要从日志里算出当前工具集。重放逻辑如果留在 agent 侧，pi-ai 就得反过来依赖本包，违反第 1 章那条"箭头全部指向上游"。放在 pi-ai，并把参数放宽成 `{ role: string }[]`，两边都能用，依赖方向不变。
