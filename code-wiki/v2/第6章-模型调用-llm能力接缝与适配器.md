# 第6章 模型调用：llm 能力接缝与适配器

## 1. 本章导读

上一章（[第5章](./第5章-Agent-Loop-让模型转动起来的引擎.md)）我们看完了 agent-loop（代理循环）：它组装请求、消费流式响应、派发工具。但有一个问题被刻意搁置了——循环里那句 `ctx.llm.stream(request)` 背后到底发生了什么？请求是怎么变成 HTTP 报文、又怎么把 SSE（Server-Sent Events）字节流变回结构化分片的？本章就拆开这个黑盒。

DSH 把"调用模型"做成了一条**能力接缝（capability seam）**：`packages/llm/` 组里的 `llm` 包声明抽象服务 `ctx.llm`，各厂商适配器（adapter）插件注册进来，agent-loop、压缩摘要器、标题生成器作为消费方调用它。学完本章，你应该能回答四个问题：

1. `ctx.llm` 到底声明了什么？一个适配器要遵守哪些契约？
2. 统一的消息与流词汇——`GenerateOptions`、`ContentBlock`、`StreamChunk`、`TokenUsage`——长什么样，为什么这么设计？
3. 能力接缝的三角色（Service Definition / Provider / Consumer）在真实文件上怎么落位？
4. 重试（`llm-retry`）与计量（`token-meter`）为什么独立成包，而不塞进 `llm` 本体？

本章的地图：第 2 节从一个"接入新厂商"的问题出发；第 3 节是正文，先解剖 `LlmRuntime`，再讲统一词汇、运行时投影、DeepSeek 适配器内幕、pi-ai 双生适配器、`prepareCall` 的"代"绑定，最后解释重试与计量的分包；第 4 节逐行走读两段核心源码；第 5 节列五个常见误区；第 6-8 节是动手环节、小结与延伸阅读。

阅读本章前，你应当读过第 3 章（Cordis 服务注入）与第 4 章（waterfall 瀑布事件），因为 `ctx.llm` 是服务、`llm/stream` 是 waterfall。

## 2. 从一个问题出发

假设明天有一家新模型厂商 "Acme" 上线，它的 HTTP API 兼容 OpenAI 的 completions 协议。你要让 DSH 支持它，需要改几处代码？

传统框架的答案你可能很熟悉：改内核。在请求构造函数里加一个 `if (provider === 'acme')`，在响应解析器里加一个分支，在重试逻辑里加一份它的错误码表……厂商越多，内核越像一团意大利面，而且每加一家都要重新发版。

DSH 的答案分两档：

- **零代码**：如果 Acme 兼容 `openai-completions`、`openai-responses`、`anthropic-messages` 三种 wire 协议之一，你只需要在 `llm-pi-ai` 的 `providers` 配置字典里加一段 YAML——这正是 [packages/llm/llm-pi-ai/src/index.ts](../../packages/llm/llm-pi-ai/src/index.ts) 文件头 JSDoc 里的第三个示例（L29-L46）：声明 `api: openai-completions`、`baseURL`、`apiKeyEnv` 和模型列表，路由即刻生效。
- **写一个适配器插件**：如果 Acme 的协议是自创的，你继承 `LlmAdapter`、实现唯一的抽象方法 `stream()`，再在插件 `apply()` 里调 `ctx.llm.registerAdapter(['acme'], adapter)`。内核一行不改。

这就是适配器模式的全部要点。你可以把 `LlmAdapter` 想象成电源适配器（adapter）：墙上插座是各厂商千奇百怪的 wire 协议，你的笔记本要的是统一的直流电——`LlmAdapter` 负责把插头形状翻译过来，插座本身（`ctx.llm`）从不为任何一家厂商改变形状。（这是本章三个比喻中的第一个，后两个会提前打招呼。）

## 3. 正文

### 3.1 能力接缝的解剖：`LlmRuntime` 声明了什么

整条接缝的服务定义（Service Definition）是 [packages/llm/llm/src/index.ts](../../packages/llm/llm/src/index.ts) 里的 `LlmRuntime`（约 L342，继承 `TypertRemoteService`）。它通过声明合并（declaration merging，见第 3 章）把 `llm` 挂到 `Context` 上，并声明了两个事件（第 4 章的事件域里它们属于 capability 域）：

```ts
// packages/llm/llm/src/index.ts（节选，L57-L78）
declare module '@deepseek-ai/cordis' {
  interface Context {
    llm: LlmRuntime
  }
  interface Events {
    'llm/stream'(this: LlmRuntime, options: GenerateOptions,
      next: () => AsyncIterable<StreamChunk>): AsyncIterable<StreamChunk>
  }
}
```

`LlmRuntime` 公开的能力可以分成四组：

| 分组 | 方法 | 作用 |
|---|---|---|
| 注册表 | `registerAdapter(providers, adapter)`（L389）、`registerConfigurableProviders`（L483）、`registerModelDiscovery`（L557） | 登记适配器路由、可配置路由目录、模型发现服务 |
| 元数据 | `listProviders` / `listModels` / `resolveModelInfo` / `resolveCallConfig` / `imageRequestPricing` / `fileRequestText` | 查询路由与模型能力，全部经过校验并返回分离副本 |
| 调用 | `stream(options)`（L1128）、`prepareCall(config, signal?)`（L929） | 发起一次流式调用；`stream` 内部经 `llm/stream` waterfall 包装 |
| 策略 | `providerRetryPolicy(provider)`（L656） | 读取注册时捕获的不可变重试策略 |

事件有两个：

- **`llm/stream`（waterfall）**：每次流式调用的拦截点，重试、重放、路由都在这里挂。waterfall 监听器必须调用 `next()` 才能到达真正的适配器流（第 4 章讲过的语义）。
- **`llm/adapters-updated`（emit）**：注册表拓扑变化的通知，声明在 [types.ts](../../packages/llm/llm/src/types.ts) L23。注意 `LlmRuntime.emitAdaptersUpdated()`（index.ts L355-L373）特意逐个 `try` 包住了每个监听器——原生 `ctx.emit` 用 `Array.map` 执行回调，一个同步抛错会饿死后面所有监听器，而注册表通知是"不容否决"的，所以必须各自隔离。

`registerAdapter` 有两个值得注意的细节。其一，同一个 provider 路由被第二个适配器认领会抛 `DUPLICATE_ADAPTER`（L431），而且是全有或全无（all-or-nothing）——候选集合先整体校验，失败则原注册纹丝不动。其二，它返回的句柄除了是释放器（disposer），还带 `replace(providers)`（L314）：同一路由集合的原子替换，整个换路由由一个同步段完成（`commitRoutes`，L456-L464），任何并发请求都观察不到"先摘掉再挂上"的空窗。为什么需要它？因为路由集合由用户配置决定——`llm-pi-ai` 插件在设置变化时就是用 `registration.replace(routes)` 原地换路由的（[llm-pi-ai/src/index.ts](../../packages/llm/llm-pi-ai/src/index.ts) L302-L305）。

三角色在真实文件上的落位，一张表说清：

| 角色 | 真实文件 | 干什么 |
|---|---|---|
| **Service Definition**（服务定义） | [llm/src/index.ts](../../packages/llm/llm/src/index.ts) 的 `LlmRuntime` + [types.ts](../../packages/llm/llm/src/types.ts) 的事件声明 | 声明接口、事件、统一词汇、注册表与错误类型 `LlmError` |
| **Service Provider**（服务提供方） | [llm-deepseek-api-key](../../packages/llm/llm-deepseek-api-key/src/index.ts)、[llm-deepseek-account](../../packages/llm/llm-deepseek-account/src/index.ts)（经 [host.ts](../../packages/llm/llm-deepseek/src/host.ts) 的 `registerDeepSeekProvider`，L18）、[llm-pi-ai](../../packages/llm/llm-pi-ai/src/index.ts) 的 `apply()`（L149） | 构造适配器实例、提供配置与认证回调、注册路由 |
| **Consumer**（消费方） | agent-loop（[agent.ts](../../packages/core/agent-loop/src/agent.ts) L436、L587）、压缩摘要器（[summarizer.ts](../../packages/compaction/compaction-basic/src/summarizer.ts) L163）、标题生成器（[session-title-llm/src/index.ts](../../packages/session/session-title-llm/src/index.ts) L281） | 组装 `GenerateOptions`，消费 `StreamChunk`，用 `BlockAssembler` 组装结果 |

`llm` 组的九个包分工如下（完整清单见 [packages/llm/README.md](../../packages/llm/README.md)）：

| 包 | 角色 |
|---|---|
| `llm` | `ctx.llm`：`LlmRuntime`——适配器注册表 + 可拦截流式 API + 全 harness 共用消息/流词汇 + `BlockAssembler` + retry policy 解析 + `LlmError` |
| `llm-deepseek` | 共享库：DeepSeek Messages 协议完整实现（序列化、SSE、翻译、配置），本身不注册路由 |
| `llm-deepseek-api-key` | 官方路由 `deepseek-official` 的 API-key 认证接入 |
| `llm-deepseek-account` | 账户 token 路由 `deepseek-account`：`x-dsh-auth-token` 头，401 映射 `ACCOUNT_TOKEN_INVALID`，配额映射 `ACCOUNT_QUOTA` |
| `llm-pi-ai` | 基于 `@earendil-works/pi/ai` 的多 provider 适配器：catalog 路由 + 手写网关路由（三种 wire 协议） |
| `deepseek-llm-api-extensions` | `ctx.deepseekLlmApiExtensions`：官方请求顶层扩展字段注册表，插件各拥独立顶层字段 |
| `plugin-package-inventory-deepseek` | 向官方请求注入 Loader 包清单（`dsh_plugin_packages` 字段） |
| `llm-retry` | 在 `agent/request-error` 扩展点上执行重试 |
| `token-meter` | `ctx.tokenMeter`：从持久日志重放测量，零模型调用 |

### 3.2 统一词汇：请求、块、分片与用量

DSH 全 harness 只有这一套模型调用词汇——循环、会话日志、压缩、UI 全都用它。（你可以把它理解为这台机器内部流通的通用语；这是第二个比喻。）定义集中在 [types.ts](../../packages/llm/llm/src/types.ts)。

**请求：`GenerateOptions`（L511）**。一次完全组装好的模型请求：

```ts
// packages/llm/llm/src/types.ts（节选，L511-L553）
export interface GenerateOptions {
  provider: string          // 已注册路由，决定用哪个适配器
  model: string
  reasoningEffort?: ReasoningEffortId
  messages: RequestMessage[]  // 有序对话历史，与 provider 看到的完全一致
  system?: string             // 一次性调用方的系统提示词；循环请求不用它
  tools?: ToolSchema[]        // 工具 schema，适配器映射到 provider 的 tools 字段
  toolHistory?: ToolHistory   // 会话折叠的工具历史，用于按路由投影
  temperature?: number
  maxTokens?: number
  stop?: string[]
  signal?: AbortSignal
  sessionId?: Branded<'SessionId'>
  purpose?: 'compaction' | 'session-title'  // 辅助调用的中性分类
}
```

`ToolSchema`（L473）是 `{name, description, parameters, deferLoading?}`——工具的 JSON Schema 描述，发给模型用。它声明在 `dsh-llm` 而不是 `dsh-tools`，正因为它是请求的一部分；产出它的 `ToolDefinition`（schema + `execute`）属于第 7 章的工具系统。

**内容块：`ContentBlock`（L150）**。从 `ContentBlockMap`（L137）派生的可合并扩展联合：`text`、`reasoning`、`image`、`file`、`tool-call`、`tool-addition`、`tool-removal`。注意 `file` 与 `tool-addition`/`tool-removal` 这三块是 DSH 自己的持久化词汇，**不是**任何 provider 的原生概念——马上会讲它们如何被投影。

**响应流：`StreamChunk`（L452）**。适配器产出的原始分片协议，七种变体：

```ts
// packages/llm/llm/src/types.ts（节选，L452-L464）
export type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: ToolCallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | { type: 'finish'; reason: FinishReason; replayState?: ReplayEnvelope }
```

三条顺序不变量写在 JSDoc 里（L444-L451）：**`usage` 先于 `finish`，`finish` 之后不再有任何分片；工具调用的 `arguments` 全程保持原始 JSON 字符串**。`index` 把交错出现的 delta 关联到所属块，`block-end` 携带组装完成的完整块——所以块重组不是每个适配器各自的问题，消费方也不需要自己拼 delta。

为什么 `usage` 必须在 `finish` 之前？因为 `finish` 是终止分片，消费方收到它就认为流结束了。把用量放在它前面，等于结账前先把账记好——组装方读到 `finish` 的那一刻，usage、结束原因、回放状态全部到手，不必再等下一段。（这是第三个比喻，说好了就这三个。）真实实现在 [translate.ts](../../packages/llm/llm-deepseek/src/translate.ts) L159-L161：先 `yield { type: 'usage', ... }` 再 `yield { type: 'finish', ... }`，紧挨着。

**结束原因：`FinishReasonMap`（L156-L162）**：`stop` / `tool-calls` / `max-tokens` / `aborted`（携带 `LlmFailure`）/ `error`（携带 `LlmFailure`）。前三个是正常结束，后两个是归一化后的失败——适配器抛出的任何异常，最终都会变成这两种之一（见 3.4 节）。

**用量：`TokenUsage`（L175）**。这是最容易踩坑的一个类型，JSDoc 用大写强调：各计数**互不重叠（DISJOINT）**：

```ts
// packages/llm/llm/src/types.ts（节选，L167-L189）
/**
 * Counts are DISJOINT: `inputTokens` is uncached input only; cached input is
 * reported separately as `cacheReadTokens`/`cacheWriteTokens` (billed input =
 * sum of the three). ...
 */
export interface TokenUsage {
  inputTokens: number     // 未缓存输入
  outputTokens: number
  totalTokens?: number
  cacheReadTokens?: number   // 缓存命中的输入
  cacheWriteTokens?: number  // 写入缓存的输入
  reasoningTokens?: number   // 信息性细节，已含在 outputTokens 里
}
```

**计费输入 = `inputTokens + cacheReadTokens + cacheWriteTokens`**，三者之和。有些 provider（如 DeepSeek 官方 API 的 `prompt_tokens`）把缓存命中折进一个总数，适配器要负责减出来。`reasoningTokens` 只是信息性字段，已经包含在 `outputTokens` 里，汇总时不能再加一遍。

**组装器：`BlockAssembler`**（[assembler.ts](../../packages/llm/llm/src/assembler.ts) L38）。把 `StreamChunk` 流折叠回 `ContentBlock`、usage、结束原因与回放状态的唯一共享实现。agent-loop 边记录原始分片边喂给它（日志保真），流结束后读 `blocks()` / `message()` / `usage` / `finish`。它有两个防御性设计：容忍只有 delta、没有 `block-start`/`block-end` 的协议（L98-L106 的 `ensure` 按需建块）；`block-end` 闭合之后迟到的 delta 直接忽略（L65、L71），一个行为不端的适配器既撑不爆内存也污染不了已完成的块。还有一个细节：`max-tokens` 截断时丢掉所有工具调用块（L137-L140）——被截断的参数 JSON 不能安全执行，而丢弃决定同时作用于块和回放元数据，两者不可能不一致。

### 3.3 运行时投影：文件、图片与工具变更

`GenerateOptions.messages` 是 DSH 的内部词汇，直接发给 provider 会出问题：`FileBlock` 是持久化的附件引用，没有哪家 API 认识它。所以 `LlmRuntime.adapterStream`（index.ts L1026）在把请求交给适配器之前，先做三类**投影（projection）**——把内部表示改写为"这个路由实际能收到的东西"。源码在 [content.ts](../../packages/llm/llm/src/content.ts)。

1. **文件投影，无条件**：`projectFilesToText`（L185）。L1058-L1061 的注释说得斩钉截铁——"Files are never dispatched natively: every route receives handle text"。每个 `FileBlock` 被替换成确定性的 handle 文本：文件名、字节数、sha256 摘要前 8 位，加上只读副本的路径和"用文件工具按需读取"的指示（`fileHandleText`，L151-L158）。持久日志保留结构化引用（供展示与授权），provider 看到的永远是文本。
2. **图片投影，按模型能力**：`projectImagesForTextModel`（L360）。当目标模型的 `inputModalities` 不含 `image` 时，每个 `ImageBlock` 替换为 `[image omitted because this model accepts text only; attachment sha256:…]`。
3. **工具变更投影，按路由模式**：`projectToolUpdates`（L424）。`tool-addition`/`tool-removal` 块属于 developer 消息；路由声明的 `toolUpdate` 模式决定它收到什么——`'in-history'` 的路由能读移除块，`'addition-only'` 的路由只能读添加块且声明列表必须剔除已停用的工具，两者都不支持的路由干脆收不到任何 developer 更新消息、每次都拿完整声明表。

这套投影解释了一个反直觉的事实：**日志里的消息 ≠ 请求里的消息**。第 8 章会从持久化侧再讲一遍这个对偶。

### 3.4 一个真实适配器：llm-deepseek 五步流水线

`llm-deepseek` 是共享库，不直接注册路由；`llm-deepseek-api-key`（路由 `deepseek-official`）和 `llm-deepseek-account`（路由 `deepseek-account`）复用它接入。核心类是 [adapter.ts](../../packages/llm/llm-deepseek/src/adapter.ts) 的 `DeepSeekAdapter`（L20，继承 `LlmAdapter`），它把一次调用组织成一条流水线：

**第一步：准备素材**。`request()`（L75）先 `prepareImages` 准备请求图片版本，`resolveAuth` 解析认证（API-key 路由给 `x-api-key` 头；账户路由给 `x-dsh-auth-token` 头），`prepareFileIds` 尝试把图片经 Files API 上传换 file id（预算 60 秒，`DEFAULT_FILES_API_TIMEOUT_MS`，[defaults.ts](../../packages/llm/llm-deepseek/src/defaults.ts) L23）——失败则退回内联 base64。

**第二步：序列化**。[serialize.ts](../../packages/llm/llm-deepseek/src/serialize.ts) 的 `serialize(options, connection, history, images, access, ..., fileIds)`（L56）把统一词汇映射为 Anthropic Messages 风格的 wire 请求：`reasoning` 块变成 `thinking`、`tool-call` 变成 `tool_use`、`tool-addition`/`tool-removal` 变成 `tool_addition`/`tool_removal`（L94-L95）。一个精妙的细节是系统更新（system update）的位置：DSH 允许 system 消息出现在任意位置（支持提示词中途变更），而 Messages 协议要求系统更新插在 user 轮之后、下一个 assistant 轮之前——`flushSystemUpdates`（L84-L88）就是干这个的，在遇到 assistant 消息前（L117）把积压的系统更新刷进去。

**第三步：HTTP 派发**（adapter.ts L120-L132）。`POST {baseURL}/v1/messages`，请求头里你能看到本章所有主角同台：

```ts
// packages/llm/llm-deepseek/src/adapter.ts（节选，L120-L131）
const response = await fetch(`${messagesApiRoot(connection.baseURL)}/messages`, {
  method: 'POST', signal, body: extensions.payload, redirect: 'error',
  headers: {
    ...attributionHeaders(),                    // 归属头，见下文
    'content-type': 'application/json', 'accept': 'text/event-stream',
    ...auth.headers,                            // x-api-key 或 x-dsh-auth-token
    'anthropic-version': '2023-06-01',
    ...betas.length === 0 ? {} : { 'anthropic-beta': betas.join(',') },
    'x-deepseek-harness-user-id': this.dependencies.resolveUserId(),
    ...options.sessionId === undefined ? {} : { 'x-deepseek-harness-session-id': String(options.sessionId) },
    ...options.purpose === 'compaction' ? { 'x-deepseek-harness-compact': '1' } : {},
  },
})
```

`attributionHeaders()`（[attribution.ts](../../packages/llm/llm/src/attribution.ts) L64）返回 `user-agent: deepseek-harness/<版本> (+仓库地址)`。`LlmAdapter` 的 JSDoc（index.ts L202-L207）把它定为硬性契约：**每个 provider HTTP 请求必须携带归属头，且要在协议级测试里证明**——白标部署可以换 `AppIdentity`，但谁也不能把它去掉。beta 头按需追加：文件引用加 `files-api-2025-04-14`，工具变更块加 `mid-conversation-tool-changes-2026-07-01`（[messages-api.ts](../../packages/llm/llm-deepseek/src/messages-api.ts) L4、L7）。

**第四步：SSE 分帧**。[sse.ts](../../packages/llm/llm-deepseek/src/sse.ts) 的 `parseSse(body, activity)`（L13）把分帧工作委托给 `eventsource-parser` 库的 `EventSourceParserStream`（L14），自己只做两件事：每个帧 `JSON.parse` 并校验 `type` 字段（失败抛 `MALFORMED_RESPONSE`），以及给看门狗"喂活动"（heartbeat 注释也算活动，L14 的 `onComment: activity`）。

**第五步：翻译**。[translate.ts](../../packages/llm/llm-deepseek/src/translate.ts) 的 `translate(events, model)`（L104）把 Messages 事件逐个翻译成 harness 分片。下面是一段（简化的）真实 SSE 文本与翻译输出的对照：

```text
event: message_start
data: {"type":"message_start","message":{"usage":{"input_tokens":12,"output_tokens":1}}}
                ↓ （无分片；usage 记入累计）
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":""}}
                ↓ block-start {index:0, blockType:'reasoning'}
event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"先想…"}}
                ↓ reasoning-delta {index:0, text:"先想…"}
event: content_block_stop
data: {"type":"content_block_stop","index":0}
                ↓ block-end {index:0, block:{type:'reasoning',…}}
event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"text","text":""}}
                ↓ block-start {index:1, blockType:'text'}
event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"text_delta","text":"答案是…"}}
                ↓ text-delta {index:1, text:"答案是…"}
event: content_block_stop
data: {"type":"content_block_stop","index":1}
                ↓ block-end {index:1, block:{type:'text',…}}
event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":9}}
                ↓ （无分片；记录 stop 原因，累计 usage）
event: message_stop
data: {"type":"message_stop"}
                ↓ usage {inputTokens:12, outputTokens:9, …}
                ↓ finish {reason:{kind:'stop'}, replayState:…}
```

`translate` 里的关键映射函数：`updateUsage`（L34-L43）把 `input_tokens`/`output_tokens`/`cache_read_input_tokens`/`cache_creation_input_tokens` 累计进 `TokenUsage` 的四个字段；`stopReason`（L90-L97）把 wire 的 `end_turn`/`stop_sequence` 映射为 `stop`、`tool_use` 映射为 `tool-calls`、`max_tokens` 映射为 `max-tokens`，不认识的直接 `MALFORMED_RESPONSE`。流在 `message_stop` 之前断掉则抛 `STREAM_CLOSED`（L165）；一个块都没有却以 `stop` 结束，归一为可重试的 `EMPTY_RESPONSE`（L149）。

**超时**。`generate()`（L51）用 `idleWatchdog`（L54）管理整个请求生命周期：只统计"一次流读挂起超过 `streamIdleTimeoutMs`"的空闲（默认 300,000ms，即 5 分钟），期间任何分片或心跳都会"喂狗"。空闲超时映射为 `TIMEOUT`，调用方提前中止映射为 `ABORTED`（L62-L66）。这是比"总时长超时"聪明得多的设计——一次合法的长思考不会被误杀，真正的死连接 5 分钟内必被发现。

**错误归一**。`LlmRuntime.adapterStream`（index.ts L1026）是最终适配器边界：适配器选择、派发、迭代器构造、迭代过程中的任何失败，统一经 `adapterFailureChunk`（L1146）转成一个终止 `finish` 分片——`signal` 已中止或 code 为 `ABORTED` 时是 `{kind:'aborted', failure}`，否则 `{kind:'error', failure}`。`LlmError`（L96，继承 `HarnessError`）携带的 `LlmFailure`（message/code/status?/providerRetryAfterMs?/requestId?/offloadImages?）是可序列化的提供方无关事实，**路由依据永远是 `code`，绝不解析 message 文本**。注意边界只归一化适配器侧失败；中间件与下游消费方的失败仍然原样抛出（L1105-L1107 的注释解释了为什么 `yield` 要放在 try 外面）。

**配置解析**。[config.ts](../../packages/llm/llm-deepseek/src/config.ts) 的 `deepSeekConfigFields`（L82）声明全部字段：`baseURL`、`thinking`（enabled/disabled）、`reasoningEffort`（off/low/high/max，默认 high）、`maxTokens`（默认 256,000）、`defaultContextWindow`（默认 1,000,000）、`models`（advisory 目录，[models.ts](../../packages/llm/llm-deepseek/src/models.ts) L6 默认 `deepseek-flash` + `deepseek-v4-pro`）、`streamIdleTimeoutMs`、图片/文件预算、`retryPolicy`。全部字段是 `Volatile`——支持热更新。`resolveAdapterOptions(config, environment?)`（L204）是显式的 resolve（解析）步骤：默认值填充与边界钳制全部在这里完成，加载时 fail-loud。这是仓库"request/spec 原则"的模板——显式优于隐式，绝不在 `run()` 里藏一个 `?? default`。`baseURL` 的三级回退链在 L290：

```ts
// packages/llm/llm-deepseek/src/config.ts（L290）
const baseURL = config.baseURL ?? environment?.get(BASE_URL_ENV)?.value ?? PUBLIC_BASE_URL
// BASE_URL_ENV = 'DEEPSEEK_BASE_URL'（L109）
// PUBLIC_BASE_URL = 'https://api.deepseek.com/anthropic'（L106）
```

**API key 解析**。[llm-deepseek-api-key/src/index.ts](../../packages/llm/llm-deepseek-api-key/src/index.ts) 的 `resolveApiKey()`（L20）：优先经 `ctx.credentials.resolve(ref)`（凭证服务，`ref` 默认 `DEEPSEEK_API_KEY`），没有凭证服务则退回启动环境变量。缺失抛 `MISSING_CREDENTIAL`；取到但格式非法（比如带了 HTTP 头放不下的字符）抛 `INVALID_CREDENTIAL`——校验在 `assertUsableApiKey`（[llm/src/index.ts](../../packages/llm/llm/src/index.ts) L151），报错信息只说"去哪改"，绝不回显密钥本身。账户路由（[llm-deepseek-account/src/index.ts](../../packages/llm/llm-deepseek-account/src/index.ts)）则是另一套：`resolveToken` 拿账户 token 放进 `x-dsh-auth-token` 头（L25），401 时拒绝该 token 并映射 `ACCOUNT_TOKEN_INVALID`（L32），配额耗尽映射 `ACCOUNT_QUOTA`（L28-L29）。

**官方请求扩展**。`ctx.deepseekLlmApiExtensions` 是个注册表：贡献插件各认领一个 `dsh_` 前缀的顶层请求字段（如 `dsh_session_log` 会话日志后缀、`dsh_plugin_packages` 包清单），适配器在序列化完基础请求后调 `prepare()` 合并字段、HTTP 2xx 后跑 `accept()` 事务。这些字段在 `messages` 之外，不占模型输入 token。完整 wire 契约见 [docs/deepseek-llm-api-wire-extensions.md](../../docs/deepseek-llm-api-wire-extensions.md)。

### 3.5 双生适配器：llm-pi-ai 与零代码路径

`llm-deepseek` 是"直连"流派的适配器：自己 fetch、自己解析 SSE。`llm-pi-ai`（[adapter.ts](../../packages/llm/llm-pi-ai/src/adapter.ts) 的 `PiAiAdapter`，L219）是"库"流派的：把 wire 协议委托给 `@earendil-works/pi-ai` 库，自己只做词汇翻译。它支持三种 wire 协议（[provider.ts](../../packages/llm/llm-pi-ai/src/provider.ts) L48-L50）：`openai-completions`、`openai-responses`、`anthropic-messages`。

它的配置是一个 `providers` 字典（[config.ts](../../packages/llm/llm-pi-ai/src/config.ts) L352-L354），键即路由名。两种路由：catalog 路由（pi-ai 装了目录的厂商，如 `openai`、`anthropic`，只补 `apiKeyEnv` 即可）和手写网关路由（pi-ai 不认识的 key，必须声明 `api` + `baseURL` + `models`）。`apiKeyEnv` 逐请求经 `ctx.credentials` 解析（[index.ts](../../packages/llm/llm-pi-ai/src/index.ts) L183-L206），一旦命名了引用而没取到，fail-loud 抛 `MISSING_CREDENTIAL`——绝不静默回落到环境里不相干的 key。

`PiAiAdapter` 的看家本领是**快照不可变性**（adapter.ts L65-L71、L330-L343）：每次解析配置都冻结出一个包含全部 profile 与模型集合的不可变快照，操作在第一个 `await` 前捕获整个快照。配置中途变了？新快照另起炉灶，进行中的请求用它出生时的那一份跑完。这让上层 `prepareCall()` 的"代"绑定一路贯穿到底——切模型在下一步生效，绝不会出现在一次回复中间。

组装层（base bundle，[cordis.patch.yml](../../packages/bundle/base/cordis.patch.yml)）把两个流派同时挂上：`llm`（L34-L35）、`llm-retry`（L91-L92）、`llm-pi-ai` 休眠挂载（L127-L128，注释写明：零路由直到 `llm-pi-ai:` 设置分节提供 profile）。哪些适配器存在是组合（composition）的事，哪些路由在跑是用户设置文档的事——第 12 章会展开这个原则。

### 3.6 `prepareCall`：绑定一"代"适配器

`prepareCall(config, signal?)`（index.ts L929）返回 `PreparedLlmCall`（L169）：`config`（深度冻结）+ `retryPolicy` + `stream(options)`。它不是缓存，而是**绑定**——绑定一次适配器注册的"代"（generation）。看它的 `stream` 实现（L957-L974）：

```ts
// packages/llm/llm/src/index.ts（节选，L957-L973）
stream: (options: GenerateOptions): AsyncIterable<StreamChunk> => {
  if (dispatched) {
    throw new LlmError('a prepared LLM call can only be dispatched once', 'INVALID_PREPARED_CALL')
  }
  if (!callConfigEquals(options, resolvedConfig)) {
    throw new LlmError('prepared LLM call config changed before adapter dispatch', 'INVALID_PREPARED_CALL')
  }
  dispatched = true
  return this.streamWithRegistration(options, { registration, config: resolvedConfig, modelInfo, dispatch: … })
},
```

两个限制：一个 prepared call 只能派发一次；请求的 config 字段必须与准备时一致，否则 `INVALID_PREPARED_CALL`。为什么这么苛刻？因为 agent-loop 在准备与派发之间要**持久化请求头**（request header，第 8 章）——`prepareCall` 保证日志里记录的那个适配器代与实际执行派发的是同一代，热更新（HMR）无法把一个适配器的能力元数据跟另一个适配器的端点拼在一起。这是"模型可见 ⟺ 已日志"不变量在调用侧的对应物：日志描述的调用与真正发出的调用必须是同一次。`adapterDefaults` 字段则标记哪些 config 值是适配器默认值填充的（区别于调用方显式设置），日志据此区分"用户选的"与"路由补的"。

`forAdapter`（L985-L999）是同一主题的另一个侧面：请求历史里 assistant 消息携带的 `replayState`（适配器私有的回放状态，如 thinking 签名）只有在"历史路由与目标路由当前注册在同一个适配器实例上"时才会传递给适配器，否则剥掉，适配器只收到提供方无关的内容。

### 3.7 重试与计量：为什么独立成包

一个初看奇怪的安排：`LlmRuntime` 明明知道每个路由的重试策略（`providerRetryPolicy`，注册时捕获的不可变 `ResolvedRetryPolicy`），却**从不重跑请求**。每次 `stream()` 调用就是一次 provider 尝试，仅此而已。重试在 [llm-retry](../../packages/llm/llm-retry/src/index.ts) 插件里，计量在 [token-meter](../../packages/llm/token-meter/src/index.ts) 插件里。为什么？

**重试属于 agent 恢复，不属于模型服务。** 重跑一次请求意味着 agent-loop 要开一个新的持久化、带编号的步骤（step），失败尝试要落日志、UI 要展示"正在重试"。这不是传输层细节，而是循环语义。所以 `llm-retry` 插件监听的是 agent 域的 `agent/request-error` 扩展点（L243），而它自己**没有任何配置**——`Config = {}`（L25），策略全部由各 provider 配置里的 `retryPolicy` 字段拥有（L30-L37 的 `validateConfig` 甚至会拒绝你在插件级写 `retryPolicy`）。策略的解析在 [retry-policy.ts](../../packages/llm/llm/src/retry-policy.ts)：`RetryPolicyConfig` 是 `normal`（有限次）/`always`（无限）两种模式的可辨识联合，`resolveRetryPolicy`（L149）解析并冻结。默认值：`maxRetries=5`、`initialDelayMs=500`、`maxDelayMs=10_000`、`jitterRatio=0.1`，可重试码 `[EMPTY_RESPONSE, RATE_LIMIT, SERVER, TIMEOUT, TRANSPORT]`（L14-L24）——注意 `INVALID_CREDENTIAL`、`ACCOUNT_QUOTA` 这类"重试一亿次也一样"的失败不在名单里。

执行时还有一个次序细节：**每次重试先落盘再等待**（L188-L190）——`agent.session.append('llm/retry', …)` 先写进会话日志，`cancellableDelay` 才开始等待，等待结束后再追加 `llm/retry-started`。重试状态持久化为会话投影（session projection）`llmRetry`（L116-L121，键是 `RetryId` 品牌类型），所以进程崩溃后重启不会把重试次数清零重数。退避算法是指数退避加对称抖动（L59-L64 的 `localDelay`），provider 若给了 `Retry-After` 且不超过 `maxDelayMs` 则优先采用（L227-L238）。

**计量必须与调用解耦。** `ctx.tokenMeter` 的测量方式是从持久会话日志**重放**（replay）：确定性地重算请求与表面压力，**零模型调用**（[token-meter/src/index.ts](../../packages/llm/token-meter/src/index.ts) L101 的 `TokenMeter` 类，构造时注册 `tokenUsage`、`contextPressure`、`contextBreakdown` 三个会话投影，见 [usage-projection.ts](../../packages/llm/token-meter/src/usage-projection.ts) L118、L174 与 [breakdown-projection.ts](../../packages/llm/token-meter/src/breakdown-projection.ts) L49）。想象一下反面设计：每次想知道"现在上下文用了多少 token"就真调一次模型 API——既花钱又不确定（估计本身要吃上下文），还没法离线分析历史会话。重放式计量对同一段日志永远给出同一个数，它是第 10 章压缩触发器的数据源：`contextPressure` 超过阈值，压缩插件才会动手摘要历史。

两个包共同体现第 12 章的原则：**插件而非循环改动**。重试和计量都是"可以不装"的——base bundle 装了它们（cordis.patch.yml L91、L338），但一个不需要重试的特殊部署可以把它摘掉，`LlmRuntime` 毫发无伤。

## 4. 源码走读

### 4.1 `adapterStream`：最终适配器边界（index.ts L1026-L1115）

这段代码值得逐行读，因为它同时完成了"词汇投影 + 错误归一 + 资源清理"三件事。

```ts
// packages/llm/llm/src/index.ts（节选，L1057-L1085）
const resolvedOptions = callConfigEquals(options, resolvedConfig)
  ? options
  : Object.isFrozen(options)
    ? deepFreeze({ ...options, ...resolvedConfig })
    : { ...options, ...resolvedConfig }
// Files are never dispatched natively: every route receives handle text.
let projectedMessages: readonly RequestMessage[] = resolvedOptions.messages
if (projectedMessages.some(message => contentHasFile(message.content))) {
  projectedMessages = projectFilesToText(projectedMessages, ref => this.fileReadPath(ref))
}
if (modelInfo.inputModalities !== undefined
  && !modelInfo.inputModalities.includes('image')
  && projectedMessages.some(message => contentHasImage(message.content))) {
  projectedMessages = projectImagesForTextModel(projectedMessages)
}
const projectedTools = projectToolUpdates(projectedMessages, resolvedOptions.tools, modelInfo.toolUpdate, resolvedOptions.toolHistory)
projectedMessages = projectedTools.messages
…
const stream = dispatch(this.forAdapter(projectedOptions, adapter))
iterator = stream[Symbol.asyncIterator]()
} catch (error: unknown) {
  yield adapterFailureChunk(error, options.signal)
  return
}
```

走读要点：

1. **L1053-L1057**：请求与解析后的 config 合并。LOOP 构建的请求是深度冻结的（`llm/stream` 的 JSDoc L67-L72 说明了原因：它的内容是会话日志的纯函数，监听器只能读不能改），合并结果保持冻结。
2. **L1058-L1070**：三类投影按序执行——文件无条件、图片按模型模态、工具变更按路由模式。投影是"只改需要改的"：没有文件就不复制消息数组，`projectFilesToText` 直接原样返回（content.ts L203）。
3. **L1080**：`this.forAdapter(projectedOptions, adapter)` 剥掉不属于本适配器代的回放状态（3.6 节）。
4. **L1082-L1085**：从适配器选择到迭代器构造的任何同步失败，统一变成一个终止 `finish` 分片——消费方从协议上根本遇不到"流建立失败"这种半开状态。
5. **L1087-L1114**：迭代循环。注意 L1105-L1107 那条注释——`yield` 特意放在适配器侧 `try` 块**之外**：消费者在消费分片时抛出的错误（比如调用方 `break` 后的清理）必须保持为抛出的插件/消费方错误，不能被归一化成流内容。`finally` 块在流未完成时调用 `iterator.return()` 释放适配器资源。

### 4.2 `translate` 的收尾：`message_stop`（translate.ts L143-L163）

```ts
// packages/llm/llm-deepseek/src/translate.ts（节选，L148-L165）
} else {
  if (reason === undefined || [...blocks.values()].some(block => !block.closed)) return malformed('message_stop without settled blocks and stop reason')
  if (blocks.size === 0 && reason.kind === 'stop') throw new LlmError('DeepSeek Messages returned no content', 'EMPTY_RESPONSE')
  // Truncated tool JSON is retained in the stream, then pruned by the shared assembler.
  if (reason.kind !== 'max-tokens') {
    for (const { content } of blocks.values()) {
      if (content.type !== 'tool-call') continue
      let parsed: unknown
      try { parsed = JSON.parse(content.arguments) } catch (_invalidProviderToolJson) { return malformed('tool input is invalid JSON') }
      object(parsed)
    }
  }
  usage.totalTokens = usage.inputTokens + usage.outputTokens + (usage.cacheReadTokens ?? 0) + (usage.cacheWriteTokens ?? 0)
  yield { type: 'usage', usage }
  yield { type: 'finish', reason, replayState: replayState(model, […blocks.values()].map(block => block.replay)) }
  return
}
```

走读要点：

1. **前置校验**：所有块必须闭合、stop 原因必须已知，否则 `MALFORMED_RESPONSE`。
2. **空响应**：零块 + `stop` = `EMPTY_RESPONSE`（可重试错误，而不是一条空空的 assistant 消息——空消息会让循环无事可做地结束这一轮，那是静默失败）。
3. **截断宽容**：`max-tokens` 截断时**不**校验工具参数 JSON——截断的 JSON 留在流里，由共享的 `BlockAssembler` 统一丢弃（3.2 节）。适配器与组装器各司其职，丢弃决定只做一次。
4. **L159**：`totalTokens` 在这里从四个不相交分桶推导——注意它**不含** `reasoningTokens`（那是 `outputTokens` 的子集）。
5. **L160-L161**：`usage` 分片先于 `finish` 分片，`finish` 携带 `ReplayEnvelope`（回放信封：响应级 + 逐块的适配器私有元数据，存进 assistant 消息的来源里，下次请求只有同一适配器代才能拿回，见 3.6 节）。

## 5. 常见误区

**误区一：以为请求里直接塞了 `FileBlock`。** 读完 3.3 节你应该已经免疫了：文件块在 `adapterStream` 里被无条件投影成确定性 handle 文本，任何路由、任何 provider 收到的都是文本。持久日志里的 `FileBlock` 是给展示与授权用的，不是给模型看的。同理，文本模型收到的是图片占位符而非 `ImageBlock`。

**误区二：以为重试逻辑在 `LlmRuntime` 里。** 它在 `llm-retry` 插件里，挂在 `agent/request-error` 扩展点上。`LlmRuntime` 只负责在注册路由时**捕获**不可变的重试策略（`providerRetryPolicy`），执行则是插件的事。直接调 `ctx.llm.stream()` 的调用方失败就是失败，只有走 agent 循环的请求才有持久化重试。把"策略声明"与"策略执行"分开，正是为了让你可以换一个重试插件而不动服务本体。

**误区三：以为 `TokenUsage.inputTokens` 就是计费输入。** 它只是**未缓存**输入。计费输入 = `inputTokens + cacheReadTokens + cacheWriteTokens`（三段互不相交）。只看 `inputTokens`，一个缓存命中率高的会话看起来便宜得离谱。另外 `reasoningTokens` 已含在 `outputTokens` 里，汇总别加第二遍。

**误区四：以为 assistant 消息存的是 delta 增量。** `StreamChunk` 里的 `text-delta`/`tool-call-delta` 是传输协议；日志里存的 assistant 消息携带组装完成的完整块，以及一份内嵌的紧凑流（compact stream，按块合并了 delta、保留精确时间戳）。delta 是过程，块是结果，日志两者都要——但形态不同。第 8 章讲持久化格式时你会看到 `AssistantStreamRecord` 的样子。

**误区五：以为 `prepareCall` 只是个缓存。** 它绑定的是适配器"代"：准备时捕获的注册必须与派发时使用的注册是同一代，config 不匹配或二次派发都抛 `INVALID_PREPARED_CALL`。它防的是一种真实的竞态：日志刚记录完"这次调用用 deepseek-official 路由、上下文窗口 1M"，热更新就换掉了路由的适配器——没有这个绑定，日志描述的调用与实际发出的调用就可能是两次不同的调用，破坏"模型可见 ⟺ 已日志"。

## 6. 动手环节

四个练习，由浅入深：

**练习 1：读组 README。** 打开 [packages/llm/README.md](../../packages/llm/README.md)，对照 3.1 节的九包表格，确认你能在"Role"一列里说出每个包的一句话职责。再点进 [docs/subsystems/llm-streaming.zh.md](../../docs/subsystems/llm-streaming.zh.md) 的"适配器约定"一节，数一数那里列了几条硬性契约（答案：九条左右，包括 usage 顺序、原始 JSON 参数、一次调用一次尝试、空闲看门狗、归属头等）。

**练习 2：追踪一个工具 schema 的旅程。** 打开 [docs/tool-catalog.md](../../docs/tool-catalog.md)，随便挑一个工具（比如 `str_replace`），记下它的 `name`、`description`、`parameters`。然后回答：这三个字段在哪一步变成 `ToolSchema`（types.ts L473）？在哪一步变成 wire 的 `tools[].input_schema`（serialize.ts L159-L166）？`deferLoading` 呢？如果目标路由是 `addition-only` 模式而该工具已被停用，它还会出现在请求里吗？（提示：content.ts L396-L412 的 `toolDeclarations`。）

**练习 3：写一个假适配器骨架。** 不需要真的跑通，把它当作对契约的记忆检验。30 行内：

```ts
// 教学骨架：把用户最后的输入原样回显
import { LlmAdapter } from '@deepseek-ai/dsh-llm'
import type { GenerateOptions, StreamChunk } from '@deepseek-ai/dsh-llm'

class EchoAdapter extends LlmAdapter {
  override providerInfo(provider: string) {
    return { id: provider, name: 'Echo（教学）' }
  }

  override async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    options.signal?.throwIfAborted()               // 契约：尊重取消信号
    const last = options.messages.at(-1)
    const heard = last?.content.flatMap(b => b.type === 'text' ? [b.text] : []).join('') ?? ''
    yield { type: 'block-start', index: 0, blockType: 'text' }
    yield { type: 'text-delta', index: 0, text: heard }
    yield { type: 'block-end', index: 0, block: { type: 'text', text: heard } }
    yield { type: 'usage', usage: { inputTokens: 1, outputTokens: 1 } }  // 先记账
    yield { type: 'finish', reason: { kind: 'stop' } }                    // 后结账
  }
}

export function apply(ctx: any) {
  ctx.llm.registerAdapter(['echo-demo'], new EchoAdapter())
}
```

写完自查四个问题：`block-end` 里的块与 delta 累计一致吗？usage 在 finish 之前吗？signal 检查在哪？忘了 `providerInfo` 会怎样（提示：`prepareRoutes` 的 `INVALID_ADAPTER` 校验，index.ts L433-L436）？

**练习 4：查配置目录。** 打开 [docs/config-catalog.md](../../docs/config-catalog.md)，找到 `@deepseek-ai/dsh-llm-deepseek-api-key` 一节（约 L1588 起）。对照 3.4 节：`apiKeyEnv` 的默认值是什么？`reasoningEffort` 有哪四个取值？把 `streamIdleTimeoutMs` 改成 `100` 会发生什么——什么时候报错，报什么错？（提示：`resolveAdapterOptions` 的 fail-loud 校验，config.ts L222-L229；想想它是加载时报还是请求时报。）

## 7. 本章小结

1. `ctx.llm`（`LlmRuntime`，[llm/src/index.ts](../../packages/llm/llm/src/index.ts) L342）是模型调用的能力接缝：适配器注册表 + 经 `llm/stream` waterfall 包装的流式 API + 全 harness 共用的消息/流词汇。
2. 两个事件：`llm/stream`（waterfall，重试/重放/路由的拦截点）与 `llm/adapters-updated`（emit，拓扑通知；监听器失败被逐一隔离，不能否决注册表变更）。
3. `registerAdapter` 全有或全无，路由冲突抛 `DUPLICATE_ADAPTER`；返回句柄的 `replace()` 提供同适配器的原子路由替换，服务配置驱动的路由集。
4. 统一词汇四件套：`GenerateOptions`（完全组装的请求）、`ContentBlock`（可合并扩展的内容块联合）、`StreamChunk`（七种分片）、`TokenUsage`（互不相交的用量分桶）。
5. `StreamChunk` 的顺序不变量：`usage` 先于 `finish`、`finish` 后零分片、工具参数全程保持原始 JSON 字符串；块重组由共享的 `BlockAssembler` 完成，不是各适配器的事。
6. 计费输入 = `inputTokens + cacheReadTokens + cacheWriteTokens`；`reasoningTokens` 是 `outputTokens` 的信息性子集。
7. 适配器边界做三类运行时投影：文件无条件投影为 handle 文本（`FileBlock` 从不原生发出）、图片按模型模态投影、工具变更按路由 `toolUpdate` 模式裁剪——日志里的消息 ≠ 请求里的消息。
8. DeepSeek 适配器是五步流水线：素材准备 → `serialize`（含 system 更新插位）→ 带 `anthropic-version`/认证/归属/beta 头的 fetch → `parseSse`（eventsource-parser 分帧 + JSON 校验）→ `translate`（wire 事件 → harness 分片）。
9. 超时用空闲看门狗（`idleWatchdog`，默认 5 分钟无活动才触发），映射为 `TIMEOUT`；调用方中止映射为 `ABORTED`；一切适配器侧失败经 `adapterFailureChunk` 归一为终止 `finish` 分片，路由永远看 `code` 不看文本。
10. 配置解析是显式的 `resolveAdapterOptions(config, environment?)` 步骤：默认值填充、边界钳制、fail-loud 校验全在此处，`run()` 里不藏 `?? default`；`baseURL` 三级回退：config → `DEEPSEEK_BASE_URL` → `https://api.deepseek.com/anthropic`。
11. 新增 provider 两条路径：零代码（`llm-pi-ai` 的 `providers` 字典，三种 wire 协议）或写 `LlmAdapter` 子类（唯一抽象方法 `stream()`），模板是 `registerDeepSeekProvider`。
12. `prepareCall` 绑定适配器"代"：一次性派发 + config 一致性检查（`INVALID_PREPARED_CALL`），保证日志与实际分发共享同一次注册——"模型可见 ⟺ 已日志"的调用侧对应物。
13. **服务从不重跑请求**：每次 `stream()` 是一次尝试；重试由 `llm-retry` 插件在 `agent/request-error` 扩展点执行（指数退避 + 对称抖动，先落盘再等待，状态持久化为 `llmRetry` 投影），策略由 provider 配置拥有。
14. `token-meter` 从持久日志重放测量（确定性、零模型调用），暴露 `tokenUsage`/`contextPressure`/`contextBreakdown` 三个投影，是第 10 章压缩触发的数据源。
15. 一次调用 = 一次尝试、策略归提供方、执行归扩展点、计量归日志重放——四个关注点四个包，这就是"llm 能力接缝"的全貌。

## 8. 必读源码回顾与延伸阅读

**必读源码回顾**（按建议阅读顺序）：

1. [packages/llm/llm/src/types.ts](../../packages/llm/llm/src/types.ts) —— 全部词汇的权威定义：`ContentBlockMap`（L137）、`FinishReasonMap`（L156）、`TokenUsage`（L175）、`ToolSchema`（L473）、`StreamChunk`（L452）、`GenerateOptions`（L511）。
2. [packages/llm/llm/src/index.ts](../../packages/llm/llm/src/index.ts) —— `LlmError`（L96）、`PreparedLlmCall`（L169）、`LlmAdapter`（L208）、`LlmRuntime`（L342）、`adapterStream`（L1026）、`adapterFailureChunk`（L1146）。
3. [packages/llm/llm/src/assembler.ts](../../packages/llm/llm/src/assembler.ts) —— `BlockAssembler`（L38）与 max-tokens 丢弃决定（L135-L150）。
4. [packages/llm/llm/src/content.ts](../../packages/llm/llm/src/content.ts) —— 三类投影：`fileHandleText`（L151）、`projectFilesToText`（L185）、`projectImagesForTextModel`（L360）、`projectToolUpdates`（L424）。
5. [packages/llm/llm-deepseek/src/](../../packages/llm/llm-deepseek/src/) —— 流水线五步：`adapter.ts`（L20）、`serialize.ts`（L56）、`sse.ts`（L13）、`translate.ts`（L104）、`config.ts`（L82、L204、L290）；宿主注册 `host.ts`（L18）。
6. [packages/llm/llm-retry/src/index.ts](../../packages/llm/llm-retry/src/index.ts) —— `agent/request-error` 监听（L243）、先落盘再等待（L188-L190）、`llmRetry` 投影（L116-L121）。
7. [packages/llm/token-meter/src/index.ts](../../packages/llm/token-meter/src/index.ts) —— 重放式测量与三个投影注册（L114-L116）。
8. [packages/llm/llm/src/retry-policy.ts](../../packages/llm/llm/src/retry-policy.ts) —— 两种模式与默认值（L14-L24）、`resolveRetryPolicy`（L149）。
9. [packages/bundle/base/cordis.patch.yml](../../packages/bundle/base/cordis.patch.yml) —— llm 组的组装行（L34-L38、L91-L92、L127-L128）。

**延伸阅读**：

- **上一章**：[第5章 Agent-Loop——让模型转动起来的引擎](./第5章-Agent-Loop-让模型转动起来的引擎.md)。本章反复出现的 `preparedCall?.stream(request)`（agent.ts L436）就是那里的主角；循环如何在 `agent/request-error` 上把失败尝试与重试衔接起来，是这两章的接缝。
- **下一章**：[第7章 工具系统——能力接缝与执行管线](./第7章-工具系统-能力接缝与执行管线.md)。`ToolSchema` 从哪来、`tool-call` 块如何被派发成真实执行、`tool_result` 如何回流，都在下一章。
- **横向**：[第8章 消息系统](./第8章-消息系统-会话日志即模型记忆.md)（assistant 消息的持久形态与内嵌紧凑流）、[第9章 上下文工程](./第9章-上下文工程-一次请求里塞了什么.md)（`messages` 数组是怎么被组装满的）、[第10章 上下文压缩](./第10章-上下文压缩-当对话太长怎么办.md)（`token-meter` 的压力数据如何触发摘要）。
- **官方文档**：[docs/subsystems/llm-streaming.zh.md](../../docs/subsystems/llm-streaming.zh.md)（适配器契约的完整清单）、[docs/deepseek-llm-api-wire-extensions.md](../../docs/deepseek-llm-api-wire-extensions.md)（`dsh_` 扩展字段的 wire 契约）、[packages/llm/README.md](../../packages/llm/README.md)（组地图）。
