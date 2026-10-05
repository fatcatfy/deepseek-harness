# 第5章 Agent Loop：让模型转动起来的引擎

## 1. 本章导读

上一章（[第4章](./第4章-事件驱动-五路派发与三个事件域.md)）我们站在事件总线的一侧，认识了五种派发模式与三个事件域。这一章我们穿过总线，站到敲钟人的位置：那个在固定节奏上派发 `agent/pre-step`、`agent/request`、`agent/request-error`、`agent/turn-stopping` 的机器本体——**Agent Loop（代理循环）**。这是全书最核心的一章：CLI、Web、Desktop、SDK 的一切"智能"行为，都是这台发动机一圈一圈转出来的。

学完本章，你应该能回答四个问题：

1. **turn（轮次）**与 **step（步）**分别是什么？为什么需要两层嵌套？
2. 从 `turn/start` 到 `turn/end`，一个轮次内部按什么顺序发生什么？
3. 想干预这台机器的插件，能拦在哪四个瀑布（waterfall）拦截点上？各自能改什么、不能改什么？
4. 循环怎么退出？取消、失败、崩溃之后，数据是怎么保住的？

本章地图：第 2 节从一个无状态模型的悖论出发；第 3 节是正文，依次讲契约分包、骨架总图、输入路由、请求组装、停止判定、拦截点、失败恢复；第 4 节按"七站"逐段走读源码；第 5 节拆五个常见误区；第 6-8 节是动手环节、小结与延伸阅读。

## 2. 从一个问题出发

你用过 ChatGPT，也知道底层真相：**模型 API 是一个无状态函数**——传进去一段消息数组，吐出来一段回复，函数自己什么都不记得。那么问题来了：

- 它怎么"记得"上一轮你们说了什么？
- 它调用工具之后，怎么知道工具返回了什么？
- 它怎么决定"继续调工具"还是"直接回答"，循环到什么时候停？

答案：模型什么都没做，**外面有一台发动机**。每一圈：把全部历史喂给模型 → 拿到回复 → 如果回复里有工具调用就执行、把结果追加进历史 → 再喂。这个圈就是 agent loop。学术界给这个范式起过名字——ReAct（Reason + Act，先推理后行动），DSH 实现里的 `ReactLoopAgent` 类名正是致敬它。这个循环的起点是一个任何大学生半小时就能写完的玩具：

```ts
// 教学玩具：最小 agent loop
while (true) {
  const reply = await callModel(messages)      // 喂历史
  messages.push(reply)
  if (reply.toolCalls.length === 0) break      // 没有工具调用 → 停
  for (const call of reply.toolCalls) {
    messages.push(await runTool(call))         // 执行工具，结果进历史
  }
}
```

十五行，能跑。现在数一数它缺什么：

1. **对话存在内存里**——进程一退全没；
2. **用户中途插话**——只能干等整圈转完；
3. **取消**——回复打到一半，用户已经看到的文字怎么办？
4. **请求失败**——要不要重试？谁说了算？
5. **工具并发**——十个读文件请求也要一个个排队吗？
6. **进程崩溃**——重启后接得上吗？
7. 换模型、审计、压缩、限流……每一项都想往循环里塞代码，而循环恰恰是整个 harness 最不该被随手改的代码。

DSH 的回答分两半：**机械的一半**写进 [agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts)（约 690 行），**策略的一半全部赶出循环**，变成第 4 章讲过的监听器（listener）。这个分工是本章的主线，我们先看它落在怎样的包结构上。

## 3. 正文

### 3.1 契约与实现分离：dsh-agent 与 dsh-agent-loop

DSH 把"什么是 Agent"和"Agent 怎么转"拆成两个包：

- [packages/core/agent](../../packages/core/agent/src/index.ts) 是**契约包**：`Agent` 接口（在 [runtime-types.ts](../../packages/core/agent/src/runtime-types.ts) L163-L243 通过接口合并声明进 [types.ts](../../packages/core/agent/src/types.ts)）、`AgentRegistry` 服务（即 `ctx.agents`，index.ts L245）、以及整张 `agent/*` 事件表（runtime-types.ts L245-L394，每条事件都注明 `@mode` 与 Scope-filtered dispatch）。它不含任何驱动逻辑。
- [packages/core/agent-loop](../../packages/core/agent-loop/README.md) 是**唯一实现**：`ReactLoopAgent` 驱动器（driver，agent.ts L98）、`AgentLoop` 服务（[index.ts](../../packages/core/agent-loop/src/index.ts) L330，`static inject = ['agents','sessions','llm','tools','systemPrompt','sessionProjections']`）。

`AgentLoop` 在构造时做三件事（index.ts L364-L369）：注册 `turnBoundary` 与 `inbox` 两个会话投影（projection），然后 `ctx.agents.setFactory(this)` 把自己登记为工厂。此后消费者只认 `ctx.agents.create()` / `resume()`，从不 import 实现包——所以[核心子系统文档](../../docs/subsystems/core.zh.md)明说：扩展插件依赖 `agent` 而绝不依赖 `agent-loop`，**循环保持可替换**。

`AgentLoop` 服务自己管两摊事。**工厂摊**：`create()`（L652）在调用方提供的 SessionId 下建新会话与新 Agent；`resume()`（L807）经持久化后端加载已有会话；两者都返回带 `dispose()` 的 `AgentHandle`——拆解（排空驱动器 → 注销 → 关写句柄）被记忆化，多个竞争的所有者等待同一个收敛点。**配置摊**：cordis.yml 里声明的 agent 条目在插件启动时自动 create 或 resume（构造函数 L374-L403），最小组合里你一行代码不写也能让一个 Agent 上线。

契约包里还有个容易忽略的角色：`AgentRegistry` 的**发起者（initiator）作用域**。它基于 Node 的 `AsyncLocalStorage` 实现 `withInitiator()` / `requireInitiator()`（index.ts L308、L327）——`wakeDriver()` 启动驱动器时用 `withInitiator(this, ...)` 包住整个 `kick()`（agent.ts L234），于是驱动链上任何异步代码（包括工具执行）都能取回"是哪个 Agent 发起的"。第 3.8 节会看到工具调度如何依赖它把子代理结果路由回正确的会话。

### 3.2 骨架总图：一台两层嵌套的发动机

先把全图铺开。这一张图是本章的地图，后面每一节都是它的某个局部放大：

```text
kick()
 └─ while (await this.turn())                 ← 驱动器边界：异常在此被包容
     │
     ├─ turn N（轮次）
     │   ├─ session.append('turn/start')
     │   │
     │   ├─ step k = 1..m（步循环，while true）
     │   │   ├─ preStep()：inbox.claim → 组装提示词 → agent/pre-step 瀑布
     │   │   │    └─ reject → turnEnds = blocked（直接进入收尾）
     │   │   ├─ session.append('step/start') + 挂 ToolCallRecovery 观察器
     │   │   ├─ step()                         ← 一次尝试 = 一次模型请求
     │   │   │   ├─ prepareRequest()：agent/request 瀑布 → llm.prepareCall
     │   │   │   ├─ 提示词准入 + 落日志（system / user / header / context）
     │   │   │   ├─ buildRequest()：deriveMessages() + deepFreeze
     │   │   │   ├─ 流式消费：AssistantStreamAttempt（start / chunk / end 帧）
     │   │   │   └─ 结算：
     │   │   │        error / aborted → assistant/attempt → agent/request-error
     │   │   │                                              └─ retry? → 同步重试
     │   │   │        正常       → assistant/message（派生历史锚点，内嵌完整流）
     │   │   │        max-tokens → {kind:'max-tokens'}
     │   │   │        无 tool-call → {kind:'completed'}
     │   │   │        有 tool-call → executeToolCalls() → null（继续下一步）
     │   │   ├─ session.append('step/end')（finally，必落）
     │   │   └─ 步已结束且 next-step 收件箱为空 → agent/turn-stopping → break
     │   │      否则 target = 'next-step' → 下一个 step
     │   │
     │   └─ finally：session.append('turn/end', {reason})（必落）
     │
     └─ 收件箱仍有 pending → 新 AbortController、step 归零 → 下一个 turn
        否则 → idle
```

读懂这张图的关键，是记住一个贯穿始终的不变量：**这台机器在内存里几乎不保存任何对话状态**。它唯一的持久状态是会话日志（session log，第 8 章整章展开）——每次请求的消息都是从日志现场派生的（`session.deriveMessages()`），每个模型可见的事实（用户消息、助手消息、工具结果）都是先落日志、再进请求。仓库规则"Model-visible ⟺ logged"（模型可见的内容必须来自日志）在这台机器上被机械地执行。

这张图也印证了[核心子系统文档](../../docs/subsystems/core.zh.md)的一句话总结——一个轮次流经六个包：driver 在 `agent-loop` 里认领排队的提示词，在会话日志（`ctx.sessions`）上开启轮次，经 `system-prompt`（`ctx.systemPrompt`）组装请求前缀并从日志派生历史，经 LLM 接缝（`ctx.llm`）流式获取模型响应，经工具注册表（`ctx.tools`）分发工具调用，再把每个模型可见的事实追加回日志，供下一步派生。循环是六包的总调度，但对每个包只持有接口——换掉任何一个实现，循环不动。

读这张图有个诀窍：先找**必落的三个事件**（`turn/start`、`step/end`、`turn/end` 都在 try/finally 的保证下），再找**四个拦截点**（`agent/pre-step`、`agent/request`、`agent/request-error` 三个瀑布，加 `agent/turn-stopping` 这个 serial 检查点），最后找**两个循环**（turn 循环与 step 内的重试循环）。抓住这三类锚点，读任何一段源码你都能立刻定位自己在图的哪一格。

### 3.3 turn 与 step：两个嵌套的时间单位

**turn（轮次）**是对用户的一次完整交代：从一条用户输入（或一条续轮输入）开始，到模型给出"不再欠答复"的最终响应为止。**step（步）**是 turn 内部的一次"请求—响应—工具"循环：一次模型请求加上它触发的工具执行。一个 turn 通常含多个 step：第 1 步模型说"我要读文件"，第 2 步读完文件继续推理，第 3 步给出答案——turn 才结束。

为什么要两层？因为**用户与模型的节奏不同**。用户按 turn 说话（一条 followup 一个 turn），模型按 step 消耗工具。两层嵌套让"用户插话"有一个天然锚点（下一个 step 边界），让"这轮结束了吗"有一个统一判定处（turn 收尾）。

驱动器内部是一个三态的**相位（phase）**状态机（agent.ts L42-L50）：

```ts
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | { kind: 'maintenance'; abort: AbortController; lastTurn: number; wakeRequested: boolean }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

- `idle`：没有驱动器在转；
- `running`：驱动器持有当前的 turn / step 计数与一个 `AbortController`（取消的载体）；
- `maintenance`：机器空闲时插入的非轮次任务（如压缩维护）——期间到来的唤醒不丢失，记在 `wakeRequested` 上，任务收敛后重放。

相位转移一览：

| 转移 | 触发 |
|---|---|
| idle → running | 唤醒输入开新驱动器（`wakeDriver` 的 idle 分支） |
| running → idle | `kick()` 边界收敛（含锁存唤醒的重放） |
| idle → maintenance | `runMaintenance()` 认领真空闲 |
| maintenance → idle | 任务收敛（锁存的唤醒此刻重放） |
| running → running | 收件箱仍有 pending：换新 AbortController、step 归零，进下一个 turn |

驱动器边界是 `kick()`（agent.ts L252）：`while (await this.turn()) {}`——`turn()` 返回 `true` 表示收件箱还有活干、继续下一轮；任何异常（哪怕是取消）都在这里被**包容**（contained）：`throwError()` 已在出错现场派发过 `agent/error`，`kick` 只负责让机器体面地回到 idle，不让异常炸穿到唤醒它的调用方。它短得值得整段读：

```ts
// agent.ts（节选，L252-L265）
private async kick(): Promise<void> {
  try {
    while (await this.turn()) {}
  } catch (_error) {
    // 已上报的失败与取消在驱动器边界被包容
  } finally {
    if (this.phase.kind === 'running') {
      const { turn, wakeRequested } = this.phase
      this.setPhase({ kind: 'idle', lastTurn: turn })
      if (wakeRequested && this.inbox.hasPending) this.wakeDriver()   // 重放锁存的唤醒
    }
  }
}
```

`maintenance` 相位值得多说一句：压缩这类维护任务必须在"真空闲"里跑（`compaction-basic` 就通过 `runMaintenance` 执行），期间到来的唤醒输入不丢弃而是锁存在 `wakeRequested` 上，任务收敛后重放；对外 status 保持 `idle`，UI 不会误显示转圈。而 `whenIdle()`（L237）用一个 do-while 循环追踪 `activityDone`——即便你观察的驱动器刚退役、新的工作立刻顶上，它也能跟到真正安静的那一刻。

### 3.4 输入从哪来：三个入口与一个持久收件箱

用户（或插件）向 Agent 说话有三个入口，语义各不相同（agent.ts L163-L173）：

| 方法 | 等价 `send()` 调用 | 语义 | 被消费的时机 |
|---|---|---|---|
| `followup(msg)` | `send(msg, 'next-turn', true)` | 排队下一个 turn：这条消息独占一个新轮次 | 轮次边界 |
| `steer(msg)` | `send(msg, 'next-step', true)` | 转向（steering）：注入下一个 step 边界，必要时唤醒驱动器 | 下一个步边界 |
| `inject(msg)` | `send(msg, 'next-step', false)` | 只排队**不唤醒**：机器空闲时就静静躺着 | 下一个步边界 |

三者的差别在"粒度"与"是否唤醒"。你让 Agent"帮我重构这个模块"，中途发现它方向错了——`steer("先别动测试文件")` 会在下一个步边界插进对话，而 `followup` 则要等本轮彻底结束。`inject` 则是给"被动补充上下文"用的：工具结果的 `additionalContexts` 就走这条路。

关键在于：**收件箱（inbox）不是内存数组**。`ReactLoopInbox`（[inbox.ts](../../packages/core/agent-loop/src/inbox.ts) L74）的每一次插入、删除、认领，都先 `session.append('agent/inbox/spliced', ...)` 落成日志事件（L235），再由 `inboxProjectionDefinition`（L27）把事件流折叠成 `{'next-turn': [...], 'next-step': [...]}` 的投影状态。也就是说：重启进程后，收件箱从日志**重建**——用户排队但还没被消费的消息不会因为崩溃而丢失。第 4 章"Session 事件是持久事实"的判据，在这里变成了具体机制。

`claim()`（L109）是循环取货的原子操作：一次认领会**清空整个 next-step 列表**；若这次认领发生在轮次边界（`target === 'next-turn'`），再**取一条** next-turn 消息拼在后面。认领是纯删除 splice（不记 `outcome`），随后逐条发出 `agent/inbox/claimed` 通知。

三个入口最终都归一到 `send()`，它的完整逻辑值得看原文——特别是"唤醒输入不能加入已中止的活动"这条改道规则：

```ts
// agent.ts（节选，L154-L161）
send(message: UserMessage, target: InboxTarget, wakeup: boolean): void {
  // 唤醒输入不能加入已中止的活动，只能排队下一个 turn
  const wakingAfterAbort = wakeup && this.phase.kind !== 'idle' && this.phase.abort.signal.aborted
  const resolvedTarget = wakingAfterAbort ? 'next-turn' : target
  this.inbox.splice(resolvedTarget, Infinity, 0, [message])
  if (wakeup) this.wakeDriver(wakingAfterAbort)
}
```

还有一个初学者容易踩的语义坑：`inject` 不保证赶上"下一个"请求——若注入到达时 pre-step 已经认领了本步的批次，这条上下文要等到**再下一个**步边界（[Agent 接口文档](../../packages/core/agent/src/runtime-types.ts) L233-L241 明说了 "It may miss a request whose pre-step already claimed its batch"）。需要精确时机就用 `steer`。

### 3.5 一次请求是如何组装的：日志的纯函数

`step()`（agent.ts L398）内部的 `while (true)` 每迭代一次就是**一次请求尝试（attempt）**。一次尝试的前半场做四件事。

**第一，解析配置**——`prepareRequest()`（L547）。种子配置来自两处之一：实例的第一个请求用 `AgentOptions`（provider / model / reasoningEffort / maxTokens）；之后的请求复用上一条日志 `request/header` 里记录的配置，但先经 `requestProposal()`（L64-L70）**剥离适配器默认值**——上次由适配器（adapter）填进来的 effort / maxTokens 不算数，让新路由重新解析。种子配置的来源切换看原文：

```ts
// agent.ts（节选，L566-L575）
const seedConfig = deepFreeze(structuredClone(
  this.requestHeaderLogged
    ? requestProposal(persistedHeader!)    // 复用上次 header，但剥离适配器默认值
    : {
      ...route,                            // AgentOptions 的 provider / model
      ...reasoningEffort === undefined ? {} : { reasoningEffort },
      ...maxTokens === undefined ? {} : { maxTokens },
    },
))
```

为什么坚持剥离适配器默认值？因为"上次适配器帮你填的 effort"不代表"这次换了模型还该用同一个"——默认值必须每次由当前路由重新解析，显式设置才会延续。然后是第一个瀑布拦截点 `agent/request`（L576）：插件可以换 provider、换模型、换 effort，但**不能改消息**（模型可见内容必须走日志通道，这是事件声明里写死的约束）。最后 `loopCtx.llm.prepareCall(proposedConfig, signal)`（L587）返回 `PreparedLlmCall`（[llm/index.ts](../../packages/llm/llm/src/index.ts) L169）——一个**一次性**的预备调用：配置与适配器注册绑定解析完毕，携带 `adapterDefaults` 与 `retryPolicy`，只能 `stream()` 一次。

**第二，提示词准入**——`systemPrompt.project()`（L410，实现在 [runtime-context.ts](../../packages/core/agent-loop/src/runtime-context.ts) L88）。组装好的系统提示词按路由能力决定进日志的方式：支持 in-history 更新的路由追加一个新的 `system/message` 节点（保住前缀缓存），不支持的路由则把内容归一化回头节点。这里你能再次看到"一切先落日志"：提示词不是请求的字段，而是日志里的 `system/message` 事件。

`project()` 的三分支决策（runtime-context.ts L88-L103）值得展开：没有头节点 → 追加（哪怕是空提示词也要占住 node 0）；路由支持 in-history 更新、系列连续、文本变了 → 在已提交的用户消息**后面**追加新节点——前缀缓存因此得以复用；其余情形（路由不支持、系列断了、提示词清空）→ 归一化：把后续非空 system 节点逐个替换为空、头节点重写为新文本。为什么如此讲究？因为提示词落在历史的哪个位置，直接决定 provider 的前缀缓存（KV cache）从哪个 token 开始失效——这个话题留给[第6章](./第6章-模型调用-llm能力接缝与适配器.md)与[第10章](./第10章-上下文压缩-当对话太长怎么办.md)。

**第三，落日志**。按顺序：`system/message`（L417）、仅首次尝试追加认领的 `user/message` 批（L421，重试不重复认领）、`request/header`（L618-L634）、工具增减时一条 `developer/message`（L643，携带 `tool-addition` / `tool-removal` 块）、`request/context`（L665，仅在 provider / model / contextWindow / systemPromptUpdate 变化时）。`request/header` 的四种 reason 值得记牢，重放与调试时全靠它区分请求系列：

| reason | 触发条件 |
|---|---|
| `initial` | 本实例第一条 header（日志里还没有 baseline） |
| `resume` | 恢复实例的第一条 header（日志里已有历史 baseline） |
| `change` | 与 baseline 相比 config 或 tools 变了 |
| `series` | header 没变，但开启了新的消息系列（`startsSeries`） |

**第四，组装请求**——`buildRequest()`（L599）。这是全章最值得咀嚼的一步：

```ts
// agent.ts（节选，L671-L685）
const boundaryMessages = session.deriveMessages()   // 从日志现场派生
for (const message of boundaryMessages) { /* deepFreeze ... */ }
const request = markAgentLoopRequest(Object.freeze({
  ...header.config,
  messages: boundaryMessages,
  toolHistory: session.toolHistory(),
  ...header.tools !== undefined ? { tools: header.tools } : {},
  sessionId: this.session.id,
  signal,
}))
```

请求是**日志的纯函数**：同样的日志前缀，必然组装出同样的请求（消息逐条深冻结，防下游监听器偷改）。`markAgentLoopRequest` 打上进程本地标记，`llm/stream` 的监听器据此识别并只读。重试**不重新组装、不重跑 pre-step**——同一份渲染产物直接再试一次。

### 3.6 停止判定：模型何时不再欠答复

停止判定的核心直觉是一句"欠账"逻辑：**模型不再欠答复（the model owes no response），turn 才能结束**。只要历史里还有没兑现的工具调用、还有没消费的转向输入，模型就仍欠着答案，机器就得再转一步。

判定分两层。**步层**看适配器的结束原因（finish reason）——`FinishReasonMap`（[llm/types.ts](../../packages/llm/llm/src/types.ts) L156-L162）：`stop` / `tool-calls` / `max-tokens` / `aborted` / `error`。`step()` 的返回值 `StepEndReason | null`（agent.ts L52 只允许 `completed` 与 `max-tokens` 两种）按此推导：

- 回复里没有工具调用块，或全部调用都已 `concludesTurn` → `{kind: 'completed'}`（L533）；
- 撞上输出 token 上限 → `{kind: 'max-tokens'}`（L530）；
- 还有未完结的工具调用 → `null`，步循环继续。

**轮次层**是 `TurnEndReasonMap`（[session/src/types.ts](../../packages/core/session/src/types.ts) L201-L229，可合并扩展）。步层的返回类型把"欠账与否"直接写进了签名：

```ts
// agent.ts（节选，L52）
type StepEndReason = Extract<TurnEndReason, { kind: 'completed' | 'max-tokens' }>
// step() 返回 StepEndReason | null：null = 模型还欠答复，步循环继续
```

| kind | 谁写入 | 语义 |
|---|---|---|
| `completed` | 循环 | 模型不再欠答复 |
| `aborted(reason)` | 循环 | 被取消；reason 是 JSON 安全的取消原因 |
| `blocked` | 循环 | pre-step 决策拒绝 |
| `error(LlmFailure)` | 循环 | 失败；`LlmError` 保留事实，其他错误拍平为 `{message, code:'UNKNOWN'}` |
| `max-tokens` | 循环 | 至少一步撞上输出上限（粘性，见下） |
| `interrupted` | 仅崩溃恢复合成 | resume 时为中断的尾部 turn 补写 |
| `forked` | 仅 fork 种子合成 | 分叉（fork）种子切断源会话的开放 turn |

两个精妙细节。其一，**max-tokens 是粘性（sticky）的**（agent.ts L336-L341）：第 1 步撞上限、第 2 步被插件续命后正常结束，turn 的最终 reason 仍是 `max-tokens` 而非 `completed`——因为第 1 步被截断的那次回答模型还欠着。其二，`agent/turn-stopping`（L359-L360）给插件最后一次反对关闭的机会：步已结束且收件箱为空时，serial 派发该事件，监听器此刻 `agent.steer(...)` 塞进新输入，机器**重读收件箱**——有货就再跑一步，没货才关 turn。数据决定结果，监听器顺序无关（第 4 章讲过的 serial 否决通道）。这两行代码把"检查—反对—重读"焊在一起：

```ts
// agent.ts（节选，L359-L364）
if (turnEnds && this.inbox.nextStep.length === 0) {
  await this.dispatch.serial('agent/turn-stopping', { turn, signal })   // 监听器可在此 steer
  signal.throwIfAborted()
}
if (turnEnds && this.inbox.nextStep.length === 0) break                  // 重读收件箱再决定
```

另一个容易误判的情形：**工具续步的空批次**。步返回 `null`（还有未完结的工具调用）时，下一步认领的 next-step 批可以为空——这个步仍然花一次模型调用，因为模型要"读"工具结果。判定停止的依据从来不是"有没有新输入"，而是"模型是否还欠答复"。

还要注意：循环**没有内建的 turn 预算**——[README 的 Known Limitations](../../packages/core/agent-loop/README.md) 明说，限制失控 turn 的策略必须从 `agent/turn-stopping` 这类扩展点取消，循环本身不预设策略。

### 3.7 四个瀑布拦截点与 "Plugins, not loop changes"

现在正式回答第 2 节埋下的问题：策略住在哪？答案是有四个拦截点（第 4 章讲过：waterfall 监听器层层包装内建行为，拥有决策者可不调 `next()` 短路）：

| 拦截点 | 声明位置 | 挂载时机 | 能做什么 |
|---|---|---|---|
| `agent/pre-step` | runtime-types.ts L320 | 每步认领消息后、组装请求前 | 拒绝整步（`{kind:'reject'}`）；改写进入消息；声明 `startsRequestSeries` |
| `agent/request` | 同上 L337 | 每次请求尝试前 | 替换调用配置（provider / model / effort…），**不能改消息** |
| `agent/request-error` | 同上 L353 | 请求失败、`assistant/attempt` 落盘后 | 返回 `{kind:'retry'}` 接管恢复；默认 `undefined` 失败即终态 |
| `tools/*`（pre/execute/post） | tools 域（[第7章](./第7章-工具系统-能力接缝与执行管线.md)） | 每个工具调用执行前后 | 审批、沙箱、超时、结果改写——循环"间接"经过的第四个拦截面 |

以 `agent/pre-step` 为例看瀑布的挂载方式——默认决策（无人拦截时的兜底）由机器提供，监听链逐层包装它：

```ts
// agent.ts（节选，L271-L283）
const claimed = this.inbox.claim(target, position.turn)          // 原子认领
const assembly = await this.loopCtx.systemPrompt.assemble(...)   // 组装提示词与工具
const context = this.runtimeContext.project(...)                 // 运行时上下文快照
const decision = await this.dispatch.waterfall(
  'agent/pre-step', { messages: claimed, ...position, signal },
  (): Promise<PreStepDecision> => Promise.resolve<PreStepDecision>({
    kind: 'enter',
    messages: context === undefined ? claimed : [...claimed, context],
  }),
)
```

顺带留意 `agent/request-error` 载荷里的 `retryPolicy`（agent.ts L500）：它来自 `prepareCall` 捕获的适配器注册——重试监听器因此能按该 provider 自己的退避策略正确等待，而不必自己猜。

注意实现包**不导出任何钩子类型**——干预面完全由契约包的事件表定义。这条"Plugins, not loop changes"的仓库铁律不是空话，你可以数一数有多少子系统是这些事件的纯消费者：

| 插件 | 消费的事件 | 职责 |
|---|---|---|
| `llm-retry` | `agent/request-error` | 重试策略 |
| `session-checkpoint-policy` | `llm/stream` + `tools/execute` + `agent/pre-step` | 三处 flush 检查点（checkpoint） |
| `compaction-basic` | `agent/pre-step` + 维护任务 | 上下文压力压缩 |
| `compaction-image-offload` | `agent/pre-step` | 图片占用卸载 |
| `model-selection` | `system-prompt/assemble` + `agent/request` + `agent/pre-step` | 模型选择 |
| `plan-mode` | `agent/pre-step` | 计划模式拦截 |
| `repeat-tool-reminder` | `tools/post-execute` | 重复工具提醒 |
| `goal-round-driver` | 空闲时 `followup` 续轮 + `agent/pre-step` | 目标驱动的多轮推进 |
| `workspace-changes` | `agent/turn-stopping` + `tools/pre-execute` | 工作区变更守卫 |
| `hooks-claude-code` / `hooks-codex` | `agent/turn-stopping` | 外部 CLI 桥接 |

仓库里大约一半的子系统以这种方式挂在循环上，而循环一行不用改。想验证的话，`grep 'agent/pre-step' packages/ --include='*.ts'` 会给你一张比本表更长的清单。

### 3.8 退出路径全景：失败、取消与崩溃恢复

这一节是本章工程密度最高的一节。退出有三类：主动取消、请求失败、进程崩溃。逐一拆。

**取消（cancel）**。`cancel(cause)`（agent.ts L175-L181）默认先清空收件箱（除非 `keepInbox`），再 `abort` 当前相位的 `AbortController`。取消原因 `AgentCancelCause`（session/src/types.ts L189-L193）是封闭四元：`user` / `parent` / `hook(reason)` / `disposed`。两个容易被忽略的细节：

- **唤醒的归宿**：活动已被取消、还没收敛时，一条新的唤醒输入不能加入垂死的活动——`send()`（L157-L158）把它改道进 `next-turn`，等活动收敛到 idle 后再开新轮。运行中的唤醒被锁存（latch）在 `wakeRequested` 上，收敛后重放。
- **原因的复制**：落日志前 `abortedCancelCause()`（L80-L95）把活的取消原因复制成 JSON 安全的形式——Node 的 fetch 会往 `AbortSignal.reason` 上贴 `stack`，直接落日志要么被拒绝要么污染数据。

先看 `cancel()` 原文，五行浓缩了"先清场再熄火"的次序：

```ts
// agent.ts（节选，L175-L181）
cancel(cause: AgentCancelCause, options: CancelOptions = {}): void {
  if (!options.keepInbox) {
    this.inbox.clear()                                    // 落 canceled splice 事件
    if (this.phase.kind !== 'idle') this.phase.wakeRequested = false
  }
  if (this.phase.kind !== 'idle') this.phase.abort.abort(cause)
}
```

**流中断的持久化**。取消打断了正在流式输出的回复怎么办？用户可能已经看到一半文字了。`step()` 的 catch（L445-L486）处理：若流已开始且 signal 已中断，取 `live.interruptedBlocks()`——**安全前缀**（只含已完整组装的块），落一条 `assistant/message` 事件并标记 `interrupted: true`（L451-L465）。于是下一个请求的历史里包含"用户已经看到的内容"，模型不会装作那半句话没说过：

```ts
// agent.ts（节选，L448-L465，有删节）
if (signal.aborted) {
  const content = live.interruptedBlocks()        // 安全前缀：只含已完整组装的块
  if (content.length > 0) {
    live.settle('assistant/message', () => this.session.append('assistant/message', {
      turn, step,
      message: createAssistantMessage({ content, /* provider / model 来源 */ }),
      interrupted: true,                          // 下一请求包含用户已见内容
      stream: live.stream,
    }, { surfaceOp: 'append' }).seq)
  } else { /* 无完整内容 → 落 assistant/attempt */ }
}
```

若中断时还没有任何完整内容，或失败不是取消（流本身出错），则落 `assistant/attempt`——保留原始流但不进派生历史。落盘动作本身失败则聚合抛 `AggregateError`（L478-L484）。

**工具中断**。取消到达工具调度时：未启动的调用由 `appendSkippedToolCall`（tool-calls.ts L250-L260）合成 `tool/call` + `tool/result` 对（`isError: true`，错误码 `ABORTED_BEFORE_DISPATCH`）——保证日志里调用与结果**配对**，重放有效；已启动的调用先排空（drain）、再按模型顺序提交结果。

**步失败恢复**。步内失败（比如工具体抛错）时，turn 的 catch（agent.ts L342-L353）请 `ToolCallRecovery.results()`（[session/src/repair.ts](../../packages/core/session/src/repair.ts) L105-L198）为每个未答复的调用补一条保守错误结果：有 `tool/call` 记录的用 `TOOL_OUTCOME_UNKNOWN`——措辞明确告诉模型"**结果未知**……只有只读或幂等操作才可重试；可能有副作用的先核实外部状态或询问用户"；无记录的用 `TOOL_NOT_STARTED`——"可按需重试"。这段文案值得逐字读（repair.ts L32-L41）：恢复机制的诚实程度，直接决定模型会不会盲目重试一个可能已扣款的转账。恢复落盘本身再失败，则聚合成 `AggregateError`（agent.ts L350-L352）——绝不静默吞掉"连修复都失败"这件事。

turn 层的收口在 `turn()` 的 catch（L366-L381）：signal 已中止 → `{kind:'aborted', reason: cause}` 后原样重抛（`kick` 包容）；否则一律结构化——`LlmError` 保留其 `failure`，任何其他错误拍平为 `{message: errorChain(error), code: 'UNKNOWN'}`，然后 `throwError()` 发出 `agent/error` 再上抛。持久层看到的 `turn/end` 永远是可序列化的判别联合，没有裸的 `Error` 对象。

**崩溃恢复（resume）**。进程整个死掉，日志尾部可能停在一个敞开的 turn 中间。`AgentLoop.resumeWith()`（index.ts L816-L890）冷读日志后调用 `interruptedTurnClosers(persisted)`（repair.ts L209）：为中断的尾部 turn 合成缺失的工具结果 + `step/end` + `turn/end {kind:'interrupted'}`，然后**作为普通批次 append**（L855-L856）：

```ts
// index.ts（节选，L852-L862，有删节）
const coldRead = await handle.read(0, undefined, { signal: fused })
const persisted = coldRead.events
const closers = interruptedTurnClosers(persisted)     // 合成缺失的工具结果 + step/end + turn/end
if (closers.length > 0) await handle.append(closers)  // 作为普通批次追加
preparation = SessionPreparation.create(this.runtime.ctx.sessions.prepare(id, {
  seed: [...persisted, ...closers],                   // 修复后的事件流即回放种子
  ...
}))
```

注意这个设计：恢复不需要专门的"恢复模式"，它只是往日志里追加普通事件——恢复路径与正常运行路径是同一条。把本节全部机制收进一张表，你会看到它们共用同一条设计公理（日志只追加、事实必配对、恢复即追加）：

| 崩溃 / 失败点 | 日志停留状态 | 修复动作 | 修复者 |
|---|---|---|---|
| 请求失败（可重试） | `assistant/attempt` | `agent/request-error` → `{kind:'retry'}` | 监听器（如 llm-retry） |
| 请求失败（终态） | `assistant/attempt` | `turn/end {kind:'error'}` | turn() catch |
| 流中途取消 | 半截流 | `assistant/message` + `interrupted: true` | step() catch |
| 工具未启动即取消 | 无 call 记录 | 合成 call + result（`ABORTED_BEFORE_DISPATCH`） | tool-calls.ts |
| 工具已启动未答复 | `tool/call` 无 result | `TOOL_OUTCOME_UNKNOWN` / `TOOL_NOT_STARTED` | ToolCallRecovery |
| 进程崩溃于 turn 中部 | 敞开的尾部 turn | 合成结果 + `step/end` + `turn/end {interrupted}` | interruptedTurnClosers |

## 4. 源码走读

建议按下面七站的顺序通读，全程对照 [docs/agent-lifecycle.md](../../docs/agent-lifecycle.md) 的时序图。所有文件都在 `packages/core/` 下。

### 4.1 第一站：入口三方法与 Phase（agent.ts L42-L181）

先读 L42-L50 的 `Phase` 联合类型，再顺读 `send` → `followup` / `steer` / `inject` → `cancel` → `runMaintenance` → `wakeDriver`（L214）。注意 `wakeDriver` 的三分支：idle 则开新驱动器（`withInitiator` 包住 `kick`）；maintenance 或已中止的活动则锁存唤醒；健康的 running 驱动器**什么都不做**——它自己会在步边界认领收件箱。这是理解"机器永不丢唤醒"的关键：

```ts
// agent.ts（节选，L215-L223）
private wakeDriver(wakeAfterAbort = false): void {
  if (this.phase.kind !== 'idle') {
    const reason = abortedCancelCause(this.phase.abort.signal)
    if (reason?.kind !== 'disposed' && (this.phase.kind === 'maintenance' || wakeAfterAbort)) {
      this.phase.wakeRequested = true                 // 锁存：收敛后重放
    }
    return
  }
  /* idle → 创建 running 相位，withInitiator 包住 kick() */
}
```

健康的 running 分支直接返回不是遗漏：活着的驱动器自己会在步边界认领收件箱，重复唤醒只会浪费。

### 4.2 第二站：收件箱（inbox.ts 全文，约 240 行）

先读 L27 的 `inboxProjectionDefinition`：`apply()` 是一个严格的折叠器（fold）——非法 splice 坐标、跨列表重复的 `MessageId` 都会让折叠抛错，并指明出错事件的 seq。再读 `ReactLoopInbox`：`claim()`（L109）的"清空 next-step + 轮次边界加取一条 next-turn"；`mutate()`（L198）里每条 splice 先落 `agent/inbox/spliced`（L235）再发 inserted / discarded 通知：

```ts
// inbox.ts（节选，L109-L114）
claim(target: InboxTarget, turn: number): UserMessage[] {
  const claimed = this.mutate('next-step', 0, this.nextStep.length, [], false)
  if (target === 'next-turn') claimed.push(...this.mutate('next-turn', 0, 1, [], false))
  for (const message of claimed) this.dispatch.emit('agent/inbox/claimed', { message, turn })
  return claimed
}
```

`mutate` 的第四个参数 `discardRemoved: false` 是关键：认领是**纯删除**（不记 `outcome: 'canceled'`、不发 discarded 通知），与用户主动取消的语义区分开。

### 4.3 第三站：turn()（agent.ts L296-L396）

逐行读这 100 行，对照 3.6 节的表。四个锚点：L305 `turn/start`；L324-L327 首步消息为空 → `completed` 关 turn **不花模型调用**（被取消的唤醒消息、被改写为空的决策都走这条路）；L331-L334 挂 `ToolCallRecovery` 观察器——它订阅 `session/event`，只记录"哪些工具调用还没得到结果"，为失败恢复备料；L382-L389 的 `finally` 保证 `turn/end` **必落**（`turnEnds!` 的非空断言靠"每条退出路径都先赋值"支撑）：

```ts
// agent.ts（节选，L382-L389）
} finally {
  try {
    this.session.append('turn/end', { turn, reason: turnEnds! })
  } catch (error: unknown) {
    this.throwError(error)
  }
}
```

连"落 `turn/end` 本身失败"都有出口——上报后抛出，让驱动器边界收敛。这台机器对失败的处理没有一处是"假装没发生"。

### 4.4 第四站：step() 上半场（agent.ts L398-L443）

从 `prepareRequest()`（L547）读起：种子配置的来源切换（L556-L575）、`agent/request` 瀑布（L576）、`prepareCall`（L587）。然后是准入与落日志的精确顺序（L410-L423）：**为什么** `system/message` 与 `user/message` 在 `request/header` 之前落？因为请求要从日志派生，日志必须先完备。最后 `buildRequest()`（L599-L687）以 `deriveMessages()` + `deepFreeze` + `markAgentLoopRequest`（L671-L685）收尾。留意 L614-L634 里 baseline 比较的细节：`headerEquals` 判断的是 config 与 tools 的规范化相等——提示词不参与比较，因为它住在派生历史里而不是 header 里；这也是为什么系统提示词变化不触发 change header，只以 `system/message` 的形式进历史。

### 4.5 第五站：结算与失败（agent.ts L445-L543 + assistant-stream.ts）

先读 [assistant-stream.ts](../../packages/core/agent-loop/src/assistant-stream.ts)（约 140 行，是 agent-loop 里最易读的文件）：`AssistantStreamAttempt`（L18）同时喂三个消费者——紧凑流累积器（持久化用）、块组装器（`blocks()` / `interruptedBlocks()`）、以及 `agent/assistant-stream` 的 start / chunk / end 帧。`settle()` 的时序最值得注意——先落日志拿 seq，再发 end 帧：

```ts
// assistant-stream.ts（节选，L78-L97）
settle(eventType, append) {
  let seq: SessionSeq
  try { seq = append() } catch (error) { this.abandon(); throw error }
  this.terminal = true
  this.emit({ type: 'end', /* ... */ outcome: { kind: 'committed', eventType, seq } })
}
```

消费者（如 Web UI）收到 committed end 帧时，对应的日志事件已保证存在——直播与回放永远不会对不上。再回头读 step() 的结算段（L487-L529）：`error` / `aborted` → `assistant/attempt` → `agent/request-error` → retry 或抛 `LlmError`；正常 → `createAssistantMessage` → `assistant/message`（L520-L529，携带 `usage` 与完整流）→ `live.settle`。三段 catch（L445、L478、L539）各管一层：流中断、落盘失败、结算失败。

### 4.6 第六站：工具调度（tool-calls.ts 全文，约 290 行）

`executeToolCalls()`（L60）先 `requireInitiator()`（L68）拿发起 Agent——工具结果要写回**它的**会话。每个调用构成 `ToolExecutionInput {callId, name, arguments, agent, signal}`（L72-L81），`parseArguments`（L105）对坏 JSON 保留原文。分组规则：`executionMode(exec).kind` 为 `parallel`（工具声明 `isConcurrencySafe(args) === true`）的连续调用进同一个池，受 `maxParallelToolCalls`（默认 10，[constants.ts](../../packages/core/agent-loop/src/constants.ts) L6）限流滚动执行；否则独占成**屏障（barrier）**。`runGroup()`（L122-L247）里盯住 `commitReady()`（L147）——结果只按**模型顺序**提交，`committed` 指针只跨连续就绪槽前进：

```ts
// tool-calls.ts（节选，L147-L160）
const commitReady = async (): Promise<void> => {
  while (committed < group.length) {
    const slot = slots[committed]
    if (slot === undefined) break                     // 只跨连续就绪槽前进
    const call = group[committed]
    const result = slot.needsPost
      ? await ctx.tools[TOOL_RUNTIME_SCHEDULER].finalize(slot.exec, slot.result)
      : ctx.tools[TOOL_RUNTIME_SCHEDULER].finish(slot.exec, slot.result)
    appendToolResult(session, turn, step, call!.block, result, callSeqs[committed]!)
    for (const context of result.additionalContexts ?? []) acceptContext(context)
    concluded ||= result.concludesTurn === true
    committed++
  }
}
```

第 3 个调用即使最先完成，也必须等第 1、2 个提交完——模型看到的工具结果顺序永远与它发起调用的顺序一致。`result.additionalContexts` 经 `acceptContext` 回填 next-step 收件箱（L157），`result.concludesTurn === true` 累积成提前收束（L158）。五阶段管线（prepare → dispatch → finalize）属[第7章](./第7章-工具系统-能力接缝与执行管线.md)，本章只看调度视角。

### 4.7 第七站：修复（repair.ts 全文，约 210 行）

`ToolCallRecovery.observe()`（L116）是一个迷你折叠器：`assistant/message` 登记 pending 的工具调用、`tool/call` 标记已启动、append 型 `tool/result` 注销。`results()`（L157）为幸存者生成保守错误结果，seq 接续最后一个观测事件。文件顶部的 `CLOSER_TEXT`（L32-L41）是写给模型的两段话，值得逐字品味。最后看 `interruptedTurnClosers()`（L209）如何被 [index.ts](../../packages/core/agent-loop/src/index.ts) L852-L862 的 `resumeWith` 当作普通批次消费——为 3.8 节的结论收口：**恢复即追加**。

## 5. 常见误区

**误区一："循环把对话存在内存里。"** 恰恰相反：循环在内存里只有 Phase 相位、几个计数器和一个防重复冻结的 WeakSet——没有任何对话内容。每次请求都现场调 `session.deriveMessages()` 派生（agent.ts L671），收件箱是日志投影，工具结果先 append 再进历史。推论：日志丢了才是真丢了一切；也正因此崩溃恢复才可能。

**误区二："工具结果是直接返回给模型的。"** 没有任何代码把工具返回值"塞给"模型。工具执行完落 `tool/result` 事件（tool-calls.ts L282-L289），**下一个 step** 组装请求时才被 `deriveMessages()` 派生进消息数组。工具与模型之间隔着一整条日志——这就是"Model-visible ⟺ logged"。

**误区三："max-tokens 是一种错误。"** 它是 `FinishReasonMap` 里的正常结束原因（llm/types.ts L159），步层返回 `{kind:'max-tokens'}`，turn 正常收尾并落 `turn/end {kind:'max-tokens'}`。它的特殊之处只是粘性：后续步骤即使正常完成也不能把 turn 的结局"降级"成 `completed`（agent.ts L336-L341），因为被截断的那次回答模型还欠着。

**误区四："取消会丢数据。"** 取消路径上每个动作都在保数据：已流出的安全前缀以 `interrupted: true` 定稿成 `assistant/message`（用户看到过的内容进历史）；未启动的工具调用补 `ABORTED_BEFORE_DISPATCH` 合成结果保证配对；`turn/end {kind:'aborted'}` 必落。取消丢的只是"还没发生的事"，已发生的事实全部留存。

**误区五："resume 需要一种特殊的恢复模式。"** `resumeWith` 冷读日志、追加 `interruptedTurnClosers` 合成的关闭事件（index.ts L855-L856），然后就走正常的 `setupAndPublish`。没有恢复标志位、没有恢复分支——修复只是往仅追加日志里追加普通事件。同理也别指望循环替你重试：`agent/request-error` 的默认动作是**失败即终态**，重试策略住在 `llm-retry` 这样的监听器里。

## 6. 动手环节

**练习一：亲眼看一台发动机起步。** 配好 `DEEPSEEK_API_KEY` 后运行：

```sh
pnpm dsh --profile headless "列出当前目录的文件，并统计一共有几个"
```

观察输出里"思考 → 调工具 → 再思考 → 回答"的节奏，对照 3.2 节的总图数一数：这次运行里有几个 turn、几个 step？

**练习二：读真实的事件序列。** 打开 [packages/test-support/session-snapshot/tests/fixtures/](../../packages/test-support/session-snapshot/tests/fixtures/) 下任意一个 fixture（如 `subagent-activation-limit.ts`），从头找 `turn/start` → `step/start` → `user/message` → `request/header` → `assistant/message` → `tool/call` → `tool/result` → `step/end` → `turn/end` 的完整链条，再找一个含 `interrupted` 或恢复结果的尾部，对照 repair.ts 的合成格式。

**练习三：数一数拦截点的消费者。** 执行 `grep -r "agent/pre-step" packages/ --include="*.ts" -l`（或用 IDE 的全局搜索），把结果与 3.7 节的表对照。哪些消费者是你意料之外的？

**练习四：写一个拒绝敏感消息的 pre-step 监听器（伪代码）。** 需求：拒绝任何包含"密码"的进入消息。形状如下：

```ts
// 教学示例：拒绝敏感消息的 pre-step 监听器（伪代码，展示监听器形状）
ctx.on('agent/pre-step', async (payload, next) => {
  const hasSecret = payload.messages.some(m =>
    m.content.some(b => b.type === 'text' && b.text.includes('密码')))
  if (hasSecret) return { kind: 'reject' }   // 拥有决策 → 不调 next() 即否决
  return await next()                        // 无异议 → 必须委托（第4章铁律）
})
```

注意两点：reject 后该 turn 以 `blocked` 收尾、被认领的消息不会重新入队；若你只想**改写**而不是拒绝，返回 `{kind:'enter', messages: [...]}` 并对不需要改的路径调 `next()`。对照 [docs/agent-lifecycle.md](../../docs/agent-lifecycle.md) 的时序图验证你的预期。

**练习五：找一次"欠账"的证据。** 在练习二的 fixtures 里找一个 `turn/end` 的 reason 为 `max-tokens` 或 `interrupted` 的会话：前者验证粘性——看后续是否存在正常完成的 step、最终 reason 是否仍是 `max-tokens`；后者对照 repair.ts 的合成事件格式——合成 tool/result 的错误码（`TOOL_OUTCOME_UNKNOWN` / `TOOL_NOT_STARTED`）、`turn/end {kind:'interrupted'}`、以及时间戳复用最后一条真实事件的特征。

## 7. 本章小结

1. Agent loop 是模型外层的发动机：喂历史 → 拿回复 → 执行工具 → 追加结果 → 再喂；`ReactLoopAgent` 之名致敬 ReAct（Reason + Act）范式。
2. 契约与实现分离：`dsh-agent` 声明 `Agent` 接口、`ctx.agents` 注册表与 `agent/*` 事件表；`dsh-agent-loop` 是唯一实现，经 `setFactory` 挂上——循环可替换，插件绝不依赖实现包。
3. 两层时间单位：**turn** 是对用户的一次完整交代，**step** 是一次"请求—工具"循环；驱动器内部是 idle / maintenance / running 三态 Phase 状态机，`kick()` 的 `while (await this.turn())` 是异常被包容的驱动器边界。
4. 三个输入入口语义不同：`followup` 排队下一个 turn；`steer` 注入下一个 step 边界；`inject` 只排队不唤醒。收件箱是持久投影——每次 splice 先落 `agent/inbox/spliced`，重启后从日志重建。
5. 请求是日志的纯函数：配置经 `agent/request` 瀑布与 `prepareCall` 解析，提示词以 `system/message` 落日志，消息由 `deriveMessages()` 现场派生并深冻结——**一切模型可见的内容都先落日志**。
6. 停止判定 = "模型不再欠答复"：步层看 `FinishReasonMap`（无工具调用或全部 concluded → completed；token 上限 → max-tokens；否则 null 续步）；轮次层是 `TurnEndReasonMap` 七种，`interrupted` 与 `forked` 只由恢复/分叉合成。
7. max-tokens 是正常结束原因且**粘性**：后续正常步骤不能把 turn 结局降级为 completed。
8. 四个瀑布拦截点：`agent/pre-step`（拒/改进入消息）、`agent/request`（换配置不改消息）、`agent/request-error`（接管重试）、`tools/*`（审批与沙箱，第 7 章）；循环本身不导出任何钩子——"Plugins, not loop changes"。
9. 约一半子系统是这些事件的纯消费者：llm-retry、session-checkpoint-policy、compaction-basic、model-selection、plan-mode、goal-round-driver、workspace-changes、hooks 桥接……循环一行未改。
10. 取消不丢数据：安全前缀以 `interrupted: true` 定稿；未启动的工具调用补 `ABORTED_BEFORE_DISPATCH` 合成结果；唤醒输入改道 next-turn，收敛后重放；取消原因经 `abortedCancelCause` 复制成 JSON 安全形式。
11. 失败恢复诚实保守：`ToolCallRecovery` 为未答复调用补 `TOOL_OUTCOME_UNKNOWN`（"结果未知，只读/幂等才可重试"）或 `TOOL_NOT_STARTED`，措辞直接决定模型敢不敢重试。
12. 崩溃恢复没有特殊模式：`resumeWith` 冷读日志后把 `interruptedTurnClosers` 的合成事件**作为普通批次追加**——恢复即追加，正常路径与恢复路径是同一条。
13. 循环无内建 turn 预算：限制失控 turn 的策略必须从 `agent/turn-stopping` 等扩展点取消——机械与策略的边界在每一处细节上都被守住。

## 8. 必读源码回顾与延伸阅读

**必读源码（本章引用过的，按学习顺序）：**

1. [packages/core/agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts)——全文件精读：L42 Phase、L64 requestProposal、L98 ReactLoopAgent、L154-L181 入口与取消、L214 wakeDriver、L252 kick、L267 preStep、L296 turn、L398 step、L547 prepareRequest、L599 buildRequest。
2. [packages/core/agent-loop/src/inbox.ts](../../packages/core/agent-loop/src/inbox.ts)——L27 投影折叠器、L74 ReactLoopInbox、L109 claim、L235 splice 落日志。
3. [packages/core/agent-loop/src/tool-calls.ts](../../packages/core/agent-loop/src/tool-calls.ts)——L60 executeToolCalls、L122 runGroup、L147 commitReady、L250 appendSkippedToolCall。
4. [packages/core/agent-loop/src/assistant-stream.ts](../../packages/core/agent-loop/src/assistant-stream.ts) 与 [runtime-context.ts](../../packages/core/agent-loop/src/runtime-context.ts)——流帧的结算时序与提示词准入。
5. [packages/core/agent/src/runtime-types.ts](../../packages/core/agent/src/runtime-types.ts) L112-L243 决策类型与 L245-L394 事件表——逐条读 `@mode` 与 `@param`，这张表就是循环的全部干预面。
6. [packages/core/agent/src/index.ts](../../packages/core/agent/src/index.ts) L245 起——AgentRegistry 与 initiator 作用域。
7. [packages/core/session/src/repair.ts](../../packages/core/session/src/repair.ts)——L32 CLOSER_TEXT、L105 ToolCallRecovery、L209 interruptedTurnClosers。
8. [packages/llm/llm/src/types.ts](../../packages/llm/llm/src/types.ts) L156 FinishReasonMap / L511 GenerateOptions；[packages/llm/llm/src/index.ts](../../packages/llm/llm/src/index.ts) L169 PreparedLlmCall / L208 LlmAdapter。

**延伸阅读：**

- **上一章**：[第4章 事件驱动](./第4章-事件驱动-五路派发与三个事件域.md)——waterfall 的 `next()` 语义、serial 的数据否决通道、scope 过滤，是理解本章四个拦截点的机制前提。
- **下一章**：[第6章 模型调用](./第6章-模型调用-llm能力接缝与适配器.md)——`prepareCall` / `PreparedLlmCall` / `LlmAdapter` 的完整世界：适配器如何解析默认值、流式协议如何定义。
- **工具纵深**：[第7章 工具系统](./第7章-工具系统-能力接缝与执行管线.md)——本章只看了调度视角的 `tools/pre-execute` → `tools/execute` → `tools/post-execute` 五阶段管线在那里展开。
- **日志纵深**：[第8章 消息系统](./第8章-消息系统-会话日志即模型记忆.md)——`SessionEventMap` 全表、`deriveMessages()` 的投影规则、surface 替换语义：本章反复依赖的"唯一真源"在那里解剖。
- **官方文档**：[docs/agent-lifecycle.md](../../docs/agent-lifecycle.md)（turn/step 时序图，本章总图的权威版）、[packages/core/agent-loop/README.md](../../packages/core/agent-loop/README.md)（请求 header 与适配器默认值、失败与取消的完整契约、Known Limitations）、[docs/subsystems/core.zh.md](../../docs/subsystems/core.zh.md)（六包主干如何流经一个轮次）、[docs/tool-execution-pipeline.md](../../docs/tool-execution-pipeline.md)（工具管线流程图）。

下一章，我们钻进 `prepareCall` 背后的 LLM 接缝，看一个"无状态函数"如何被适配器包装成可流式、可重试、可审计的能力。
