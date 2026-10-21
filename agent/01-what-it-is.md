# 第 1 章 · 它是什么：一个被做成库的 agent 循环

## 从一个场景开始

你在写一个应用，想在里面嵌入一个能"自己干活"的 AI agent：你给它一句话，它调模型，模型说要调工具，它执行工具，把结果喂回模型，如此往复，直到模型说"完了"。中间每一步你都想实时看到——文字是一个字一个字流出来的，工具是一个一个执行的——因为你要把这些渲染到 UI 上。

这就是 `pi-agent-core` 要解决的全部问题。它的 README 用一句话概括：

```markdown
// README.md:3
Stateful agent with tool execution and event streaming. Built on `@earendil-works/pi-ai`.
```

拆开看，它给你两样东西，外加一个小配件：

1. **一个 agent 循环**（`src/agent-loop.ts`）：prompt 进来，事件流出去，中间是"调模型 → 执行工具 → 再调模型"的往复。
2. **一层状态**（`src/agent.ts`）：`Agent` 类替你持有对话历史、当前正在流的消息、正在执行的工具，让你在任意时刻都能回答"它现在到哪一步了"。
3. **一个代理流**（`src/proxy.ts`）：给浏览器应用用的模型调用通道——请求发到你自己的服务器，由服务器持有密钥、转发给厂商。

没有第四样。持久化会话、上下文压缩、内置的文件和 shell 工具、skill 加载——这些曾经都在这个包里，1.0.0 之后全部搬走了（本章末「为什么不去」）。现在的 `pi-agent-core` 是一个只有六个源文件、两千五百行的包。

## 它拒绝做什么

理解一个库，边界和能力同样重要。`pi-agent-core` 有四条明确的拒绝，每一条都能从代码里核实。

**拒绝一：不认识任何模型厂商。** 包里搜不到一个 provider 的名字。循环拿到的模型接口是一个函数类型：

```typescript
// src/types.ts:33
export type StreamFn = (
	model: Model<Api>,
	context: TranscriptContext,
	options?: SimpleStreamOptions,
) => AssistantMessageEventStream | Promise<AssistantMessageEventStream>;
```

它只约定形状：给我 `Model` 和一份规范化的对话记录（`TranscriptContext`），还我一个事件流。谁来提供这个函数？调用方。`Agent` 的构造参数里 `streamFn` 是必填项（`src/agent.ts:118`）；嫌每次都传麻烦，宿主可以装一个全局默认值：

```typescript
// src/stream-fn.ts:5
/**
 * Configure the fallback used by Agent and low-level loops when callers omit streamFn.
 *
 * Hosts that provide a default model runtime can install its stream function here
 * without making pi-agent-core depend on a provider catalog or compatibility layer.
 */
export function setDefaultStreamFn(streamFn: StreamFn | undefined): void {
	defaultStreamFn = streamFn;
}
```

两个都没有，循环会在开跑时直接抛错，而不是悄悄挑一个厂商（`src/stream-fn.ts:17`："No default stream function configured"）。

**拒绝二：不碰 UI。** 包里没有一行渲染代码。它对外沟通的方式只有一种：发事件。全部事件就是一个可辨识联合，十种：

```typescript
// src/types.ts:516（删减：两个工具事件的字段）
export type AgentEvent =
	// Agent lifecycle
	| { type: "agent_start" }
	| { type: "agent_end"; messages: AgentMessage[] }
	// Turn lifecycle - a turn is one assistant response + any tool calls/results
	| { type: "turn_start" }
	| { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
	// Message lifecycle - emitted for system, user, assistant, and toolResult messages
	| { type: "message_start"; message: AgentMessage }
	// Only emitted for assistant messages during streaming
	| { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
	| { type: "message_end"; message: AgentMessage }
	// Tool execution lifecycle
	| { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
	| { type: "tool_execution_update"; /* ... */ }
	| { type: "tool_execution_end"; /* ... */ };
```

文字流、工具进度、生命周期，全在这十种里。你想画成终端、网页还是日志文件，是你的事。

**拒绝三：不做持久化。** `Agent` 类的对话历史就是一个内存数组：

```typescript
// src/agent.ts:83（createMutableAgentState 内）
let tools = initialState?.tools?.slice() ?? [];
let messages = initialState?.messages?.slice() ?? [];
```

进程退出，一切归零。想要能恢复的会话，有两条路：把这个包当循环用、在外面自己存（coding-agent 就是这样，它在 `initialState.messages` 里塞进从会话文件读出的历史，在钩子里把权威上下文换回来）；或者换一个从地基起就为持久化设计的运行时（`@earendil-works/pi-durable`，本章「上游与下游」）。包本身对这两种选择一无所知。

**拒绝四：不碰运行时 API。** 六个源文件里没有任何 `node:fs`、`node:child_process`，也没有任何 `node:*`。不信可以在 `packages/agent/src/` 里 grep `from "node:`，零命中。`package.json` 的 `exports` 也只剩一个入口：

```json
// package.json:8
"exports": {
	".": {
		"types": "./dist/index.d.ts",
		"import": "./dist/index.js"
	},
	"./package.json": "./package.json"
},
```

旧版本这里有四个入口——`./node` 专门放唯一的 Node 实现，`./experimental` 隔离进行中的代码。把"哪部分代码敢碰运行时 API"提升到包边界，是那时的架构声明。现在这个声明更简单了：**整个包都不敢碰**。能碰文件和进程的东西——内置工具、会话存储——随 harness 一起搬出去了。所以这个包可以直接跑在浏览器里，`src/proxy.ts` 就是为此准备的。

## 拒绝的另一面：留出来的位置

每一条拒绝都留下了一个位置，由调用方来填。按耦合从松到紧，一共四类：

1. **模型调用：`streamFn`。** 循环对模型层的全部依赖是一个函数形状（「拒绝一」引过）。换厂商、走代理、塞 mock，都是传一个不同的函数进来。最松：每个 `Agent` 实例都可以不一样，`prepareRequest` 甚至能在一个 run 中途换掉模型（第 2 章）。
2. **边界上的翻译：`transformContext` 和 `convertToLlm`。** 循环内部说的是 `AgentMessage`，一种可以由应用扩展的消息类型；在调模型的那一刻，这两个函数把它剪裁、翻译成模型听得懂的 `Message`。应用的自定义消息（通知、工件卡片）能和模型消息住在同一个数组里，靠的就是这两道闸门（第 3 章）。
3. **生命周期上的钩子。** `beforeToolCall` / `afterToolCall` 拦截工具调用，`finishTurn` 决定一圈之后停不停，`prepareNextTurn` / `prepareRequest` 在请求前换上下文、换模型。它们都是 `Agent` 上可赋值的公开属性，宿主可以在构造后随时叠一层（第 2 章「岔路」一节有 coding-agent 的真实例子）。
4. **执行体：`AgentTool.execute`。** 包里一个工具都不带。工具是什么、能碰什么，全由调用方写在 `execute` 里；循环只负责校验参数、按顺序或并发调用、把结果包成消息。"不碰运行时 API"能成立，正是因为碰运行时的代码全在这个位置后面。

`proxy.ts` 是第 1 类的一个现成填法。它实现的就是 `StreamFn` 的形状，只是把请求发给你自己的服务器：

```typescript
// src/proxy.ts:107
export function streamProxy(
	model: Model<any>,
	context: TranscriptContext,
	options: ProxyStreamOptions,
): AssistantMessageEventStream {
```

注释里交代了它多做的一件事："The server strips the partial field from delta events to reduce bandwidth. We reconstruct the partial message client-side."（`src/proxy.ts:74`）——为了省带宽，服务器只发增量，完整的 partial 消息在浏览器这一侧重建。用法就是把它塞进 `streamFn`：

```typescript
// README.md:515（Proxy Usage 一节的示例，删减）
import { Agent, streamProxy } from "@earendil-works/pi-agent-core";

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: "...",
      proxyUrl: "https://your-server.com",
    }),
});
```

它和直连厂商的 `streamFn` 在循环眼里没有区别。这正是"注入一个函数形状"的意义：浏览器支持不是一个特性开关，只是第 1 类位置的又一种填法。

## 文件图：一个包，六个文件

现在把镜头拉远，看整个包。旧版本这里是一张两层的图：核心层两千多行，harness 层一万多行。现在只剩一层：

```text
                    ┌─────────────────────────────┐
                    │   @earendil-works/pi-ai     │  (另一个包：消息类型、事件流、
                    │                             │   对话记录工具、参数校验)
                    └──────────────┬──────────────┘
                                   │ 类型 + 少量无副作用的工具函数
   ┌───────────────────────────────┼────────────────────────────────┐
   │ @earendil-works/pi-agent-core ▼                    共 2519 行   │
   │                                                                │
   │   types.ts        539 行   全部公开类型：消息、工具、事件、配置   │
   │   stream-fn.ts     20 行   默认 streamFn 的全局插槽              │
   │   agent-loop.ts   949 行   无状态循环：runAgentLoop / agentLoop  │
   │   agent.ts        613 行   Agent 类：状态、队列、run 生命周期    │
   │   proxy.ts        393 行   经服务器转发的 streamFn 实现          │
   │   index.ts          5 行   出口桶                               │
   └────────────────────────────────────────────────────────────────┘
```

五个文件各管一件事，第六个只负责导出：

```typescript
// src/index.ts:1
export * from "./agent.ts";
export * from "./agent-loop.ts";
export * from "./proxy.ts";
export { setDefaultStreamFn } from "./stream-fn.ts";
export * from "./types.ts";
```

注意 `stream-fn.ts` 只导出了 `setDefaultStreamFn`，`getDefaultStreamFn` 留在包内——外面只能装，不能读。

## 依赖：一棵很矮的树

只看 import 关系（`typebox` 等外部包略去）：

```text
types.ts      ◀─────────────── stream-fn.ts
   ▲                                ▲
   │                                │
agent-loop.ts ──────────────────────┘
   ▲
   │
agent.ts                    proxy.ts（只依赖 pi-ai，与另外四个文件无关）
```

箭头全部指向上游，没有环：`types.ts` 是地基，`agent-loop.ts` 在其上，`agent.ts` 是循环的一种包装。`proxy.ts` 是一片孤立的叶子——它连本包的类型都不引，只实现 pi-ai 定义的 `StreamFn` 形状。`agent.ts` 从循环拿的值只有两个函数：

```typescript
// src/agent.ts:14
import { runAgentLoop, runAgentLoopContinue } from "./agent-loop.ts";
```

也就是说，`Agent` 不是循环的一部分，而是循环的**第一个消费者**。循环本身不知道 `Agent` 存在；你完全可以绕过 `Agent`，直接调 `agentLoop()` 拿事件流（第 2 章「emit 的另一头」一节对比了两种用法）。

### 对 pi-ai 的依赖

图里最粗的那条箭头指向包外。旧版本的说法是"本包对 pi-ai 的唯一依赖就是几个类型"，现在不成立了：`package.json` 的 dependencies 里写着它，循环也从它导入真正的函数。

```typescript
// src/agent-loop.ts:6
import {
	type AssistantMessage,
	EventStream,
	getCurrentTools,
	getToolStateChanges,
	normalizeContext,
	type SystemMessage,
	type ToolResultMessage,
	type ToolStateChanges,
	toToolDeclaration,
	validateToolArguments,
} from "@earendil-works/pi-ai";
```

这些值分三类：事件流的容器（`EventStream`）、对话记录的工具函数（`normalizeContext`、`getCurrentTools`、`getToolStateChanges`、`toToolDeclaration`）、工具参数的 schema 校验（`validateToolArguments`）。没有一个和具体厂商有关。

"依赖 pi-ai"和"不认识厂商"能同时成立，是因为 pi-ai 自己也分了层。它的主入口文件头写明了只放什么：

```typescript
// pi-ai/src/index.ts:4
// Core only, side-effect free: no generated catalogs, no provider factories,
// no api-registry, no OAuth implementations, no compat. Provider factories
// live under "@earendil-works/pi-ai/providers/*", API implementations under
// "@earendil-works/pi-ai/api/*", the old global API under
// "@earendil-works/pi-ai/compat".
```

本包只 import 这个主入口。厂商实现在 `providers/*` 和 `api/*` 子路径里，本包一个都不碰——依赖的是 pi-ai 的**协议层**，不是它的**厂商层**。

## 上游与下游

把视野再放大一圈，看这个包在仓库里的位置。

**上游是 pi-ai。** 它承担的比以前多：除了消息类型和事件流，对话记录怎么重放、system prompt 怎么从消息里读出来、工具声明怎么比较，这些逻辑现在都在 pi-ai 里（第 3 章）。原因很实际：provider 也要用它们，放在本包里就得让 pi-ai 反过来依赖本包。

**下游有两种用法，正好代表两种态度。**

第一种是 **coding-agent：包一层**。它直接 `new Agent(...)`，把自己的东西塞进本章「拒绝的另一面」列出的那四类位置：

```typescript
// packages/coding-agent/src/core/sdk.ts:411（仓库根起，删减）
const agent = new Agent({
	initialState: {
		systemPrompt: "",
		model,
		thinkingLevel,
		tools: [],
		messages: existingSession.messages,
	},
	convertToLlm: convertToLlmWithBlockImages,
	streamFn: async (model, context, options) => {
		const requestOptions = buildRequestOptions(model, options);
		// ...（缓存预热）
		return modelRuntime.streamSimple(model, context, requestOptions);
	},
	// ...
	transformContext: async (messages) => {
		const runner = extensionRunnerRef.current;
		if (!runner) return messages;
		return runner.emitContext(messages);
	},
	// ...
});
```

历史消息从会话文件读出来塞进 `initialState`，`streamFn` 接到自己的模型运行时上，`transformContext` 交给扩展系统。构造之后它还会在 `finishTurn`、`prepareNextTurn`、`prepareRequest` 上各叠一层（第 2 章）。持久化、压缩、扩展、工具——全部在本包之外，经这些位置接进来。顺带一提，它还替旧扩展装了一个全局默认流函数，注释把分寸说得很清楚：

```typescript
// packages/coding-agent/src/core/sdk.ts:43（仓库根起）
// Preserve the pre-0.81 fallback for extensions that construct Agent instances
// or invoke low-level agent loops without supplying streamFn. Agent core remains
// provider-agnostic and does not import pi-ai/compat itself.
setDefaultStreamFn(streamSimple);
```

第二种是 **pi-durable：另起一个**。它就是 1.0.0 时从本包搬出去的那套 harness 的去处，自称 "A durable agent harness"——对话、模型回合、工具调用都先写进存储再显示，进程中途死掉，重新打开就从断点继续。它的依赖列表里没有本包：

```json
// packages/durable/package.json:146（仓库根起）
"dependencies": {
	"@earendil-works/chord": "1.3.0",
	"@earendil-works/pi-ai": "1.3.0",
	"@earendil-works/pi-codemode": "1.3.0",
	"@earendil-works/pi-telemetry": "1.3.0",
	"@silvia-odwyer/photon": "0.3.3",
	"diff": "8.0.4",
	"typebox": "1.3.27"
},
```

它有自己的生成逻辑，直接调 pi-ai 的 `streamSimple`：

```typescript
// packages/durable/src/harness/generation.ts:484（仓库根起）
const events = runtime.models.streamSimple(model, { messages: [...messages] }, options);
```

这和旧版本的结构不一样。那时 `Agent` 和 `AgentHarness` 是同一个循环原语上并排的两层组合，都调 `runAgentLoop`。现在持久化运行时不再复用这个循环：它要的是"每一步先落盘、可以从任意断点恢复"，而本包的循环是一个跑在内存里、从头跑到尾的 `async` 函数，这两种执行模型套不到一起。于是 pi-durable 只和本包共享上游（pi-ai 的消息与流协议），不共享循环。

两种用法放在一起，本包的定位就清楚了：**它是给"包一层"的人用的。** 你要一个能嵌进应用的循环，自己决定存储、工具和 UI，用它。你要的是持久化、可恢复的运行时，用 pi-durable，它和本包讲同一种消息语言，但不是本包的上层。

## 一章小结

- `pi-agent-core` 是一个 agent 循环库：一个无状态循环、一层状态（`Agent`），外加一个浏览器代理流。六个文件，两千五百行。
- 四条拒绝：不认识厂商、不碰 UI、不做持久化、不碰运行时 API。后两条比旧版本更彻底——相关代码在 1.0.0 整体搬出了本包。
- 每条拒绝都留了位置：`streamFn`、边界上的两道翻译、生命周期钩子、工具的 `execute`。
- 依赖图是一棵矮树，箭头全部朝上游：`types.ts` 是地基，循环在其上，`Agent` 是循环的第一个消费者，`proxy.ts` 是孤立的叶子。
- 运行时依赖 pi-ai，但只依赖它无副作用的协议层，不碰厂商层。
- 下游两种态度：coding-agent 包一层，pi-durable 另起一个循环。本包服务前者。

下一章我们不再谈定位，跟着一次真实的 `prompt()` 调用，把循环从头到尾走一遍。

## 为什么不去

> **为什么把 harness 搬出去，而不是留在包里当可选部分？** 旧版本已经用子路径（`./node`、`./experimental`）把它隔开了，看起来够用。但 harness 和循环的演化速度差得太远：它一路长出了 session 存储、压缩、内置工具、pico 系列内核，体量是核心的四五倍，契约还在频繁变动；而核心是一个需要稳定的小库。commit `7fd478a2e` 的说明列出了被移走的全部内容，最后一句指出了去处：
>
> ```text
> // commit 7fd478a2e（2026-10-01）
> pi-agent-core now contains only Agent, the agent loop, the proxy stream, and
> their types. Removed the harness, sessions and session storage, pico3, harness
> tools, compaction, skills, prompt templates, telemetry schemas, search types,
> and the uuidv7 and pi-telemetry re-exports, plus the ./node, ./harness/* and
> ./experimental/pico3 subpath exports. Durable sessions live in
> @earendil-works/pi-durable.
> ```
>
> 连同一起移除的 `packages/session-backends` 和两个实验性前端，这个提交删掉了十万多行。分开之后，两边各按自己的节奏走：pi-durable 的 README 开头就标着 "Experimental. The API changes without notice between releases"；`pi-agent-core` 在同一版本进入 1.0.0。

> **既然不认识厂商，为什么还要运行时依赖 pi-ai，而不是只依赖类型？** 因为消息语言本身就带着行为。system prompt 搬进对话记录之后（第 3 章），"当前提示词是什么""当前有哪些工具"都要靠重放消息算出来，这套算法 provider 也要用，只能放在两边共同的上游。复制一份到本包，两边迟早算出不同的答案。代价被 pi-ai 的分层压住了：主入口只放无副作用的协议代码，厂商实现留在子路径里。本包在 0.8x 时专门处理过这个问题：先是把可选的默认 `streamFn` 改成必填，免得 `pi-ai/compat` 连同所有内置 provider 被拉进只想用一两家厂商的打包结果（#6851）；随后又把"可省略"找了回来，但默认值改由宿主安装，本包自己不再引用 compat：
>
> ```markdown
> // CHANGELOG.md:181
> - Restored the `Agent` `streamFn` option and host-configurable fallback for omitted agent-loop stream functions without reintroducing a `pi-ai/compat` dependency ([#6915](https://github.com/earendil-works/pi/issues/6915)).
> ```
>
> 依赖一个协议，而不是依赖一个目录——这是 `StreamFn` 和 `setDefaultStreamFn` 存在的原因，也是"依赖 pi-ai"和"不认识厂商"能同时成立的原因。
