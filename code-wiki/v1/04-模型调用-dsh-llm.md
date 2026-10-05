# 04 模型调用 dsh-llm

> 官方参考：[../packages/llm/README.md](../packages/llm/README.md)、[../docs/subsystems/llm-streaming.md](../docs/subsystems/llm-streaming.md)、加适配器手册 [../docs/cookbook/adding-an-llm-adapter.md](../docs/cookbook/adding-an-llm-adapter.md)

## 1. llm 组包清单

| 包 | 角色 |
|---|---|
| [llm](../packages/llm/llm/)（`ctx.llm`） | Provider 中立的调用服务 `LlmRuntime`：适配器注册表 + 可拦截流式 API；定义全 harness 共用的消息/流/usage 词汇表 |
| [llm-deepseek](../packages/llm/llm-deepseek/) | DeepSeek Messages 协议完整实现（共享库） |
| [llm-deepseek-api-key](../packages/llm/llm-deepseek-api-key/) | 官方路由 `deepseek-official` 的 API-key 认证（`DEEPSEEK_API_KEY`） |
| [llm-deepseek-account](../packages/llm/llm-deepseek-account/) | 账户 token 路由认证（`x-dsh-auth-token`） |
| [llm-pi-ai](../packages/llm/llm-pi-ai/) | 基于 pi-ai 库的多 provider 适配器（OpenAI/Anthropic 兼容网关 + 三种 wire 协议手写路由） |
| [llm-retry](../packages/llm/llm-retry/) | 在 `agent/request-error` 扩展点按各 provider 策略重试 |
| [token-meter](../packages/llm/token-meter/)（`ctx.tokenMeter`） | 从持久日志重放测量 token 用量/上下文压力（零模型调用） |
| [deepseek-llm-api-extensions](../packages/llm/deepseek-llm-api-extensions/) | 官方请求顶层扩展字段注册表 |

## 2. 接缝：LlmRuntime 与 LlmAdapter

这是 capability seam 的标准样本（接缝模式详见 [05-工具系统](05-工具系统.md)）：

- **Service Definition**：`LlmRuntime`（[llm/src/index.ts](../packages/llm/llm/src/index.ts)），服务名 `ctx.llm`
- **Provider**：`DeepSeekAdapter`、`PiAiAdapter` 等注册进来的适配器
- **Consumer**：agent-loop、compaction 摘要器、session-title 生成器

`LlmRuntime` 的关键 API：

```ts
registerAdapter(providers: string[], adapter: LlmAdapter): AdapterRegistrationHandle  // handle.replace() 原子换路由
registerConfigurableProviders(entries)   // 声明可配置路由（settings 界面激活）
registerModelDiscovery(ns, discover)     // 端点模型探测
prepareCall(config, signal?): Promise<PreparedLlmCall>  // 一次性 dispatch、绑定适配器"代"
stream(options: GenerateOptions): AsyncIterable<StreamChunk>  // 经 'llm/stream' waterfall
```

`PreparedLlmCall` 只能 dispatch 一次（config deep-frozen；不匹配抛 `INVALID_PREPARED_CALL`），保证**日志与实际分发共享同一适配器代**——这是"模型可见 ⟺ 已日志"在调用侧的对应物。

**`LlmAdapter` 的唯一必需方法**是 `stream(options): AsyncIterable<StreamChunk>`；可选覆写 `providerInfo` / `providerRetryPolicy` / `listModels` / `prepareCall`（动态适配器应覆写以绑定代）等。

## 3. 请求构建：GenerateOptions

```ts
{ provider, model, reasoningEffort?, messages: RequestMessage[],
  system?, tools?: ToolSchema[], toolHistory?, temperature?, maxTokens?,
  stop?, signal?, sessionId?, purpose?: 'compaction' | 'session-title' }
```

- `ToolSchema` = `{ name, description, parameters, deferLoading? }`——发给模型的 JSON Schema，由 `ctx.tools.schemas()` 投影而来。
- 运行时边界做三类**投影**：`projectFilesToText`（FileBlock 永不原生发给 provider）、`projectImagesForTextModel`（纯文本路由的图片占位）、`projectToolUpdates`（按路由能力裁剪工具变更块）。
- Loop 构建的请求整体 deepFreeze；`llm/stream` waterfall 的监听器只可读。

## 4. 流式协议：StreamChunk 与装配

```
block-start → text-delta* / reasoning-delta* / tool-call-delta*
           → block-end → ... → usage → finish{reason}
```

- 协议顺序不变量：`usage` 先于 `finish`，`finish` 后无内容；工具参数保持模型原始 JSON 字符串。
- `BlockAssembler`（[assembler.ts](../packages/llm/llm/src/assembler.ts)）把 chunk 装配为 `ContentBlock`（text / reasoning / tool-call / file / image），容忍 delta-only 协议、忽略闭合后的迟到 delta。
- `TokenUsage`：`inputTokens`（未缓存输入）/ `outputTokens` / `cacheReadTokens` / `cacheWriteTokens` / `reasoningTokens`，三段输入不相交（计费输入 = 三者之和）。

## 5. DeepSeek Provider 内幕

[llm-deepseek](../packages/llm/llm-deepseek/src/) 的流水线：

1. **serialize.ts** `serialize(options, ...)`：映射为 Anthropic Messages 风格 wire 请求（`tool_use` / `thinking` / `tool_addition` / `tool_removal`，system 更新插在 user 轮后）。
2. **adapter.ts** `DeepSeekAdapter`：fetch 到 `{baseURL}/messages`，头含 `anthropic-version: 2023-06-01`、`x-api-key` 或 `x-dsh-auth-token`、`attributionHeaders()`、beta 头、会话 id；每请求生命周期由 `idleWatchdog` 管理。
3. **sse.ts** `parseSse()`：eventsource-parser 分帧 + JSON 校验（失败 → `MALFORMED_RESPONSE`）。
4. **translate.ts** `translate(events, model)`：Anthropic 事件 → harness StreamChunk；累计 usage；stopReason 映射（`tool_use→tool-calls`、`max_tokens→max-tokens`…）。

**配置来源**（全部 `Volatile` 支持热更，加载时 fail-loud 校验）：

| 配置 | 来源与默认 |
|---|---|
| `baseURL` | config → `DEEPSEEK_BASE_URL` → `https://api.deepseek.com/anthropic` |
| `reasoningEffort` | off/low/high/**high(默认)**/max |
| `maxTokens` | 默认 256,000 |
| `defaultContextWindow` | 默认 1,000,000 |
| `models` | advisory 目录，默认 `deepseek-flash` + `deepseek-v4-pro` |
| `streamIdleTimeoutMs` | 默认 300,000 |
| API key | `ctx.credentials.resolve(ref)` → 环境变量 `DEEPSEEK_API_KEY`；缺失 `MISSING_CREDENTIAL` |

`resolveAdapterOptions(config, environment)` 是显式的 resolve 步骤——默认值填充与边界钳制在拥有方实现里显式完成，不在 `run()` 里藏 `?? default`（仓库 request/spec 原则的模板之一）。

## 6. 重试、超时与错误归一

- **策略**：[retry-policy.ts](../packages/llm/llm/src/retry-policy.ts) `resolveRetryPolicy()`；默认 `maxRetries=5`、指数退避 500ms→10s、jitter 0.1，可重试码 `RATE_LIMIT / SERVER / TIMEOUT / TRANSPORT / EMPTY_RESPONSE`。
- **执行**：[llm-retry](../packages/llm/llm-retry/src/index.ts) 监听 `agent/request-error`，每次重试**先落盘再等待**（重试状态持久化为 session projection）；`llm-retry` 自身无配置。
- **超时**：`idleWatchdog`（空闲 5 分钟）+ Files API 60s。
- **归一**：`LlmRuntime.adapterStream` 是最终适配器边界——选择/分发/迭代失败统一转为终止 `finish` chunk；`LlmError extends HarnessError` 携带 `LlmFailure`（message/code/status/providerRetryAfterMs/requestId）。**服务本身从不重跑请求**——每次流是一次 provider 尝试，重试策略属于扩展点。

## 7. 新增 provider 的两条路径

1. **零代码**：用 `dsh-llm-pi-ai` 的 `providers` dict 配置任意 OpenAI/Anthropic 兼容网关（`api` + `baseURL` + `models` + `apiKeyEnv`）。
2. **写适配器插件**：继承 `LlmAdapter`，在 `apply()` 中 `ctx.llm.registerAdapter([route], adapter)`；模板见 [llm-deepseek/src/host.ts](../packages/llm/llm-deepseek/src/host.ts) 的 `registerDeepSeekProvider`（含热替换 retry policy）。每个 provider HTTP 请求必须带 `attributionHeaders()`。

逐步手册：[../docs/cookbook/adding-an-llm-adapter.md](../docs/cookbook/adding-an-llm-adapter.md)。

## 8. token-meter：确定性的用量测量

[token-meter](../packages/llm/token-meter/)（`ctx.tokenMeter`）从持久会话日志**重放**测量请求/上下文压力，零模型调用；提供 `tokenUsage` / `contextPressure` / `contextBreakdown` 三个 session projection。它是压缩触发（见 [09](09-上下文压缩.md)）与 Web 用量显示的数据源。

## 9. 关键文件索引

| 内容 | 路径 |
|---|---|
| `LlmRuntime` / `LlmAdapter` / `PreparedLlmCall` / `LlmError` | [packages/llm/llm/src/index.ts](../packages/llm/llm/src/index.ts) |
| `GenerateOptions` / `StreamChunk` / `TokenUsage` / `FinishReasonMap` | [packages/llm/llm/src/types.ts](../packages/llm/llm/src/types.ts) |
| `BlockAssembler` | [packages/llm/llm/src/assembler.ts](../packages/llm/llm/src/assembler.ts) |
| 消息构造器（`createAssistantMessage` / `createToolResultMessage`） | [packages/llm/llm/src/message.ts](../packages/llm/llm/src/message.ts) |
| 重试策略 | [packages/llm/llm/src/retry-policy.ts](../packages/llm/llm/src/retry-policy.ts) |
| DeepSeek 适配器 / 序列化 / SSE / 翻译 | [packages/llm/llm-deepseek/src/](../packages/llm/llm-deepseek/src/) |
| pi-ai 适配器 | [packages/llm/llm-pi-ai/src/adapter.ts](../packages/llm/llm-pi-ai/src/adapter.ts) |
| 重试执行器 | [packages/llm/llm-retry/src/index.ts](../packages/llm/llm-retry/src/index.ts) |
| 用量测量 | [packages/llm/token-meter/src/index.ts](../packages/llm/token-meter/src/index.ts) |
