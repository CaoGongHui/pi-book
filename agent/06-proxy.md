# 第 6 章 · 代理流：把模型调用搬到服务器后面

前五章里的 `streamFn` 都是一个黑盒：给它模型和对话记录，它还你一个事件流。这一章打开本包里唯一一个 `streamFn` 实现——`src/proxy.ts`。它解决一个具体的问题：**浏览器里的应用不能持有厂商的 API key**。key 放在前端代码里，等于公开给每一个打开页面的人。

解法是在中间放一台服务器：浏览器把请求发给它，它带上 key 转发给厂商，再把事件流传回来。`streamProxy` 就是浏览器这一侧的那一半：

```typescript
// src/proxy.ts:1
/**
 * Proxy stream function for apps that route LLM calls through a server.
 * The server manages auth and proxies requests to LLM providers.
 */
```

读这个文件，要回答三个问题：

1. 什么能过网线，什么不能？
2. 服务器为了省带宽丢掉的东西，客户端怎么补回来？
3. 网络会断、服务器会挂，怎么保证 `StreamFn` 的"不许抛"契约？

## 什么能过网线

请求是一次普通的 `fetch`：

```typescript
// src/proxy.ts:147（streamProxy 内）
const response = await fetch(`${options.proxyUrl}/api/stream`, {
	method: "POST",
	headers: {
		Authorization: `Bearer ${options.authToken}`,
		"Content-Type": "application/json",
	},
	body: JSON.stringify({
		model,
		context,
		options: buildProxyRequestOptions(options),
	}),
	signal: options.signal,
});
```

三样东西进了请求体：模型描述、对话记录、一组选项。认证用的是 `authToken`——你自己的用户令牌，不是厂商的 key。服务器拿它确认"这是我的用户"，再用自己保管的厂商 key 去调模型。

`context` 是 `TranscriptContext`，第 3 章讲过它只有一个 `messages` 字段，系统提示词和工具声明都在消息里。所以一次 `JSON.stringify` 就把整个对话的状态发了过去，服务器不需要知道 agent 的任何别的东西。

选项要挑。`SimpleStreamOptions` 里有些东西过不了网线：`signal` 是一个本地对象，`onPayload` / `onResponse` 是回调函数，`apiKey` 根本不该从浏览器发出去。所以先用类型定一份白名单：

```typescript
// src/proxy.ts:48
type ProxySerializableStreamOptions = Pick<
	SimpleStreamOptions,
	| "temperature"
	| "samplingParams"
	| "maxTokens"
	| "reasoning"
	| "cacheRetention"
	| "sessionId"
	| "headers"
	| "metadata"
	| "transport"
	| "thinkingBudgets"
	| "maxRetryDelayMs"
>;
```

再用一个函数逐字段抄过去：

```typescript
// src/proxy.ts:91（删减：中间七个字段）
function buildProxyRequestOptions(options: ProxyStreamOptions): ProxySerializableStreamOptions {
	return {
		temperature: options.temperature,
		samplingParams: options.samplingParams,
		// ...
		maxRetryDelayMs: options.maxRetryDelayMs,
	};
}
```

为什么不直接 `{ ...options }` 再删掉几个键？因为黑名单会漏。`SimpleStreamOptions` 来自 pi-ai，它加一个新字段，展开运算符就会把它带上网线——哪怕那是个函数或者一个不该外传的值。白名单反过来：新字段默认不过，有人想让它过，得在两个地方明确写上。这份白名单是被 bug 逼出来的，CHANGELOG 记了它的来历：

```markdown
// CHANGELOG.md:400
- Fixed `streamProxy()` to preserve the proxy-safe serializable subset of stream options, including session, transport, retry-delay, metadata, header, cache-retention, and thinking-budget settings ([#3512](https://github.com/badlogic/pi-mono/issues/3512))
```

在那之前，代理只转发了少数几个选项，`sessionId` 之类的设置在浏览器里设了也白设。修法是把"能安全序列化的子集"一次列全。

## 服务器丢掉的 partial

第 2 章讲过，pi-ai 的每个流事件都带一个 `partial`：到目前为止的整条消息。这对本地调用很方便，对网络却很浪费——模型每吐一个 token，就要把整条越来越长的消息重新发一遍。所以代理协议把它去掉了：

```typescript
// src/proxy.ts:20（删减：done / error 两个成员的字段）
/**
 * Proxy event types - server sends these with partial field stripped to reduce bandwidth.
 */
export type ProxyAssistantMessageEvent =
	| { type: "start" }
	| { type: "text_start"; contentIndex: number }
	| { type: "text_delta"; contentIndex: number; delta: string }
	| { type: "text_end"; contentIndex: number; contentSignature?: string }
	| { type: "thinking_start"; contentIndex: number }
	| { type: "thinking_delta"; contentIndex: number; delta: string }
	| { type: "thinking_end"; contentIndex: number; contentSignature?: string }
	| { type: "toolcall_start"; contentIndex: number; id: string; toolName: string }
	| { type: "toolcall_delta"; contentIndex: number; delta: string }
	| { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall }
	| {
			type: "done";
			// ...
	  }
	| {
			type: "error";
			// ...
	  };
```

对照第 2 章引的 `AssistantMessageEvent`：成员一一对应，只是每个都少了 `partial`，增量事件只带 `delta`。这是一个**增量协议**：服务器只发变化，状态由接收方累积。

### 客户端重建

`streamProxy` 在开始读流之前，先造一个空的 assistant 消息当作累积器（`src/proxy.ts:116`，`stopReason: "pending"`、`content: []`），然后每收到一个代理事件，就交给 `processProxyEvent` 改写累积器、再还原成一个带 `partial` 的标准事件：

```typescript
// src/proxy.ts:257（processProxyEvent，删减：thinking 三件与 toolcall 三件、error 分支）
function processProxyEvent(
	proxyEvent: ProxyAssistantMessageEvent,
	partial: AssistantMessage,
): AssistantMessageEvent | undefined {
	switch (proxyEvent.type) {
		case "start":
			return { type: "start", partial };

		case "text_start":
			partial.content[proxyEvent.contentIndex] = { type: "text", text: "" };
			return { type: "text_start", contentIndex: proxyEvent.contentIndex, partial };

		case "text_delta": {
			const content = partial.content[proxyEvent.contentIndex];
			if (content?.type === "text") {
				content.text += proxyEvent.delta;
				return {
					type: "text_delta",
					contentIndex: proxyEvent.contentIndex,
					delta: proxyEvent.delta,
					partial,
				};
			}
			throw new Error("Received text_delta for non-text content");
		}
		// ...

		case "done":
			partial.stopReason = proxyEvent.reason;
			partial.usage = proxyEvent.usage;
			if (proxyEvent.providerThinkingLevel !== undefined) {
				partial.providerThinkingLevel = proxyEvent.providerThinkingLevel;
			}
			return { type: "done", reason: proxyEvent.reason, message: partial };
		// ...
	}
}
```

规则很机械：`*_start` 在 `contentIndex` 处放一个空块，`*_delta` 往那个块上追加，`*_end` 补上签名之类的收尾字段。每次返回的 `partial` 都是**同一个对象**——这正好符合 pi-ai 对 `partial` 的定义（第 2 章引过："the shared live response-so-far helper, not an event-time snapshot"）。循环拿到事件时会浅拷贝一份再发给订阅者，所以这里放心地原地改。

错配的事件（往一个 thinking 块上追加 text）直接抛错。这个错不会逃出 `streamProxy`，下一节会看到它被谁接住。

### 工具调用参数：边流边解析

工具调用的参数是一段 JSON，模型一个字符一个字符地吐。UI 想在参数还没吐完时就显示"正在读 src/ma…"，就得解析一段**不完整**的 JSON：

```typescript
// src/proxy.ts:329（processProxyEvent 内，toolcall 三件）
case "toolcall_start":
	partial.content[proxyEvent.contentIndex] = {
		type: "toolCall",
		id: proxyEvent.id,
		name: proxyEvent.toolName,
		arguments: {},
		partialJson: "",
	} satisfies ToolCall & { partialJson: string } as ToolCall;
	return { type: "toolcall_start", contentIndex: proxyEvent.contentIndex, partial };

case "toolcall_delta": {
	const content = partial.content[proxyEvent.contentIndex];
	if (content?.type === "toolCall") {
		(content as any).partialJson += proxyEvent.delta;
		content.arguments = parseStreamingJson((content as any).partialJson) || {};
		partial.content[proxyEvent.contentIndex] = { ...content }; // Trigger reactivity
		return {
			type: "toolcall_delta",
			contentIndex: proxyEvent.contentIndex,
			delta: proxyEvent.delta,
			partial,
		};
	}
	throw new Error("Received toolcall_delta for non-toolCall content");
}

case "toolcall_end": {
	const content = partial.content[proxyEvent.contentIndex];
	if (content?.type === "toolCall") {
		Object.assign(content, proxyEvent.toolCall);
		delete (content as any).partialJson;
		return {
			type: "toolcall_end",
			contentIndex: proxyEvent.contentIndex,
			toolCall: content,
			partial,
		};
	}
	return undefined;
}
```

三步：

- **start**：块上挂一个类型之外的私有字段 `partialJson`，用来攒原始文本。`satisfies ... as ToolCall` 这一串是在说"我知道多了一个字段，先检查它确实满足 ToolCall 再假装没有"。
- **delta**：文本追加到 `partialJson`，再用 pi-ai 的 `parseStreamingJson` 整段重新解析一次。这个函数专门处理半截 JSON：先尝试修补，再退到部分解析，都不行就返回空对象。所以 `arguments` 在流的过程中一直是"目前能看懂的那部分"。解析完还把整个块换成一个新对象，注释写了原因：触发 UI 框架的响应式更新，和第 5 章 `pendingToolCalls` 每次换新 Set 是同一个考虑。
- **end**：服务器在结束事件里给出**权威的**完整工具调用，`Object.assign` 盖到块上，删掉私有字段。之前边流边解析出来的 `arguments` 只是预览，最终以服务器给的为准。

`toolcall_end` 里的 `toolCall` 字段是后加的。早期协议的结束事件只带 `contentIndex`，客户端唯一的信息来源就是自己边流边解析出来的参数；provider 挂在工具调用上的其他字段到不了浏览器。OpenAI Responses 给工具调用加了 `namespace` 之后，这个缺口就成了 bug：namespace 在代理这一跳丢了（CHANGELOG 0.84.2："Fixed `streamProxy()` dropping finalized tool-call metadata such as OpenAI Responses namespaces"）。修法是 commit `02bd2d1c6`：协议里给结束事件加上完整的 `toolCall`，客户端整体 `Object.assign` 过来。用整体覆盖而不是只抄 `namespace`，以后 provider 再加字段也能自动过来。


## 不许抛：网络上的契约

第 1 章引过 `StreamFn` 的契约：请求、模型、运行时的失败**都不许抛**，要编码进返回的流里，以一条 `stopReason` 为 `error` 或 `aborted` 的消息收尾。对直连厂商的 `streamFn`，这由 pi-ai 的 provider 负责；`streamProxy` 自己就是一个 `streamFn`，这份责任落在它头上。网络上能出错的地方很多：服务器返回 500、连接中途断开、用户按了停止、某一行事件解析失败。它的做法是把所有这些都收进一个 `try`。

先看整体形状。函数同步返回一个流，真正的工作放进一个立刻执行、不被 await 的异步函数：

```typescript
// src/proxy.ts:107（streamProxy，删减：partial 初始化、读流主体与 catch 体）
export function streamProxy(
	model: Model<any>,
	context: TranscriptContext,
	options: ProxyStreamOptions,
): AssistantMessageEventStream {
	const stream = createAssistantMessageEventStream();

	(async () => {
		// ...
		try {
			// ...（fetch、逐行读、处理事件）
			stream.end();
		} catch (error) {
			// ...（把异常变成一个 error 事件）
		} finally {
			if (options.signal) {
				options.signal.removeEventListener("abort", abortHandler);
			}
		}
	})();

	return stream;
}
```

调用方拿到流的那一刻，请求可能还没发出去。之后发生的一切，只能通过流里的事件告诉调用方。

### 三种出错的方式

**服务器拒绝。** 非 2xx 的响应先尝试读服务器给的错误信息，读不到就用状态码，然后**主动抛出**：

```typescript
// src/proxy.ts:161（streamProxy 内）
if (!response.ok) {
	let errorMessage = `Proxy error: ${response.status} ${response.statusText}`;
	try {
		const errorData = (await response.json()) as { error?: string };
		if (errorData.error) {
			errorMessage = `Proxy error: ${errorData.error}`;
		}
	} catch {
		// Couldn't parse error response
	}
	throw new Error(errorMessage);
}
```

在一个"不许抛"的函数里主动 `throw`，看起来矛盾。其实这是把各种失败汇到同一个出口：抛出去的异常一定落进外层的 `catch`，在那里统一变成事件。`processProxyEvent` 遇到错配事件时抛的错，走的也是这条路。

**用户中止。** 中止有两条线。`fetch` 本身接收 `signal`，请求还没回来时按停止，`fetch` 会 reject。响应已经开始流了，就靠一个监听器取消读取器：

```typescript
// src/proxy.ts:136（streamProxy 内）
const abortHandler = () => {
	if (reader) {
		reader.cancel("Request aborted by user").catch(() => {});
	}
};

if (options.signal) {
	options.signal.addEventListener("abort", abortHandler);
}
```

读循环里也在每块数据之后检查一次 `signal.aborted`，按下了就抛 `"Request aborted by user"`。几条线最终都汇到 `catch`。`finally` 里把监听器摘掉，避免同一个 signal 被多次请求复用时越挂越多。

**连接中途断开。** 这一种最隐蔽，因为它**不报错**。服务器进程崩了，TCP 连接正常关闭，`reader.read()` 返回 `done: true`，读循环正常退出——但最后一个事件 `done` 或 `error` 根本没来。旧代码这时直接 `stream.end()`，流结束了，却没有最终消息。commit `ebc374490` 的说明把后果写得很清楚：

```text
// commit ebc374490（2026-09-02），提交说明节选
streamProxy dropped a final SSE event that was not newline-terminated
(the residual line buffer was never flushed after the read loop) and
called stream.end() without a result when no done/error event had been
seen, leaving EventStream.result() pending forever. Consumers like
agentLoop awaiting response.result() hung indefinitely when the server
closed the connection mid-response.
```

`result()` 永远不 resolve，循环永远卡在 `streamAssistantResponse` 里（第 2 章：收到 `done` / `error` 时它才 `await result()`；流结束而没有终止事件，它走到函数末尾的兜底分支，同样 `await result()`）。run 不结束，`waitForIdle()` 也不结束——一次服务器崩溃，变成了整个 agent 的永久挂起。

修法分两步。先把缓冲区里最后一行没有换行符的数据处理掉：

```typescript
// src/proxy.ts:212（streamProxy 内）
// The final event may not be newline-terminated; flush the decoder and
// process whatever is left in the buffer.
buffer += decoder.decode();
if (buffer) {
	processLine(buffer);
}
```

再检查有没有见过终止事件，没有就自己造一个：

```typescript
// src/proxy.ts:219（streamProxy 内）
if (!sawTerminalEvent) {
	// A clean EOF without a done/error event means the server dropped the
	// response mid-stream. Surface it as an error instead of leaving
	// consumers waiting on a result that never arrives.
	partial.stopReason = "error";
	partial.errorMessage = "Connection closed by proxy server before the response completed";
	stream.push({
		type: "error",
		reason: "error",
		error: partial,
	});
}
```

`sawTerminalEvent` 在 `processLine` 里设置：每处理一个 `done` 或 `error` 事件就置真（`src/proxy.ts:186`）。

### 统一出口

所有抛出的异常最后都到这里：

```typescript
// src/proxy.ts:233（streamProxy 内）
} catch (error) {
	const errorMessage = error instanceof Error ? error.message : String(error);
	const reason = options.signal?.aborted ? "aborted" : "error";
	partial.stopReason = reason;
	partial.errorMessage = errorMessage;
	stream.push({
		type: "error",
		reason,
		error: partial,
	});
	stream.end();
```

`reason` 不看异常是什么，只看 signal 有没有被按下。这是一个务实的判断：中止引起的异常形状各异（`fetch` 的 `AbortError`、读取器被取消、自己抛的那句话），统一按 signal 判，就不用逐个识别。

注意 `error` 事件带的是 `partial`——已经累积的那部分内容。模型说到一半连接断了，用户依然能看到那半句话，消息带着 `stopReason: "error"` 进入记录。第 2 章讲过，循环看到 `error` 就走硬出口：发 `turn_end`、`agent_end`，run 结束。

## 服务器那一半

本包只有客户端。服务器要做的事可以从协议里读出来：

- 在 `POST {proxyUrl}/api/stream` 上接收 `{ model, context, options }`，校验 `Authorization` 里的用户令牌；
- 用自己保管的厂商 key 调 pi-ai 的 `streamSimple(model, context, options)`；
- 把每个 `AssistantMessageEvent` 去掉 `partial`，以 `data: <json>\n` 的格式写回（`processLine` 只认 `data: ` 开头的行，`src/proxy.ts:180`）；
- 结束时写一个 `done` 或 `error` 事件，带上 `usage`。

这台服务器是一个对公网开放的模型调用入口，等于替用户花你的钱。用户认证、限流、模型白名单（不能让前端随便指定一个昂贵的模型）、对 `context` 大小的上限，都得服务器自己做。`streamProxy` 只管把令牌带上，不替你做任何访问控制。

## 一章小结

- `streamProxy` 是本包唯一的 `streamFn` 实现：浏览器把请求发给你自己的服务器，服务器持有厂商 key 转发。它和直连的 `streamFn` 在循环眼里没有区别。
- 请求体是 `{ model, context, options }`。选项走白名单（`ProxySerializableStreamOptions`），函数、signal、key 都过不了网线。
- 代理协议是增量的：服务器去掉每个事件的 `partial`，客户端用一个累积器重建。工具参数边流边用 `parseStreamingJson` 解析，结束时以服务器给的完整 `toolCall` 为准。
- 所有失败——服务器拒绝、用户中止、事件错配——都汇到一个 `catch`，变成一个带已累积内容的 `error` 事件。
- 连接正常关闭但没有终止事件，会被识别出来并补发一个错误；在修复之前，这种情况会让整个 agent 永久挂起。

到这里，`pi-agent-core` 的六个文件都看完了。接下来两章不再按文件走，而是讲两件横跨全包的事。

## 为什么不去

> **为什么服务器要去掉 `partial`，而不是原样转发 pi-ai 的事件？** `partial` 是到当前为止的整条消息。一条一千 token 的回答大约产生一千个增量事件，原样转发就是把一条越来越长的消息发一千遍，传输量和回答长度的平方成正比。去掉之后，每个事件只带自己那一小段 `delta`，总量和回答长度成正比。代价是客户端要自己重建，而重建规则是机械的（start 放空块、delta 追加、end 收尾），写一次就够。类型注释把这个取舍写在协议定义的第一行："server sends these with partial field stripped to reduce bandwidth"（`src/proxy.ts:21`）。

> **为什么 EOF 时补发一个错误，而不是安静地结束流？** 因为"安静地结束"违反了 `StreamFn` 的契约：契约要求流以一条最终消息收尾，`result()` 必须能 resolve。流不报错地结束，下游没有任何办法知道出了问题，只能一直等。补发的错误把一个"不可见的挂起"变成"可见的失败"：消息记录里多一条带 `errorMessage` 的 assistant 消息，UI 能显示，用户能重试（`agent.continue()`，第 5 章）。对一个跑在不可靠网络上的 `streamFn` 来说，"永远有一个终止事件"比"永远不报错"更重要。
