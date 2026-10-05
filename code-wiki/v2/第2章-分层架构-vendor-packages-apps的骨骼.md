# 第2章 分层架构：vendor、packages、apps 的骨骼

> 前置：你会 JS/TS、用过 ChatGPT，但从未读过 agent 框架源码。本章只需要第1章的全景印象，不需要任何 Cordis 细节。

## 本章导读

第1章你从远处看了一眼 DeepSeek Harness（下称 DSH）：一个"一切皆插件"的 agent 运行时。本章把镜头拉近到它的**物理结构**——打开仓库根目录，你会看到三个平行的世界：`vendor/`、`packages/`、`apps/`。这不是随手分的文件夹，而是整个仓库的骨骼：三种包有三种身份，身份决定它可以依赖谁、被谁依赖、以及它的类型如何影响别人。

本章回答三个问题，它们也是后续所有章节的地基：

1. **三种包的身份差异**：内嵌框架（vendor）、能力（packages）、产品（apps）各是什么，为什么必须区分？
2. **依赖方向的硬规则**：谁可以 import 谁？这些规则靠什么强制执行？
3. **类型如何跨包扩展**：两个互不相识的包，如何给同一个接口"添字段"（declaration merging）而不冲突？

读完本章，你应该能回答："bash 工具想调用 agent loop，为什么会被架构拒绝？"以及："我想给会话日志加一种新事件，类型上要动几个包？"

依赖注入、Context 的运行时机制留到[第3章](第3章-Cordis内核-服务依赖注入与上下文.md)；本章只讲**结构**，不讲机制。

## 从一个问题出发

假设你执行了 `npm install -g @deepseek-ai/dsh`，然后在终端敲 `dsh`——一个能读写文件、跑命令、调用模型的 agent 跑起来了。现在问：**这些代码住在哪？**

打开 [apps/cli/package.json](../../apps/cli/package.json) 你会发现，`@deepseek-ai/dsh` 这个 npm 包小得可疑：`bin` 只指向一个 `lib/bin.js`（L14-16）。agent loop 呢？bash 工具呢？Web 界面呢？都不在这个包里。

再追问三个更尖锐的问题，它们正是本章的引线：

- 为什么 Cordis 框架不直接 `npm install cordis`，而是把源码整份拷进 `vendor/`？
- 为什么 [packages/shell/tool-bash/package.json](../../packages/shell/tool-bash/package.json)（bash 工具）里，`@deepseek-ai/dsh-agent-loop` 只出现在 `devDependencies`（L47），而不在运行时依赖里？工具不驱动模型吗？
- [packages/core/session](../../packages/core/session/src/index.ts) 给 `Context` 接口加了 `sessions` 字段，[packages/shell/shell](../../packages/shell/shell/src/index.ts) 又给它加了 `shell` 字段——两个包改了"同一个"接口，TypeScript 为什么不报重复定义？

答案指向同一件事：**分层不是目录美学，而是依赖方向的工程纪律**。目录只是纪律的物理投影。

## 正文

### 3.1 三种包的身份：vendor=内嵌框架 / packages=能力 / apps=产品

"三种包"的严格定义不在任何文档里，而在 [pnpm-workspace.yaml](../../pnpm-workspace.yaml) 的 glob 里。DSH 是一个单仓库（monorepo），pnpm 用 workspace 把多个包联成一个可互相引用的整体：

```yaml
# pnpm-workspace.yaml:1-15
packages:
  - vendor/*
  - packages/*/*
  # The Landlock launcher is developed with its harness consumers but keeps
  # its native build and publication scripts under native/system.
  - native/system
  - native/system/packages/*
  # Product assemblies over the package tier; apps/cli owns the `dsh` bin.
  - apps/*
  # Private package owning repository-level benchmark dependencies.
  - benchmarks
  - website
  # Deploy root of the single-exe build: a pure dependency manifest whose
  # closure is what the exe bundles and what the Python runtime distributes.
  - python/sdk-runtime
```

三种身份对照：

| 目录 | npm 命名 | 身份 | 代表成员 |
|---|---|---|---|
| `vendor/*` | `@deepseek-ai/cordis` 等（重命名自上游） | 内嵌框架（vendored framework） | cordis、loader、schemastery……共 9 个 |
| `packages/<group>/<pkg>` | `@deepseek-ai/dsh-<name>` | 能力（capability） | core/session、shell/tool-bash、llm/llm…… |
| `apps/*` | `@deepseek-ai/dsh` 等 | 产品入口（product entry） | cli、web、desktop、desktop-host |

**vendor/*：仓库自己持有的框架层。**
[vendor/README.md](../../vendor/README.md) 开头一句话讲清动机：这些包是 Cordis 框架及其基础库的**源码拷贝**（pinned，即钉在确定的上游 commit 上），拷进本仓库而不是走 npm 依赖，是为了让 harness "fully owns its framework layer (auditable, patchable, pinned)"——可审计、可打补丁、版本钉死。9 个成员（L13-23 的清单表）：`cordis`（框架核心，上游 4.0.0-rc.7，钉在 commit `56b3d4f7…`）、`loader`（`cordis.yml` 加载）、`include`（entry 列表与 patch 算法）、`group`、`hmr`、`timer`、`logger-console`、`schemastery`（配置 schema）、`cosmokit`（基础工具）。

一个常见误解要当场纠正：**vendor 包不是 `private: true` 的私有包，它们公开发布**。因为 DSH 的每个能力包都把 `@deepseek-ai/cordis` 声明为对等依赖（peer dependency，见 3.3），发布能力包就必须连框架层一起发布；若沿用上游名字 `cordis`，会在 npm registry 上抢占一个不属于本项目的名字。所以全部改作用域（scope）`@deepseek-ai/*`，名字映射见 [docs/rescope.md](../../docs/rescope.md)。另外注意版本的"双层"结构：清单表记录的是**上游**版本（4.0.0-rc.7），而 `vendor/cordis/package.json` 自己的 `version` 字段（当前 4.0.5-alpha.1）是本仓库的发布版本——rescope.md L25 明确了这一区分。vendoring 的边界也画得干净：vendor 包的第三方依赖（`js-yaml`、`chokidar`、`picomatch` 等）仍留在 npm 上，`reggol`、`@cordisjs/utils` 等经验证未用的包刻意不拷（vendor/README.md L25-27）——拷贝是手段，不是越多越好。

**packages/*/*：能力的原子单位。**
每个包是一个 npm 包，命名 `@deepseek-ai/dsh-<name>`，物理路径 `packages/<group>/<pkg>/`。当前有 54 个组（组表见 [packages/README.md](../../packages/README.md) L29-84）、三百多个包（实测 319）：`core/`（session、agent、tools……）、`llm/`（适配器）、`shell/`、`fs/`、`web/`、`subagent/`、`boot/`、`client/`……每个包自带 README 三件套（`README.md` / `README.zh.md` / `README.i18n.yaml`）。能力族内部普遍遵循三角色模式——服务定义（Service Definition）声明接口、服务提供者（Service Provider）实现、消费者（Consumer）使用（通常是模型可调用的工具）——[第7章](第7章-工具系统-能力接缝与执行管线.md)专门展开，本章 3.3 只用它的依赖含义。

**apps/*：产品入口。**
- `apps/cli` 就是 npm 包 `@deepseek-ai/dsh`，`dsh` 命令本体。调用链：`src/bin.ts` 的 `runCli()`（L26-74）→ `src/args.ts` 的 `parseDshArgs()`（L146）→ 动态 import `src/profile-boot.ts` 的 `runProfile()`（bin.ts L33-35）。
- `apps/web`（`@deepseek-ai/dsh-web-frontend`）是 Vite 壳：[src/main.ts](../../apps/web/src/main.ts) 在浏览器环境下只做一件事——找到 `#root`、`new AppWebEntry(el)`、`entry.run()`（L18-37）；桌面环境下多一段 boot 注入握手。
- `apps/desktop` 是 Electron 壳（[src/main.ts](../../apps/desktop/src/main.ts) 的 `main()` L315 起，管窗口、更新、Host 子进程）；`apps/desktop-host` 是它拉起的私有 Host。
- 其余 workspace 成员各有专职：`native/system`（Landlock 启动器）、`benchmarks`、`website`、`python/sdk-runtime`（单文件可执行构建的部署根）。

给你一张防迷路的对照表——同一个词 "web" 在三层里各有所指，初学者最容易在这里混淆：

| 路径 | 是什么 |
|---|---|
| `apps/web` | 产品：Vite 构建的浏览器入口（`@deepseek-ai/dsh-web-frontend`） |
| `packages/web/` | 能力组：web 搜索/抓取接缝与工具（`dsh-web`、`dsh-tool-web`、`dsh-web-search-exa`……） |
| `packages/client/web/` | 客户端运行时：浏览器侧插件装配，`AppWebEntry` 就住在这里（`dsh-client-web`） |

**为什么否决替代方案？** 把所有代码并成一个 `dsh` 大包，会失去"可替换性"——DSH 的核心主张是 agent loop 本身也是插件、可从配置换掉（见 [docs/architecture.md](../../docs/architecture.md)），前提就是能力按包粒度拆开。而框架不走 npm、不 fork 成独立仓库，理由见 3.6。还有一条硬约束兜底：仓库禁止包自带可执行入口，只有 `dsh` profile 能启动 Node 应用（`scripts/verify-application-entrypoints.ts` 强制），所以"apps"这一层在物理上也是唯一的启动出口。

### 3.2 workspace 的物理布局与包命名

组的权威清单在 [packages/README.md](../../packages/README.md)：每个包恰好属于一个组，新包进现有组，新组要同时更新组 README 和总表（L27）。命名本身就是角色说明，读包名即可预判内容：

| 命名模式 | 角色 | 例 |
|---|---|---|
| `tool-*` | 模型可调用的工具（Consumer） | `dsh-tool-bash`、`dsh-tool-fs` |
| `*-local` / `*-ssh` / `*-sandbox` | 同一接缝的不同 Provider | `dsh-bash-local`、`dsh-fs-ssh`、`dsh-sandbox-local` |
| `session-format-vN-to-vN+1` | 单步会话格式迁移（[第11章](第11章-会话管理-持久化格式迁移与分叉.md)） | `dsh-session-format-v1-to-v2` |
| 裸名 | 接缝的 Service Definition | `dsh-agent`、`dsh-shell`、`dsh-fs` |
| 裸名 + `-loop` 等实现后缀 | 接缝的默认实现 | `dsh-agent-loop`（与 `dsh-agent` 契约/实现分离，3.3 主角） |
| `test-support/`、`util/` | 支撑层：测试基础设施、零依赖工具 | `dsh-agent-loop-testkit`、`dsh-brand` |

一个精妙的布局约定：**包目录名跨组唯一**。[tsconfig.base.json](../../tsconfig.base.json) 用一条通配符把所有 `@deepseek-ai/dsh-*` 映射到各自源码（L185-190 的注释解释了为什么一条 wildcard 就够、以及为什么聚合配置仍需显式列出）。这意味着加包不需要改根 tsconfig——目录结构自身就是解析表。

包之间的引用版本有两种协议，混用会被门禁拒绝：

- DSH 包之间用 `workspace:*`：开发时解析到 workspace 内的本地包，发布时锁定为精确版本；
- 引用 vendor/native 包用 `workspace:~`：[vendor/README.md](../../vendor/README.md) L5 给出理由——发布范围允许同一 minor 内的补丁升级（published ranges permit patch updates within the same minor version），因为 vendor 层可能带着紧急补丁独立小步发布。

这条规则不是靠自觉，而是写在 [scripts/verify-package-dependencies.ts](../../scripts/verify-package-dependencies.ts) 里的机器逻辑（L23-25）：

```ts
function workspaceRange(name: string): 'workspace:*' | 'workspace:~' {
  return name === '@deepseek-ai/dsh' || name.startsWith('@deepseek-ai/dsh-') ? 'workspace:*' : 'workspace:~'
}
```

最后记住一个习惯：**别背依赖，查图**。`docs/module-graph.md` 是 [scripts/gen-module-graph.ts](../../scripts/gen-module-graph.ts) 从所有 manifest 生成的依赖图（CI 校验其新鲜度），它像一张地铁线路图——你不需要记住每一站，只需要会查。

### 3.3 依赖方向的四条铁律与门禁

**铁律一：框架是 peer，不是硬依赖。**
看 [packages/shell/tool-bash/package.json](../../packages/shell/tool-bash/package.json) L29-48（节选）：

```json
"peerDependencies": {
  "@deepseek-ai/dsh-agent": "workspace:*",
  "@deepseek-ai/dsh-shell": "workspace:*",
  // ……（其余 dsh-* 同型，略）
  "@deepseek-ai/cordis": "workspace:~"
},
"dependencies": {
  "@deepseek-ai/schemastery": "workspace:~"
},
"devDependencies": {
  "@deepseek-ai/dsh-agent": "workspace:*",
  "@deepseek-ai/dsh-agent-loop": "workspace:*",
  // ……（测试用具，略）
}
```

`@deepseek-ai/cordis` 在 peerDependencies（以及 devDependencies，供本地测试），**从不在 dependencies**。为什么？peer 的语义是"我要用 cordis，但请让我和宿主共享**同一个实例**"。Cordis 的 Context、事件总线、服务注册表都是进程内单例——如果 tool-bash 自带一份 cordis，就会出现两个 Context、两套事件，插件树直接裂成两半。谁最终"落位"这份 peer？装配根：`apps/cli` 的 `dependencies` 里赫然有 `@deepseek-ai/cordis`（其 package.json L23）。能力包声明需求，apps 满足需求——这正是 peer dependency 的教科书用法。

**铁律二：版本协议分治。** DSH 互引 `workspace:*`，vendor/native 用 `workspace:~`，理由见 3.2，机器强制见 `verify-package-dependencies.ts`。

**铁律三：依赖契约，不依赖实现。**
`packages/core/agent`（`dsh-agent`）声明 `Agent` 接口、活动注册表和 `agent/*` 事件；`packages/core/agent-loop`（`dsh-agent-loop`）是**唯一默认实现**。[packages/README.md](../../packages/README.md) L100 说得直白："Extension plugins depend on Service Definitions, never concrete providers. `dsh-agent-loop` is swappable; UI, hook, and tool plugins use `dsh-agent`."

回看开篇第二个问题：tool-bash 的 devDependencies 里有 `dsh-agent-loop`，但那只是测试需要；运行时它只 peer 依赖 `dsh-agent`（上例 JSON 第一行）。它拿到的 `Agent` 类型来自契约包（[tool-bash/src/index.ts](../../packages/shell/tool-bash/src/index.ts) L21：`import type { Agent } from '@deepseek-ai/dsh-agent'`）。于是把 agent-loop 换成别的实现，bash 工具毫发无伤。

同一条铁律的推论：**仓库里约一半子系统是循环事件的纯消费者**——`llm-retry`、`compaction-basic`、`plan-mode`、`goal-round-driver`、`session-checkpoint-policy`、`repeat-tool-reminder`、`workspace-changes`、`hooks-claude-code`、`hooks-codex`……它们完全不 import 循环代码，只监听 `agent/*` 等事件（[第4章](第4章-事件驱动-五路派发与三个事件域.md)展开事件域）。工具族（shell/fs/web/lsp……）只依赖 core 的接缝（`ctx.tools`、`ctx.shell`），同样不碰 agent-loop。

**铁律四：产品面只经服务面消费内核。**
client/host/api/sdk 这些包不直接伸手进循环内部，而是通过 Remote/服务接口（JSON-RPC、控制器）消费内核。这让 Web、Desktop、SDK 三张产品脸共享同一套内核语义（[第13章](第13章-产品面-CLI-Web-Desktop-SDK.md)）。

**门禁（gate）：纪律的执行者。**
方向规则不是 code review 里的口头约定，而是一组静态检查脚本，CI 必跑：

| 脚本 | 检查什么 |
|---|---|
| [verify-package-dependencies.ts](../../scripts/verify-package-dependencies.ts) | 依赖分区与版本协议（`workspace:*` vs `workspace:~`），可自动修复 |
| [package-graph.ts](../../scripts/package-graph.ts) | workspace 包图的发现与排序，供各生成器共用 |
| [verify-runtime-closure.ts](../../scripts/verify-runtime-closure.ts) | 可执行部署清单覆盖 preset 引用的每个插件和每个必需 peer |
| [verify-npm-install-layout.ts](../../scripts/verify-npm-install-layout.ts) | 发布包的安装布局 |
| [verify-cordis-config.ts](../../scripts/verify-cordis-config.ts) | `cordis.yml` 里裸插件名必须出现在其 resolver manifest 的 dependencies 中——防止配置引用了一个"装不上"的插件 |

为什么用静态门禁而不是人的警觉？因为依赖方向是**可机械判定的不变量**，而"每个 reviewer 都记得规则"不是。试着预演三种违规的结局：把 `@deepseek-ai/cordis` 塞进某个能力包的 `dependencies`——`verify-package-dependencies.ts` 的分区检查直接改回/报错；在 `cordis.yml` 里写一个 resolver manifest 没声明的裸插件名——`verify-cordis-config.ts` 拒绝；某个发布 preset 用到的 peer 没进部署清单——`verify-runtime-closure.ts` 在打包前拦下。这是本仓库反复出现的方法论：把纪律写成可执行检查（misconfiguration fails loud，配置错误要响亮地失败在加载时，而不是悄悄运行）。

### 3.4 类型如何跨包流动：declaration merging 三例

开篇第三个问题在这里回答。TypeScript 的**声明合并**（declaration merging）允许对同名接口/同名模块的多处声明被编译器合并成一个——特例是**模块增强**（module augmentation）：`declare module '某个包' { interface X { … } }` 给那个包已导出的接口追加成员，而不修改它的源码。你可以把它想象成给一本公共字典添词条：字典只有一个，每个包往里写自己的词条，谁也不复印整本字典。

合并的规则要精确掌握，这是后续所有章节读源码的语法基础：不同名的成员直接累积（`Context` 同时有 `sessions` 和 `shell`）；**同名成员的类型必须一致**，否则编译错误——所以 `sessions` 这个键全仓库只能有一种类型，由 session 包独占所有权；同名的方法签名则形成重载。整个过程纯编译期，`declare module` 块不产生任何 JS 代码。

**例一：服务键——`interface Context`。**
[packages/core/session/src/index.ts](../../packages/core/session/src/index.ts) L38-41：

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    sessions: SessionStore
  }
```

同样的句式出现在 [packages/shell/shell/src/index.ts](../../packages/shell/shell/src/index.ts) L30-34（`shell: ShellExecutor`）和 [packages/core/agent/src/index.ts](../../packages/core/agent/src/index.ts) L27-31（`agents: AgentRegistry`）。三个包、三次增强、零冲突——编译后这些声明完全消失，**运行时没有任何对应物**；运行时 `ctx` 是个 Proxy，按属性键查服务表（`ctx.sessions` 命中 `sessions` 键），机制在[第3章](第3章-Cordis内核-服务依赖注入与上下文.md)。类型层与运行时层是两条独立但平行的线：类型靠 declaration merging 汇合，运行时靠服务注册表汇合。

**例二：事件名——`interface Events`。**
同一个 session 包，L43-83 给 Cordis 的 `Events` 接口合并了四个事件（节选）：

```ts
  interface Events {
    // ……（JSDoc 略）
    'session/created'(this: Scoped<Session>, session: Session): void
    // ……（session/disposed、session/event 同型，略）
    'session/flush'(this: Scoped<Session>, session: Session): Promise<void> | void
  }
}
```

注意签名上方的 JSDoc 带 `@mode emit` / `@mode parallel`（L52、L81）——这是仓库级约定：事件文档必须标注派发模式与载荷参数。`session/created` 是同步 emit（抛错可否决创建并回滚），`session/flush` 是被 await 的并行检查点。还有一类 waterfall 事件，监听器必须调用 `next()` 才能把链传递下去——语义细节留给第4章，此处只需记住：**事件名和它的类型签名，由事件的生产者包通过 declaration merging 声明，全仓库共享**。下游写 `ctx.on('session/created', (session) => …)` 时，参数类型自动来自这条合并声明——事件的消费不需要 import 生产者的任何运行时代码。

**例三：会话事件载荷——`interface SessionEventMap`。**
这是 DSH 里最重要的一张可扩展类型表。[packages/core/session/src/types.ts](../../packages/core/session/src/types.ts) L281-428 定义了 `SessionEventMap`——每种会话事件名映射到它的载荷类型：

```ts
// packages/core/session/src/types.ts:288-301（节选，JSDoc 略）
'turn/start': { turn: number }
'turn/end': { turn: number; reason: TurnEndReason }
'step/start': { turn: number; step: number }
'step/end': { turn: number; step: number }
```

此外还有 `'user/message'`、`'developer/message'`、`'system/message'`、`'assistant/message'`、`'assistant/attempt'`、`'tool/call'`、`'tool/result'`、`'request/header'`……它们是会话日志的全部词表（[第8章](第8章-消息系统-会话日志即模型记忆.md)）。这张表是**可扩展映射**（merge-extensible map）：别的包可以继续往里合并新事件。[packages/compaction/compaction/src/types.ts](../../packages/compaction/compaction/src/types.ts) L17-24 就是实例——上下文压缩包（[第10章](第10章-上下文压缩-当对话太长怎么办.md)）合并了四个 `compaction/*` 事件：

```ts
declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    /**
     * Marks the start of a compaction — log-only, holds the lock until
     * `compaction/end`. A numbered owner is strictly enclosed by that open turn;
     * `null` identifies a standalone manual transaction between turns.
     */
    'compaction/start': { compactionId: CompactionId; sourceCommandId?: CommandId; turn: number | null }
    // ……（compaction/summary、compaction/end、compaction/prune，见 L34-89）
```

合并之后，`SessionEventType = keyof SessionEventMap`（types.ts L430-431，注释明言 "plugin-merged extensions included"）自动包含新事件——会话日志的追加 API、校验、重放全部立刻认识 `compaction/start`，**session 包本身一行未改**。这就是"给会话日志加一种新事件"的完整答案：一个 `declare module` 块，零个核心包改动。

**可扩展联合 vs 封闭联合：两条配套纪律。**
类型表不是无限开放的。仓库约定（根 [AGENTS.md](../../AGENTS.md)）区分两类判别联合（discriminated union）：

- **merge-extensible（可扩展联合）**：如 `TurnEndReasonMap`（types.ts L201-229）——"为什么这个 turn 结束了"的变体表（`completed`/`aborted`/`blocked`/`error`/`max-tokens`/`interrupted`/`forked`），插件可以合并新变体进去（L231-232 的 `TurnEndReason = TurnEndReasonMap[keyof TurnEndReasonMap]` 取并集）。对它做 switch 时**必须**走一个写明文档的 default 分支，因为编译器不知道还有谁会扩展它。
- **closed（封闭联合）**：语义上穷尽、不打算被扩展的联合，switch 必须以 `assertNever`（穷尽性断言）收尾——少处理一个 case 就是编译错误。

配套的持久化纪律：`SessionEventMap` 成员默认 required-on-read——不认识某事件类型的构建会拒绝读取该日志，除非事件带 `ignorable: true` 信封；只有结构性格式变更才提升 `SESSION_FORMAT_VERSION`。可扩展性与向前兼容是被一起设计的（[第11章](第11章-会话管理-持久化格式迁移与分叉.md)）。

**为什么否决替代方案？** 用"中央 d.ts 声明所有服务与事件"会把所有权搅浑：session 的事件凭什么写在别人的文件里？用继承（`interface MyContext extends Context`）会造出无数个互不相容的 Context 变体。用泛型注入（`Context<S, E>`）会让类型参数传染整个调用图。declaration merging 的赢面在于：**所有权归生产者，可见性归所有人，核心包零改动**。

### 3.5 source plane 与 artifact plane、双编译面

仓库有一条容易忽略但对日常开发影响巨大的规则（根 [AGENTS.md](../../AGENTS.md)）：**源码面（source plane）与构建产物面（artifact plane）永不混合**。静态门禁（测试、typecheck、文档链接检查）把 workspace import 经 tsconfig 的 `paths` 解析到**源码 `src/*.ts`**，因此在一棵从未构建过的干净树上也能全绿；需要消费构建产物 `lib/` 的门禁必须显式声明这一依赖。禁止"只在 built 树上通过"的配置——否则你无法区分"代码是对的"和"上次构建的残留是对的"。

实现就在 [tsconfig.base.json](../../tsconfig.base.json)：L27-29 的注释写明 "Source-level resolution for every repo-local graph"，`paths` 把每个包名指到源码，例如 `"@deepseek-ai/cordis": ["./vendor/cordis/src"]`（L82）、`"@deepseek-ai/dsh-session": ["./packages/core/session/src"]`（L426）。

第二个要点是**双编译面**。看根 [tsconfig.json](../../tsconfig.json) 全文（15 行）：

```json
{
  // Solution file: the whole-repo graph for `tsc -b tsconfig.json` and the
  // tsserver entry. … `files: []` keeps it program-less, so the host/client
  // cordis Context merges never meet. …
  "extends": "./tsconfig.base.json",
  "files": [],
  "references": [
    { "path": "./tsconfig.host.json" },
    { "path": "./tsconfig.client.json" }
  ]
}
```

它是**解文件**（solution file）：自身 `files: []` 不编译任何东西，只通过项目引用（project references）指向两个聚合（aggregate）——[tsconfig.host.json](../../tsconfig.host.json)（Node 侧：apps、packages、scripts、tests）和 [tsconfig.client.json](../../tsconfig.client.json)（浏览器侧：packages/client 的测试与基准）。为什么要劈成两面？host/client 两份注释（tsconfig.host.json L2-4、tsconfig.client.json L2-6）给出了同一个理由：**两侧对 cordis `Context` 的同名键做了不同的 declaration merging**——host 侧的 `sessions` 是服务实现，client 侧是远程代理。一个 TypeScript program 不可能同时接受同一键的两种合并，所以两套 program 各自持有自己的合并结果，共享的叶子包（session、llm、tools……）各建一次、被两边引用。仓库术语是 "face-specific leaf configs, solution-only root"（[docs/development.md](../../docs/development.md) 的 TypeScript project layout 一节）：有 Host/Client 双面的包暴露 `tsconfig.host.json`/`tsconfig.client.json` 叶子配置加一个 solution-only 根；普通包一个 tsconfig 即可。

落到单个包，[packages/AGENTS.md](../../packages/AGENTS.md) 给出可操作的细则：包 tsconfig 继承 `tsconfig.base.json`（Client 面继承 `tsconfig.base.client.json`），设 `rootDir: src`、`outDir: lib/types`，用项目引用（project references）指向 workspace 依赖，并注册进两个聚合之一；`src/types.ts` 只放类型不放运行时代码——3.4 的 `SessionEventMap` 之所以住在 `types.ts`，正是为了让别的包能零运行时成本地类型导入并合并它；测试放在包级 `tests/`，而非 `src/__tests__/`。

### 3.6 vendored 框架的修改纪律：pinned + 补丁日志

vendoring 最大的风险是"拷贝漂移"：本地改了 300 行，三个月后上游发新版，谁还记得改了什么、为什么改？DSH 的回答是一套**修改纪律**：

1. **清单表**（[vendor/README.md](../../vendor/README.md) L13-23）：每个 vendor 包记录上游仓库名、上游版本、**精确的源码 commit SHA**。`cordis` 钉在 `56b3d4f725681cf4556c1a8695a709cc3b6eed74`。要回答"我们的 cordis 和上游差多少"，看表即知。
2. **详尽的本地修改日志**（L29-59，当前 **23 条**）：文件开头明令 "Keep this log exhaustive — every divergence from upstream must be listed"（每一处分歧都必须列出）。条目不是一句话了事，而是写清改了什么、为什么、上游问题链接、由哪些测试覆盖。举四例感受其颗粒度：
   - `include` 的 patch 算法被抽成导出的纯函数 `applyEntryPatches`（L43），让 `dsh --dump-config` 无需启动插件树就能复用同一算法，杜绝工具与框架的算法漂移；
   - entry 的 `disabled` 字段支持 `!!js` 表达式插值（L50），且是唯一允许插值的元数据字段；
   - `cordis/src/fiber.ts` 生命周期加固（L38）：封堵三个可重入销毁缺口；
   - 跨四个包的 volatile config 支持（L57）：让携带引用的配置字段在相等比较时被跳过。
3. **同步流程**（L61-69）：更新一个 vendor 包固定五步——记上游 SHA → 拷 `src/` → **重放或退役**上述日志条目（无论哪种都更新日志）→ 更新清单表的版本与 SHA → 根目录 `pnpm install && pnpm run test && pnpm run build`。
4. **改名自动化**：rescope 由 [scripts/rescope-vendor.ts](../../scripts/rescope-vendor.ts) 机器执行（`pnpm run rescope-vendor --apply`），同步后重放，禁止手改名字。

Rescope 的边界同样值得学（[docs/rescope.md](../../docs/rescope.md) L23-31）：改名只动 manifest 的 `name`、依赖键和 import 说明符，**不动**目录名（`vendor/hmr/` 仍叫 `hmr/`）、依赖版本范围、Loader 的 `cordis:` 协议前缀（`cordis:include` 是协议名不是包名）、上游运行时标识符（如 Schemastery 的 `Symbol.for('schemastery')`），以及 `docs/` 之外的既有散文。改动面被刻意压到最小——每多改一样东西，与上游的 diff 就多一份对账成本。

为什么不 fork 成独立仓库？因为本地修改与 harness 的测试、门禁需要**原子地共同演进**——fiber 生命周期加固的验证测试就住在 `packages/boot/app-boot/tests/` 里，跨仓协调会让每次修改变成一次发版仪式。单仓 vendoring 用"拷贝的廉价"换"演进的原子"，代价（同步成本）被日志和 SHA 严格定价。

## 源码走读

场景：模型决定调用 bash 工具。我们从消费者出发，沿着 import 链走到框架基类，亲眼看一遍"包边界=接缝边界"。先给出全程地图：

```text
[Consumer]              [Service Definition]            [Provider]              [Service Definition]
tool-bash    ──peer──→  shell（ctx.shell）      ←──实现──  bash-local    ──peer──→  subprocess（ctx.subprocess）
 模型面 bash 工具        ShellExecutor 抽象类             本地 bash 执行器         SubprocessRuntime 抽象类
                                                                                      │ 实现
                                                                                      ↓
                                                                            subprocess-local（本地进程树）

     ShellExecutor 与 SubprocessRuntime 的共同父类：Service（vendor/cordis/src/service.ts:11）
```

**第 1 站 · Consumer：`packages/shell/tool-bash`。**
[src/index.ts](../../packages/shell/tool-bash/src/index.ts) 的模块注释自报家门（L2）："Model-facing Consumer of the `ctx.shell` capability seam." 它的类型进口只有契约：

```ts
// packages/shell/tool-bash/src/index.ts:21,28-29（相邻节选）
import type { Agent } from '@deepseek-ai/dsh-agent'
import { DSH_ENV_PREFIX } from '@deepseek-ai/dsh-shell'
import type { ShellExecRequest, ShellExecSpec, ShellExecution, ShellRunResult } from '@deepseek-ai/dsh-shell'
```

运行时它只通过 `ctx.shell` 调用，从不 import 任何 executor 实现：

```ts
// packages/shell/tool-bash/src/index.ts:527
const foreground = await ctx.shell.execute(ctx.shell.resolve({ ...request, signal: exec.signal }))
```

注意 `resolve` → `execute` 的两段式：先把调用者的松散请求解析成完整规格，再执行——这正是根 AGENTS.md "defaulting is an explicit `resolve(request): Spec` step" 约定的实例。

**第 2 站 · Service Definition：`packages/shell/shell`。**
[shell/src/index.ts](../../packages/shell/shell/src/index.ts) 声明接缝本身：

```ts
// packages/shell/shell/src/index.ts:30-34
declare module '@deepseek-ai/cordis' {
  interface Context {
    shell: ShellExecutor
  }
}
```

```ts
// packages/shell/shell/src/index.ts:64-67
export abstract class ShellExecutor extends Service {
  constructor(ctx: Context) {
    super(ctx, 'shell')
  }
```

抽象类只规定两个抽象方法：`resolve(request): ShellExecSpec`（L84）和 `execute(spec): Promise<ShellExecution>`（L93），并把大量行为契约写在 JSDoc 里（`result` 只在基础设施故障时 reject、非零退出码是正常结算……L49-62）。`dsh-shell` 的 package.json 里没有任何 executor 依赖——**定义不知道实现的存在**。

**第 3 站 · Service Provider：`packages/shell/bash-local`。**
本地实现 [bash-local/src/index.ts](../../packages/shell/bash-local/src/index.ts) 自述（L91）："Local bash executor over `ctx.subprocess`." 它把进程创建委托给下一个接缝：

```ts
// packages/shell/bash-local/src/index.ts:259
running = this.ctx.subprocess.spawn(this.spawnSpec(spec, argv, spec.stdoutMaxBytes, spawnSignal))
```

**第 4 站 · 又一层 Service Definition：`packages/subprocess/subprocess`。**
[subprocess/src/index.ts](../../packages/subprocess/subprocess/src/index.ts) 用同样的句式声明 `ctx.subprocess`（L82-83）并导出抽象类 `SubprocessRuntime extends Service`（L117）；本地进程树由同组的 `subprocess-local` 实现，SSH 侧由 `packages/ssh/subprocess-ssh` 实现——**换一个 subprocess provider，bash 工具就从本机搬到了远程沙箱**，上层零改动。

**第 5 站 · 框架基类：`vendor/cordis`。**
`ShellExecutor` 与 `SubprocessRuntime` 共同的父类来自 [vendor/cordis/src/service.ts](../../vendor/cordis/src/service.ts) L11：`export abstract class Service<out T = never>`。你终于踩到了 vendor 层的地板：注册、生命周期、销毁语义都在这里（第3章细讲）。

走读结论：这条链上每一站都是一个包边界，同时也是一个角色边界——Consumer → Definition → Provider → Definition → Provider → 框架。`tool-bash/package.json` 的 peerDependencies（`dsh-shell`）就是第 1→2 站的物理投影；第 3 站之后各层同理。**依赖方向沿着接缝单向流动，从不逆行。**

## 常见误区

1. **"vendor 包是私有包，不发布。"** 错。它们以 `@deepseek-ai/*` 作用域公开发布（vendor/cordis/package.json 带 `publishConfig.access: public`），原因恰恰是能力包把框架声明为 peer——发布能力包必须连框架层一起发（[docs/rescope.md](../../docs/rescope.md)）。"private" 的是 `benchmarks` 这类仓库内成员。
2. **"要用 Agent 就得 import agent-loop。"** 错，且是方向性错误。`dsh-agent-loop` 是默认实现，消费者应依赖契约包 `dsh-agent`（接口与 `agent/*` 事件）。tool-bash 的 package.json 就是标准答案：`dsh-agent` 在 peer，`dsh-agent-loop` 只在 dev（测试用）。依赖了实现，就失去了换掉它的资格。
3. **"declaration merging 修改了 cordis 的代码/有运行时开销。"** 错。它是纯编译期模块增强，`declare module` 块不产生任何运行时代码；前提是你的文件确实（直接或间接）加载了被增强模块的类型。运行时的 `ctx.sessions` 能用，靠的是服务注册表 + Context 代理，与类型合并是两套平行机制。
4. **"tsconfig 的 `paths` 是给 Node 运行时用的。"** 错。`paths` 只影响类型检查与源码模式工具链；运行时解析靠 pnpm 的 workspace 符号链接（开发）或构建产物 `lib/`（发布）。把两者混为一谈，就会出现"类型检查过了、运行时 import 报错"的错位。
5. **"依赖方向靠 reviewer 把关。"** 错。分区协议、运行时闭包、cordis.yml 引用可解析性都是脚本门禁（3.3 的表格），CI 强制；gen-module-graph 的新鲜度本身也被 CI 校验。方向违规在合并前就会响亮失败。
6. **"`SessionEventMap` 是 session 包的私有类型，加事件要改核心。"** 错。它是 merge-extensible map，插件在自己的 `src/types.ts` 里 `declare module '@deepseek-ai/dsh-session/types'` 即可合并（compaction 包已示范）。但要知道纪律边界：新成员默认 required-on-read，不认识它的旧构建会拒读日志。

## 动手环节

以下练习在你的克隆里做，均不需要修改仓库文件：

1. **数三种包**：`ls vendor`（9 个包目录，另有 README/AGENTS）、`ls -d packages/*/*/package.json | wc -l`（三百多个）、`ls apps`（cli、desktop、desktop-host、web）。再打开 [pnpm-workspace.yaml](../../pnpm-workspace.yaml) 对上号：哪些 glob 对应哪种身份？
2. **查图不背图**：打开 [docs/module-graph.md](../../docs/module-graph.md)，在 mermaid 图里找到 `tool-bash` 节点，确认它的 peer 边指向 `shell`、`agent` 而不是 `agent-loop`；再找到 `bash-local` → `subprocess` 的边。然后试试 `pnpm run gen-module-graph`，理解"生成物 + 新鲜度门禁"的工作方式。
3. **验证铁律一**：随机抽三个 `packages/*/*/package.json`，检查 `@deepseek-ai/cordis` 是否只出现在 peerDependencies/devDependencies；再看 [apps/cli/package.json](../../apps/cli/package.json) 的 dependencies——谁最终落位了框架？
4. **亲眼看见 declaration merging**：在编辑器里打开 [packages/core/session/src/index.ts](../../packages/core/session/src/index.ts)，跳到 L38 的 `declare module` 块；然后打开任意一个 import 过 `@deepseek-ai/cordis` 且 import 了 `@deepseek-ai/dsh-session` 的文件，输入 `ctx.` 看补全里有没有 `sessions`。再对照 [packages/compaction/compaction/src/types.ts](../../packages/compaction/compaction/src/types.ts) L17，体会"词典变厚"。
5. **读一次启动链**：从 [apps/cli/src/bin.ts](../../apps/cli/src/bin.ts) 的 `runCli`（L26）开始，跳进 `parseDshArgs`（args.ts L146），再回 bin.ts L33-35 看动态 import 的 `runProfile`。思考：为什么 `runProfile` 是动态 import 而不是顶层 import？（提示：`plugin`、`dump-config` 等子命令不需要加载 profile 启动路径。）
6. **精读一条 vendor 修改日志**：在 [vendor/README.md](../../vendor/README.md) 挑第 11 条（`applyEntryPatches` 的抽取，L43）精读，回答三个问题：改了什么？为什么必须在本仓库改、而不是提给上游或另写一份？哪些测试覆盖它？能答对这三问，说明你已理解 vendoring 纪律的实质。

## 本章小结

1. DSH 是 pnpm monorepo；"三种包"的物理定义就是 pnpm-workspace.yaml 的 glob：`vendor/*`、`packages/*/*`、`apps/*`（另有 `native/system`、`benchmarks`、`website`、`python/sdk-runtime`）。
2. vendor 是内嵌框架层：Cordis 及配套库的 pinned 源码拷贝，可审计、可打补丁、钉在上游 commit 上；以 `@deepseek-ai` 作用域发布，因为能力包 peer 依赖它。
3. packages 是能力原子：`@deepseek-ai/dsh-<name>`，按 `packages/<group>/<pkg>` 分组（54 组），组表权威在 packages/README.md，包目录名跨组唯一。
4. apps 是产品入口：cli（`dsh` bin）、web（Vite 壳）、desktop（Electron 壳）；只有 `dsh` profile 能启动 Node 应用，门禁强制。
5. 版本协议分治：DSH 互引 `workspace:*`，vendor/native 用 `workspace:~`；由 `verify-package-dependencies.ts` 的 `workspaceRange` 机器强制。
6. 框架是 peer 不是硬依赖：能力包共享宿主的同一份 cordis 实例；装配根 apps/cli 负责落位。
7. 依赖契约不依赖实现：`dsh-agent`（契约）与 `dsh-agent-loop`（默认实现）分离；约一半子系统是循环事件的纯消费者。
8. declaration merging 让类型跨包流动：`Context`（服务键）、`Events`（事件名）、`SessionEventMap`（事件载荷）三例，所有权归生产者、可见性归所有人、核心包零改动。
9. 可扩展联合走 documented default（如 `TurnEndReasonMap`），封闭联合 switch 以 `assertNever` 收尾；`SessionEventMap` 新成员默认 required-on-read。
10. 源码面与构建产物面不混合：静态门禁经 tsconfig `paths` 解析到 `src/*.ts`，干净树可全绿；消费 `lib/` 的门禁需显式声明。
11. 双编译面：根 tsconfig.json 是 solution-only（`files: []`），host/client 两个聚合各自持有 cordis Context 的不同合并；叶子配置按面拆分。
12. vendor 修改纪律 = 清单表（上游 SHA）+ 23 条详尽修改日志 + 五步同步流程 + 机器化 rescope；不 fork 独立仓库是为了让修改与测试原子演进。
13. 依赖图是生成物：别背依赖，查 docs/module-graph.md（`pnpm run gen-module-graph` 再生成，CI 校验新鲜度）。

## 必读源码回顾与延伸阅读

**本章引用过的源码，建议按此顺序重读：**

- [pnpm-workspace.yaml](../../pnpm-workspace.yaml)（L1-15）——三种包的物理定义；
- [packages/README.md](../../packages/README.md)（L29-84 组表、L96-101 依赖规则）——组的权威清单；
- [vendor/README.md](../../vendor/README.md)（清单表、23 条修改日志、同步流程）——vendoring 纪律全书；
- [packages/shell/tool-bash/package.json](../../packages/shell/tool-bash/package.json)（L29-48）——peer/dev 分区的一手证据；
- [packages/core/session/src/index.ts](../../packages/core/session/src/index.ts)（L38-83）——`Context` 与 `Events` 两次合并的原文；
- [packages/core/session/src/types.ts](../../packages/core/session/src/types.ts)（L201-232、L281-431）——`TurnEndReasonMap` 与 `SessionEventMap`；
- [packages/compaction/compaction/src/types.ts](../../packages/compaction/compaction/src/types.ts)（L17-91）——跨包合并会话事件的范例；
- [tsconfig.json](../../tsconfig.json) + [tsconfig.host.json](../../tsconfig.host.json) + [tsconfig.client.json](../../tsconfig.client.json)——solution 与双编译面；配套读 [packages/AGENTS.md](../../packages/AGENTS.md)（包级 tsconfig 细则）与 [docs/development.md](../../docs/development.md) 的 TypeScript project layout 一节；
- 走读链五站：[tool-bash/src/index.ts](../../packages/shell/tool-bash/src/index.ts) → [shell/src/index.ts](../../packages/shell/shell/src/index.ts) → [bash-local/src/index.ts](../../packages/shell/bash-local/src/index.ts) → [subprocess/src/index.ts](../../packages/subprocess/subprocess/src/index.ts) → [vendor/cordis/src/service.ts](../../vendor/cordis/src/service.ts)。

**延伸阅读：**

- 上一章：[第1章 开篇：DeepSeek Harness 总览](第1章-开篇-DeepSeek-Harness总览.md)——如果"一切皆插件"还没有画面感，回去补；
- 下一章：[第3章 Cordis 内核：服务依赖注入与上下文](第3章-Cordis内核-服务依赖注入与上下文.md)——本章埋下的所有伏笔（`ctx` 代理、Service 注册、fiber 生命周期）在那里兑现；
- 向前跳读：[第4章 事件驱动](第4章-事件驱动-五路派发与三个事件域.md)（`Events` 合并的运行时一面）、[第7章 工具系统](第7章-工具系统-能力接缝与执行管线.md)（能力接缝三角色）、[第12章 扩展系统](第12章-扩展系统-配置即组合一切皆插件.md)（分层如何被配置组合成产品）。
