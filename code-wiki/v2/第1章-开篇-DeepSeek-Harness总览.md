# 第1章 开篇：DeepSeek Harness 总览

## 本章导读

本章回答四个问题：DeepSeek Harness（下称 DSH）**是什么**、**能干什么**、**为什么值得你逐行读它的源码**、以及**它由多少零件组成、为什么恰好是这么多**。

在全书 13 章里，本章属于地基部分（第 1–4 章）的总入口。全书的分部逻辑如下，本章结尾的"延伸阅读"会把相邻章串起来：

| 部分 | 章 | 主题 |
|---|---|---|
| 地基（1–4） | 第1章 总览 / 第2章 分层架构 / 第3章 Cordis 内核 / 第4章 事件驱动 | 项目是什么、代码组织、框架内核、通信机制 |
| 运行时主干（5–8） | 第5章 Agent Loop / 第6章 模型调用 / 第7章 工具系统 / 第8章 消息系统 | 循环、模型、工具、记忆 |
| 专题（9–11） | 第9章 上下文工程 / 第10章 上下文压缩 / 第11章 会话管理 | 一次请求塞了什么、对话太长怎么办、持久化与分叉 |
| 组合与产品（12–13） | 第12章 扩展系统 / 第13章 产品面 | 配置即组合、CLI/Web/Desktop/SDK |

**先修要求**：会 JavaScript/TypeScript，用过 ChatGPT 或 Claude，知道什么是 npm 包。不需要读过任何 agent 框架源码——本书默认你没读过。

**读完本章你能做什么**：用自己的话向同学解释"harness（挽具）"这个词为什么准确；在仓库里 30 秒定位任何一个能力所在的包组；亲手数出全仓库的包数并解释这个数字的来历；跑起 Web UI 与无密钥测试；知道接下来 12 章每一章对应仓库的哪一块。

## 从一个问题出发

上学期期末，你让 ChatGPT 帮你修一个测试挂掉的 bug。它给了你一段代码，你贴进编辑器，跑测试，还是红的，再把报错贴回去，它又给一段……三个来回之后你意识到：**你成了模型的"手"**。模型负责思考，你负责一切执行：打开文件、跑命令、看输出、把结果搬回对话框。

现在把问题反过来问：要让模型**自己**长出手和脚，缺的是什么？列一下清单：一个能替它执行 shell 命令的进程；一套读写文件的权限规则；一个记得住整个对话的存储；一条"模型发起 tool call（工具调用）→ 执行 → 把结果喂回去 → 模型继续"的循环；还得有人管沙箱、人工审批、日志、子任务分发、超时与重试。业界把这一整套"模型之外、产品之内"的基础设施叫做 **harness（挽具）**——这个词选得很准：挽具是套在马身上、把马的力气传导到车轮上的那套皮革与缰绳，它不提供马力（马力在模型），但决定了力往哪儿传、以什么姿态传。

DSH 就是 DeepSeek 开源的一整套挽具（npm 包名 `@deepseek-ai/dsh`，命令行就叫 `dsh`）。它最特别的地方不是功能多，而是一条贯彻到偏执的架构决策：**一切皆插件（everything is a plugin）**——连驱动模型的循环本身都只是一个普通插件。这条决策的后果——好的和需要适应的——会贯穿全书。

同一个仓库，值得用三个透镜各看一遍：当**产品**用（3.2），当**教材**读（3.3），当**平台**嵌（3.4）。三个透镜看到的是同一批代码的三种切面，而且互为印证——能当教材的代码才敢当平台让人嵌入，能当平台嵌入的产品才算把边界画对了。

## 3.1 DSH 是什么：一段话定义与两大支柱

**定义**：DSH 是一个建立在 Cordis 框架之上的"全插件"（all-plugin）编程 Agent 挽具，用 TypeScript 写成，提供浏览器/桌面 UI、CLI、TypeScript 与 Python SDK、ACP 服务器等多种形态，让大模型在受控环境中替你读写代码、执行命令、完成任务。

支撑这个系统的是两大支柱（pillar），都写在 [docs/architecture.md](../../docs/architecture.md) 开头，此处先记住结论：

**支柱一：没有特权核心（no privileged core）。** 文档原话是 *"There is no privileged core to patch"*——模型适配器、工具注册表、会话日志、Agent 循环本身，全都是插件树上可替换的普通节点，你想换掉任何一个，就是在配置里挂一个新插件到它旁边，而不是去改"内核"。传统框架通常反着做：一个单体核心加若干 hook 点。为什么否决那种做法？因为 hook 点是**枚举出来的**——框架作者预见到的扩展点才有，没预见到的就得等上游发版；而插件树上每个节点生来平等，替换的可能性是结构自带的、不用枚举的。代价是你必须一开始就把"谁定义服务、谁提供实现、谁消费"想清楚，这正是第 3 章和第 7 章的主题。

**支柱二：模型可见即可重建（model-visible means logged）。** 文档原话：*"Every model request must be reconstructable from the log"*——凡是发到模型那边的内容，都必须能从会话日志（session log）里重建出来。任何一个模型请求，你拿着日志就能完整复现它当时看到了什么。这条不变量把"agent 的记忆"从各处散落的内存状态收拢成一条持久事件流，第 8 章整章讲它。替代方案是常见的两种："内存态对话 + 事后落盘"或"只保存对话文本"——都被否决：前者崩溃即失忆、无法审计；后者丢掉 tool 结果、上下文注入、系统提示变化，你永远无法回答"模型当时到底看到了什么"。而 agent 系统里最难调试的 bug 恰恰是"模型为什么会**这么**说"——支柱二把这个问题从玄学变成一次日志回放。

一句话记住两者的分工：支柱一管**怎么组装**（组装是配置行为），支柱二管**怎么记账**（历史是唯一事实源）。第 3.6 节还会给一段预览。

### 先认识 Cordis：站在哪个框架上

后文你会不断看到 "Cordis" 这个名字，先交代清楚。Cordis 是 cordiverse 社区的 TypeScript 应用框架，其设计写成了论文《A Programming Paradigm for Spatiotemporal Composability》（arXiv:2608.25512，[README](../../README.md) 引用）。它的核心机制一句话（转述 [docs/architecture.md](../../docs/architecture.md) 第 11 行）：**插件（plugin）向一个共享上下文（Context）贡献服务（services）、类型化事件（typed events）与可逆效果（reversible effects）**。

如果你在后端课上学过 Spring 或 Angular 的依赖注入（dependency injection），Cordis 是同族物种，但有两个关键字是那类框架没有的：**可逆**——每次注册都是一个效果（effect），插件卸载时效果自动退回，系统不留残渣；**可重组装**——配置层变动可以触发整棵树的一次序列化重载（`dsh-hmr` 负责，见下节）。DSH 不从 npm 安装 Cordis，而是把 9 个相关包 vendor 进 `vendor/` 并改写作用域，钉在源码上——对框架的态度是"带补丁的同行者"而非"版本号的消费者"，第 2 章展开。第 3 章是 Cordis 的整章课程，第 4 章讲它的五路事件派发。

## 3.2 视角一：一个可用的编程 Agent

先别管架构，把它当产品用。你的第一天大概是这样的：

```sh
npx @deepseek-ai/dsh web        # 或源码运行：pnpm dsh web
```

浏览器打开 `http://127.0.0.1:3080`（[README](../../README.md) 承诺的默认地址），你面对一个聊天界面，输入"帮我看看这个仓库的测试为什么挂了"。接下来发生的事，就是本章开头那个"三个来回"问题的解法。先看**一个回合（turn）里发生什么**：你的消息入队；agent 循环从会话日志推导出下一发模型请求（系统提示 + 历史 + 上下文）；模型要么回话、要么发起 tool call；被批准的工具调用走执行管线，结果作为 tool 结果写回日志；循环回到"推导下一发请求"，直到模型不再要工具、本轮收尾。这个"日志→请求→工具→日志"的闭环就是第 5 章的主角，现在只需记住：**你看到的聊天，只是这条闭环投到屏幕上的影子**。

再看模型的手——工具族。模型看不到你的屏幕，但它有一排工具，每一个都登记在生成的 [docs/tool-catalog.md](../../docs/tool-catalog.md) 里（参数、权限、输出截断规则俱全），而且**工具名与包的对应关系一眼可查**：

| 工具 | 所在包 | 干什么 |
|---|---|---|
| `bash` / `pwsh` | `packages/shell/tool-bash`、`tool-pwsh` | 执行命令（POSIX / Windows） |
| `read` / `write` / `edit` / `read_image` | `packages/fs/tool-fs` | 读写改文件、读图 |
| `glob` / `grep` | `packages/fs/tool-fs-search` | 按名/按内容搜索 |
| `str_replace_editor` | `packages/fs/tool-str-replace-editor` | 精确字符串替换编辑 |
| `web_search` / `web_fetch` | `packages/web/tool-web` | 搜索与抓取网页 |
| `lsp` | `packages/lsp/` | 拿语言服务器的类型信息 |
| `terminal_open` 等 | `packages/terminal/` | 常驻 PTY 终端操作 |
| `run_code` | PTC 模式（`packages/ptc-runtime/`） | 整段程序进沙箱执行 |
| `present` | `packages/deliverables/tool-present` | 显式声明本轮交付物 |
| `todo_write` | `packages/todo/tool-todo` | 维护任务清单 |
| `exit_plan_mode` | `packages/plan/plan-mode` | 计划模式收尾 |
| `create_goal` 等 | `packages/goal/tool-goal` | 会话目标与自动续轮 |
| `schedule_*` | `@deepseek-ai/dsh-tool-schedule` | 定时后续动作 |
| `ask_user_question` | `packages/interaction/tool-ask-user` | 反过来问你 |

工具之外，还有一圈"协作与治理"能力，同样各占一组包：**沙箱与审批**——默认权限预设（permission preset）是 `workspace-write`（工作目录内可写、目录外受限、危险操作走人工审批），另一档是名字就带警告的 `danger-full-access`（[packages/interaction/permission-presets/README.md](../../packages/interaction/permission-presets/README.md)）；**子代理（subagent）**——主 agent 把活派给子 agent，派发方式本身是一个插件族：进程内 fork/spawn、经 ACP、经 dsh 自己的 SDK（`subagent-dsh-sdk`）、甚至调 Claude Code 或 Codex 当子代理（`packages/subagent/` 下各有桥接包）；**skills**——可加载的技能目录（`packages/skill/`）；**后台作业**（`packages/jobs/`）、**工作流引擎**（`packages/workflow/`，含 PTC 流程引擎）、**MCP 外部服务器接入**（`packages/mcp/`，把外部 Model Context Protocol 服务器暴露成本地工具）、**Claude Code / Codex 钩子桥**（`packages/hooks/`）；甚至有**运行时自我修改**——`packages/extensions/` 让 agent 检查活着的插件/服务并挂载/卸载插件。

还有一个容易忽视但极实用的能力：**会话检索**。`packages/session-query/` 提供跨会话的逻辑语料、有界读取、谱系（lineage）与 SQLite 全文搜索——你的历史会话不是躺在磁盘上的死数据，而是可查询的资产。

值得单独点名的是**沙箱（sandbox）**：`packages/sandbox/` 是进程限制（process confinement）的接缝，后端有 bwrap 与 Landlock（Linux）、Seatbelt（macOS）三种实现（[packages/README.md](../../packages/README.md) 的表）。`workspace-write` 预设的背后就是它。动手运行前请先读根目录的 [SAFETY.md](../../SAFETY.md)——[README](../../README.md) 在显眼位置要求了这件事。

所有这些能力不是硬编码在某个 `main.ts` 里的，而是**按 profile（档案）组装**出来的。一个 profile 就是一份进程组合模板：`web`（浏览器应用）、`headless`（一次性运行器，跑完打印答案就退出）、`sdk`（JSON-RPC stdio 服务器）、`sdk-minimal`（不走 `dsh-base` 大树的独立最小树）、`acp`（automation-only 的 ACP 服务器）。`desktop` 这个名字保留给 Electron 应用。`dsh` 是**唯一**受支持的 Node 应用启动器——这不是君子协定，`scripts/verify-application-entrypoints.ts` 这道门禁会在 CI 里拒绝任何想另开入口的包。为什么管这么严？因为入口一多，"组装顺序由谁说了算"就会重新变成一笔糊涂账，支柱一就名存实亡了。

### 组合的次序：一棵树如何长出来

profile 只回答"叠哪些 bundle（束）"；真正的组合是四层 patch（补丁）在**空根**上的依次叠加（[apps/cli/README.md](../../apps/cli/README.md) 拥有权威描述）：

```text
空根
 ↓ ① 各 bundle 的 patch —— 按 dsh.profile.bundles 列出的顺序
 ↓ ② profile 自己的 cordis.patch.yml
 ↓ ③ home 级 $DSH_HOME/cordis.patch.yml
 ↓ ④ --patch 命令行覆盖层（可多个，按 argv 顺序）
```

每个 patch 按 id 命中一行配置：要么整行替换，要么插入新行。要点在于**你的覆盖永远在最上层**——官方发的东西对你而言也只是一层 patch，这就是"没有特权核心"在配置层面的落地。想看组合结果而不启动进程，用 `dsh --profile web --dump-config`；配置还能热重载（hot reload）：`dsh-hmr` 监视 profile 清单与两级 patch 文件，变更经一次序列化重载完成整棵树的重组装（headless、SDK、ACP 默认关掉它，`sdk-minimal` 干脆不含它）。第 12 章会把"配置即组合"讲透。

## 3.3 视角二：一部 agent 系统的工程教科书

如果你只把 DSH 当产品用，会错过它更值钱的那一半。这个仓库本身是一门"如何工程化一个 agent 系统"的课程，而且教材、板书、甚至错题本都齐了。

**分层文档**（[docs/](../../docs/)）：[architecture.md](../../docs/architecture.md) 是总纲（改 `packages/` 前必读）；`docs/subsystems/` 一子系统一页，共 64 页英文页，从 `agent-team` 到 `webhook` 每个子系统都有独立参考；`docs/cookbook/` 顶层 11 页步骤手册（加一个工具、加一个包怎么做）；`docs/user/` 是产品文档；[docs/development.md](../../docs/development.md)、[docs/testing.md](../../docs/testing.md)、[docs/defensive-patterns.md](../../docs/defensive-patterns.md) 是开发者必读三件。

**九份生成目录**——全部由脚本从源码生成、CI 校验新鲜度。你可以把它们理解为"永不撒谎的索引"：代码改了目录没跟上，门禁就红。每份的用途：

| 目录 | 回答什么问题 |
|---|---|
| `config-catalog` | 每个插件有哪些配置项 |
| `tool-catalog` | 每个模型可见工具的参数与行为 |
| `persistence-catalog` | 哪些数据被持久化、格式如何 |
| `module-graph` | 包与包的依赖全景图 |
| `capability-seams` | 全仓库的能力接缝清单 |
| `event-producer-consumer` | 每个事件谁发谁收 |
| `agent-lifecycle` | agent 从生到死的状态迁移 |
| `tool-execution-pipeline` | 一次工具调用经过哪些站点 |
| `graph-atlas` | 上述图集的地图册 |

生产这些目录与执行纪律的是 `package.json` 里**59 个 `verify-*` 门禁脚本与 16 个 `gen-*` 生成器**——从 `verify-md-links`（文档链接有效性）到 `verify-client-ui-i18n`（客户端 UI 文案必须走本地化字典）。这个数字本身就是一课：DSH 把"工程规范"从 review 意见变成了可执行程序。

**决策记录**（[.agents/notes/](../../.agents/notes/README.md)）：这是我最想让你注意的部分。Implemented（已实施）229 篇、proposed（提议中）37 篇、rejected（已否决）8 篇、archived（已归档）925 篇，每篇记录**为什么这么做、放弃了什么**。例如 `.agents/notes/implemented/architecture/2026-06-13-capability-seams.md` 定义了全书反复出现的"能力接缝（capability seam）"模型。那 8 篇 rejected 与 925 篇 archived 尤其珍贵——教科书只教你正确的路，这里连走错的路都标了路牌。归档笔记冻结只读、永不当作现行权威，这是明文政策。

**事故复盘（postmortem）**：[docs/postmortem/](../../docs/postmortem/README.md) 有 4 篇编号复盘（如 `0002-js-expression-disabled-filesystem-tools`），每篇都是双语三件套。**双语三件套**：`docs/` 下几乎每页都是 `foo.md` + `foo.zh.md` + `foo.i18n.yaml`，配对与新鲜度同样有门禁（`verify-translation-pairing`）。**动手教程**：[docs/cordis-tutorial/](../../docs/cordis-tutorial/index.md) 共 7 章，从写第一个插件一路讲到把 Cordis 用进 harness 本身。**维护者技能**（`.agents/skills/`，15 个）：给"用 agent 开发本仓库"的 agent 看的操作手册——这个仓库连"如何让 AI 帮我维护我自己"都文档化了。

**升级指南与基准**：`docs/upgrade-guide/` 按 version 记录每个外部可感知的破坏性变更——[AGENTS.md](../../AGENTS.md) 要求"立即记录"，不是"下个里程碑补"；`docs/persistence-changes/` 把持久化类型变更变成显式的评审流程；顶层 `benchmarks/` 目录放性能门禁（`pnpm run test:bench`），`snapshots/` 顶层树归 session 驱动的录制用例（[snapshots/AGENTS.md](../../snapshots/AGENTS.md) 立有所有权规则）。一个连"性能回归"与"快照归属"都有成文宪法的仓库，这就是它当教材的资格。

**测试策略**也值得先记两条：`pnpm run test:coverage` 是 CI 覆盖门禁（对 `packages/*/*/src` 要求逐文件 100%——注意不是 `pnpm run test`）；每个非平凡的模型可见或用户可见改动都要更新无密钥的录制会话快照。规则详见 [docs/testing.md](../../docs/testing.md)。

把你以后使用这套资料的**固定路线**也定下来：遇到任何陌生的包，先查 [packages/README.md](../../packages/README.md) 锁定组 → 读组 README 拿到包清单 → 读包 README（用途、配置、扩展点、已知限制四段是门禁强制的）→ 需要更深就查对应的 `docs/subsystems/` 页 → 还想问"为什么"就去 `.agents/notes/` 搜决策记录。五步之内必有答案，这本身就是一个值得借鉴的文档工程。

把这个仓库当解剖课的标本来看：一个生产级 agent 系统的每一块器官——循环、模型适配、工具管线、记忆、压缩、持久化、扩展——都切好了摆在正确的位置上，还贴了标签。后面 12 章就是解剖顺序。

## 3.4 视角三：一个 SDK 与平台

第三视角：DSH 是可以被别的程序"开走"的平台。三种嵌入方式，按由重到轻：

**TypeScript SDK**（`@deepseek-ai/dsh-sdk-client`）。你的 TS 程序起一个完整的 DSH 子进程，通过 stdio 上的 JSON-RPC 驱动它。核心 API 就一个类加一个方法（示例摘自 [packages/sdk/client/README.md](../../packages/sdk/client/README.md)）：

```ts
import { DeepSeekHarness } from '@deepseek-ai/dsh-sdk-client'

await using harness = new DeepSeekHarness({
  profile: 'sdk',
  provider: 'deepseek-official',
  model: 'deepseek-v4-flash',
})
const result = await harness.run('say hi')
console.log(result.finalResponse)
```

注意 `await using`：这个类实现了 `AsyncDisposable`，作用域结束自动收割子进程——SDK 层面也在贯彻"效果可逆"的框架纪律。SDK 家族本身也是三角色拆包：`packages/sdk/protocol`（JSON-RPC 协议定义）、`server`（服务器端）、`client`（客户端）。

`run()` 的一次调用背后（[client README](../../packages/sdk/client/README.md) 的行为契约）值得现在就细读，因为它预告了第 4、8 章的主题。子进程**懒启动**：`start()` 记忆化一次有界的 `initialize` 握手，携带工作目录、provider/model 路由、可选的 `reasoningEffort` 与 `maxTokens`。`run()` 把提示词入队，**等到它的 message id 出现在持久化收件箱（durable inbox）的回执里**，然后一直收集到下一次 whole-agent `idle`，返回 `RunResult { sessionId, finalResponse, events, notifications }`。一个容易踩的坑：`finalResponse` 是该区间内最后一条已提交的根会话助手文本，**不**是因果上专属于你这条提示词的回答——中途的 steering（转向插话）、注入上下文都可能贡献内容。`close()` 走一段 stdin-EOF → SIGTERM → SIGKILL 的升级阶梯且幂等。连"关进程"都写成了纪律——这是贯穿全书的态度。

**Python SDK**（`deepseek-harness-sdk`，模块名 `deepseek_harness`）。高层 API 与 TS 版对称：`DeepSeekHarnessConfig`、`DeepSeekHarness.run()`、`Session`。它的搭档 `deepseek-harness-runtime-bin`（[python/sdk-runtime/README.md](../../python/sdk-runtime/README.md)）把 `dsh` CLI 及其封闭 Node 依赖树打成原生可执行文件，发布 5 个平台 wheel（Linux x64/arm64、macOS x64/arm64、Windows x64），**宿主机不需要装 Node.js**。一个值得学的细节：每次启动必须显式传 `dsh_home`，Python 侧绝不静默读写 `~/.dsh`——SDK 不暗动用户全局状态，这是文档明文承诺的行为契约。

**协议服务器**。`dsh --profile acp` 起 ACP（Agent Client Protocol，标准 agent 客户端协议）服务器，automation-only——只面向自动化、不含人机交互面；`packages/webhook/` 提供经校验的外部事件触发会话（fire-and-forget 的 Workspace Session）；`headless` 的一次性运行支持 `--json`（NDJSON 事件流投影）与 `--session-id`（续接既有持久会话），适合接进任何调度系统。

`sdk` 与 `sdk-minimal` 的分工值得多说一句：前者组装 `dsh-base` 这棵大树（全量能力），后者是**不经 dsh-base 的独立最小树**——嵌入方只想要"模型 + 工具 + 会话"的最小闭环时用。这再次印证 3.5 节的逻辑：包是装载单位，profile 是装载方案，所以"最小"可以是真的小。

还有一条彩蛋级的线索：`packages/subagent/subagent-dsh-sdk/` 让一个 DSH 实例通过 SDK 把**另一个 DSH 实例**当子代理使唤。这叫吃自己的狗粮（dogfooding）——平台提供的嵌入能力，平台自己就是第一个重度用户。

## 3.5 关键数字与包的算术

现在直面那个吓退过很多人的数字。以下统计你在仓库根目录可以亲手复现：

| 统计项 | 数值 | 说明 |
|---|---|---|
| `packages/*/*/` 包目录 | **319** | `ls -d packages/*/*/ \| wc -l` |
| 包组（capability family） | 54 | `packages/` 一级目录，[packages/README.md](../../packages/README.md) 的表 |
| `vendor/` 包 | 9 | cordis、cosmokit、group、hmr、include、loader、logger-console、schemastery、timer |
| `apps/` | 4 | cli（`dsh` npm 包）、web（Vite 前端）、desktop（Electron）、desktop-host |
| `docs/` 根级英文页 | 22 | 每页配 `.zh.md` 与 `.i18n.yaml` 成三件套 |
| `docs/subsystems/` 英文页 | 64 | 一子系统一页，含 README |
| `docs/cookbook/` 顶层英文页 | 11 | 步骤手册 |
| 决策记录 | 229/37/8/925 | implemented / proposed / rejected / archived |
| 维护者技能 | 15 | `.agents/skills/` |
| 事故复盘 / Cordis 教程 | 4 篇 / 7 章 | 双语 |
| 门禁 / 生成器脚本 | 59 / 16 | `package.json` 里 `verify-*` 与 `gen-*` |

顺带把 workspace 的全貌说清：根 [package.json](../../package.json) 的 `workspaces` 字段是 `vendor/*`、`packages/*/*`、`native/system`（及其子包）、`apps/*`、`website`——所以除了 319 个能力包，还有 9 个 vendor 包、原生插件（`@deepseek-ai/node-addon-system`）、应用壳与文档站同处一个 pnpm workspace。`python/` 是例外：它有自己的构建体系，不进 workspace。构建也按"面（face）"切：`build:lib:host`（`tsc -b tsconfig.host.json` + tsdown，再捆桌面资源）与 `build:lib:client`（`tsc -b tsconfig.client.json` + tsdown）是两套显式编译面——同时有 Host 与 Client 程序的包各暴露一个叶子 tsconfig，根 solution 只做聚合；你跑 `pnpm run typecheck`，实际是先编 Host 面再编 Client 面。为什么分两套程序？浏览器代码与 Node 代码的目标运行时不同，第 2 章展开。

再看**组的规模分布**（一条 shell 就能复现：`for g in packages/*/; do find "$g" -mindepth 1 -maxdepth 1 -type d | wc -l; done` 配合排序）：

| 组 | 包数 | 一句话 |
|---|---|---|
| `client` | 63 | Web GUI 的浏览器半边（shell、wire、对象服务、slots、`ui-*` 插件）——UI 是这个仓库最重的资产 |
| `experimental` | 24 | 预稳定原型，默认公开、显式私有例外 |
| `session` | 20 | 持久化数据面，含四个版本迁移包 |
| `util` | 17 | 零依赖工具（`Branded<B>`、路径、超时、留存策略……） |
| `subagent` / `shell` | 10 / 10 | 见下文展开 |
| `llm` / `host` / `api` | 9 / 9 / 9 | 模型族 / GUI 宿主 / 远程 BFF 与 RPC 网关 |
| `core` | 8 | 产品 API 脊柱：`agent`、`agent-loop`、`session`、`tools`、`system-prompt` 等 |
| `test-support` / `fs` | 7 / 7 | 测试基建 / 文件系统族 |
| `web` / `skill` / `context` / `bundle` | 6 / 6 / 6 / 6 | 其余前列组 |
| 其余 38 组 | 共 102 | 长尾：平均每组不到 3 个包，一能力一组 |

两个观察送给你：其一，最大组竟是 `client`——"一切皆插件"连 UI 都算在内，浏览器半边自己就是一大片插件森林（第 13 章解剖）；其二，长尾占 38 组却只有约三成包——**多数能力是"接缝 + 一两个提供方 + 一两个工具"的小 trio**，大组才是例外。这再次印证：包数是架构规则的输出，不是管理成本的输入。

**为什么是 319 个包？** 这不是官僚分层，而是"一切皆插件"做乘法之后的算术结果。拆开看一个包组你就明白了。以 `packages/shell/` 为例，它有 10 个包：

| 包 | 角色 |
|---|---|
| `shell` | **接缝定义**：`dsh-shell` 只声明"执行一条 shell 请求"的服务接口（Service Definition），不含任何实现 |
| `bash-local` / `bash-sandbox` | **提供方**（Provider）：本机直跑 bash / 沙箱内跑 bash |
| `pwsh-local` / `pwsh-sandbox` | 提供方：同上，换成 Windows 的 PowerShell |
| `shell-env` | 提供方配套：环境变量拼装 |
| `tool-bash` / `tool-pwsh` | **消费方**（Consumer）：把执行能力包装成模型可见的 `bash`/`pwsh` 工具 |
| `tool-bash-persistent` / `tool-pwsh-persistent` | 消费方变体：常驻 shell 会话版本 |

看出来了吗？"执行 shell 命令"这一个人类概念，在"一切皆插件"下必须拆成**角色 × 平台 × 沙箱**的矩阵：接缝一个包，`{bash, pwsh} × {local, sandbox}` 四个提供方包，工具包装若干包。同样的算术随处可见——`fs/` 组 7 个包（`fs` 接缝、`fs-local`/`fs-sandbox` 提供、`fs-observation-policy` 策略、`tool-fs`/`tool-fs-search`/`tool-str-replace-editor` 消费）；`web/` 组 6 个包（`web` 接缝、`web-fetch-http` 抓取、**三个**搜索提供方 `web-search-deepseek`/`web-search-exa`/`web-search-perplexity`——同一接缝三家供应商，各自成包）；`llm/` 组 9 个包（`llm` 接缝、`llm-deepseek`/`llm-pi-ai` 适配器、`llm-retry`、`token-meter` 计量……）；`session/` 组 20 个包，光格式迁移就按版本各占一包（`session-format-v0-to-v1` 直到 `v3-to-v4`）；`subagent/` 组 10 个包，五种派发方式各占其位。把 54 个组都这么乘开，319 这个数字就一点也不神秘了。规模最大的是 `experimental/`（预稳定原型，公开默认、显式私有的例外）与 `test-support/`（测试基建）——探索与验证本身也按包管理。

**为什么值得各自成包，而不是一个大包里的 319 个目录？** 四个理由，每个都能在仓库里找到实证：

1. **接缝即包边界**：`dsh-shell` 的 `package.json` 依赖列表本身就声明了"谁依赖接缝、谁提供实现"，依赖方向静态可查——`pnpm run gen-module-graph` 生成的 [docs/module-graph.md](../../docs/module-graph.md) 就是全景图，CI 校验新鲜度。规则明文写在 packages/README.md：**扩展插件依赖 Service Definition，从不依赖具体 Provider**。
2. **按需装载**：`sdk-minimal` profile 不装 `dsh-base`，就真的不必把 319 个包拉进进程——包是装载单位，目录不是。
3. **独立演进与测试**：每个包自带测试与 README 契约（用途、配置、扩展点、已知限制都有门禁校验），换掉 `web-search-deepseek` 不需要重新测试 `web-search-exa`。
4. **发布节奏独立**：pre-stable 的 API 变化只需同步更新实际的消费者，消费者集合由包依赖图给出，不用全库搜字符串。

反过来问：**能不能合并成 50 个"大区"包？** 能，但你会立刻丢掉 1 和 2——依赖图退化成大区之间的粗粒度箭头，"只装接缝不装实现"变成空话。DSH 的选择是把包边界当架构表达来用。

## 3.6 两大支柱预览

两条支柱各给一个**可检验的推论**——科学课教过的：不能被观测推翻的命题不是好命题。

**推论一（没有特权核心）**：既然一切皆插件，你就应该能（a）用 `dsh --profile web --dump-config` 看到**整棵**运行树的每一行来处；（b）在不动 `packages/` 任何源码的前提下，用一个 YAML patch 替换或禁用某个官方能力；（c）观察到插件卸载后不留残渣。三条本章动手环节都能粗验。第 3 章讲 Cordis 如何用"插件 + 共享上下文（Context）"让每个注册（registration）都是可逆的效果（effect）——插件卸载时自动回收；第 5 章带你读 `ReactLoopAgent`，你会看到连"驱动模型的循环"也只是实现 `Agent` 接口的一个普通插件，理论上你可以在配置里换成自己写的。

**推论二（模型可见即可重建）**：既然每个请求都从日志推导，你就应该能拿着一份会话日志文件（`packages/session/` 的持久化产物，第 11 章讲格式）重放出模型当时收到的完整输入——包括 tool 结果、注入的上下文、系统提示。`packages/core/agent-loop/src/agent.ts` 的第一行注释就是这句承诺的代码形态：*"Every request is derived from the session log"*（每个请求都从会话日志推导而来）。第 8 章讲会话日志的事件词汇表。此刻记住推论：**调试一个 DSH agent 不需要"复现现场"，日志就是现场**。

## 源码走读

现在按顺序读六个真实文件。带上 [packages/README.md](../../packages/README.md) 当地图，全程只看不改。走读前约定三件事：一，**行号以撰写时的仓库为准**，它们会漂移，但符号名（`runCli`、`parseDshArgs`、`ReactLoopAgent`……）是稳定锚点，用编辑器的"转到符号"总能找到；二，每个片段不超过 30 行，截断处自己打开文件补全；三，读不懂的地方先记下来——它大概率正是后面某一章的标题。

**第 1 站：唯一入口 `apps/cli/src/bin.ts`（全文仅 78 行）**。`runCli()`（第 26–74 行）就是 `dsh` 命令的全部逻辑——注意它有多薄：

```ts
// apps/cli/src/bin.ts:26-31
export async function runCli(options: RunCliOptions = {}): Promise<void> {
  const version = getDshRuntimeVersion()
  const { manageDesktopProfile, ...profileOptions } = options
  const invocation = parseDshArgs(process.argv.slice(2), version, manageDesktopProfile)

  switch (invocation.mode) {
```

`switch` 的四个分支（`profile` / `plugin` / `dump-config` / `dump-config-schema`）每个都只 `await import()` 对应模块再委托，入口不含任何业务逻辑；兜底分支用 `invocation satisfies never`（第 71 行）让编译器证明判别联合已穷尽。第 76–78 行的 `if (import.meta.main)` 才是进程真身。**没有特权核心，从第一行就开始兑现**：入口是个调度员，不是皇帝。

**第 2 站：命令文法 `apps/cli/src/args.ts`（212 行）**。第 21–61 行定义了一个判别联合（discriminated union）`DshInvocation`，四种调用各是一个 interface，比如：

```ts
// apps/cli/src/args.ts:21-30
/** Boot a named profile and hand it the invocation's inner arguments. */
interface ProfileInvocation {
  mode: 'profile'
  profile: string
  fromDefaultProfile?: string | undefined
  patches: string[]
  args: string[]
}
```

`parseDshArgs()` 在第 146 行。文件头注释（第 4–11 行）讲清了一个精妙设计：启动器**只**解析自己的 flag（`--profile`、`--patch`、dump 系列与 `--from-default-profile`），其后的一切原样交给被组装的应用插件去解析——所以 `dsh --profile web -h` 打印的是 web 应用的帮助而不是启动器的。全仓库处理封闭联合以 `assertNever` 收尾，这是 [AGENTS.md](../../AGENTS.md) 的明文规范。

**第 3 站：分组表 `packages/README.md`（第 29–84 行）**。54 行表 = 54 个包组，每行一句话职责。以后找任何能力，先查这张表锁定组，再进组的 README 查包。把 `shell`、`fs`、`web`、`session`、`subagent` 五行多读两遍，对照 3.5 节的展开。表下方"Dependencies"一节的两句话值得背下来：依赖图是生成的；**扩展插件依赖 Service Definition，从不依赖具体 Provider**。

**第 4 站：Agent 循环 `packages/core/agent-loop/src/agent.ts`（688 行，本章只看骨架）**。整个 `src/` 目录只有 7 个文件，先扫一遍文件名再进主文件：`inbox.ts`（`ReactLoopInbox`，排队输入）、`runtime-context.ts`（`RuntimeContextProjection` 与 `SystemPromptProjection`，把日志投影成请求上下文）、`assistant-stream.ts`（`AssistantStreamAttempt`，流式响应尝试）、`tool-calls.ts`（`executeToolCalls`，工具调用执行）、`constants.ts`、`index.ts`，以及主角 `agent.ts`——第 1–5 行模块注释就写着 *"Default Agent driver over queued turns and step-boundary input. Every request is derived from the session log."* 支柱二从口号变成第一行代码注释。第 42–50 行的 `Phase` 类型是循环的状态机骨架：

```ts
// packages/core/agent-loop/src/agent.ts:42-50
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | {
    kind: 'maintenance'
    abort: AbortController
    lastTurn: number
    wakeRequested: boolean
  }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

`idle / maintenance / running` 三态，外加轮（turn）与步（step）两个计数、贯穿三态的 `abort` 与 `wakeRequested`——取消与插话是循环的一等公民。第 97–130 行的 `ReactLoopAgent implements Agent` 是实现主角——注意它 implements 的是一个来自 `dsh-agent` 包的接口：循环实现的是接缝，而非垄断接缝。构造器参数 `(loopCtx, id, options, session)` 告诉你它天生绑定一个 Session。细节留给第 5 章。

**第 5 站：headless 应用的参数面 `packages/bundle/headless/src/startup.ts`**。3.4 节说的 `--json` 与 `--session-id` 就定义在这里——注意它是个插件（第 16 行 `export const name = 'headless-startup'`，第 19 行 `inject = ['cmdlineArgs']`），不是启动器的一部分：

```ts
// packages/bundle/headless/src/startup.ts:43-51
  .option('--json', 'write newline-delimited run events to stdout instead of the final message')
  .option('--session-id <id>', 'adopt the persisted Session with this id; an unknown id is an error')
```

第 50–51 行的示例注释直接给了用法：`dsh --profile headless --json "run the tests"` 输出机器可读事件流、`--session-id session-…` 续接既有会话。启动器薄、应用插件厚——第 2 站的设计在这里落地。

**第 6 站：两张 SDK 门面**。TypeScript 侧 `packages/sdk/client/src/api.ts` 第 22 行：`export class DeepSeekHarness implements AsyncDisposable`——3.4 节示例的真身。Python 侧 `python/sdk/src/deepseek_harness/api.py`（模块目录只有 `__init__.py`、`api.py`、`client.py`、`errors.py`、`models.py` 五个文件，门面干净得近乎苛刻）：

```python
# python/sdk/src/deepseek_harness/api.py:14,31,49,124
@dataclass(slots=True)
class DeepSeekHarnessConfig:
    ...
    dsh_home: str | None = None

class DeepSeekHarness:
    ...
    def run(self, ...
```

`DeepSeekHarnessConfig` 里那个 `dsh_home` 就是 3.4 节说的"必须显式选择 Harness home"的落点。两个 SDK 的 API 形状几乎逐符号对称——这不是巧合，[AGENTS.md](../../AGENTS.md) 要求"两个 SDK 都投影循环"（Both SDKs project the loop）：循环或事件格式改动必须在同一个 PR 里同步两侧的预期输出。

## 常见误区

**误区一："319 个包 = 过度工程。"** 数量本身不说明任何问题，要看数字的来历。3.5 节拆给你看过：角色 × 平台 × 沙箱的乘法，加上"接缝独立成包"的边界纪律，乘出来就是这个量级。真正该问的问题是"依赖方向是否静态可查、能否按需装载"——两个答案都是肯定的（`module-graph.md` 与 `sdk-minimal`）。刚入门的同学容易把"包多"与"耦合多"划等号，实际恰好相反：包边界在这里是**解耦的执行机制**。

**误区二："agent 的核心就是一个 while 循环调 API。"** 那是教科书上的玩具。生产循环要处理：排队输入、步边界上的插话、取消（注意 `ReactLoopAgent` 的 `AbortController` 与 `Phase` 三态）、错误重试、以及"每个请求从日志推导"的纪律。第 5 章会精确展示 688 行都在为什么付费。

**误区三："`vendor/` 是普通的第三方依赖。"** 不是。它是 Cordis 及其配套（9 个包）的**源码固定拷贝**，被改写作用域（rescope）为 `@deepseek-ai/*` 私有包，钉在上游特定 SHA 上（清单见 [vendor/README.md](../../vendor/README.md)）。DSH 对框架不是"npm install 的使用者"，而是"带补丁的同行者"。第 2 章展开。

**误区四："没有 DEEPSEEK_API_KEY 就什么都学不了。"** 诚实边界在这里：真实模型调用确实需要 key（headless/SDK 的完整体验）。但 `pnpm run test:e2e` 无 key 自动跳过，`pnpm run test:snapshot` 是**无 key** 的录制会话回放，而且 `packages/test-support/llm-mock-server` 提供一个可编排的 Messages 兼容故障服务器（`pnpm run mock:llm`），能让你**不带 key 看到完整的模型调用回合**，包括限流、坏块、工具调用等脚本化行为。你可以先学完这本书的大半，再决定要不要花钱跑真模型。

**误区五："Makefile 和 package.json 是两套构建系统。"** Makefile 只有 5 个 target（`build`/`web`/`desktop`/`dev-web`/`dev-desktop`），头两行注释写明它只是 `package.json` scripts 的**别名**，真源永远是 scripts（`make help` 自己也这么说）。看到 `make web`，就去查 `"start:web"`。

**误区六："desktop 是第六个 profile。"** `desktop` 这个名字被 Electron 应用**保留**（CLI 会拒绝以它启动），它的运行时打包在签名资源里、由 `apps/desktop` 自治。五种可启动 profile 是 `web`/`headless`/`sdk`/`sdk-minimal`/`acp`，一个不多一个不少。

**误区七："根目录的 CLAUDE.md 是给别的工具的重复文档。"** 它是 `AGENTS.md` 的符号链接（symlink），`packages/` 下还有一个同样的——目的是让不同 agent 工具读到同一部仓库宪法。要改内容请编辑 `AGENTS.md` 真身；这条规则 AGENTS.md 自己写着（"edit the real file"）。

## 动手环节

先给你一张**命令速查表**（全部来自根 [package.json](../../package.json) 的 scripts；`make` 系列只是别名；"要 key?"指 `DEEPSEEK_API_KEY`）：

| 命令 | 作用 | 要 key? |
|---|---|---|
| `pnpm install` | 安装 pnpm workspace 依赖（Node ^22.19 或 ≥24） | 否 |
| `pnpm run build` | Host/Client 两个编译面的完整构建 | 否 |
| `pnpm dsh web` | 起 Web UI，默认 `127.0.0.1:3080` | 起服务否；对话要 |
| `pnpm dsh --profile headless "task"` | 一次性运行（源码经 tsx ESM hook 启动） | 是 |
| `pnpm run test` / `test:coverage` | 单元测试 / CI 覆盖门禁（逐文件 100%） | 否 |
| `pnpm run test:e2e` | 真 API 测试 | 是；无 key 自动跳过 |
| `pnpm run test:snapshot` | 录制会话的无密钥回放 | 否 |
| `pnpm run typecheck` / `lint` | 先编 Host 面再做类型检查 / oxlint | 否 |
| `pnpm run doc-sync` | 文档门禁聚合 | 否 |
| `pnpm run mock:llm` | 起 Messages 兼容的模拟 LLM 服务器 | 否 |

按顺序做，每步都有明确现象：

1. **亲手数包**（建立对 3.5 节的身体记忆）：
   ```sh
   ls -d packages/*/*/ | wc -l        # 319
   find packages -maxdepth 1 -mindepth 1 -type d | wc -l   # 54
   ls packages/shell/                 # 对照三角色拆分
   grep -c '"verify-' package.json    # 59 道门禁
   # 组规模分布（应看到 client 63、experimental 24、session 20 居首）：
   for g in packages/*/; do echo "$(find "$g" -mindepth 1 -maxdepth 1 -type d | wc -l) $(basename "$g")"; done | sort -rn | head
   ```
2. **装依赖并构建**（Node ^22.19 或 ≥24）：`pnpm install && pnpm run build`。首次构建较慢——它在跑 Host/Client 两个编译面（`tsc -b tsconfig.host.json` 与 `tsconfig.client.json`，各接一步 tsdown 打包；Host 面还要捆桌面资源）。
3. **起 Web UI**：`pnpm dsh web`（或 `make web`）。现象：浏览器打开 `http://127.0.0.1:3080`。试着发一条消息——没有配 `DEEPSEEK_API_KEY` 时观察它如何**响亮失败**（misconfiguration fails loud 是仓库明文规范），而不是转圈装死。
4. **看组装树，不启动**：`pnpm dsh --profile web --dump-config | head -60`。你看到的是 profile 的完整插件组合结果——支柱一的可视化：整棵树，行行来自配置层叠加。
5. **无 key 跑测试**：`pnpm run test:snapshot`。现象：录制会话在本地回放并通过——不需要任何密钥。
6. **无 key 看一个真回合**：开一个终端跑 `pnpm run mock:llm`（可编排的模拟 LLM 服务器），再把 `DEEPSEEK_BASE_URL` 指向它起一个 profile——你将看到一次不花一分钱的完整"模型请求→工具调用→结果回填"回合。细节读 [packages/test-support/llm-mock-server/README.md](../../packages/test-support/llm-mock-server/README.md)。
7. **（有 key 才做）一次性运行**：`pnpm dsh --profile headless "用一句话总结这个仓库的 README"`。现象：stdout 只打印最终答案（中间工具输出默认不打印）；加 `--json` 换成 NDJSON 事件流；换 `--session-id` 可以接着上一次继续。
8. **读一张生成图**：打开 [docs/module-graph.md](../../docs/module-graph.md)，找到 `dsh-shell`，数一数有几个包依赖它、它又依赖谁——3.5 节"接缝即包边界"的现场验证。
9. **读两篇"错题本"**：挑一篇 [docs/postmortem/](../../docs/postmortem/README.md)（建议 `0002-js-expression-disabled-filesystem-tools`）和一篇 rejected 决策记录，注意它们如何写"当时以为、后来发现、于是改为"——比任何教科书都接近真实工程。
10. **（可选）跑 Python 最小示例**：打开 `python/sdk/examples/minimal.py`，注意它如何显式指定 `dsh_home`（指向一个临时目录即可）并选择 `sdk-minimal` profile——3.4 节两条行为契约（显式 home、最小树）的现场对照。

## 本章小结

把本章压成十五条，贴在显示器边上（每一条都能在正文找到出处）：

1. Harness（挽具）是"模型之外、产品之内"的全部基础设施：工具执行、权限、记忆、循环、组装。
2. DSH 是全插件（all-plugin）的 Cordis agent harness：没有特权核心，一切可替换，组装即配置。
3. 支柱一：*"There is no privileged core to patch"*——连 agent 循环本身都是插件树上的普通节点。
4. 支柱二：*"Model-visible means logged"*——一切模型可见内容都是会话日志里的持久事件，任何请求可从日志重建。
5. 产品形态由 profile 决定：`web` / `headless` / `sdk` / `sdk-minimal` / `acp`，`desktop` 保留给 Electron；`dsh` 是唯一受门禁保护的 Node 应用启动器。
6. 一个回合是"日志→推导请求→工具执行→写回日志"的闭环；聊天界面只是它的投影。
7. 工具族与包一一对应（`bash`↔`tool-bash`、`web_search`↔`tool-web`……），完整清单在生成的 `tool-catalog.md`。
8. 安全默认：权限预设 `workspace-write`（工作区可写、外部受限、危险操作审批），显式弃权才升到 `danger-full-access`。
9. 学习资源密度罕见：64 页子系统参考、九份生成目录、59 道门禁脚本、229 篇已实施决策记录（外加 37 提议、8 否决、925 归档）、4 篇事故复盘、7 章 Cordis 教程，全部双语三件套。
10. 319 个包 = 54 组 ×（角色 × 平台 × 沙箱）的乘法；包边界就是接缝边界，依赖方向静态可查（`module-graph.md`）。
11. TS SDK 与 Python SDK API 逐符号对称（`DeepSeekHarness` / `run()` / `Session`）；Python runtime wheel 打包原生可执行，5 平台无需系统 Node，且强制显式 `dsh_home`。
12. `headless` 支持 `--json`（NDJSON 投影）与 `--session-id`（续接会话）；`subagent-dsh-sdk` 让 DSH 吃自己的狗粮。
13. `bin.ts` 入口只有 78 行且零业务逻辑；判别联合 + 类型级穷尽检查（`satisfies never` / `assertNever`）是全仓库处理封闭集合的统一姿势。
14. 无 `DEEPSEEK_API_KEY` 也能完成本仓库绝大部分学习：单测、快照回放、构建、门禁、甚至 `mock:llm` 模拟的完整回合，全部 keyless。
15. Makefile 只是别名，`package.json` scripts 是唯一真源。

## 必读源码回顾与延伸阅读

**本章提到的必读文件**（按出场顺序）：

- [README.md](../../README.md) — 项目门面与运行方式
- [AGENTS.md](../../AGENTS.md) — 仓库宪法：布局、命令、约定（值得整篇通读，之后每一章都会回头引用它）
- [docs/architecture.md](../../docs/architecture.md) — 两大支柱原文（第 9–13 行与第 127 行附近）
- [packages/README.md](../../packages/README.md) — 54 组分组表与"依赖接缝不依赖实现"规则
- [apps/cli/README.md](../../apps/cli/README.md) — 入口模式表、profile 与 bundle 机制
- [apps/cli/src/bin.ts](../../apps/cli/src/bin.ts)、[apps/cli/src/args.ts](../../apps/cli/src/args.ts) — 入口与命令文法
- [packages/core/agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts) — 循环骨架（只读头 130 行即可）
- [packages/bundle/headless/src/startup.ts](../../packages/bundle/headless/src/startup.ts) — 应用插件解析自身 flag 的样本
- [packages/sdk/client/README.md](../../packages/sdk/client/README.md) — TS SDK 的 `await using` 示例
- [python/README.md](../../python/README.md)、[python/sdk-runtime/README.md](../../python/sdk-runtime/README.md) — Python 双包与五平台 wheel
- [python/sdk/examples/minimal.py](../../python/sdk/examples/minimal.py) — 48 行看懂 SDK 的全部行为契约（显式 `dsh_home`、`sdk-minimal`、`run()`/`final_response`）
- [SAFETY.md](../../SAFETY.md)、[vendor/README.md](../../vendor/README.md) — 运行前安全须知；vendor 钉定清单与上游 SHA

**延伸阅读**：下一章 [第2章-分层架构-vendor-packages-apps的骨骼](第2章-分层架构-vendor-packages-apps的骨骼.md) 沿着本章的地图下潜一层，解剖 `vendor/` 的钉定与改作用域、`packages/` 的 54 组内部结构、`apps/` 三种产品壳如何共享同一棵插件树。如果你已经急着想知道"插件"到底在代码里长什么样，也可以先跳 [第3章-Cordis内核-服务依赖注入与上下文](第3章-Cordis内核-服务依赖注入与上下文.md)，再回第 2 章补地形。
