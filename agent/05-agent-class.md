# 第 5 章 · Agent 类：状态、队列与收场

第 1 章说 `Agent` 是循环的第一个消费者。第 2 章跟着它走了一遍 `prompt()`：拍快照、装配配置、建生命周期、逐个 await 监听器。这一章换个角度，不看一次调用的流程，而看 `Agent` 这个对象**持有什么**、**怎么改它**：

1. 状态由哪些字段组成，谁能写，什么时候写？
2. 两条消息队列怎么存、怎么取，`one-at-a-time` 和 `all` 差在哪？
3. `continue()` 为什么有三条分支？
4. run 中途抛了异常，状态和事件序列怎么收场？

## 状态：一半归你，一半归循环

公开状态的接口：

```typescript
// src/types.ts:382（删减：JSDoc 只留首句）
export interface AgentState {
	/** Current system prompt, replayed from the transcript's system messages. ... */
	readonly systemPrompt: string;
	/** Active model used for future turns. */
	model: Model<any>;
	/** Requested reasoning level for future turns. */
	thinkingLevel: ThinkingLevel;
	/** Executable tools. Assigning a new array copies the top-level array. ... */
	set tools(tools: AgentTool<any>[]);
	get tools(): AgentTool<any>[];
	/** Conversation transcript. Assigning a new array copies the top-level array. ... */
	set messages(messages: AgentMessage[]);
	get messages(): AgentMessage[];
	/** True while the agent is processing a prompt or continuation. ... */
	readonly isStreaming: boolean;
	/** Partial assistant message for the current streamed response, if any. */
	readonly streamingMessage?: AgentMessage;
	/** Tool call ids currently executing. */
	readonly pendingToolCalls: ReadonlySet<string>;
	/** Error message from the most recent failed or aborted assistant turn, if any. */
	readonly errorMessage?: string;
}
```

字段按"谁能写"分成两组：

| 组 | 字段 | 写入方 |
|---|---|---|
| 配置 | `model`、`thinkingLevel`、`tools`、`messages` | 你，随时可赋值 |
| 运行时 | `isStreaming`、`streamingMessage`、`pendingToolCalls`、`errorMessage` | 只有 `Agent` 自己 |
| 派生 | `systemPrompt` | 没人写，读时从 `messages` 重放（第 3 章） |

运行时那组在接口上是 `readonly`，但内部用的是另一个类型——把这四个字段的 `readonly` 去掉：

```typescript
// src/agent.ts:70
type MutableAgentState = Omit<AgentState, "isStreaming" | "streamingMessage" | "pendingToolCalls" | "errorMessage"> & {
	isStreaming: boolean;
	streamingMessage?: AgentMessage;
	pendingToolCalls: Set<string>;
	errorMessage?: string;
};
```

`agent.state` 返回的就是这个内部对象本身（第 2 章「emit 的另一头」引过访问器），只是类型上收窄成 `AgentState`。这是 TypeScript 里常见的"对外只读、对内可写"写法：不拷贝、不包代理，靠类型约束。

配置组里的 `tools` 和 `messages` 用访问器实现，赋值时拷贝顶层数组：

```typescript
// src/agent.ts:94（createMutableAgentState 返回对象内）
get tools() {
	return tools;
},
set tools(nextTools: AgentTool<any>[]) {
	tools = nextTools.slice();
},
get messages() {
	return messages;
},
set messages(nextMessages: AgentMessage[]) {
	messages = nextMessages.slice();
},
```

读写不对称：**赋值拷贝，读取不拷贝**。所以 `agent.state.messages = myArray` 之后再改 `myArray`，影响不到 agent；而 `agent.state.messages.push(x)` 会直接改到 agent 的数组。README 专门提醒了后一种（"Mutating the returned array mutates the current agent state"）。

### 运行时字段怎么变

运行时那四个字段只在两个地方被写：`processEvents` 归约事件，`runWithLifecycle` / `finishRun` 开关一个 run。归约部分是一个 `switch`：

```typescript
// src/agent.ts:565（Agent.processEvents，删减：末尾调用监听器的部分，见第 2 章「收尾」）
private async processEvents(event: AgentEvent): Promise<void> {
	switch (event.type) {
		case "message_start":
			this._state.streamingMessage = event.message;
			break;

		case "message_update":
			this._state.streamingMessage = event.message;
			break;

		case "message_end":
			this._state.streamingMessage = undefined;
			this._state.messages.push(event.message);
			break;

		case "tool_execution_start": {
			const pendingToolCalls = new Set(this._state.pendingToolCalls);
			pendingToolCalls.add(event.toolCallId);
			this._state.pendingToolCalls = pendingToolCalls;
			break;
		}

		case "tool_execution_end": {
			const pendingToolCalls = new Set(this._state.pendingToolCalls);
			pendingToolCalls.delete(event.toolCallId);
			this._state.pendingToolCalls = pendingToolCalls;
			break;
		}

		case "turn_end":
			if (event.message.role === "assistant" && event.message.errorMessage) {
				this._state.errorMessage = event.message.errorMessage;
			}
			break;

		case "agent_end":
			this._state.streamingMessage = undefined;
			break;
	}
	// ...
}
```

三处值得停顿：

- **消息只在 `message_end` 入数组。** `message_start` 和 `message_update` 只更新 `streamingMessage`。所以 `state.messages` 里永远是完整的消息，正在流的那条单独放在一边——UI 渲染时把两者拼起来，就是"历史 + 正在打字"。
- **`pendingToolCalls` 每次换新对象。** 加一个 id、删一个 id，都是 `new Set` 拷贝再赋值，而不是原地 `add` / `delete`。原因是给 UI 框架用：React 之类按引用比较状态，原地修改一个 Set，引用不变，界面不会刷新。消息数组却是原地 `push` 的——它太大，每次拷贝不划算；UI 要的"有新消息了"信号，由 `message_end` 事件本身提供。
- **`errorMessage` 只在 `turn_end` 时看。** 一个出错的 assistant 消息会带着 `errorMessage`，归约把它抄到状态上；下一个 run 开始时 `runWithLifecycle` 把它清掉（第 2 章引过）。

注意这里的归约和循环自己维护的 `currentContext.messages` 是**两份数组**。循环往自己的工作副本里推消息，`Agent` 在 `message_end` 时往 `state.messages` 推同一条消息。两份同步靠的是事件，不是共享引用——快照在 run 开始时就拷贝过了（`createContextSnapshot` 的 `slice()`，第 2 章）。

## 两条队列

`steer()` 和 `followUp()` 各往一条队列里放消息：

```typescript
// src/agent.ts:298
/** Queue a message to be injected after the current assistant turn finishes. */
steer(message: AgentMessage): void {
	this.steeringQueue.enqueue(message);
}

/** Queue a message to run only after the agent would otherwise stop. */
followUp(message: AgentMessage): void {
	this.followUpQueue.enqueue(message);
}
```

两条队列是同一个类的两个实例：

```typescript
// src/agent.ts:143
class PendingMessageQueue {
	private messages: AgentMessage[] = [];
	public mode: QueueMode;

	constructor(mode: QueueMode) {
		this.mode = mode;
	}

	enqueue(message: AgentMessage): void {
		this.messages.push(message);
	}

	hasItems(): boolean {
		return this.messages.length > 0;
	}

	peek(): AgentMessage[] {
		if (this.mode === "all") return this.messages.slice();
		const first = this.messages[0];
		return first ? [first] : [];
	}

	drain(): AgentMessage[] {
		const drained = this.peek();
		this.messages = this.messages.slice(drained.length);
		return drained;
	}

	clear(): void {
		this.messages = [];
	}
}
```

全部语义集中在 `peek()`：`all` 模式取全部，`one-at-a-time` 模式只取最老的一条。`drain()` 就是 `peek()` 再把取走的那几条切掉。把"取哪些"和"取走"拆成两个方法，`peekQueuedMessages()` 才能做"预览但不消费"。

两条队列的默认模式都是 `one-at-a-time`：

```typescript
// src/agent.ts:247（Agent 构造函数内）
this.steeringQueue = new PendingMessageQueue(runtimeOptions.steeringMode ?? "one-at-a-time");
this.followUpQueue = new PendingMessageQueue(runtimeOptions.followUpMode ?? "one-at-a-time");
```

两种模式的区别落到循环上，是"一次插队消耗几圈"。用户在 agent 工作时连发三条 steering，`all` 模式下三条一起进下一圈，模型一次读完、回一次；`one-at-a-time` 模式下每圈只进一条，模型逐条回应，要多跑两圈。默认选的是后者。代码里没写理由，但效果很清楚：用户连发的消息常常是"补充、纠正、再纠正"，逐条处理时每一条都能拿到一个明确的回应；想省圈数，构造时传 `steeringMode: "all"`。

### 队列怎么接进循环

循环不知道队列存在，它只认 `AgentLoopConfig` 上的两个函数。`createLoopConfig` 把队列接成这两个函数：

```typescript
// src/agent.ts:496（Agent.createLoopConfig 内）
getSteeringMessages: async () => {
	if (skipInitialSteeringPoll) {
		skipInitialSteeringPoll = false;
		return [];
	}
	return this.steeringQueue.drain();
},
getFollowUpMessages: async () => this.followUpQueue.drain(),
```

`skipInitialSteeringPoll` 是个一次性开关：为真时，循环开头那次 steering poll 返回空，之后照常。它为 `continue()` 的一条分支准备，下一节就会看到。

预览函数把两条队列的优先级写得很直白：

```typescript
// src/agent.ts:329
/** Preview the messages selected for the next turn without consuming them. */
peekQueuedMessages(): AgentMessage[] {
	const steering = this.steeringQueue.peek();
	return steering.length > 0 ? steering : this.followUpQueue.peek();
}
```

有 steering 就只看 steering，没有才看 follow-up。这和循环里的顺序一致：steering 在每圈之后问，follow-up 要等内层彻底停了才问（第 2 章「外层循环」）。所以"下一圈会处理哪些消息"的答案，永远先是 steering。

## continue()：三条分支

`prompt()` 是"带着新消息开始一个 run"。`continue()` 是"不带新消息，从现有记录接着跑"——典型场景是上一个 run 出错了，用户按"重试"。

```typescript
// src/agent.ts:384（Agent.continue）
async continue(): Promise<void> {
	if (this.activeRun) {
		throw new Error("Agent is already processing. Wait for completion before continuing.");
	}

	const lastMessage = this._state.messages[this._state.messages.length - 1];
	if (!lastMessage || this._state.messages.every((message) => message.role === "system")) {
		throw new Error("No messages to continue from");
	}

	if (lastMessage.role === "assistant") {
		const queuedSteering = this.steeringQueue.drain();
		if (queuedSteering.length > 0) {
			await this.runPromptMessages(queuedSteering, { skipInitialSteeringPoll: true });
			return;
		}

		const queuedFollowUps = this.followUpQueue.drain();
		if (queuedFollowUps.length > 0) {
			await this.runPromptMessages(queuedFollowUps);
			return;
		}

		throw new Error("Cannot continue from message role: assistant");
	}

	await this.runContinuation();
}
```

先看两道护栏。记录为空，或者只有系统消息（第 3 章：系统消息是开头那条提示词，不算对话），都不能继续——模型没东西可回应。

然后按最后一条消息的角色分流：

1. **最后一条不是 assistant**（是 user 或 toolResult）：正常的续跑，走 `runContinuation`，底层是 `runAgentLoopContinue`。模型的上一句话还没说，现在补上。
2. **最后一条是 assistant，但 steering 队列里有消息**：模型已经说完了，直接续跑会被 provider 拒绝（不能让 assistant 接着 assistant 说话）。但如果有人在排队，就把排队的消息当作新 prompt 开一个 run。
3. **最后一条是 assistant，steering 空、follow-up 有**：同上，用 follow-up 开 run。
4. **都没有**：抛错。

分支 2 里的 `skipInitialSteeringPoll: true` 就是上一节那个开关的用处。这里已经 `drain()` 过一次 steering 了；如果循环开头再 poll 一次，`one-at-a-time` 模式下第二条 steering 会被立刻取出、和第一条挤进同一圈。开关让循环跳过开头那次 poll，第二条留到下一圈，逐条语义得以保持。测试按两种模式各验一次：

```typescript
// test/agent.test.ts:843（"continue() keeps $mode steering semantics for assistant-tail fallback" 内）
agent.state.messages = [createUserMessage("Initial"), createAssistantMessage("Initial response")];
agent.steer(createUserMessage("Steering 1"));
agent.steer(createUserMessage("Steering 2"));

await expect(agent.continue()).resolves.toBeUndefined();

expect(requests).toHaveLength(expectedRequests);
```

`one-at-a-time` 期望两次请求，`all` 期望一次。分支 3 不需要这个开关：follow-up 不会在循环开头被 poll。

这几条分支是逐步修出来的。CHANGELOG 里有一条直接对应它：

```markdown
// CHANGELOG.md:571
- Fixed `continue()` to resume queued steering/follow-up messages when context currently ends in an assistant message, and preserved one-at-a-time steering ordering during assistant-tail resumes ([#1312](https://github.com/badlogic/pi-mono/pull/1312) by [@ferologics](https://github.com/ferologics))
```

引入分支 2 和 3 的提交是 `b050c582a`，标题交代了场景："resume queued messages after auto-compaction"。当时 coding-agent 在 run 之间做自动压缩，压缩完的记录停在一条 assistant 消息上，用户在压缩期间排的队要靠 `continue()` 接着处理。


## 收场：run 抛异常时

第 1 章说 `StreamFn` 的契约是"不许抛"：失败要编码进流里，以一条 `stopReason: "error"` 的消息收尾。但契约是约定，不是保证——自己写的 `streamFn` 可能有 bug，`convertToLlm` 可能抛，某个钩子可能抛。这些异常一路冒到 `runWithLifecycle` 的 `catch`（第 2 章引过），落到这里：

```typescript
// src/agent.ts:532
private async handleRunFailure(error: unknown, aborted: boolean): Promise<void> {
	const failureMessage = {
		role: "assistant",
		content: [{ type: "text", text: "" }],
		api: this._state.model.api,
		provider: this._state.model.provider,
		model: this._state.model.id,
		usage: EMPTY_USAGE,
		stopReason: aborted ? "aborted" : "error",
		errorMessage: error instanceof Error ? error.message : String(error),
		timestamp: Date.now(),
	} satisfies AgentMessage;
	await this.processEvents({ type: "message_start", message: failureMessage });
	await this.processEvents({ type: "message_end", message: failureMessage });
	await this.processEvents({ type: "turn_end", message: failureMessage, toolResults: [] });
	await this.processEvents({ type: "agent_end", messages: [failureMessage] });
}
```

它做的事是**伪造一个正常的结局**：造一条空内容、带错误信息的 assistant 消息，然后手动走一遍 `message_start` → `message_end` → `turn_end` → `agent_end`。这条消息的形状和 provider 正常报错时返回的完全一样（`stopReason` 是 `error` 或 `aborted`，`errorMessage` 有值），所以下游分不出这次失败来自 provider 还是来自 bug：

- `message_end` 让它进了 `state.messages`；
- `turn_end` 让归约把 `errorMessage` 抄到状态上；
- `agent_end` 让监听器照常收尾，`finishRun()` 照常 resolve。

订阅者看到的事件序列依然完整。测试用一个直接抛异常的 `streamFn` 验证了这一点：

```typescript
// test/agent.test.ts:330（删减：消息断言）
it("emits full lifecycle events for thrown run failures", async () => {
	const agent = new Agent({
		streamFn: () => {
			throw new Error("provider exploded");
		},
	});
	const events: string[] = [];
	agent.subscribe((event) => {
		events.push(event.type);
	});

	await agent.prompt("hello");

	expect(events).toEqual([
		"agent_start",
		"turn_start",
		"message_start",
		"message_end",
		"message_start",
		"message_end",
		"turn_end",
		"agent_end",
	]);
	// ...
	expect(agent.state.errorMessage).toBe("provider exploded");
});
```

前两对 `message_start`/`message_end` 来源不同：第一对是循环为 prompt 消息发的，第二对是 `handleRunFailure` 为伪造的失败消息发的。`await agent.prompt("hello")` 也没有抛——**异常被转成了状态**，调用方要看 `state.errorMessage` 或最后一条消息的 `stopReason` 才知道失败了。

两个边界情况值得知道：

- `agent_end` 里的 `messages` 只有那条失败消息，不含本次 run 里已经产生的其他消息（循环的 `newMessages` 随异常丢了）。但那些消息在各自的 `message_end` 时已经进了 `state.messages`，状态本身是完整的。
- 如果异常发生在 `agent_end` 之后（比如某个 `agent_end` 监听器抛了），`handleRunFailure` 会再发一个 `agent_end`。监听器最好别抛。

### 运行中不许重置

还有一道护栏在 `reset()` 上（第 3 章引过它的主体）。run 进行中调用直接抛错：

```typescript
// test/agent.test.ts:695（"should reject reset while processing without corrupting the transcript" 内）
expect(agent.state.isStreaming).toBe(true);
expect(agent.state.messages.map((message) => message.role)).toEqual(["user"]);
expect(() => agent.reset()).toThrow("Agent is already processing. Wait for completion before resetting.");
expect(agent.state.isStreaming).toBe(true);
```

和 `prompt()`、`continue()` 一样，凡是会和正在跑的 run 争抢 `state.messages` 的操作，一律在入口检查 `activeRun`。能在运行中做的只有不冲突的几件事：往队列里放消息、改配置字段（下一次请求生效）、`abort()`。

## 一章小结

- `AgentState` 的字段分三组：配置组（`model`、`thinkingLevel`、`tools`、`messages`）你随时可写；运行时组（`isStreaming`、`streamingMessage`、`pendingToolCalls`、`errorMessage`）只有 `Agent` 写；`systemPrompt` 是重放出来的。
- `tools` 和 `messages` 赋值时拷贝顶层数组，读取时不拷贝。
- 归约规则：消息在 `message_end` 才入数组，正在流的放在 `streamingMessage`；`pendingToolCalls` 每次换新 Set，方便 UI 按引用比较。
- 两条队列是同一个 `PendingMessageQueue` 的两个实例，默认都是 `one-at-a-time`；`peek` 决定取哪些，`drain` 取走。steering 优先于 follow-up。
- `continue()` 按最后一条消息分流：不是 assistant 就续跑；是 assistant 就用排队的消息开新 run，`skipInitialSteeringPoll` 保住逐条语义。
- run 里冒出的异常被 `handleRunFailure` 转成一条失败消息和完整的事件序列，`prompt()` 本身不抛。

下一章看本包最后一个文件：`proxy.ts`，一个跑在浏览器里、经服务器转发的 `streamFn`。

## 为什么不去

> **为什么 run 失败不让 `prompt()` 直接 reject？** 因为 `Agent` 的使用者大多是 UI。UI 的渲染由事件驱动，如果失败走 Promise rejection、成功走事件，UI 就得维护两套路径，还得处理"事件发到一半、Promise 先 reject 了"的中间态。把失败也做成一条消息、一组事件，UI 只有一套逻辑：最后一条 assistant 消息的 `stopReason` 是 `error`，就显示错误。provider 正常报错本来就是这个形状（`StreamFn` 的契约要求），`handleRunFailure` 只是让 bug 引起的失败也长成同一个样子。

> **为什么队列放在 `Agent` 里，而不是让循环自己管？** 循环是无状态的：一个 `async` 函数，跑完就结束，什么都不留。队列却必须活得比一次 run 长——用户在 run 结束前排的 follow-up，可能要等下一次 `continue()` 才被处理；`finishTurn` 投了 `end`，排着的消息也得原样留到下一个 run（`test/agent.test.ts` 的 "keeps queues when finishTurn ends the run"）。所以队列只能放在有状态的那一层，循环通过 `getSteeringMessages` / `getFollowUpMessages` 两个函数来问。换一个宿主，可以用完全不同的方式实现这两个函数——比如从数据库里读——循环一行都不用改。
