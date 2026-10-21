# 第 8 章 · 测试：用假模型驱动循环

上一章讲失败，这一章讲另一件横跨全包的事：**怎么在不调真模型的情况下验证这个循环**。

这件事能做成，靠的是第 1 章那条拒绝：循环不认识厂商，它对模型的全部依赖是一个函数形状 `StreamFn`。函数可以随手写一个。仓库里的测试正是这么做的，按"假得有多像"分成三层：

1. **手写流**：在测试里直接造一个事件流，几行代码，完全可控。
2. **假 provider**：pi-ai 提供的 faux provider，按脚本回放消息，像真模型一样逐块流出来。
3. **假网络**：给 `streamProxy` 换掉全局 `fetch`，测它对服务器各种行为的反应。

## 第一层：手写一个流

最小的 `streamFn` 只需要三行逻辑：造一个流，下一个微任务里推一个 `done` 事件，返回流。

```typescript
// test/agent-loop.test.ts:80（删减：断言部分）
it("uses the configured default when a legacy caller omits streamFn", async () => {
	let calls = 0;
	setDefaultStreamFn(() => {
		calls++;
		const stream = createAssistantMessageEventStream();
		queueMicrotask(() => {
			stream.push({
				type: "done",
				reason: "stop",
				message: createAssistantMessage([{ type: "text", text: "fallback" }]),
			});
		});
		return stream;
	});
	// ...
```

`queueMicrotask` 不是装饰。`StreamFn` 同步返回流，循环拿到之后才开始 `for await`；如果在返回之前就把事件推进去，事件会先进队列、等迭代器来取，结果一样能跑通——但那就测不到"流是异步到达的"这件事了。推迟一个微任务，让测试里的时序和真实 provider 一致：先返回，后到达。

`createAssistantMessageEventStream` 来自 pi-ai，它造的就是真 provider 用的那个 `AssistantMessageEventStream`。测试和生产共用同一个流实现，测试里的假只假在"内容是写死的"，不假在"流的行为"上。

这一层的价值在于**能精确构造任何情形**。前几章引过的很多测试都是这个写法：

- 第 3 章的工具切换测试，`streamFn` 不回答问题，只把每次请求里的系统消息记下来，用来断言模型看到了什么（`test/agent.test.ts:163`）。
- 第 4 章的 `beforeToolCall` 改参数测试，第一次调用返回一个工具调用，第二次返回文字，用一个计数器切换（`test/agent-loop.test.ts:528`）。
- 第 5 章的失败测试更极端：`streamFn` 直接 `throw`，故意违反契约，看 `Agent` 能不能兜住（`test/agent.test.ts:330`）。

最后一个例子说明了这一层的另一个用途：**测试契约被违反时的行为**。真 provider 守约，用它测不出防御代码有没有用；手写流想怎么违约就怎么违约。

## 第二层：假 provider

手写流有一个盲区：它通常一步推出最终消息，没有中间的 `text_delta`、`toolcall_delta`。而第 2 章「回流」一节讲的那些就地更新、`message_update` 事件、`streamingMessage` 状态，都发生在增量事件上。要测它们，需要一个会"逐块说话"的模型。

pi-ai 的 faux provider 就是这个。它的测试用法：

```typescript
// test/e2e.test.ts:186（Agent integration with faux provider 内，前两个用例）
describe("Agent integration with faux provider", () => {
	it("handles a basic text prompt", async () => {
		const faux = createFauxRegistration();
		faux.setResponses([fauxAssistantMessage("4")]);
		await basicPrompt(faux.getModel());
	});

	it("executes tools and tracks pending tool calls", async () => {
		const faux = createFauxRegistration();
		faux.setResponses([
			fauxAssistantMessage(
				[
					fauxText("Let me calculate that."),
					fauxToolCall("calculate", { expression: "123 * 456" }, { id: "calc-1" }),
				],
				{ stopReason: "toolUse" },
			),
			fauxAssistantMessage("The result is 56088."),
		]);
		await toolExecution(faux.getModel());
	});
```

用法是"写剧本"：`setResponses` 放进一个消息队列，每次请求按顺序取一条。第二个用例的剧本是"先说一句话并调计算器，再报结果"，正好是一次完整的模型 → 工具 → 模型往复。`fauxText`、`fauxToolCall` 只是构造内容块的小函数：

```typescript
// pi-ai/src/providers/faux.ts:54
export function fauxText(text: string): TextContent {
	return { type: "text", text };
}

export function fauxThinking(thinking: string): ThinkingContent {
	return { type: "thinking", thinking };
}

export function fauxToolCall(name: string, arguments_: ToolCall["arguments"], options: { id?: string } = {}): ToolCall {
	return {
		type: "toolCall",
		id: options.id ?? randomId("tool"),
		name,
		arguments: arguments_,
	};
}
```

### 它守契约

faux provider 自己也是一个 `streamFn` 实现，所以它同样要守第 7 章那条契约。剧本用完了怎么办？不是抛异常，而是流出一条错误消息：

```typescript
// pi-ai/src/providers/faux.ts:522（stream 内，删减：deferred 分支）
const stream: StreamFunction<string, SimpleStreamOptions> = (requestModel, context, streamOptions) => {
	const outer = createAssistantMessageEventStream();
	const step = pendingResponses.shift();
	state.callCount++;

	queueMicrotask(async () => {
		try {
			await streamOptions?.onResponse?.({ status: 200, headers: {} }, requestModel);
			if (!step) {
				let message = createErrorMessage(
					new Error("No more faux responses queued"),
					api,
					provider,
					requestModel.id,
				);
				message = withUsageEstimate(message, context, streamOptions, promptCache);
				outer.push({ type: "error", reason: "error", error: message });
				outer.end(message);
				return;
			}
			// ...

			const message = await resolveResponse(step, context, streamOptions, requestModel);
			await streamWithDeltas(outer, message, minTokenSize, maxTokenSize, tokensPerSecond, streamOptions?.signal);
		} catch (error) {
			const message = createErrorMessage(error, api, provider, requestModel.id);
			outer.push({ type: "error", reason: "error", error: message });
			outer.end(message);
		}
	});

	return outer;
};
```

结构和第 6 章的 `streamProxy` 几乎一样：同步返回流，工作放进异步块，所有失败汇到一个 `catch` 变成 `error` 事件。这不是巧合——两者是同一份契约的两个实现。一个测试多发了一次请求，看到的是一条 `"No more faux responses queued"` 的错误消息进了记录，而不是测试进程崩掉或挂住。

### 逐块与时间

`streamWithDeltas` 把剧本里的整条消息拆成小块，依次推出 `*_start`、`*_delta`、`*_end`。每块之间的等待由一个函数决定：

```typescript
// pi-ai/src/providers/faux.ts:348
function scheduleChunk(chunk: string, tokensPerSecond: number | undefined): Promise<void> {
	if (!tokensPerSecond || tokensPerSecond <= 0) {
		return new Promise((resolve) => queueMicrotask(resolve));
	}
	const delayMs = (estimateTokens(chunk) / tokensPerSecond) * 1000;
	return new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

默认只等一个微任务：测试跑得飞快，但事件依然是一个一个异步到达的。给了 `tokensPerSecond`，就换成真实的定时器，模拟一个慢模型。这个开关专门为测中止准备：

```typescript
// test/e2e.test.ts:208（Agent integration with faux provider 内）
it("handles abort during streaming", async () => {
	const faux = createFauxRegistration({
		tokensPerSecond: 20,
		tokenSize: { min: 2, max: 2 },
	});
	faux.setResponses([
		fauxAssistantMessage(
			"one two three four five six seven eight nine ten eleven twelve thirteen fourteen fifteen",
		),
	]);
	await abortExecution(faux.getModel());
});
```

每秒 20 个 token、每块 2 个 token，一条十五个词的回答要流一秒多。`abortExecution` 在 30 毫秒后调 `agent.abort()`，这时消息一定还在流。faux provider 在每块之后检查 signal（第 2 章「插叙：executor 与 signal」讲的那根电线一直通到这里），按下了就流出一条 `stopReason: "aborted"` 的消息。测试断言的正是这个：

```typescript
// test/e2e.test.ts:124（abortExecution 内，删减：前面的构造与等待）
const lastMessage = agent.state.messages[agent.state.messages.length - 1];
if (lastMessage.role !== "assistant") throw new Error("Expected assistant message");
expect(lastMessage.stopReason).toBe("aborted");
expect(lastMessage.errorMessage).toBeDefined();
expect(agent.state.errorMessage).toBe(lastMessage.errorMessage);
```

没有假 provider，这个测试要么依赖真模型（慢、花钱、不稳定），要么用手写流自己实现一遍"逐块推送 + 检查 signal"。faux provider 把这件事做成了一个参数。

### 注册方式

e2e 测试走的是 pi-ai 的兼容入口：`registerFauxProvider` 把假 provider 注册进全局的 API 注册表，`streamSimple` 按模型的 `api` 字段找到它。

```typescript
// test/e2e.test.ts:20
function createFauxRegistration(options: Parameters<typeof registerFauxProvider>[0] = {}): FauxProviderRegistration {
	const registration = registerFauxProvider(options);
	registrations.push(registration);
	return registration;
}
```

全局注册意味着要清理：同一个文件里的 `afterEach` 逐个 `unregister()`，否则上一个用例的剧本会漏到下一个用例里。pi-ai 也提供了不走全局的版本 `fauxProvider()`，配合显式的 `Models` 集合使用（`pi-ai/src/providers/faux.ts:695` 的注释给了用法）。两者共用同一个 `createFauxCore`，区别只在挂到哪里。

注意 `Agent` 在这里拿到的 `streamFn` 就是 `streamSimple`——生产代码用的同一个函数。从 `Agent` 往下，经 `runAgentLoop`、`streamAssistantResponse`、`normalizeContext`、pi-ai 的分发，一直到 provider 的 `stream`，整条链路都是真的；只有最末端那个"说什么"是剧本。所以文件叫 `e2e.test.ts`。

## 第三层：假网络

`streamProxy` 的下游不是一个函数，而是 `fetch`。测它就换掉 `fetch`：

```typescript
// test/proxy.test.ts:99
it("emits an error instead of hanging when the stream ends without a terminal event", async () => {
	const body = `data: ${JSON.stringify({ type: "start" })}\n\n`;
	vi.stubGlobal(
		"fetch",
		vi.fn(async () => new Response(body, { status: 200 })),
	);

	const stream = streamProxy(model, normalizeContext({ systemPrompt: "", messages: [] }), {
		authToken: "test-token",
		proxyUrl: "https://proxy.example.com",
	});
	const events: AssistantMessageEvent[] = [];
	for await (const event of stream) events.push(event);
	const result = await stream.result();

	expect(events.map((event) => event.type)).toEqual(["start", "error"]);
	expect(result.stopReason).toBe("error");
	expect(result.errorMessage).toContain("Connection closed by proxy server");
});
```

响应体只有一个 `start` 事件，没有 `done`——这正是第 6 章讲的"服务器中途崩溃"。测试本身也说明了 bug 的形态：修复之前，`for await` 那一行会永远等下去，测试超时失败；修复之后，流以一个补发的 `error` 收尾。

`Response` 是标准 Web API，`new Response(body)` 造出一个真正的可读流响应，`streamProxy` 里的 `getReader()`、`TextDecoder` 全都照常工作。所以这一层的假也只假在"服务器发了什么"，不假在"怎么读"。

同一个文件里的另外两个用例，各对应一次修过的 bug：

- `"processes terminal metadata when the event is not newline-terminated"`（`test/proxy.test.ts:78`）——最后一行没有换行符的 `done` 事件曾被丢掉。
- `"preserves tool-call metadata received only on toolcall_end"`（`test/proxy.test.ts:33`）——第 6 章那个 `namespace` 丢失的问题。

第二个用例前面有一行注释把它们和 issue 绑在一起：

```typescript
// test/proxy.test.ts:77
// Regression tests for https://github.com/earendil-works/pi/issues/8996
```

每修一个 bug 就加一个能复现它的用例，这是测试文件最常见的增长方式。翻这类用例的名字，往往比翻 CHANGELOG 更快找到一段代码为什么长成这样。

## 测"等待"本身

前面的测试大多断言"最后得到了什么"。第 2 章讲过一类更难测的性质：**谁在等谁**——监听器没跑完，`prompt()` 不能 resolve；`agent_end` 发出了，`waitForIdle()` 还不能 resolve。这类测试的写法是放一道闸门：

```typescript
// test/agent.test.ts:410（删减：streamFn 与闸门的构造）
agent.subscribe(async (event) => {
	if (event.type === "message_end" && event.message.role === "assistant") {
		await barrier.promise;
	}
});

const promptPromise = agent.prompt("hello");
let idleResolved = false;
const idlePromise = agent.waitForIdle().then(() => {
	idleResolved = true;
});

await new Promise((resolve) => setTimeout(resolve, 10));
expect(idleResolved).toBe(false);
expect(agent.state.isStreaming).toBe(true);

barrier.resolve();
await Promise.all([promptPromise, idlePromise]);

expect(idleResolved).toBe(true);
```

`barrier` 是一个手动 resolve 的 Promise——第 2 章「结算与 waitForIdle」讲的同一个模式。监听器卡在闸门上，测试等 10 毫秒确认 `waitForIdle()` 确实没 resolve、`isStreaming` 确实还是真；然后开闸，两个 Promise 一起落地。断言的是"在闸门开之前，什么都不该发生"。

这里的 10 毫秒不是证明，只是采样：它能抓住"根本没等"的 bug，抓不住"等了但等得不够久"的 bug。对异步代码，这是测试能做到的常见上限。

## 一章小结

- 循环可测，是因为它对模型的依赖只是一个 `StreamFn` 形状。仓库里的测试按"假得多像"分三层。
- 手写流：`createAssistantMessageEventStream` 加一个 `queueMicrotask`，能精确构造任何情形，包括故意违反契约。
- faux provider：按剧本回放消息，逐块流出增量事件；默认每块等一个微任务，给 `tokensPerSecond` 就用真定时器，专门用来测中止。它自己也守 `StreamFn` 契约，剧本用完时流出错误消息而不是抛。
- 假网络：`vi.stubGlobal("fetch", ...)` 配合标准 `Response`，测 `streamProxy` 对服务器各种行为的反应。每个用例往往对应一次修过的 bug。
- 测"谁在等谁"用闸门：监听器卡在一个手动 resolve 的 Promise 上，先断言什么都没发生，再开闸断言一切落地。
- 三层的共同点：假只假在"内容"，流的实现、读取的方式、循环本身都是真的。

到这里，本书对 `pi-agent-core` 的导读告一段落：第一部分画出了整体，第二部分拆开了六个文件，第三部分讲了两条横跨全包的规则——失败变成值，以及怎么验证它。

## 为什么不去

> **为什么测试里不 mock `runAgentLoop` 或 `Agent` 的内部方法，而是从 `streamFn` 这一层注入？** 因为 `streamFn` 是这个包唯一的外部依赖。在它这里注入，被测的就是包里的全部真实代码：循环、管线、状态归约、事件分发。往里面 mock 任何一层，那一层的 bug 就测不到了。这也是第 1 章"拒绝认识厂商"的回报之一：如果循环里直接 import 了某家厂商的 SDK，测试就得 mock 那个 SDK——那是别人的代码、别人的接口，升级一次就可能全部重写。依赖一个自己定义的函数形状，测试只需要满足这个形状。

> **为什么 faux provider 放在 pi-ai 里，而不是本包的测试目录？** 因为它的用户不止本包。任何依赖 pi-ai 的包——coding-agent、pi-durable、第三方应用——要测"模型说了某句话之后会怎样"，都需要同一个东西。放在 pi-ai 的公开导出里（`pi-ai/src/index.ts` 导出了 `./providers/faux.ts`），它就和真 provider 站在同一个注册表、同一套契约下，谁都能用，行为也和真 provider 保持一致。
