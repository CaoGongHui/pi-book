# 第 7 章 · 失败变成值：贯穿全包的一条契约

前六章各讲一个部件。这一章讲一件不属于任何单一部件、却在每个部件里都出现的事：**出错的时候怎么办**。

翻一遍 `src/types.ts`，"Contract: must not throw or reject" 这句话出现了五次，`StreamFn` 的注释里还有一句更长的版本。第 4 章的 `runToolCall` 写着 "Never rejects for tool failures"，第 5 章的 `handleRunFailure` 把异常改写成一条消息，第 6 章的 `streamProxy` 把网络断开改写成一个事件。这些不是各自独立的小决定，是同一条规则在不同位置的落实：

> **失败是数据，不是控制流。**

这一章把这条规则摊开：它要求了谁，它在哪里被兑现，以及它在哪里没被兑现。

## 为什么是这条规则

先看循环的下游是谁。循环的输出有两个读者：订阅事件的 UI，和下一轮请求里的模型。

- **对 UI**，一次失败最好的样子是一个事件：它和成功走同一条渲染路径，最后一条 assistant 消息的 `stopReason` 是 `error`，界面就显示错误。如果失败走 Promise rejection，UI 就得维护两套路径，还要处理"事件发到一半、Promise 先 reject 了"的中间态。
- **对模型**，一次失败最好的样子是一条消息：工具返回"文件不存在"，模型下一轮就能换个路径；参数校验失败，模型读到错误列表就能改。异常往上抛，run 就断了，模型永远不知道发生过什么。

两个读者都要"值"，不要"异常"。所以规则的方向是：**尽可能早地把异常捕获下来，转成这两个读者能读懂的形状**。

## 契约的两种写法

全包有两种方式落实这条规则：要求别人不抛，或者自己兜住。

### 要求：写在类型注释里的六份契约

`StreamFn` 的契约最完整：

```typescript
// src/types.ts:27（StreamFn 的 JSDoc 节选）
 * Contract:
 * - Must not throw or return a rejected promise for request/model/runtime failures.
 * - Must return an AssistantMessageEventStream.
 * - Failures must be encoded in the returned stream via protocol events and a
 *   final AssistantMessage with stopReason "error" or "aborted" and errorMessage.
```

三句话分别规定：不许抛；必须返回流；失败必须编码成流里的事件，并以一条特定形状的最终消息收尾。第三句最关键，它规定了失败**长什么样**——不是随便一个错误对象，而是一条和正常回答同类型的 `AssistantMessage`，只是 `stopReason` 不同。这让失败可以原样进入对话记录。

`AgentLoopConfig` 上另有五个回调签了"不许抛"：

| 回调 | 失败时应该返回 | 位置 |
|---|---|---|
| `convertToLlm` | 一个安全的回退值 | `src/types.ts:203` |
| `transformContext` | 原消息或其他安全回退值 | `src/types.ts:231` |
| `getApiKey` | `undefined` | `src/types.ts:252` |
| `getSteeringMessages` | `[]` | `src/types.ts:291` |
| `getFollowUpMessages` | `[]` | `src/types.ts:304` |

每一份都告诉你出错时**该返回什么**，而不只是"别抛"。`convertToLlm` 的那份还写了后果：

```typescript
// src/types.ts:203（AgentLoopConfig.convertToLlm 的 JSDoc 节选）
	 * Contract: must not throw or reject. Return a safe fallback value instead.
	 * Throwing interrupts the low-level agent loop without producing a normal event sequence.
```

"without producing a normal event sequence"——这句话是理解整章的钥匙，下一节会看到它具体指什么。

### 兜底：循环自己捕获的三处

另一种写法是不信任调用方，自己 `try/catch`。全包的 `agent-loop.ts` 里只有三个 `catch`，全在工具管线上（`src/agent-loop.ts:770`、`:845`、`:898`）：

- `prepareToolCall` 的 `catch`：找工具、修补参数、校验、`beforeToolCall` 抛的任何东西；
- `executePreparedToolCall` 的 `catch`：工具 `execute()` 抛的任何东西；
- `finalizeExecutedToolCall` 的 `catch`：`afterToolCall` 抛的任何东西。

三处都把异常变成同一个形状——`createErrorToolResult` 造的结果：

```typescript
// src/agent-loop.ts:912
function createErrorToolResult(message: string): AgentToolResult<any> {
	return {
		content: [{ type: "text", text: message }],
		details: {},
	};
}
```

只有 `content`，因为它的读者是模型。

为什么工具这里自己兜、别处却要求调用方不抛？看谁写代码。`streamFn`、`convertToLlm` 这些回调由**宿主**写，一个应用通常只写一份，写的人清楚自己在做什么。工具由**很多人**写：宿主写、扩展作者写、甚至模型生成的脚本间接调用。工具数量多、质量参差、经常抛异常是常态。在这里要求"不许抛"等于没要求，只能自己兜。`beforeToolCall` / `afterToolCall` 的 JSDoc 只要求它们"honor the abort signal"（`src/types.ts:324`、`:339`），没有写不许抛的契约——循环本来就会兜住它们。

## 契约断了会怎样

现在回到那句"without producing a normal event sequence"。如果一个签了契约的回调还是抛了，会发生什么？

看循环的调用点：`runLoop` 里调用 `getSteeringMessages`、`prepareNextTurn`、`prepareRequest`、`finishTurn`、`getFollowUpMessages`，`streamAssistantResponse` 里调用 `transformContext`、`convertToLlm`、`getApiKey`，全都是裸 `await`，周围没有 `try`（`src/agent-loop.ts:176` 到 `:401` 之间的十一处调用）。异常直接冒出 `runLoop`、冒出 `runAgentLoop`。

接下来怎么样，取决于你用的是哪个入口。

### 用 Agent：兜得住

第 5 章讲过：`Agent.runWithLifecycle` 有一个 `catch`，异常进 `handleRunFailure`，被改写成一条失败消息和一组补发的事件（`message_start` → `message_end` → `turn_end` → `agent_end`）。订阅者看到的序列依然完整，`prompt()` 正常 resolve，`state.errorMessage` 带着错误文本。

所以对 `Agent` 的使用者，契约断了的代价是"run 以一个错误结束"——和 provider 报错的效果一样。

### 用裸 agentLoop：挂住

低层入口没有这层兜底。看 `agentLoop` 的实现：

```typescript
// src/agent-loop.ts:38
export function agentLoop(
	prompts: AgentMessage[],
	context: AgentContext,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
	const stream = createAgentStream();

	void runAgentLoop(
		prompts,
		context,
		config,
		async (event) => {
			stream.push(event);
		},
		signal,
		streamFn,
	).then((messages) => {
		stream.end(messages);
	});

	return stream;
}
```

`runAgentLoop(...).then(...)`，没有 `.catch`。异常让这个 Promise reject，`then` 里的 `stream.end(messages)` 永远不执行。再看流那一侧——`EventStream` 来自 pi-ai：

```typescript
// pi-ai/src/utils/event-stream.ts:60（EventStream 内，删减：push 方法）
end(result?: R): void {
	this.done = true;
	if (result !== undefined) {
		this.resolveFinalResult(result);
	}
	// Notify all waiting consumers that we're done
	while (this.waiting.length > 0) {
		const waiter = this.waiting.dequeue()!;
		waiter({ value: undefined as any, done: true });
	}
}

async *[Symbol.asyncIterator](): AsyncIterator<T> {
	while (true) {
		if (this.queue.length > 0) {
			yield this.queue.dequeue()!;
		} else if (this.done) {
			return;
		} else {
			const result = await new Promise<IteratorResult<T>>((resolve) => this.waiting.enqueue(resolve));
			if (result.done) return;
			yield result.value;
		}
	}
}
```

迭代器在队列空、流未结束时，挂一个等待者进 `waiting` 队列，等 `push` 或 `end` 来唤醒它。`end` 永远不来，`agent_end` 事件也永远不来（`push` 里只有收到 `agent_end` 才会把 `done` 置真），这个等待者就永远挂着。调用方的 `for await` 卡死，`stream.result()` 也永不 resolve。而那个被 reject 的 Promise 前面有个 `void`，没有人接，在 Node 里会变成一个 unhandled rejection。

所以"without producing a normal event sequence"的具体含义是：**用低层 API 时，契约断了的代价不是一个错误，而是永久挂起**。这正是第 6 章 `streamProxy` 那个 EOF bug 的同一种病：流没有终止事件，消费者永远等。

### 两层的分工

把两种情况放在一起，分工就清楚了：

- **循环**（`agent-loop.ts`）只承诺：在所有回调守约的前提下，事件序列完整、流一定结束。它把守约的责任交给调用方，自己不加 `try`，保持控制流简单。
- **`Agent`**（`agent.ts`）承诺得更多：哪怕某个回调失约，也给出一个完整的事件序列。代价是一个 `catch` 和一个伪造失败消息的函数。

这和第 1 章的定位一致：`Agent` 是循环的第一个消费者，替你做掉了一部分防御。直接用 `agentLoop()` 的人拿到了更多控制权，也接过了这份防御的责任——要么保证回调不抛，要么自己包一层：

```typescript
// 演示：给低层入口加一层兜底（应用侧代码）
const safeConfig: AgentLoopConfig = {
	...config,
	convertToLlm: async (messages) => {
		try {
			return await config.convertToLlm(messages);
		} catch {
			return messages.filter((m) => m.role === "user" || m.role === "assistant" || m.role === "toolResult" || m.role === "system");
		}
	},
};
```

## 失败的形状

最后看失败最终长成什么样。全包的失败只有两种形状，对应两个读者：

**给 UI 和记录的：一条 `stopReason` 为 `error` 或 `aborted` 的 assistant 消息。** 来源有四个，形状一致：

| 来源 | 谁造的 |
|---|---|
| provider 报错 | pi-ai 的 provider，按 `StreamFn` 契约 |
| 网络断开、服务器拒绝 | `streamProxy`（第 6 章） |
| 用户中止 | provider 或 `streamProxy`，看 `signal` |
| 回调失约、bug | `Agent.handleRunFailure`（第 5 章） |

循环对这条消息只有一个反应：走硬出口（第 2 章「agent 循环什么时候停」）——`finishTurn` 被调用但结论被忽略，发 `turn_end`、`agent_end`，run 结束。下游分不出失败来自哪里，也不需要分。

**给模型的：一条 `isError: true` 的 `toolResult` 消息。** 来源是工具管线的五道关卡和三个 `catch`（第 4 章）。循环对它没有特殊反应：这一圈照常结束，结果回喂模型，下一圈继续。

两种形状的差别正是"失败了还能不能继续"的差别。模型调用失败，没有东西可以喂给下一轮，run 必须停；工具调用失败，失败本身就是下一轮的输入，run 应该继续。规则只有一条，落到两个位置，就长出了两种语义。

## 一章小结

- 全包的失败处理遵循一条规则：失败是数据，不是控制流。它的两个读者——UI 和模型——都要"值"，不要"异常"。
- 落实方式有两种：对宿主写的回调，用类型注释里的契约要求它们不抛，并规定失败时返回什么（六份契约）；对工具，循环自己用三个 `catch` 兜住，因为工具数量多、作者多、抛异常是常态。
- 循环本身不加 `try`，只在回调守约的前提下保证事件序列完整。契约断了：用 `Agent`，`handleRunFailure` 兜成一个普通错误；用裸 `agentLoop()`，流永远不结束，消费者永久挂起。
- 失败最终只有两种形状：`stopReason: "error" | "aborted"` 的 assistant 消息（run 停），`isError: true` 的 toolResult（run 继续）。

## 为什么不去

> **为什么循环不在每个回调调用点都包一层 `try`，像 `Agent` 那样兜住？** 因为循环兜住之后不知道该怎么办。`convertToLlm` 抛了，该用什么回退？原样发送所有消息，可能把自定义消息送进 provider；发送空列表，模型会收到一个没有上下文的请求。`getSteeringMessages` 抛了，是当作没有消息、还是停下来？每个回调的合理回退都不一样，而只有写回调的人知道。所以契约里写的不是"别抛"，而是"出错时返回 X"——回退值由最了解它的人提供。循环自己只兜一种情况：工具。工具失败的回退是唯一确定的——告诉模型失败了。

> **为什么 `agentLoop` 的 `.then` 后面不补一个 `.catch` 来结束流？** 从代码里能读出的只是现状：没有补。补上之后，异常会变成一个提前结束、没有 `agent_end` 的流，消费者至少不会挂住，但也拿不到一个完整的事件序列，还是得自己判断"流结束了但没有 `agent_end`，说明出错了"。`Agent` 选择了更完整的做法：伪造一个失败消息，把序列补齐。低层入口把这个选择留给调用方。这是本书唯一一处我们读出了取舍、却在仓库里没找到明确记录的地方，所以这里只陈述行为，不替作者给理由。
