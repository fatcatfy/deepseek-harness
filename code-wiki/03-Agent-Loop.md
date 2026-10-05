# 03 Agent Loop —— 让模型转动起来的引擎

> 官方参考：[../packages/core/agent-loop/README.md](../packages/core/agent-loop/README.md)、[../docs/agent-lifecycle.md](../docs/agent-lifecycle.md)（时序图）、[../docs/architecture.md](../docs/architecture.md) 的 Turn flow 节

## 1. 结构：契约与实现分离

Agent Loop 不是单一包，而是"契约包 + 实现包"分层，全部在 `packages/core/`：

| 包 | 角色 |
|---|---|
| [dsh-agent](../packages/core/agent/) | 公共契约：`Agent` 接口、`AgentRegistry`（`ctx.agents`）、全部 `agent/*` 事件声明 |
| [dsh-agent-loop](../packages/core/agent-loop/) | 唯一具体实现：`ReactLoopAgent` 驱动器 + `AgentLoop` 服务（工厂） |
| [dsh-session](../packages/core/session/) | 持久会话日志，循环的唯一事实来源 |
| [dsh-tools](../packages/core/tools/) | 工具注册表与执行管线 |
| [dsh-scope](../packages/core/scope/) | 作用域原语，实现 agent 级事件过滤 |

源码映射（agent-loop 包内）：

| 文件 | 职责 |
|---|---|
| [src/agent.ts](../packages/core/agent-loop/src/agent.ts) | 驱动器主体（约 690 行） |
| [src/index.ts](../packages/core/agent-loop/src/index.ts) | 插件入口、`AgentLoop` 服务、配置、create/resume |
| [src/inbox.ts](../packages/core/agent-loop/src/inbox.ts) | 持久收件箱 |
| [src/tool-calls.ts](../packages/core/agent-loop/src/tool-calls.ts) | 工具调度器 |
| [src/assistant-stream.ts](../packages/core/agent-loop/src/assistant-stream.ts) | 流式帧处理 |
| [src/runtime-context.ts](../packages/core/agent-loop/src/runtime-context.ts) | 系统提示/运行时上下文投影 |

## 2. 核心类与函数

### ReactLoopAgent（驱动器）

定义于 [agent.ts](../packages/core/agent-loop/src/agent.ts)，`export class ReactLoopAgent implements Agent`（约 L98）。内部状态机 `Phase`：`idle` / `maintenance` / `running`（各带 `AbortController`、turn/step 计数、wake 请求标记）。

| 方法 | 作用 |
|---|---|
| `kick()` | 驱动器边界：`while (await this.turn()) {}`；异常在此被包容，退出回归 idle 并重放被锁存的 wake |
| `turn()` | 一个完整 turn（返回是否继续下一个 turn） |
| `preStep()` | 步前决策：认领消息、组装提示词、跑 `agent/pre-step` waterfall |
| `step()` | 一次模型调用 + 工具执行，返回 StepEndReason 或 null（继续） |
| `prepareRequest()` | 解析请求配置并绑定适配器 |
| `buildRequest()` | 落日志 request/header 并构造冻结请求 |
| `followup/steer/inject/cancel` | 公开输入与取消入口 |

### AgentLoop（服务/工厂）

定义于 [index.ts](../packages/core/agent-loop/src/index.ts)，`class AgentLoop extends Service implements AgentFactory`，`static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt', 'sessionProjections']`。关键方法：`create()`、`resume()`、`createAgent()`。构造时注册两个 session projection（`turnBoundary`、`inbox`），并经 `ctx.agents.setFactory(this)` 成为 `ctx.agents` 背后的工厂。

### AgentRegistry（契约侧）

定义于 [dsh-agent/src/index.ts](../packages/core/agent/src/index.ts)。值得注意的方法是 `withInitiator()` / `requireInitiator()`——基于 `AsyncLocalStorage` 的因果归因：工具执行时用它取回"发起这个工具的 Agent"，子代理的工具结果因此能路由回正确的会话。

## 3. 输入路由：持久收件箱

Agent 的三个输入入口语义不同：

| 入口 | 行为 |
|---|---|
| `followup(msg)` | 排队**下一个 turn** |
| `steer(msg)` | 注入**下一个 step 边界**的转向 |
| `inject(msg)` | 只排队不唤醒（等待唤醒消息） |

收件箱**不是内存数组**：`ReactLoopInbox`（[inbox.ts](../packages/core/agent-loop/src/inbox.ts)）的每次 splice 都先落持久事件 `agent/inbox/spliced`，`inboxProjectionDefinition` 把事件折叠成 `{ 'next-turn': [...], 'next-step': [...] }` 状态——重启后收件箱从日志重建。

## 4. 三层循环详解

### 4.1 turn()

1. `session.append('turn/start', { turn })`
2. 步循环 `while (true)`：
   - `preStep()`：收件箱**原子认领**（清空 next-step + 首步取一条 next-turn）→ `systemPrompt.assemble()` 组装 sections/contexts/tools → 运行时上下文投影 → `agent/pre-step` waterfall（插件可 `reject` 或改写 `enter` 消息）
   - 被拒绝 → turn 以 `blocked` 收束；首步消息为空 → 直接 `completed` 关 turn，**不花模型调用**
   - `session.append('step/start')` → `step(decision)`
3. 步结束且无新输入时：`agent/turn-stopping`（serial）——监听器此刻可 `agent.steer(...)` 反对关闭
4. `finally` 中 `session.append('turn/end', { reason })` —— turn/end **必落日志**
5. 收件箱仍有 pending → 换新 AbortController，进入下一个 turn

### 4.2 step()（一次请求尝试）

```
waterfall('agent/request') ──── 插件可换 provider/model/effort
   └▶ llm.prepareCall() ─────── 绑定适配器"代"，一次性的 PreparedLlmCall
systemPrompt.project() ──────── 提示词准入（按路由能力决定 append 还是归一化）
落日志：system/message、user/message(仅首次尝试)、request/header、request/context
deriveMessages() ───────────── 从日志派生模型历史
markAgentLoopRequest(冻结) ─── 请求是日志的纯函数，llm/stream 监听器只读
llm.stream(request) ────────── 流式消费 → agent/assistant-stream 帧序列
   结算：
   - error/aborted → assistant/attempt 落日志 → agent/request-error waterfall
       └─ 监听器返回 {kind:'retry'} → 重试（复用同一组装，不重跑 pre-step）
   - 正常 → assistant/message（内嵌完整流 + usage）落日志 ← 派生历史的锚点
   - 无 tool-call 块 → completed；max-tokens → max-tokens
   - 有 tool-call → executeToolCalls() → null（继续下一步）
```

关键不变量：

- **请求是日志的纯函数**：`messages = session.deriveMessages()` 逐条 deepFreeze；重试不重复组装。
- **提示词以 history 旅行**：渲染后的 system prompt 是 `system/message` 事件（surface 节点 0），不是请求头字段；空渲染会清空所有活动 system 节点。
- **`agent/request` 不能改消息**——一切模型可见内容必须走日志通道（"model-visible ⟺ logged"）。

### 4.3 工具调度（executeToolCalls）

[tool-calls.ts](../packages/core/agent-loop/src/tool-calls.ts) 的 `executeToolCalls()`：

1. 从 `ctx.agents.requireInitiator()` 取回发起 Agent；每个调用构成 `ToolExecutionInput { callId, name, arguments, agent, signal }`（arguments 解析 JSON，坏 JSON 保留原文本）
2. 按 `ctx.tools.executionMode()` 分组：`isConcurrencySafe(args) === true` → 并行（默认上限 10，`maxParallelToolCalls` 配置），否则独占屏障
3. 每个调用：`tool/call` 落日志 → `tools/pre-execute` → `tools/execute` → 工具体 → `tools/post-execute` → `tool/result` 落日志（`sourceEventSeqs` 引用 call 事件）
4. `result.additionalContexts` 回填进 next-step 收件箱供下一 step；`result.concludesTurn === true` 可提前收束 turn

管线五阶段的完整展开见 [05-工具系统](05-工具系统.md)。

## 5. 停止判定

| 层 | 词汇 | 取值 |
|---|---|---|
| 适配器 finish | `FinishReasonMap`（[dsh-llm types.ts](../packages/llm/llm/src/types.ts)） | `stop` / `tool-calls` / `max-tokens` / `aborted` / `error` |
| Step | StepEndReason | `completed` / `max-tokens` / null（继续） |
| Turn | `TurnEndReasonMap`（[session types.ts](../packages/core/session/src/types.ts)，merge-extensible） | `completed` / `aborted(reason)` / `blocked` / `error` / `max-tokens` / `interrupted`(仅崩溃恢复合成) / `forked`(仅 fork 种子合成) |

终止的核心信号是"**模型不再欠答复**"：无活动 tool-call 且收件箱无新 steering，`agent/turn-stopping` 跑完即关 turn。循环没有内建 turn 预算——限制失控 turn 的策略从 `agent/turn-stopping` 等扩展点取消（README 的 Known Limitations 明示）。

## 6. 中断、取消与错误恢复

这是 agent-loop 设计最密的部分：

- **取消入口** `cancel(cause)`：cause ∈ `user` / `parent` / `hook` / `disposed`；默认清空收件箱再 abort。运行中的 wake 请求被锁存，活动收敛后重放；abort 后到达的输入改道 next-turn。
- **流中断的持久化**：若流已开始且 signal aborted，把已送达的安全前缀落成 `assistant/message` + `interrupted: true`——下一请求包含用户已见内容；否则落 `assistant/attempt`（保留原始流，不进模型历史）。
- **工具中断**：未启动的调用合成 `ABORTED_BEFORE_DISPATCH` 错误结果（保证重放有效）；已启动的先排干再按序提交。
- **步失败恢复**：`ToolCallRecovery`（[repair.ts](../packages/core/session/src/repair.ts)）为每个未答复的 call 落保守错误结果——有记录的用 `TOOL_OUTCOME_UNKNOWN`（"outcome is unknown… 只读/幂等才可重试"），无记录的用 `TOOL_NOT_STARTED`。
- **崩溃恢复（resume）**：`AgentLoop.resumeWith` 冷读日志后，`interruptedTurnClosers()` 为中断的尾部 turn 合成缺失工具结果 + `turn/end {kind:'interrupted'}`，作为普通批次 append——恢复不需要专门的"恢复模式"。

## 7. 循环的全部扩展点（"Plugins, not loop changes"）

循环实现不 export 任何钩子类型给消费者；一切干预面是 [dsh-agent](../packages/core/agent/src/runtime-types.ts) 声明的事件表，配合 dsh-scope 的 scope 过滤分发（子代理只见自己的事件）：

| 事件 | 模式 | 语义 |
|---|---|---|
| `agent/created` | serial | 创建后按序初始化，失败回滚创建 |
| `agent/pre-step` | **waterfall** | 拒绝或改写进入 step 的消息；可声明 `startsRequestSeries` |
| `agent/request` | **waterfall** | 替换冻结前的调用配置（provider/model/effort/maxTokens） |
| `agent/request-error` | **waterfall** | 返回 `{kind:'retry'}` 接管恢复，或 `next()` 委托 |
| `agent/turn-stopping` | serial | turn 关闭前最后一问；监听器可 steer 续命 |
| `agent/assistant-stream` | emit | 进程内流帧（start/chunk/end），供 UI |
| `agent/status` / `agent/error` / `agent/inbox/*` / `agent/disposed` | emit | 观测 |

**原则的证明**：仓库约一半子系统是这些事件的纯消费者，全部不改循环——`llm-retry`（request-error）、`compaction-basic`（pre-step + 维护）、`plan-mode`（pre-step）、`goal-round-driver`（自动续轮）、`session-checkpoint-policy`（llm/stream + tools/execute 前落盘）等。换驱动器只需再实现一个 `AgentFactory`。

## 8. 与任务结构插件的交互

todo / plan / goal 三者都是纯插件消费者：

- **todo**（[tool-todo](../packages/todo/tool-todo/src/index.ts)）：`todo_write` 工具 + `todos` session projection；工具 execute 内直接 `session.append('todo/write')`——工具结果与状态更新同事务落日志。
- **plan**（[plan-mode](../packages/plan/plan-mode/src/index.ts)）：`agent/pre-step` waterfall 写日志-only 的 `plan/mode` 事件；`plan:policy` 提示词节按模式条件渲染；`exit_plan_mode` 工具；离开模式的叙述经 `agent.inject` 注入。
- **goal**（[goal](../packages/goal/goal/src/index.ts) + goal-round-driver）：goal 状态机全部落 `goal/*` 会话事件；round-driver 在 Agent 空闲且无竞争输入时 `agent.followup()` 投递续轮消息，pre-step 校验认领的 round 仍有效（无效则 reject 并归还消息）。

## 9. 数据流一图流

```
followup/steer/inject
   └→ Inbox.splice ──append('agent/inbox/spliced')──▶ Session log
wakeDriver → kick → turn()
   turn/start ─▶ preStep: inbox.claim ─ systemPrompt.assemble ─ agent/pre-step
   step/start ─▶ agent/request ─ llm.prepareCall ─ system/message ─ user/message
                 deriveMessages → 冻结请求 → llm.stream
                 assistant/message(+usage+内嵌流)  ← 派生历史锚点
                 tool/call → pre-execute → execute → post-execute → tool/result
                 additionalContexts → next-step inbox
   agent/turn-stopping → turn/end
回到 turn()，直至收件箱清空 → idle
```

## 10. 关键文件索引

| 内容 | 路径 |
|---|---|
| 驱动器 `ReactLoopAgent` | [packages/core/agent-loop/src/agent.ts](../packages/core/agent-loop/src/agent.ts) |
| 服务/工厂 `AgentLoop` | [packages/core/agent-loop/src/index.ts](../packages/core/agent-loop/src/index.ts) |
| `Agent` 契约 + `agent/*` 事件 | [packages/core/agent/src/runtime-types.ts](../packages/core/agent/src/runtime-types.ts) |
| `AgentRegistry` + initiator 归因 | [packages/core/agent/src/index.ts](../packages/core/agent/src/index.ts) |
| 持久收件箱 | [packages/core/agent-loop/src/inbox.ts](../packages/core/agent-loop/src/inbox.ts) |
| 工具调度 | [packages/core/agent-loop/src/tool-calls.ts](../packages/core/agent-loop/src/tool-calls.ts) |
| 失败恢复 | [packages/core/session/src/repair.ts](../packages/core/session/src/repair.ts) |
| 循环级守卫 | [packages/guard/timeout-policy](../packages/guard/timeout-policy/src/index.ts)、[repeat-tool-reminder](../packages/guard/repeat-tool-reminder/src/index.ts) |
