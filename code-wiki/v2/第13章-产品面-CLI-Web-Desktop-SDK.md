# 第13章 产品面：CLI、Web、Desktop、SDK

## 1. 本章导读

前面十二章，你一直在拆一台机器：Cordis 内核（[第3章](./第3章-Cordis内核-服务依赖注入与上下文.md)、[第4章](./第4章-事件驱动-五路派发与三个事件域.md)）、Agent 循环（[第5章](./第5章-Agent-Loop-让模型转动起来的引擎.md)）、模型接缝（[第6章](./第6章-模型调用-llm能力接缝与适配器.md)）、工具管线（[第7章](./第7章-工具系统-能力接缝与执行管线.md)）、会话日志（[第8章](./第8章-消息系统-会话日志即模型记忆.md)）、上下文工程（[第9章](./第9章-上下文工程-一次请求里塞了什么.md)）、压缩（[第10章](./第10章-上下文压缩-当对话太长怎么办.md)）、持久化（[第11章](./第11章-会话管理-持久化格式迁移与分叉.md)）、配置组合（[第12章](./第12章-扩展系统-配置即组合一切皆插件.md)）。现在把它们拼回去，问一个产品问题：

> 同一套 agent 内核，怎么同时变成网页应用、一次性命令行、桌面软件、两种语言的 SDK、以及被程序驱动的自动化服务？

本章回答这个问题，它也是全书唯一的"收束章"。读完你应该能：

1. 说清 `dsh` 为什么是**唯一**受支持的 Node 应用启动器，五种产品形态由什么决定；
2. 画出 Web 的"三段式"——浏览器半边 / Remote 层 / Host 半边——并解释鉴权栏挡在谁和谁之间；
3. 在 Web 里从"会话事件"一路追到"屏幕上一个卡片"，并知道"加一个 Chat 节点"要注册什么；
4. 区分 Desktop 与 Web（前者不是另一种实现，而是同一运行时的壳）、TS 与 Python SDK（都走 stdio JSON-RPC，不走 HTTP）、ACP 与完整 DSH（前者是刻意的 automation-only 子集）。

本章先修要求：读到这里即可，另外最好已在动手环节跑过 [第1章](./第1章-开篇-DeepSeek-Harness总览.md) 的 `pnpm dsh web`。

## 2. 从一个问题出发

教学虚构一下。假设你写了一个最小 agent，全部逻辑塞在一个 `main.ts` 里：一个循环、一个模型客户端、几个工具。某天产品经理说要网页界面。你只好把 `main.ts` 复制成 `web-main.ts`，再写一个 HTTP 服务器、一套前端、一处 CORS、一处登录……。第二天又要一个跑脚本的 CLI，你又复制一份。两周后仓库里有五个长得越来越像又各自漂移的入口，任何 bug 要修五遍。

这正是 [第1章](./第1章-开篇-DeepSeek-Harness总览.md) 讲的"没有特权核心"要避免的病：**入口一多，"组装顺序由谁说了算"重新变成糊涂账**。DSH 的解法分两步走：

- **组合（[第12章](./第12章-扩展系统-配置即组合一切皆插件.md)）**：一棵插件树由四层 patch 叠出来，产品形态只是"叠哪些 bundle"，而不是"写哪个 main"。`web` / `headless` / `sdk` / `sdk-minimal` / `acp` 五种 profile 对应五份 bundle 清单。
- **投射（本章）**：同一棵树上的服务面，通过**三条通道**长到不同进程边界之外。

三条投射通道分别是：

| 通道 | 边界 | 使用者 |
|---|---|---|
| **进程内 / 浏览器 Remote（remote）** | Client ↔ Host，走 HTTP 与 WebSocket | Web、Desktop |
| **stdio JSON-RPC** | 外部进程 ↔ DSH 运行时，走标准输入输出 | TS SDK、Python SDK |
| **进程内直接调用** | 同一进程内的普通 Cordis 服务调用 | 所有形态共享的底座 |

把内核、组合、投射叠起来，就是本章的主图。借一棵树打比方（这是本章第 1 处比喻）：**内核是根与干（第3–11章），组合是枝干分叉（第12章），五片叶子就是五种产品形态。**

```text
                 内核（第3–11章：服务、事件、循环、工具、会话、压缩、持久化）
                                      │
                              组合（第12章：profile + bundle + 四层 patch）
                                      │
        ┌──────────────┬──────────────┼──────────────┬──────────────┐
        ▼              ▼              ▼              ▼              ▼
      CLI           Web          Desktop        TS/Python SDK      ACP
   (headless)   (浏览器 Remote)  (Electron 壳)   (stdio JSON-RPC)  (stdio JSON-RPC)
```

记住这张图的读法：**右下方两种（SDK、ACP）走 stdio，左上方两种（Web、Desktop）走 Remote，CLI 是同一个启动器的一个 profile。** 下面分节展开。

## 3. 正文

### 3.1 唯一启动器：`dsh` 与 profile

五种形态共享同一个命令：`dsh`。`apps/cli/` 就是 npm 包 `@deepseek-ai/dsh` 的本体。入口薄得近乎苛刻——[apps/cli/src/bin.ts](../../apps/cli/src/bin.ts) 全文只有 78 行，`runCli()`（L26–74）拿到 argv 后只做一件事：解析出调用模式，再**动态 import** 对应模块：

```ts
// apps/cli/src/bin.ts:31-33
switch (invocation.mode) {
  case 'profile': {
    const { runProfile } = await import('./profile-boot.ts')
```

四个分支（`profile` / `plugin` / `dump-config` / `dump-config-schema`）每个都只 `await import()` 再委托，入口不含任何业务逻辑；兜底分支用 `invocation satisfies never`（L71）让编译器证明判别联合已穷尽，进程真身是 L76 的 `if (import.meta.main)`。命令文法唯一的所有者是 [apps/cli/src/args.ts](../../apps/cli/src/args.ts) 的 `parseDshArgs()`（L146）：启动器**只**解析自己的 flag（`--profile`、`--from-default-profile`、`--patch`、dump 三兄弟），其后的一切原样装进 `ProfileInvocation.args`（L21–30），交给被组装的应用插件去解析。所以 `dsh --profile web --help` 打印的是 Web 应用的帮助，不是启动器的。

`--profile` 名字指向 `$DSH_HOME/profiles/<name>`。五种 shipped profile 的 bundle 清单写在 [packages/boot/app-boot/src/profile.ts](../../packages/boot/app-boot/src/profile.ts) 的 `PROFILE_TEMPLATES`（L179–195）：

| profile | bundles | 形态 |
|---|---|---|
| `web` | `dsh-base` + `dsh-web-app` | 浏览器应用 |
| `headless` | `dsh-base` + `dsh-headless` | 一次性运行器 |
| `sdk` | `dsh-base` + `dsh-sdk-app` | JSON-RPC stdio 服务器 |
| `sdk-minimal` | `dsh-sdk-minimal`（**不经 dsh-base**） | 最小独立树 |
| `acp` | `dsh-base` + `dsh-acp-app` | automation-only ACP 服务器 |

`DEFAULT_PROFILE_BUNDLES`（L213）是 `['@deepseek-ai/dsh-base']`；`OPTIONAL_BUNDLES`（L223–228）列出默认关闭、可在插件管理器里打开的额外 bundle（agent-team、voice-input、auto-review、inspector）。第五个名字 `desktop` **保留**给 Electron 应用，CLI 明确拒绝启动它——`args.ts` 的 `rejectElectronProfile()`（L83–87）会在解析时就报错。

**唯一启动器不是君子协定。** [scripts/verify-application-entrypoints.ts](../../scripts/verify-application-entrypoints.ts) 在 CI 里把三类突破口全部堵死：`MANIFEST_BIN_ALLOWLIST`（L27–30，只允许 `apps/cli` 与一个 build-only 的 WebWorker packer 声明 `bin`）、`EXECUTABLE_SOURCE_ALLOWLIST`（L33–49，带 shebang 的源文件必须逐条分类）、`ROOT_LAUNCHER_POLICIES`（L56–61，`demo:*`、`start:web`、`dev:web` 必须显式指向 `apps/cli/src/bin.ts` 或分类过的包装脚本）。任何想另开入口的包，门禁都会拒绝。

真正的组装在 [apps/cli/src/profile-boot.ts](../../apps/cli/src/profile-boot.ts) 的 `runProfile()`（L244）里：它调 [packages/boot/app-boot/src/index.ts](../../packages/boot/app-boot/src/index.ts) 的 `boot()`（L973），按四层 patch（bundle 层 → profile 自己的 `cordis.patch.yml` → home 级 `$DSH_HOME/cordis.patch.yml` → `--patch` 覆盖层）叠在一个**空根**上（`PROFILE_ROOT_CONFIG`，profile-boot.ts L81–85）。`boot()` 挂载 `cordis:include` 与 `cordis:group` 两个内置（`mountRootInclude`，index.ts L539），等整棵树 settle，再由 `auditStartupEntries()`（L926）审计：

- **required 条目**（`requiredStartupEntryIds`，L747–755：`agent-loop`、`webserver`、`modules`、`connection`、`headless-runner`、`acp`、`sdk-jsonrpc-server`）没激活就抛 `StartupError`，整个应用 dispose 并以非零退出；
- 普通可选条目失败只记一条 warning，成功的兄弟继续跑。

启动前还有两条纪律值得知道：`loadLayeredEnv()`（L235）按"继承环境 > 调用目录 `.env` > Harness home `.env`"加载，且**拒绝**任何决定进程如何启动的变量（`PATH`、`DSH_*` 等）从 `.env` 出现；`installFailLoud()`（L680）把未处理的 rejection / 未捕获异常变成一条带 `util.inspect` 的 stderr 诊断并 `exit(1)`，绝不静默半死不活。

### 3.2 CLI 的两条辅助命令族：`plugin` 与 dump

`dsh` 除了启动 profile，还管两件事，都在 `runCli` 的 switch 里。

**`dsh plugin --profile <name> <pnpm args>`**（[apps/cli/src/plugin.ts](../../apps/cli/src/plugin.ts) 的 `runPlugin()`，L70）：把后续参数原样转发给 profile 目录里的 pnpm，用来安装/卸载外部插件 bundle。它还独占一组 DSH 自己的子命令 `allow-version` / `revoke-version` / `version-exemptions`（`versionCommand()`，L17），用来对"不兼容的插件版本"做**显式、按精确版本**的风险确认——不兼容的插件默认被跳过并报到 `skippedBundles`，不静默放行。

**dump 命令**用于审计组合树而不启动进程：`--dump-config` 打印完整组合、`--dump-config-schema` 打印 JSON Schema、`--dump-default-config` 只保留 bundle 层（用作坏 patch 时的恢复诊断）。实现见 [apps/cli/src/dump-config.ts](../../apps/cli/src/dump-config.ts) 的 `collectConfigDumpLayers()`（L51–75）：bundle 层 → profile patch → home patch → argv overlays，逐层标注来源文件。dump 刻意**不运行命令行 provider**，所以它明确拒绝接受应用参数（`resolveBoot`，args.ts L125–127）。

现在进入本章主干——Web。

### 3.3 Web 的三段式：浏览器半边 / Remote 层 / Host 半边

Web 是五片叶子里最重的一片，也是初学者最容易困惑的一片。请先记下它的**三段式结构**（[docs/subsystems/web-client.zh.md](../../docs/subsystems/web-client.zh.md) 的分层表是权威描述）：

| 段 | 主要 owner | 职责 |
|---|---|---|
| **Host 半边**（[packages/host/](../../packages/host/README.md)） | `webserver`、`frontend-static`、`directory-picker-*`、`open-in-app`、`plugin-inventory`、`product-telemetry-otel` | 权威状态、持久化、mutation 顺序、HTTP 服务与 SPA 静态资源 |
| **Remote 层**（[packages/api/](../../packages/api/README.md) + [packages/typert/](../../packages/typert/README.md)） | `api-remotes`、`api-gateway`、`*-controller` | 把 Host 服务暴露成类型化的 Client 方法、流、转发事件 |
| **浏览器半边**（[packages/client/](../../packages/client/README.md)，60+ 包） | `client/web`、`client/modules`、`client/connection`、`client/store`、40+ 个 `ui-*` | 内核 boot、RPC、对象层、Slots 与 React 渲染 |

依赖方向单向：**Host 状态 → Remote 传输 → Client model → UI adapter → Conversation/presentation → Slots → React**（web-client.zh.md L20）。用户操作沿 callback 反向回到注入的 Client 服务或生成的 Remote namespace。**前端绝不直连内核**——它只能通过 Remote 层或某个精确的 HTTP 路由（如文件下载）说话。

Host 半边里 `webserver` 提供命名路由与回退座位（`ctx.webServer`），`frontend-static` 坐在回退座位上把 SPA 的 `dist` 发出去。[apps/web](../../apps/web/package.json)（`@deepseek-ai/dsh-web-frontend`）就是一个 Vite 应用，入口 [apps/web/src/main.ts](../../apps/web/src/main.ts) 只做一件事——`new AppWebEntry(el).run()`（L37，实现在 `packages/client/web`）；它的 `dist` 由 `dsh web` 服务。

"发送一条消息"要穿过哪些层？[docs/subsystems/web-client.zh.md](../../docs/subsystems/web-client.zh.md) 的数据通路表给了完整顺序：

```text
Host Session log（第8章的仅追加日志）
   │ packed Remote follow/page 历史
   ▼
Client SessionEventLikeEntry 窗口（api/session-controller/client）
   │ Conversation Context（match / start / update）
   ▼
target 快照（chat / trajectory / ...）
   │ Slot view
   ▼
React 渲染
```

反向的"用户 command"路径是：component callback → 注册项 inject face 或 Slot owner → `ctx.sessions` / `ctx.workspaces` / 生成的 scoped Remote → Host Controller → 权威 update → 再经 stream 或 event projection 回到 Client。请盯住两个端点：**权威状态永远在 Host**，Client 只维护"最新可用的本地投影"（[packages/api/session-controller](../../packages/api/session-controller/README.md) 的 `ClientSessions → SessionManager → Session`）。

### 3.4 Remote 层：Typert RPC 与连接

Remote 层回答"浏览器怎么调到 Host 的能力"。它的编程模型在 [docs/api-gateway.md](../../docs/api-gateway.md)：

- Host 业务服务继承 `TypertRemoteService`，用 `@Remote` 或 `@RemoteScope` 标注要暴露的方法（未标注的不进生成的 Client 类型）。`@Remote` 调根 Context 上的服务；`@RemoteScope(key)` 先经 `ctx.typert.contexts` 解析出一个 scoped Context 再调。
- **复杂对象不能直接过线**：业务包必须用 `TypertLookupMap` 声明它和某个 wire 身份（identity）的关联。例如 Host 签名里一个叫 `agent` 的 `Agent` 参数，线上会变成 `agentId` 字段，Gateway 在调用前把它解析回真实的 Host 对象。
- 构建期由 `dsh-typert-generator` 从 Host 的 `ts.Program` 严格分析 Remote 签名，生成 Host 契约与 Host-for-Client 契约——即 **InvocationDescriptor（调用描述符）**：严格 schema、runtime codec、declaration merge、source map，写进各业务包的 `lib/typert.host.js` 与 `lib/typert.remote-client.js`。

传输分两种。**一元（unary）调用**走 HTTP：`connection.rpc.call('/api', '<namespace>/<method>', { args }, signal)`，HTTP carrier 映射成 `POST /api/<namespace>/<method>`。**流（stream）**（`@Remote({ mode: 'stream' })`）走 Gateway 拥有的 **`/api/remote.mux` WebSocket**：多条逻辑流复用这一条物理连接，每条逻辑流还能携带 Client→Host 的**上行帧**（uplink frame：`open` / `item` / `end` / `cancel`）。为避免空闲被中间网络设备断开，Host 默认每 **2 秒**发一次 Ping（`websocketHeartbeatIntervalMs`），浏览器在 WebSocket 协议层回 Pong；上一个 Ping 没回的 socket 会在下一个周期被终止。

几条工程细节值得记住（[packages/api/gateway/README.md](../../packages/api/gateway/README.md)）：

- **一元结果永不为 carrier 问题 reject**：每个一元调用返回 `RemoteResult<T>` = `{ ok: true, value }` 或 `{ ok: false, error }`；断线被折进 error 分支，只有装配层错误才 reject。
- **重连分两层**：Gateway mux 负责恢复物理 WebSocket；Connection 发布新 generation 后，每个 `RemoteStream` 各自重开自己的逻辑源。
- `RemoteSnapshotStream` 提供"一份快照 + 后续增量"；`RemoteJournalStream` 提供"先订阅后分页"的开场、分页、重连补齐与缺口修复。普通转发事件**不重放**——需要可靠恢复的有状态领域必须自带 baseline / cursor。

**开发时的 SRC 回退**：当 Host 从源码经 `node --import tsx/esm` 启动时，Typert 编译器插件不跑。此时标准装饰器初始izer 会把方法名与调用模式记进一个版本化描述符，Gateway 据此构造一个较弱的临时描述符（SRC fallback），从活函数里解析简单的参数名并只校验 JSON 安全。SRC 只管"源码运行的 Host 如何派发"；Client 的类型、codec、注册值永远来自最近一次生成的 `lib/typert.remote-client.*` 产物。

**生成的次序不能乱。** Remote 的严格契约是**构建产物**，所以根构建按固定顺序跑（[docs/api-gateway.md](../../docs/api-gateway.md) 的 "Strict generation pipeline"）：`build:lib:host`（`tsc -b tsconfig.host.json` + Host 面 tsdown，Typert generator 在此运行，Host aggregate 是它唯一的 `ts.Program` 种子）→ `build:lib:client`（`tsc -b tsconfig.client.json` + Client 面 tsdown，消费刚生成的 Remote Client 声明）→ `build:web`。因此：只改一个 Remote 方法的**实现体**、不改契约，不必重新生成；一旦动了装饰器、命名空间、参数、返回值、lookup、Context 或取消签名，就必须重跑有序的 lib 构建。`pnpm run typecheck` 也会先跑 Host lib 阶段再编 Client——**"Host 先生成、Client 后编译"是编译期硬纪律**，clean 工作树无法跳过 Host 阶段。

### 3.5 鉴权栏：浏览器请求信任

Remote 层之外还有一条**信任栏**，位于 [packages/client/connection](../../packages/client/connection/README.md#browser-authentication-and-request-trust)。它是初学者最容易忽略、也最该讲清的部分。

要点：**每一次 Host RPC 方法与 WebSocket 流都需要一个浏览器会话，没有针对单个方法的 loopback 特例。** 流程如下：

1. 每个进程启动时生成一个随机 **launch token**；`dsh-web-app` 打印并打开带 `?token=...` 的应用 URL。
2. `frontend-static` 把根请求委派给 `ctx.connection.authorizeIndex`，它**只在 `GET /`** 上接受这个 token，写下一个**authority 绑定的签名 cookie**，然后重定向到干净的 `./`（丢掉 token）。静态资源始终公开。
3. Cookie 的签名密钥是 `ctx.credentials` 里 owner-scoped 的 `client-connection/browser-session` grant 记录，本地 provider 存在 `$DSH_HOME/.credentials.yaml` 里，Connection 激活时载入内存，所以请求鉴权是同步的。Cookie 默认 30 天有效（`cookieMaxAgeDays`），名字与签名载荷里都绑定归一化的 `host:port`，且是 host-only、`Path=/`、`HttpOnly`、`SameSite=Strict`——刻意**不带 `Secure`**，因为官方服务器跑的是 loopback HTTP。
4. 鉴权之前，每个请求还要过 [packages/client/connection/src/api-request-trust.ts](../../packages/client/connection/src/api-request-trust.ts)：`Host` 必须是 loopback 或命中 `trustedHosts`；带了 `Origin` 就必须等于 `Host`；`sec-fetch-site: cross-site` 一律拒绝。这些检查防的是 **DNS rebinding（域名重绑定）** 与跨站请求，它们**从不建立身份**。Host/Origin 检查失败返回 403，可信但未鉴权返回 401。`dsh web --host 0.0.0.0` 依然不支持。

cookie 的每个属性都有理由，值得列成一张表：

| 属性 | 值 | 为什么 |
|---|---|---|
| 名称与签名载荷 | 绑定归一化 `host:port` | 换端口或域名即失效，防串用 |
| 有效期 | 默认 30 天（`cookieMaxAgeDays`） | 本机开发不必反复换 token |
| `HttpOnly` | 是 | 脚本读不到会话凭据 |
| `SameSite` | `Strict` | 跨站请求不带它 |
| `Path` | `/` | 覆盖整个应用 |
| `Secure` | **不带** | 官方传输是 loopback HTTP |

三条由它推出的产品事实：**静态资源公开**；**根交换之外不接受 query token，也不接受 Authorization 头 token**；**没有 logout 操作**——清浏览器 cookie 只结束一个浏览器会话，删除 owner 凭据记录并重启 `dsh` 才撤销全部会话。

**重连的"代"（generation）**由内部 `$events` 逻辑流作**唯一** generation source。Host 先同步挂好所有增量监听，再发一条 `{ type: 'ready', clientId, host: { home } }`，`ConnectionController` 收到 ready 后才发布 generation——所以"baseline 读取"永远追不过"增量观察"。断线时在 50%–100% 抖动下按 500ms、1s、2s、4s、8s、10s 的阶梯重试，到顶后维持 10s 直到恢复；浏览器 `offline` 暂停自动重试，下一次 `online` 从 500ms 档重启。

### 3.6 浏览器半边：boot 注入与 Conversation 节点

浏览器里跑的是另一个 Cordis 应用。它由 [packages/client/web](../../packages/client/web/README.md) 的 `AppWebEntry` 两段式 boot：

- **模块阶段**：采用 Host 注入的起始批次，用 Host 提供的 **boot graph（`window.__DSH_BOOT__`）** 建好模块系统，并预取 `immediately` 层。
- **插件阶段**：挂载 vendored Cordis Loader，为图中每个 entry 创建一行，等全部激活后再把 boot DOM 交给 `ui-renderer` 去 hydrate 并切到完整 UI。

失败一个 entry 就保留那张无框架的 boot page 并逐条报错——**不支持部分可用**，宁可显示"谁挂了"也不留白屏。共享模块表 `PLATFORM_MODULES`（`src/platform.ts`）给每个动态 bundle 提供 React、Cordis 与静态 UI 库作为隐式外部依赖。

Host 如何知道该加载哪些浏览器插件？靠每个包 `package.json` 里的 **`dsh.client`** 声明（[packages/client/modules/README.md](../../packages/client/modules/README.md)）：`platform: 'web'`、必须导出 `./client`、可选的 `dsh.client.external` 申请非基线的共享模块。Host 半边扫描启用的 Loader 条目、组成 boot graph，经 **`/plugins`** 路由把每个 `lib/client.js` 发出去；浏览器惰性地加载它们——**运行一个 bundle 只注册工厂，模块体副作用（含 CSS 注入）在 materialize 时才跑**。

`dsh.client` 的语义只含三点，值得逐条记牢（[packages/client/AGENTS.md](../../packages/client/AGENTS.md)）：`platform: 'web'` 固定；声明必须配一个 `./client` 导出（扫描缺它会抛错）；`inject` 列的是**信息性**的包名依赖边（供 preflight 显示与 HMR diffing），它**不**排序 entry 激活——激活顺序只由 Cordis fiber 等服务决定。模块共享另有机制：`dsh.client.external` **不是功能插件的依赖机制**，只有基础设施、传输或生成的 assembly 才能申请非基线的共享模块身份，普通第三方实现库"沉默即私有拷贝"。Desktop 复用同一套注入：Electron 启动时把 boot 注入交给浏览器客户端，client 维护"连接代"、Host 先挂监听再发 ready，上面的加载链条一字未变。

最后一个关键接缝是**会话事件如何变成屏幕上的东西**。核心是 [packages/client/ui-conversation](../../packages/client/ui-conversation/README.md) 的 `UiConversation` 服务：

- `ctx.uiConversation.events` 是事件 Definition 的唯一注册表；`ctx.uiConversation.views` 是 target 快照 builder 的唯一注册表；`ctx.uiConversation.binding(bindingOrSessionId)` 返回某个 Session 的、身份稳定的 binding；`ctx.uiConversation.groups` 为每个 target 注册一个可选的分组 Definition。
- target 包（如 Chat）声明合并自己的 snapshot / Location 数据表，用 `ctx.uiConversation.events.register(...)` 与 `ctx.uiConversation.views.register(...)` 注册，再用 `binding(...).target(targetId)` 读自己的源。

具体到"怎么把一个 Session 事件翻成一个节点"，是 **ConversationNodeDefinition**（[packages/client/ui-conversation/src/client/contract/conversation.ts](../../packages/client/ui-conversation/src/client/contract/conversation.ts) L203）：

```ts
// packages/client/ui-conversation/src/client/contract/conversation.ts:203-212
export interface ConversationNodeDefinition<State = unknown> {
  readonly kind: string
  /** Sole view target owned by this Definition; omitted for state-only Contexts. */
  readonly target?: string
  match(event: SessionEventLike): ConversationMatchResult | null
```

`match(event)` 只用"当前这一个事件"抽出一个稳定的业务身份（identity），`start` / `update` 把 Match 折成 State，装配器再按 Turn/Step 算出 **Location** 并 materialize 成 target 节点快照 `ConversationSnapshot`。**同一家族的事件不会互相扫描整窗**：append 热路径与渲染器都只累加 State，公开同 Turn/Step 的事实走 `buildLocationData()`。

Chat target 自己注册全部节点，入口在 [packages/client/ui-chat/src/client/conversation-nodes/register.ts](../../packages/client/ui-chat/src/client/conversation-nodes/register.ts)（L22 的 `registerConversationNodes(ctx)`）：inbox / message / request-prompt / assistant / turn-process / tool / command / compaction / retry / turn-error / turn-max-tokens / turn-tail / fallback，外加一个 `processGroupDefinition`。

如果你要**给 Chat 加一个新节点**，路径不长（这也正是 [docs/architecture.md](../../docs/architecture.md) L158 "Add a Web Client Chat node" 指向的机制）：

1. 写一个 `ConversationNodeDefinition`，注册进 `ctx.uiConversation.events`；
2. 写一个 keyed renderer，注册进 `conversation.chat.node` —— 该 slot 由 `ui-chat` 声明。

注意这只影响**展示**：客户端 UI 是纯表现层，"怎么画"永远不进会话日志；只有**模型可见**的新输入才需要新的 Session 事件（第8章的不变量）。

工具卡片同理：[packages/client/ui-tool](../../packages/client/ui-tool/README.md) 不消费 Host 的 presenter 值，卡片由原始 `tool/call` / `tool/result` 事件 + 持久化 `result.meta` 派生，按 wire 工具名经 keyed slot **`tool.call.toolview`** 分发，未注册名走通用卡——这正是 [第7章](./第7章-工具系统-能力接缝与执行管线.md) 讲的"Host presenter 与 Client 卡片是两条独立渲染路线，共享的只有事件与 `meta`"。

### 3.7 Desktop：给同一运行时套一层 Electron 壳

[第1章](./第1章-开篇-DeepSeek-Harness总览.md) 立过一条规矩：`desktop` 不是第六个 profile，它保留给 Electron 应用。读 [apps/desktop/README.md](../../apps/desktop/README.md) 后你会看到它到底"包"了什么：

> The desktop application is an Electron shell around the complete dsh Web application.

关键机制：

- **不是另一种实现**。Electron 用一个 **RunAsNode** 子进程（`ELECTRON_RUN_AS_NODE=1`、`--expose-internals`）启动**共享的 profile runner**，即 `@deepseek-ai/dsh/profile-boot`（`runProfile` 也导出给 Desktop host 用，见 [apps/cli/README.md](../../apps/cli/README.md) L60）；窗口立即加载打包好的 Web 入口 `dsh-app://app/`，等 boot 注入到位后再在**同一个 document** 里激活 client 插件，不导航到另一个文档。
- **两条通道分工**：Node IPC 只承载 boot 注入、就绪与关机；**Web 拥有 RPC 与流**——desktop carrier 只把本地页面接到那个已经过鉴权的 Host 上（desktop 页面的 `streamBaseUrl` 与 HTTP 传输各自独立选择）。
- **版本纪律**：Electron 与 `@deepseek-ai/dsh` **永远同版本**；一次 dsh 升级就是一次 Desktop 发布，即使壳代码没变。
- **状态归属**：Desktop 独占 `$DSH_HOME/profiles/desktop`，与 CLI 共享 `$DSH_HOME` 下的会话/设置/凭据/工作区，但**绝不共享可执行包、插件激活、lockfile、node_modules**；CLI 不能启动这个保留 profile，只有 Desktop 安装的命令能在应用完全退出后管理它的插件。
- **端口与打包**：默认让 OS 分配端口（因此不会和 Web 的 3080 撞车），可用 `webserver.config.port` patch 覆盖；自带独立 Node、Python、pnpm 发行版，插件事务用捆绑的 pnpm 在 Electron Node 模式里跑；Windows 走 EV 签名，macOS 走公证（细节以 README 为准）。

一句话：**Desktop = 同一棵插件树 + 一个把浏览器载入与本地能力（目录选择、托盘、更新、OS 集成）接起来的 Electron 壳**。

### 3.8 TypeScript SDK：用 stdio JSON-RPC 开走运行时

换一条通道：[packages/sdk](../../packages/sdk/README.md) 家族让**另一个进程**通过换行分隔（newline-delimited）的 JSON-RPC 2.0 驱动一个完整的 DSH 运行时——**不走 HTTP**。三个包各司其职：`protocol`（wire 协议）、`client`（客户端）、`server`（服务器）。

**客户端**（[packages/sdk/client/README.md](../../packages/sdk/client/README.md)）有两层。高层是 `DeepSeekHarness`，用法极简：

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

注意 `await using`：`DeepSeekHarness` 实现 `AsyncDisposable`，作用域结束自动收割子进程。`run()` 拥有一个**活动区间**：把提示词入队，**等到它的 message id 出现在持久收件箱（durable inbox）的回执里**，再一直收集到下一次 whole-agent `idle`，返回 `RunResult { sessionId, finalResponse, events, notifications }`。一条容易踩的坑：`finalResponse` 是该区间内最后一条已提交的根会话助手文本，**不**是因果上专属于你这条提示词的回答——中途的 steering 与注入上下文都可能贡献内容。

低层是 `HarnessClient`：显式 `start()` / `initialize()` / `prompt()` / `request()` / `close()` 加订阅（`subscribe(filter?)`、`subscribeSessionTree(id)`）。失败模式有四个导出错误类：`JsonRpcResponseError`、`RequestTimeoutError`、`SdkProtocolError`、`TransportClosedError`。`close()` 走一条 `shutdown` → stdin-EOF → SIGTERM → SIGKILL 的升级阶梯且幂等——连"关进程"都写成了纪律。

**服务器**（[packages/sdk/server/README.md](../../packages/sdk/server/README.md)）是 `jsonrpc` 插件，在 stdio 上服务 SDK 客户端。要点有三：`initialize` 是**就绪边界**——它等当前插件树 settle 后才回复，因此首次 prompt 能看见诸如初始 MCP 工具发现这类异步兄弟能力，返回的 wire-stable 身份是 `deepseek-harness-sdk-runtime`；每个 `sessionId` 一个 agent；`session/prompt` 立即返回 `{ messageId }`（只代表入队）。一条硬约束：**stdout 只能有 JSON-RPC 帧**，所以部署里绝不能组合一个 stdout logger，诊断一律走 stderr。

`protocol` 定义了这套方法集：三个 client→server 请求（`initialize`、`session/prompt`、`shutdown`）加四个 server→client 通知（`session.event`、`session.status`、`subagent.started`、`subagent.finished`）。还有个彩蛋：`packages/subagent/subagent-dsh-sdk` 是 DSH 内部消费者——用 SDK 把**另一个 DSH 实例**当子代理使唤（吃自己的狗粮）。

### 3.9 Python SDK：把运行时一起打包

[python/](../../python/README.md) 有两包：`deepseek-harness-sdk`（模块 `deepseek_harness`，高层 `turns` API + 低层 JSON-RPC client）与 `deepseek-harness-runtime-bin`（捆绑 `dsh` CLI 可执行文件与 native sidecars）。蝴蝶效应在这里最明显：**运行时不要求宿主机装 Node.js**——runtime wheel 发布五个平台目标（linux x64/arm64、macOS arm64/x64、Windows x64）。

两条与 TS 版刻意对称、但更严的行为契约（[python/sdk/README.md](../../python/sdk/README.md)）：

1. **每次启动必须显式选择 Harness home**（`dsh_home` 或非空 `DSH_HOME`）。Python 侧**绝不**静默读写 `~/.dsh`——SDK 不暗动用户的全局状态。
2. `Session.run()` 返回 `RunResult(session_id, final_response, finish_reason, events, notifications)`：`final_response` 是区间内最后一条已提交的根会话助手文本，`finish_reason` 是最后一次根会话 `turn/end` 的 `kind`（如 `completed` / `max-tokens` / `error`），没有 turn 结束时为 `None`。一个缺字符串 `data.reason.kind` 的 `turn/end` 会违反协议并抛 `SdkProtocolError`。

其余要点：`profile` 可选 `sdk` 或 `sdk-minimal`（后者是**独立的**最小树，不经 `dsh-base`，只提供平台选择的持久 shell、本地执行与 JSONL 会话）；`patches=(...)` 按序转发；持久定制走 `--dump-default-config` 初始化后 `dsh plugin --profile sdk add file:...` 安装外部 bundle。贡献者工作流（构建产物、pytest、烟测、源码模式）见 `python/development.md`。

### 3.10 ACP：刻意的 automation-only 子集

最后一片叶子是 [packages/acp](../../packages/acp/README.md)：一个实现标准 **Agent Client Protocol（agentclientprotocol.com）** 的服务器，`pnpm dsh --profile acp` 启动，让程序与自动化通过 JSON-RPC stdio 驱动持久 DSH agent——创建/列出/恢复/关闭会话、挂标准 MCP 服务器、选模型与 reasoning effort、发文本/图片 prompt、收语义更新、答权限请求、取消工作，**全程无人类参与**。

它最该记住的一句话是：**它刻意只暴露标准 ACP v1 面**。DSH 特有的展示卡、plan、todo、标题、终端、客户端文件系统操作、elicitation、`session/load`、删除、fork、附加目录……全部**不在**这个自动化面上。选择它的场景是进程外子代理、测试运行器、脚本化控制器；而"人需要 DSH 特有交互"的场景应该用 Web。配置只有 `provider`、`model`、`sessionListPageSize`（默认 `100`）三项（[packages/acp/acp/README.md](../../packages/acp/acp/README.md) L45–49），配套客户端是 `packages/subagent/subagent-acp`。

一句话对比两种 stdio 服务器：**SDK 是"把整个 harness 当运行时开走"，ACP 是"按标准协议把它接进别人的自动化流水线"**——前者的协议是 DSH 自己的，后者的是外部的。

### 3.11 人机交互简述：五种形态共用的那一圈

五种形态共享同一圈人机交互能力（[packages/interaction](../../packages/interaction/)、[packages/feedback](../../packages/feedback/)）：

- `commands`：slash 命令，**免模型回合**即可派发（`ctx.commands`）。
- `user-approval`：一次性 `allow` / `reject`（`ctx.approval`）；**没有 answerer 时 fail-closed**。
- `permission-presets`：把沙箱模式与审批策略捆成一个 Permissions 选择器（`ctx.permissionPresets`）。
- `user-questions`：校验问答 schema + scoped answerer waterfall，agent 可暂停等人（`ctx.userQuestions`）；模型侧工具是 `tool-ask-user` 的 `ask_user_question`。
- `command-feedback`（会话级 `/feedback`）与 `message-feedback`（逐消息评分/备注）：**随会话存储，永不进模型历史或遥测**。

它们的产品面落在不同形态：Web 是 `ui-approval` / `ui-user-questions` / `ui-message-feedback` 等插件；CLI/headless 走命令行 provider；ACP 只映射到标准的 `session/request_permission`。这就再次印证"一切皆插件"的好处：**同一能力，不同产品形态各自接上不同的表面（surface）**，内核完全不变。

## 4. 源码走读

按顺序读七站，全程只看不改。约定同全书：行号以撰写时仓库为准，符号名才是稳定锚点；每段不超过 30 行。

**第 1 站：唯一入口与命令文法。** [apps/cli/src/bin.ts](../../apps/cli/src/bin.ts) 的 `runCli()`（L26–74）全文读一遍——记住它有多薄，四个分支都在 `await import()`。再看 [apps/cli/src/args.ts](../../apps/cli/src/args.ts) 的 `parseDshArgs()`（L146）与 `ProfileInvocation`（L21–30）：文件头注释（L4–11）讲清了"只解析自己的 flag，其余原样交给应用插件"这条中央设计，`HELP_EXAMPLES`（L90–100）给了六个真实用法。这就是 [第12章](./第12章-扩展系统-配置即组合一切皆插件.md) 那句"应用插件厚、启动器薄"的代码形态。

**第 2 站：profile 生命周期。** [apps/cli/src/profile-boot.ts](../../apps/cli/src/profile-boot.ts)：`homePatchPath()`（L73）告诉你 home 级 patch 在哪，`composeProfile()`（L197）是四层 patch 的装配点，`runProfile()`（L244）是共享给 Desktop 的完整生命周期。重点读 L244–318：`installProxyFromEnvironment` 在第一个插件挂载前装好代理、`createProcessShutdown` 与 SIGTERM/SIGINT 的分工、`provideCmdline(hostCtx, { args, exit, ready })` 如何把启动器的事实交给树、以及 `appReady.commit()` 只在 boot 成功且 fiber ACTIVE 时才提交。

**第 3 站：boot 与 fail-loud。** [packages/boot/app-boot/src/index.ts](../../packages/boot/app-boot/src/index.ts) 的 `boot()`（L973–1039）：看清两阶段失败标签（`host preparation failed` / `plugin tree failed to load`）、`requiredStartupEntryIds`（L747）的全局 required 列表、`auditStartupEntries()`（L926）如何区分"required 失败 → 停止启动"与"可选失败 → 记 warning"。再读 `loadLayeredEnv()`（L235）与 `installFailLoud()`（L680）——这两个函数定义了 DSH 的启动纪律。

**第 4 站：Remote 的两种传输。** 先读 [docs/api-gateway.md](../../docs/api-gateway.md) 的 "Programming model" 与 "Runtime invocation" 两节，建立 `@Remote` → 生成描述符 → `/api` 调用的全链路。再进 [packages/api/gateway/README.md](../../packages/api/gateway/README.md) 看 `/api/remote.mux` 的分帧、`streamInboxBytes`（默认 262144）与 uplink 背压、`$events` 的 `ready` 帧契约。

**第 5 站：信任栏。** [packages/client/connection/README.md](../../packages/client/connection/README.md) 的 "Browser authentication and request trust"（README 内锚点 `#browser-authentication-and-request-trust`）整节精读，代码在 [packages/client/connection/src/api-request-trust.ts](../../packages/client/connection/src/api-request-trust.ts)：cookie 的每个属性、403 与 401 的分界、`trustedHosts` 与 `sec-fetch-site` 的作用，一句话都不要跳过。

**第 6 站：两条 stdio 协议。** [packages/sdk/client/src/api.ts](../../packages/sdk/client/src/api.ts) 第 22 行 `export class DeepSeekHarness implements AsyncDisposable` 是 3.8 节示例的真身；[packages/sdk/protocol/README.md](../../packages/sdk/protocol/README.md) 的方法表是 wire 契约；[packages/sdk/server/src/server.ts](../../packages/sdk/server/src/server.ts) 的 `HarnessSdkJsonRpcServer` 是服务端。对照 [packages/acp/acp/README.md](../../packages/acp/acp/README.md) 的协议表，确认 ACP 只暴露标准面。

**第 7 站：事件到节点的最后一跳。** [packages/client/ui-conversation/src/client/contract/conversation.ts](../../packages/client/ui-conversation/src/client/contract/conversation.ts) 从 L195 读到 `ConversationNodeDefinition`（L203）结束，体会 `match` / `start` / `update` 三段式；再翻 [packages/client/ui-chat/src/client/conversation-nodes/register.ts](../../packages/client/ui-chat/src/client/conversation-nodes/register.ts) 的 `registerConversationNodes()`（L22），把 3.6 节列的十三个节点和真实函数对上号。

## 5. 常见误区

**误区一："`packages/api` 是公网服务端。"** 不是。它是 **Client ↔ Host 的 Remote 层**：`api/remotes` 决定暴露哪些能力，`api/gateway` 承载一元调用、多路流与转发事件。它跑在 Host 进程里，服务的对象是同一个应用的前端，不是互联网。真正的 HTTP 服务器在 `packages/host/webserver`。

**误区二："Web 前端能直连 agent 内核。"** 不能。两端隔着信任栏与 Remote 层：一元走 `POST /api/...`，流走 `/api/remote.mux` 的 WebSocket，非 RPC 的响应（如文件下载）必须注册**精确的** Fetch 路由。浏览器代码不知道任何 Host 服务对象，只认识生成的 namespace 方法。

**误区三："Desktop 是独立实现。"** 恰恰相反，Desktop 是**套壳**：它用 RunAsNode 子进程跑同一个共享 profile runner，把打包好的 Web 入口加载进 Electron 窗口，然后等 Host 通过 Node IPC 把 boot 注入交过来。它拥有的是壳能力（目录选择、托盘、更新、OS 集成）与独占的 `desktop` profile，不是第二份 agent 逻辑。

**误区四："SDK 走 HTTP。"** 不走。TS SDK 与 Python SDK 都通过 **stdio 上的换行分隔 JSON-RPC 2.0** 驱动子进程；`dsh --profile sdk` 起的 `jsonrpc` 插件就是服务端。这也是为什么"stdout 只能有 JSON-RPC 帧"是硬约束——它是一条管道协议，不是网络服务。

**误区五："ACP 就是完整的 DSH 面。"** 不是，ACP 是**刻意的 automation-only 子集**：只暴露标准 ACP v1，DSH 的展示卡、plan、todo、标题、终端、elicitations 全部不在。想让自动化拿到完整 DSH 交互面，应该用别的通道（例如 SDK + 自定义处理），而不是指望 ACP。

**误区六："`desktop` 是第六个 profile。"** `desktop` 这个名字被 Electron 应用**保留**，CLI 会拒绝启动或 dump 它（`rejectElectronProfile`）。五种可启动 profile 是 `web` / `headless` / `sdk` / `sdk-minimal` / `acp`，一个不多一个不少。

**误区七："生成的 Remote 方法是 JavaScript Proxy。"** 不是。Client 侧 `ctx.remote.<namespace>` 挂在**普通对象的具体函数**上（每个 namespace 是一个已追踪的 Cordis 子服务），类型由 declaration merge 提供。没有 Proxy，也就没有"运行期才发现的魔法"。

## 6. 动手环节

按顺序做，前四步无 key 即可。

1. **起 Web 并看两条通道。** `pnpm dsh web`（默认 `http://127.0.0.1:3080`，本机会自动开浏览器；`--no-open` 只起服务）。打开 DevTools 的 Network 面板：发一条消息，观察一元调用走 `POST /api/<namespace>/<method>`（注意响应是 `RemoteResult` 的 `{ ok, value }` 或 `{ ok, error }`），而流开在 **`/api/remote.mux`** 这条 WebSocket 上。再切到 Application 面板看那个 host-only、`SameSite=Strict` 的会话 cookie。
2. **审计组装树，不启动。** `pnpm dsh --profile web --dump-config | head -60`，数一数每一行的来源注释 `# ==`（bundle / profile patch / home patch / `--patch`），再跑 `pnpm dsh --profile web --dump-default-config` 对比"只留 bundle 层"的差别——这是 [第12章](./第12章-扩展系统-配置即组合一切皆插件.md) "配置即组合"的可视化。
3. **看一次性运行器的第二张脸。** （有 key）`pnpm dsh --profile headless --json "用一句话总结这个仓库的 README"`：stdout 变成 NDJSON 事件流（就是第8章的 Session 事件投影）；换 `--session-id session-…` 续接既有会话。没有 key 时改用 `pnpm run mock:llm` 观察一次完整回合。
4. **用 TS SDK 跑一次 run。** 照 3.8 节的示例写一个小脚本（`profile: 'sdk'`），`await using` 一个 `DeepSeekHarness` 并 `run('say hi')`。观察 `result.finalResponse`，再把 `onNotification` 加上，看看 `session.event` / `session.status` 通知的到达顺序。无 key 时它会响亮失败——体会"misconfiguration fails loud"。
5. **对照第12章看一行浏览器插件怎么被插进来。** 打开 [packages/bundle/web-app/cordis.patch.yml](../../packages/bundle/web-app/cordis.patch.yml)，找一个 `ui-*` 行，回到它的 `package.json` 看 `dsh.client` 声明；再对照 [apps/web/tests](../../apps/web/tests/) 里同名 `*.e2e.ts` + `expected/` 的浏览器快照，理解"一个浏览器插件 = 一个包 + 三处注册（tsconfig.client 聚合、web-app 的 patch 行、web-app 的依赖）"。
6. **（纸面）设计一个 Chat 节点。** 挑一个现在没有专门卡片的场景（例如"某工具连续失败三次"），按 3.6 节的三步写出：`ConversationNodeDefinition` 的 `match` / `start` / `update` 各读到哪个事件的哪个字段？它注册进 `ctx.uiConversation.events` 后，renderer 注册进哪个 slot 的哪个 key？最后回答：这个改动要不要新增 Session 事件？（提示：如果只是"怎么画"，不要。）

## 7. 本章小结

本章压成十五条。前十条是本章要点，后五条把全书串成一句话。

1. **产品面不由入口决定，由 profile 决定**：`web` / `headless` / `sdk` / `sdk-minimal` / `acp` 是五份 bundle 清单，`desktop` 保留给 Electron。
2. **`dsh` 是唯一受支持的 Node 应用启动器**，由 `scripts/verify-application-entrypoints.ts` 的三张 allowlist 在 CI 里强制；`bin.ts` 只有 78 行且零业务逻辑。
3. **组合在空根上叠四层 patch**：bundle 层 → profile → home → `--patch`；`boot()` 在 required 条目（含 `modules`、`connection`）失败时 dispose 整个应用并非零退出。
4. **Web 是三段式**：Host 半边（权威状态与 HTTP 服务）/ Remote 层（Typert RPC）/ 浏览器半边（60+ 包的内核与 UI）；依赖方向单向，前端绝不直连内核。
5. **Typert 的编程模型**是 `@Remote` / `@RemoteScope` 标注 + 构建期生成的 `InvocationDescriptor`；复杂对象靠 `TypertLookupMap` 声明 wire 身份；源码运行时有 SRC 回退。
6. **Remote 两种传输**：一元走 `POST /api/...` 返回 `RemoteResult<T>`；流走多路复用的 `/api/remote.mux` WebSocket（默认 2s 心跳，逻辑流可带上行帧），并分"物理重连"与"逻辑重开"两层。
7. **信任栏**由 launch token → authority 绑定签名 cookie（30 天、HttpOnly、SameSite=Strict、不带 Secure）与 `api-request-trust` 的 Host/Origin/`sec-fetch-site` 检查组成；失败分 403（不可信）与 401（可信未鉴权），防 DNS rebinding。
8. **浏览器 boot 是两段式**（模块阶段 + 插件阶段），由 `dsh.client` 声明与 `/plugins` 路由驱动，`PLATFORM_MODULES` 提供隐式外部依赖；失败一个 entry 就不进完整 UI。
9. **事件到节点是 `ConversationNodeDefinition`**（`match` / `start` / `update` → State + Location → `ConversationSnapshot`）；加 Chat 节点 = 注册一个 Definition + 一个 keyed renderer；`ui-tool` 卡片经 keyed slot `tool.call.toolview` 分发。
10. **Desktop 是同一运行时的 Electron 壳**（RunAsNode 子进程 + `dsh-app://app/` + Node IPC 注入），与 dsh 永远同版本、独占 `desktop` profile、默认 OS 端口。
11. **TS SDK** 用 `DeepSeekHarness.run()` 拥有"回执 → 下一次 idle"的活动区间并返回 `RunResult`；服务器是 stdio `jsonrpc` 插件，`initialize` 是等树 settle 的就绪边界，stdout 只许 JSON-RPC 帧。
12. **Python SDK** 与 TS 版逐符号对称，且**强制显式 Harness home**、绝不静默用 `~/.dsh`；runtime wheel 五个平台自带可执行文件，不要求系统 Node。
13. **ACP 是 automation-only 的标准 ACP v1 子集**，不含 DSH 展示面；配置只有 `provider` / `model` / `sessionListPageSize`（默认 100）。
14. **人机交互是共享的一圈能力**（commands / user-approval / permission-presets / user-questions / feedback），不同产品形态各接各的 surface，内核不变。
15. **在 DSH 里加产品面特性，先问"这是组合、Remote 方法，还是新事件"**：能靠配置与插件扩展点解决的，绝不改循环、绝不新开入口。

**全书收束——把 13 章串成一句话：**

- **地基（第1–4章）**：DSH 是一棵"没有特权核心"的 Cordis 插件树，一切注册都是可逆效果，事件分五路派发、三个事件域。
- **主干（第5–8章）**：Agent 循环从会话日志推导每个请求，模型经 llm 接缝被调用，工具经执行管线落地，而"模型可见即可重建"把记忆收拢成一份仅追加事件日志。
- **专题（第9–11章）**：一次请求里塞什么由上下文工程组装，对话太长由压缩以 `replace` 遮蔽（对模型遗忘、对用户全存），持久化格式按版本迁移、会话可 fork 与 resume。
- **组合（第12章）**：产品形态不是写死的入口，而是从四层 patch 叠出的插件树，你的覆盖永远在最上层。
- **产品（本章）**：同一棵树经三条投射通道长出 CLI、Web、Desktop、TS/Python SDK 与 ACP——**内核一份，形态五片，组合决定它长成哪片。**

## 8. 必读源码回顾与延伸阅读

**本章必读源码（按出场顺序）：**

- [apps/cli/src/bin.ts](../../apps/cli/src/bin.ts) — 唯一入口的全部逻辑（78 行）；[apps/cli/src/args.ts](../../apps/cli/src/args.ts) — 命令文法唯一所有者 `parseDshArgs`。
- [apps/cli/src/profile-boot.ts](../../apps/cli/src/profile-boot.ts) — `runProfile`（L244）共享 profile 生命周期，Desktop 复用；[apps/cli/src/plugin.ts](../../apps/cli/src/plugin.ts)、[apps/cli/src/dump-config.ts](../../apps/cli/src/dump-config.ts) — 另两条命令族。
- [packages/boot/app-boot/src/index.ts](../../packages/boot/app-boot/src/index.ts) — `boot`（L973）、`auditStartupEntries`（L926）、`loadLayeredEnv`（L235）、`installFailLoud`（L680）；[packages/boot/app-boot/src/profile.ts](../../packages/boot/app-boot/src/profile.ts) — `PROFILE_TEMPLATES`（L179）。
- [docs/subsystems/web-client.zh.md](../../docs/subsystems/web-client.zh.md) — Web 三段式与数据通路；[docs/api-gateway.md](../../docs/api-gateway.md) — Typert Remote 编程模型与生成管线。
- [packages/api/README.md](../../packages/api/README.md)、[packages/api/gateway/README.md](../../packages/api/gateway/README.md) — Remote 层与 mux 传输。
- [packages/client/connection/README.md](../../packages/client/connection/README.md)、[packages/client/connection/src/api-request-trust.ts](../../packages/client/connection/src/api-request-trust.ts) — 信任栏。
- [packages/client/web/README.md](../../packages/client/web/README.md)、[packages/client/modules/README.md](../../packages/client/modules/README.md) — 两段式 boot 与 `dsh.client`。
- [packages/client/ui-conversation/src/client/contract/conversation.ts](../../packages/client/ui-conversation/src/client/contract/conversation.ts)（`ConversationNodeDefinition` L203）、[packages/client/ui-chat/src/client/conversation-nodes/register.ts](../../packages/client/ui-chat/src/client/conversation-nodes/register.ts)（L22）、[packages/client/ui-tool/README.md](../../packages/client/ui-tool/README.md) — 事件到卡片。
- [packages/host/README.md](../../packages/host/README.md) — Host 半边；[apps/desktop/README.md](../../apps/desktop/README.md) — Electron 壳与关键决策表。
- [packages/sdk/README.md](../../packages/sdk/README.md)、[packages/sdk/client/README.md](../../packages/sdk/client/README.md)、[packages/sdk/protocol/README.md](../../packages/sdk/protocol/README.md)、[packages/sdk/server/README.md](../../packages/sdk/server/README.md) — TS SDK；[python/README.md](../../python/README.md)、[python/sdk/README.md](../../python/sdk/README.md) — Python SDK。
- [packages/acp/README.md](../../packages/acp/README.md)、[packages/acp/acp/README.md](../../packages/acp/acp/README.md) — automation-only ACP。
- [scripts/verify-application-entrypoints.ts](../../scripts/verify-application-entrypoints.ts) — "唯一启动器"的机械执行者；根 [package.json](../../package.json) 的 scripts 是命令真源。

**延伸阅读：**

- 回看 [第1章-开篇-DeepSeek-Harness总览](./第1章-开篇-DeepSeek-Harness总览.md)：本章讲的五种形态、三个视角、两大支柱在开篇都已预告；重读一遍，你会发现自己现在能看懂当时略过的每一个断言。
- 紧接 [第12章-扩展系统-配置即组合一切皆插件](./第12章-扩展系统-配置即组合一切皆插件.md)：本章反复出现的"bundle / profile / 四层 patch / `dsh.client`"都是它的展开；想动产品面，先在那里把组合模型吃透。
- 若要继续深入某条通道：[第6章-模型调用-llm能力接缝与适配器](./第6章-模型调用-llm能力接缝与适配器.md)、[第7章-工具系统-能力接缝与执行管线](./第7章-工具系统-能力接缝与执行管线.md)、[第8章-消息系统-会话日志即模型记忆](./第8章-消息系统-会话日志即模型记忆.md)、[第11章-会话管理-持久化格式迁移与分叉](./第11章-会话管理-持久化格式迁移与分叉.md) 分别是 SDK 的 provider 路由、`tool.call.toolview` 的数据来源、`$events` 投影与 resume 语义的地基。
- 全书地图：地基（[第1章](./第1章-开篇-DeepSeek-Harness总览.md)–[第4章](./第4章-事件驱动-五路派发与三个事件域.md)）→ 主干（[第5章](./第5章-Agent-Loop-让模型转动起来的引擎.md)–[第8章](./第8章-消息系统-会话日志即模型记忆.md)）→ 专题（[第9章](./第9章-上下文工程-一次请求里塞了什么.md)–[第11章](./第11章-会话管理-持久化格式迁移与分叉.md)）→ 组合（[第12章](./第12章-扩展系统-配置即组合一切皆插件.md)）→ 产品（本章）。读到这里，欢迎回来把 13 章当一份完整教材再走一遍。