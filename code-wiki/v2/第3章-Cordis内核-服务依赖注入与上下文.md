# 第3章 Cordis 内核：服务、依赖注入与上下文

> 本章读法：请打开 [vendor/cordis/src/](../../vendor/cordis/src/) 目录对照阅读，讲义中的每个行号都指向仓库里的真实源码。配套官方读物是 [Cordis Primer](../../docs/cordis-primer.md) 与 [Cordis 教程](../../docs/cordis-tutorial/index.md)；上一章我们看清了 [vendor/packages/apps 的分层骨骼](第2章-分层架构-vendor-packages-apps的骨骼.md)，这一章要下到最深的一层。

## 1. 本章导读

[第2章](第2章-分层架构-vendor-packages-apps的骨骼.md)说过，DSH 是一个"全插件"系统：agent 循环、工具注册表、模型适配器，甚至你将来写的每一个扩展，都是插件树上的普通节点。让这套说法成立的底座叫 **Cordis**——它以源码 vendoring 的方式进入仓库（[vendor/](../../vendor/README.md)，pinned 上游版本，[vendor/README.md](../../vendor/README.md) 记录了 23 条本地修改），DSH 对它拥有完整的审计与修补权。

Cordis 的设计思想可以用 [docs/cordis-primer.md](../../docs/cordis-primer.md) 开篇的五个概念概括（该文件 L9-13）：

1. **插件是实现 Service 的对象**——带 `inject`/`apply` 字段的函数，或 `Service` 子类；
2. **context 是服务仓库**——插件用稳定的 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`）按键取服务，而不是 import 某个具体实现；
3. **`inject` 声明依赖**——所需服务齐备插件才激活，加载顺序由服务需求表达，而非手动 boot 排序；
4. **类型化事件**（[第4章](第4章-事件驱动-五路派发与三个事件域.md)展开）；
5. **注册是可逆的效应**——一切安装都经 `ctx.effect()` / `ctx.on()`，能展开也能收回。

本章吃透其中的 1、2、3、5，也就是回答四个问题：

- `ctx` 到底是什么？为什么 `ctx.tools` 不用 import 就能用？
- 服务如何声明（`Service` 子类 + 声明合并）与注入？
- `inject` 如何决定加载顺序？服务换了提供者，依赖方会发生什么？
- 为什么每一次注册都必须是可回退的效应？

学完本章，你应该能读懂 DSH 任何一个插件源码的第一屏——那一屏几乎总是 `declare module`、`static inject` 和 `apply`/`Service` 子类的组合。

## 2. 从一个问题出发

想象你在给 agent 写"读文件"这个能力。最快的写法：

```ts
import { readFile } from './fs-local.ts'
```

问题在这一行 import 落笔的瞬间就出现了。DSH 确实需要一个本地文件系统（[packages/fs/fs-local/](../../packages/fs/fs-local/)），但总有一天你会遇到"文件在远程沙箱里"的场景——仓库里真的躺着另一个提供者 [packages/fs/fs-sandbox/](../../packages/fs/fs-sandbox/)（它的 `static inject = ['sandboxPolicy']`，见 [packages/fs/fs-sandbox/src/index.ts:56](../../packages/fs/fs-sandbox/src/index.ts)）。`import` 把实现焊死在调用点上，于是：

- **换实现要改代码**：从 `fs-local` 切到 `fs-sandbox`，所有调用点都得动；
- **无法按配置选择**：`cordis.yml` 只能选"哪些插件加载"，选不了"import 哪条路径"；
- **无法在运行中替换**：进程起来后，import 关系一毫秒都不会变。

你也许会退一步："那我定义一个 `FileSystem` 接口，再写个工厂函数返回实现。" 这解决了第一宗罪，但马上冒出三个新问题：插件们谁先启动、谁后启动？`fs` 的提供者中途被卸载或替换时，正在使用它的插件怎么办？进程退出时，谁负责把每个插件注册的定时器、监听器逐一清理干净？

Cordis 的答案是四个机制的组合，每一个对应一节正文：

| 问题 | 机制 | 正文 |
|---|---|---|
| 能力如何命名与实现解耦 | 服务（service）+ 上下文（context） | 3.2、3.3 |
| 谁先启动、缺了怎么办 | `inject` 依赖声明 | 3.4 |
| 换提供者时如何安全过渡 | 依赖跟踪 + fiber 重启 | 3.4 |
| 如何干净地拆除 | 可逆效应（effect）+ disposer | 3.5 |

这不是假想的需求。shell 接缝就有 [packages/shell/bash-local/](../../packages/shell/bash-local/src/index.ts)（L98 声明 `static inject = ['subprocess']`）、[packages/shell/pwsh-local/](../../packages/shell/pwsh-local/src/index.ts)（L124）等多个提供者；官方教程明说（[docs/cordis-tutorial/03-services.md:78](../../docs/cordis-tutorial/03-services.md)）：卸掉一个 `shell` 提供者、换挂另一个，所有注入 `'shell'` 的插件会"干净地对着新实现重启"。这句话背后的一整套机制，就是本章。

## 3. 正文

### 3.1 插件（plugin）：一切能力的装载单位

**直觉。** 插件是一个自描述的盒子：它随身携带"我叫什么"（`name`）、"我需要哪些服务"（`inject`）、"我提供哪些服务"（`provide`）、"我配置长什么样"（`Config`）以及"装载后做什么"（函数体 / `apply` / 构造器）。框架不关心盒子里面装的是工具注册表还是模型适配器，只要它满足三种形态之一。

**严格定义。** [vendor/cordis/src/registry.ts](../../vendor/cordis/src/registry.ts) L92-95：

```ts
export type Plugin<T = any> =
  | Plugin.Function<T>      // L121: 函数插件，签名 (ctx, config) => any
  | Plugin.Constructor<T>   // L126: 类插件，new (ctx, config)
  | Plugin.Object<T>        // L131: 对象插件，带 apply(ctx, config) 方法
```

三种形态共享同一份元数据接口 `Plugin.Base`（L100-111），其中 `inject?: Inject`（L106）声明所需服务、`provide?: string | string[]`（L108）声明对外提供的服务名。判断一个值是不是对象插件，registry 只做一件事（L8-10）：`typeof object.apply === 'function'`。

**源码位置与运行时形态。** `ctx.plugin()`（registry.ts L316）装载插件时，会为"这一次装载"创建一个 **fiber**——插件的一次运行时实例（[vendor/cordis/src/fiber.ts](../../vendor/cordis/src/fiber.ts) L184），它持有自己的 context、config、生命周期状态机（`FiberState`：PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED，另有 FAILED，fiber.ts L147-154）。同一个插件类可以被装载多次，每次一个独立 fiber。

**为什么这样设计。** DSH 需要把"哪些能力、什么配置"写进 `cordis.yml`，由 Loader 在运行时装配成树（[第12章](第12章-扩展系统-配置即组合一切皆插件.md)）；还需要 HMR 把单个插件作为更新单位热替换。两者都要求"能力的最小单位"有统一形态、可独立装载/卸载。被否决的替代方案是"一个巨大的 Application 类 + 手写 main() 装配"：配置组合、运行时插拔、按需重载都无从谈起。

### 3.2 Context：一座带 Proxy 门面的服务仓库

**直觉。** context 是一座"名字 → 服务"的仓库，同时又是你手里这一层的取货凭证：`ctx.tools`、`ctx.llm` 这些读法，都是在向仓库按键要货。

**严格定义。** Cordis 的 context 有两副面孔。编译期的公开形态是 `interface Context`（[vendor/cordis/src/context.ts](../../vendor/cordis/src/context.ts) L16-33）：

- `[symbols.isolate]`（L18）：服务名 → scope 标签的隔离映射；
- `[symbols.intercept]`（L20）：服务名 → 拦截 config 的映射；
- `root` / `events` / `logger` / `reflect` / `registry`（L22-32）：根上下文、事件总线、日志、反射层、插件注册表。

关键在于：这个接口是**声明合并（declaration merging）**的——任何包都可以写 `declare module '@deepseek-ai/cordis' { interface Context { tools: ToolRuntime } }` 往里加成员，比如 DSH 的 [packages/core/tools/src/index.ts:137-139](../../packages/core/tools/src/index.ts)。这就是"ctx 上永远能读出新键、且类型不丢"的原因。

运行时形态是 `class Context`（context.ts L42），但它本体是个 **Proxy**（L74）：

```ts
const self = new Proxy<this>(this, ReflectService.handler)  // context.ts L74
```

**`ctx.tools` 为什么能直接用——读取的完整路径。** 这是你必须掌握的机制，它发生在 [vendor/cordis/src/reflect.ts](../../vendor/cordis/src/reflect.ts) 的 `ReflectService.handler.get`（L136-171）：

1. **特殊属性直通**（L137-139 + `isSpecialProperty` L86-91）：symbol 键、`prototype`/`then`、`_` 开头的属性绕过服务解析；
2. **Context 自有属性直通**（L140-142）：`ctx.events`、`ctx.registry`、`ctx.fiber` 等真实存在于类实例上，直接返回；
3. **accessor 转发**（L147-149）：`ctx.on`、`ctx.plugin`、`ctx.effect`、`ctx.get` 这些方法并不长在 Context 类上，而是 `ReflectService` 构造时用 `mixin` 把 `events`/`registry`/`fiber`/`reflect` 的方法转发上来的（reflect.ts L219-222），每次读取都经 accessor 计算属性转发到对应服务；
4. **服务解析**（L152-167）：普通属性走 `internal/get` waterfall——从**你当前 fiber** 开始沿父链向上，逐层查 `fiber.store?.[prop]`（该 fiber 已注入且当前可用的服务快照）；祖先 fiber 注入过的服务，后代 context 也能读到。每向上走一层都要校验 isolate 标签一致（L164）。一路到根都没有，就抛出那句著名错误（L144）：

```text
cannot get property "tools" without inject
```

而如果一个 fiber 声明了依赖但服务当前不可用，错误换成 L160-161 的 `cannot get required service "..." in inactive context`。读到值后还会包一层 traceable 代理（`getTraceable`，[vendor/cordis/src/utils.ts](../../vendor/cordis/src/utils.ts) L117-125），让方法调用里的 `this.ctx` 绑定到调用方 context——这个细节本章点到即止。

**子 context：`extend` / `isolate` / `intercept`。** 这三个方法都不改变父 context，而是以原型链创建子层：

- `extend(meta)`（context.ts L99-107）：以父 context 为原型创建子对象，`meta` 里的键成为子层自有属性——fiber 装载插件时就是这样把 `fiber` 挂上去的；
- `isolate(name, label?)`（L121-125）：为服务名 `name` 建立新的 scope 标签（默认新造一个 symbol；传同一个 `label` 则并入同一 realm）。服务注册表 `ReflectService.store` 以这个标签为槽位键（reflect.ts L286-292），`_getImpl` 读取时用读取方自己的标签找槽（L237-243）——标签一致才互相可见。这是 [第12章](第12章-扩展系统-配置即组合一切皆插件.md) preset 隔离 realm 的底层原语；
- `intercept(name, config)`（L141-145）：给子层添加一条"服务名 → 拦截 config"，后续在子层之下启动的插件解析该服务 config 时会把它合并进去（合并算法见 3.3 的 `resolveConfig`）。

**一张示意图记住 context 与 fiber 的关系。** 根 context 只有一个；每次 `ctx.plugin()` 都经 `extend({ fiber })` 派生一个子 context，同时新建一个 fiber；`isolate()` 再派生一层携带新标签的 context。原型链承载 isolate/intercept 映射，服务解析则沿 fiber 父链向上：

```text
root context ── root fiber
 │
 ├─ ctx A（fiber A：插件 A 的运行时，root 上 ctx.plugin() 装载产生）
 │   │
 │   ├─ ctx A' = ctx A.isolate('tools')      ← 本节刚讲的 isolate()
 │   │   └─ ctx A1（fiber A1：在 ctx A' 之下装载的消费者插件）
 │   │        它读 ctx.tools → 解析到 A' 的新 scope 槽位
 │   └─ ctx A2（fiber A2：未经隔离的另一个消费者）
 │
 └─ ctx B（fiber B：另一个插件，与 A 互不影响）
```

读 `ctx_A1.tools` 时，Proxy 从 fiber A1 出发沿父链逐层找 store 快照，每一步都校验 isolate 标签（reflect.ts L164）。[第12章](第12章-扩展系统-配置即组合一切皆插件.md)的 preset 正是用 `isolate` 让每个会话拥有独立的 `ctx.tools` realm。

另外 `Context.is(value)`（L61-63）用 `Symbol.for('cordis.is')`（L66）做跨 realm 判定——即使页面里同时存在两份 cordis 拷贝，`instanceof` 会失效而它不会。

**为什么这样设计。** 服务的集合在编译期是开放的：每个包都可能往 `ctx` 上贡献新键。Proxy 让"新增服务"不需要改框架代码；声明合并让新增键有类型。被否决的替代方案是"把所有服务写成 Context 类的真实属性"——框架每认一个服务都要改一次核心文件，与全插件架构背道而驰。

### 3.3 Service：把能力挂到 ctx 上

**直觉。** Service 是一个"有名有姓"的对象：构造完成的那一刻就自动上架（注册进仓库），随拥有它的 fiber 卸下（自动注销）。

**严格定义。** [vendor/cordis/src/service.ts](../../vendor/cordis/src/service.ts) L11：

```ts
export abstract class Service<out T = never> {
```

子类在构造器里 `super(ctx, name)`，父类构造器（L42-59）做三件事：给实例标上 `name`、把 `ctx` 存为 `this.ctx`、然后一行完成注册（L57）：

```ts
self.ctx.reflect.provide(name, self, this[Service.check])
```

`ReflectService.provide`（reflect.ts L277-305）内部把整个注册包成 `ctx.fiber.effect(...)`（L278）：注册成功写入 `props`/`store`，返回一个异步 disposer（L297-303）负责注销并唤醒所有依赖方。所以"构造即注册、随 fiber 卸载自动注销"不是约定，是 `provide` 的实现本身。同一 scope 内重复注册会直接抛错（L289-291：`service "..." has been registered at <fiber>`）——**misconfiguration fails loud**。

`Service` 类还定义了一组静态 symbol 键，每个都有明确职责：

- `Service.init`（L13）：**构造之后**运行的异步初始化方法，必须是 `async*` generator，每个 `yield` 出去的 disposer 都会被登记。为什么不用构造器？因为 JS 构造器不能 `await`。真实例子是 [packages/preset/agent-preset/src/index.ts:27-29](../../packages/preset/agent-preset/src/index.ts)：

```ts
async* [Service.init]() {
  yield await this.ctx.agentPresets.register(this.config)
}
```

- `Service.check`（L14）：可用性谓词。依赖方每次重估时，Cordis 会调用提供者的 check（fiber.ts `_checkImpl` L597-609），false 或抛错都视为"服务暂时不可用"，依赖方保持等待。这让"已注册但还没登录好"这类中间态可以表达；
- `Service.config`（L17）：intercept config 的类型幻影参数——它只在类型层出现，运行时不存在；
- `Service.invoke`（L19）：让服务本身可调用（`ctx.logger(name)` 就是这种形态）；
- `Service.resolveConfig`（L25，实现 L86-102）：沿祖先链把各层 `intercept()` 写入的 config 依序合并（根方向优先），有 `Config.merge` 用之，否则浅合并——3.2 的 `intercept()` 与 3.4 的对象形态 inject 最终都汇到这里。

**最小示例。** 教程第三章的标准写法（[docs/cordis-tutorial/03-services.md:11-35](../../docs/cordis-tutorial/03-services.md)）：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {   // 编译期：并入 Context 接口
  interface Context { greeter: GreeterService }
}

export class GreeterService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'greeter')                // 运行期：构造即注册
  }
  greet(who: string) { return `Hello, ${who}!` }
}

export const apply = (ctx: Context) => ctx.plugin(GreeterService)
```

注意两段代码各管一摊：`declare module` 只产生类型，不产生任何运行时代码；`super(ctx, 'greeter')` 只产生运行时注册，不产生任何类型。**两者都写，服务才"既能用又有类型"**。

**DSH 里的真实服务键。** 认识 `ctx` 是读懂 DSH 的第一步，下面这份清单全部来自仓库里的真实声明合并（挑你会最先遇到的）：

| ctx 键 | 声明位置 | 职责（后续章节） |
|---|---|---|
| `ctx.agents` | [packages/core/agent/src/index.ts:27-29](../../packages/core/agent/src/index.ts) | Agent 注册表 |
| `ctx.agentLoop` | [packages/core/agent-loop/src/index.ts:215-217](../../packages/core/agent-loop/src/index.ts) | 默认 agent 驱动器（[第5章](第5章-Agent-Loop-让模型转动起来的引擎.md)） |
| `ctx.sessions` | [packages/core/session/src/index.ts:40](../../packages/core/session/src/index.ts) | 会话日志存储（[第8章](第8章-消息系统-会话日志即模型记忆.md)） |
| `ctx.tools` | [packages/core/tools/src/index.ts:137-139](../../packages/core/tools/src/index.ts) | 工具注册表与管线（[第7章](第7章-工具系统-能力接缝与执行管线.md)） |
| `ctx.systemPrompt` | [packages/core/system-prompt/src/index.ts:15](../../packages/core/system-prompt/src/index.ts) | 系统提示词分段 |
| `ctx.llm` | [packages/llm/llm/src/index.ts:57-59](../../packages/llm/llm/src/index.ts) | 模型调用接缝（[第6章](第6章-模型调用-llm能力接缝与适配器.md)） |
| `ctx.shell` | [packages/shell/shell/src/index.ts:32](../../packages/shell/shell/src/index.ts) | 命令执行器 |
| `ctx.fs` | [packages/fs/fs/src/index.ts:47](../../packages/fs/fs/src/index.ts) | 文件系统 |
| `ctx.skills` | [packages/skill/skill/src/index.ts:285](../../packages/skill/skill/src/index.ts) | Skill 加载 |
| `ctx.compaction` | [packages/compaction/compaction/src/index.ts:90](../../packages/compaction/compaction/src/index.ts) | 压缩引擎（[第10章](第10章-上下文压缩-当对话太长怎么办.md)） |
| `ctx.sessionQuery` | [packages/session-query/session-query/src/index.ts:87](../../packages/session-query/session-query/src/index.ts) | 会话检索/导出 |
| `ctx.settings` | [packages/settings/settings/src/index.ts:41](../../packages/settings/settings/src/index.ts) | 用户设置 |
| `ctx.credentials` | [packages/credentials/credentials/src/index.ts:150](../../packages/credentials/credentials/src/index.ts) | 凭据 |
| `ctx.storage` | [packages/storage/storage/src/index.ts:32](../../packages/storage/storage/src/index.ts) | 非会话存储 |
| `ctx.agentPresets` | [packages/preset/agent-preset-registry/src/index.ts:25-29](../../packages/preset/agent-preset-registry/src/index.ts) | preset 注册表 |

同模式的还有 `ctx.web`（[packages/web/web/src/index.ts:37](../../packages/web/web/src/index.ts)）、`ctx.lsp`（[packages/lsp/lsp/src/index.ts:40](../../packages/lsp/lsp/src/index.ts)）、`ctx.sandbox`、`ctx.subagents`、`ctx.goals`、`ctx.spillStore`、`ctx.tokenMeter`（[packages/llm/token-meter/src/index.ts:96](../../packages/llm/token-meter/src/index.ts)）、`ctx.commands`、`ctx.jobs`、`ctx.sessionProjections`、`ctx.webhookRuntime`、`ctx.typertGateway` 等。

### 3.4 inject：用依赖声明代替启动顺序

**直觉。** 声明了 `inject` 的插件像一班等乘客齐全才发车的车：不发车也不报错，就是安静地停在那里；乘客到齐立刻发车，中途有乘客下车，它还会退回站台重新等。

**严格定义。** [vendor/cordis/src/registry.ts](../../vendor/cordis/src/registry.ts) L19：

```ts
export type Inject<M = Dict> = (keyof M)[] | { [K in keyof M]?: M[K] }
```

- **数组形式**（如 `static inject = ['agentPresets']`，[packages/preset/agent-preset/src/index.ts:13](../../packages/preset/agent-preset/src/index.ts)）：只声明依赖；
- **对象形式**（如 `{ llm: {…} }`）：额外携带**每个服务的 intercept config**。`Inject.resolve`（L71-88）把两种形态归一成"服务名 → config 或 null"的映射；Fiber 构造时把这些非空 config 写进子 context 的 intercept 原型链（fiber.ts L238-245），最终由 3.3 的 `resolveConfig` 合并。

声明可以写在三个位置：函数插件的 `export const inject`、类插件的 `static inject`（两者都是 `Plugin.Base.inject`，L106），以及 `@Inject()` 装饰器（L37-60）。用在类上时等价于 `static inject`；用在方法上则更精巧——方法调用被推迟到所需服务可用之后（L45-55），框架在背后以 `this.ctx.inject(inject, () => method.call(this))` 的形式为这个方法单开一道依赖门控：

```ts
class Reporter {
  @Inject('sessions')        // 声明方法级依赖
  report() {                 // sessions 可用后才被调用；
    // …                       // sessions 消失时门控随之卸载
  }
}
```

还有一个糖：`ctx.inject(deps, callback)`（L300-302）等价于 `ctx.plugin({ inject: deps, apply: callback })`，回调体在依赖变化时会整段重跑。

**加载顺序是如何被"表达"出来的。** fiber.ts 构造器的收尾（L314-319）对每个依赖调用 `_checkImpl`，然后 `_refresh()`（L611-623）：

```ts
for (const name of Object.keys(this.inject)) {
  const impl = this._store[name]
  if (!impl) { epoch = INACTIVE; break }   // 缺任何一个 → 不激活
  epoch += ':' + impl.fiber.uid            // 注意：记的是"哪个 fiber 提供的"
}
```

这个 `epoch` 字符串就是依赖状态的指纹：

- 有缺失 → epoch 为 INACTIVE → fiber 停在 **PENDING**，插件回调一行都不执行；
- 全齐备 → 进入 LOADING，`_reload()`（L646-673）先拍下依赖快照（L647 `this.store = {...this._store}`）再执行插件体；
- 运行期提供者**消失或换人**（uid 变了，L620）→ epoch 改变 → `_setEpoch`（L625-639）触发 `_unload()` 再视新 epoch 决定 `_reload()`。提供者被替换时，依赖方先整体卸载、再对着新实现重载——这就是"干净地重启"的机制本体。触发源是 `ReflectService.notify`（reflect.ts L314-336）：任何 provide/provide-撤销都会让所有声明了该名字的 fiber 重新 `_checkImpl` + `_refresh`。

**没有 optional。** 读到这你可能会问："依赖可以标成可选吗？"这份 vendored 的 Cordis 里 `inject` 只有硬需求语义（grep 全文找不到 optional 标记）。官方教程同样明说（[docs/cordis-tutorial/03-services.md:82-89](../../docs/cordis-tutorial/03-services.md)）：可缺的能力**不写进 inject**，改在使用点用 `ctx.get('greeter')` 探测（reflect.ts `get` L233-235，不在时返回 `undefined`）。

**为什么这样设计。** 被否决的替代方案是手动 boot 排序——在某个 main 里 `await initFs(); await initShell(); …`。它死于三处：`cordis.yml` 里插件行的顺序是数据而非代码，用户可以乱写；HMR 会在运行中增删提供者，静态顺序无法响应；preset 隔离 realm（[第12章](第12章-扩展系统-配置即组合一切皆插件.md)）里同一服务在不同 scope 有不同提供者，全局顺序根本不存在。声明式 `inject` 让每个 fiber 各自维护一张依赖表，框架负责把它们拼成动态的加载图。

### 3.5 effect：注册必须是可逆的

**直觉。** 每次向系统的注册（挂一个监听器、开一个定时器、贡献一段提示词）都签一张"回执"——回执就是 **disposer**，调用它（或拥有它的插件卸载时），这次注册被原样撤回。

**严格定义。** [vendor/cordis/src/fiber.ts](../../vendor/cordis/src/fiber.ts) 的 `Fiber.effect()`（L415-561）：

- `execute` **立即执行**（L522 调 `_execute`），产出的 disposer 被收集（L424-454）；
- 显式调用返回的 disposer、或 fiber 卸载，**先到先触发**；重复调用是 no-op（L427-429）；
- 收集的 disposers **逆序执行**——`dispose()` 里 `disposables.splice(0).reverse()`（L431），fiber 卸载时 `_unload()` 用 `DisposableList.clear()` 拿到的也是反转列表（fiber.ts L676 + utils.ts L27-31）；
- effect 体支持四种形态（L83-93）：返回一个函数、返回 Promise<函数>、同步迭代器、异步迭代器——generator 形态允许"边产出边登记"，适合一次注册多件东西；
- fiber 已 DISPOSED 或处于 UNLOADING 时再建 effect 会抛 `CordisError('INACTIVE_EFFECT')`（L419-421，错误码定义 L171-174）。

把 generator 形态写开——`yield` 的顺序就是注册顺序，拆除时逆序：

```ts
ctx.effect(function* () {
  const watcher = startWatching()            // 第 1 个注册
  yield () => watcher.stop()                 // 回执：拆除时后执行
  const timer = setInterval(poll, 1000)      // 第 2 个注册
  yield () => clearInterval(timer)          // 回执：拆除时先执行
})
```

框架自己就是这个形态的重度用户：`ctx.mixin` 为目标服务的每个成员逐个 `yield` 出 accessor 的 disposer（reflect.ts L366-390）。

**为什么要逆序。** 效应像一对括号：打开的必须闭合，闭合顺序与打开顺序相反。注册动作经常嵌套：先打开数据库连接（第一个注册），再挂一个使用这条连接的事件监听（第二个注册）。卸载时必须先摘监听器、再关连接——逆序执行正好把资源依赖反转成安全的拆除序。若卸载顺序重要，官方指引（[docs/cordis-primer.md](../../docs/cordis-primer.md) 实用规则一节）建议把相关工作放进**同一个** effect，让拆除按 yield 的逆序展开。

**`ctx.on()` 也是 effect。** 事件监听的安装同样包在 `ctx.fiber.effect` 里（[vendor/cordis/src/events.ts:256-259](../../vendor/cordis/src/events.ts)）：注册后返回的函数注销该监听器。所以插件作者挂的每一样东西——`ctx.effect` 里的定时器、`ctx.on` 的监听器、注册表 `register()` 退回的 disposer——都挂在自己 fiber 的拆除清单上。

**仓库纪律。** [AGENTS.md](../../AGENTS.md) 把这件事写成了硬规则（Conventions 一节）：

> **Registrations are effects**: every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer.

提示词分段、工具 schema、模型适配器、监听器，无一例外。为什么这么严格？对比一下"忘了清理"的经典 bug：定时器没人 clear、事件监听没人 remove，进程越跑越重、回调打到已死的对象上。在普通程序里这只是泄漏，在 DSH 里是灾难——HMR（[第12章](第12章-扩展系统-配置即组合一切皆插件.md)）每次热替换、preset 每次切换都会重跑插件体，任何没纳入 effect 的注册都会**每轮重载泄漏一份**。可逆性是整个扩展系统安全的根基。

**DSH 的本地加固。** 上游 Cordis 的 effect 模型在这里打过三个补丁（[vendor/README.md](../../vendor/README.md) 本地修改 #6，该文件 L38）：

- effect 的 owner 清单包装在 `execute` 执行**之前**登记（fiber.ts L504-520），从 setup 内部就开始的卸载也等得到 setup 及其清理；
- fiber 处于 UNLOADING 时拒绝新 effect（L420-421），防止清理期注册逃过卸载快照；
- 子 fiber 在对外发布 `internal/plugin` 之前就拿到由父 fiber 持有的 disposer（L265-297），同步观察者提前 dispose 任何一方都不会泄漏状态。

**为什么不用 GC 或引用计数。** JS 的垃圾回收时机不确定、`FinalizationRegistry` 不可依赖，而监听器、定时器这类注册恰恰是 GC 追踪不到的（事件总线持有回调 = 强引用，永远不会"自动消失"）。显式 disposer 把拆除变成确定性的、可逐条 await 的过程——确定性正是卸载与 HMR 需要的性质。

### 3.6 连起来：一个 15 行插件走完声明→注入→激活→卸载

把两个小文件放进教程的 `tmp/cordis-tutorial/` 目录（环境与启动命令见 [docs/cordis-tutorial/index.md](../../docs/cordis-tutorial/index.md)，L32-34 的 `node --import tsx ../../vendor/cordis/bin.js`）：

```ts
// greeter.ts —— 声明 + 提供
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {        // ① 声明：类型并入 Context
  interface Context { greeter: GreeterService }
}

export class GreeterService extends Service {
  constructor(ctx: Context) { super(ctx, 'greeter') }   // ② 提供：构造即注册
  greet(who: string) { return `Hello, ${who}!` }
}

export const apply = (ctx: Context) => ctx.plugin(GreeterService)
```

```ts
// consumer.ts —— 注入 + 使用 + 可逆注册
import type { Context } from '@deepseek-ai/cordis'

export const inject = ['greeter']              // ③ 注入：greeter 齐备才激活

export function apply(ctx: Context) {
  ctx.effect(() => {                          // ④ 注册为效应
    const timer = setInterval(() => console.log(ctx.greeter.greet('agent')), 1000)
    return () => clearInterval(timer)         //    卸载回执：Cordis 逆序调用
  })
}
```

```yaml
# cordis.yml
- name: './greeter.ts'
- name: './consumer.ts'
```

四个时刻分别对应正文四节：①声明合并（3.2/3.3）、②`super` 注册（3.3）、③inject 门控（3.4）、④effect 回执（3.5）。把两行配置换个顺序再跑——输出不变，因为顺序根本不由 YAML 决定。删掉 `greeter.ts` 那行——consumer 永远安静地 PENDING，连进程都会静默退出（[docs/cordis-tutorial/03-services.md:72](../../docs/cordis-tutorial/03-services.md)：PENDING 的 fiber 不会让 Node 事件循环保持存活）。

再留意一个陷阱：**不要把 provide 和 inject 写在同一个插件里**。fiber 在自己的回调运行**之前**就要依赖齐备，而它自己要提供的服务只有在回调运行后才存在——于是它等一个只能由自己提供的服务，永远 PENDING。提供者和消费者必须是两棵 fiber。

DSH 里这对"提供者 + 消费者"的工业级版本遍地都是：压缩引擎声明 `static inject = ['llm', 'tokenMeter', 'sessions']`（[packages/compaction/compaction-basic/src/index.ts:114](../../packages/compaction/compaction-basic/src/index.ts)）；agent 循环声明 `static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt', 'sessionProjections']`（[packages/core/agent-loop/src/index.ts:331](../../packages/core/agent-loop/src/index.ts)）——第5章的主角，原来也是排队等服务的普通插件。

## 4. 源码走读

现在把一个插件从装载到拆除的完整旅程串起来，每一步都给出真实的文件与行号。建议你打开这几个文件边读边走。

**第 0 站：根 context 诞生。** `new Context()`（context.ts L71-84）依次创建 isolate/intercept 两个原型映射（L72-73）、把自身包成 Proxy（L74）、创建根 fiber（L77），再安装四个内建服务：`ReflectService`、`RegistryService`、`EventsService`、`LoggerService`（L78-81）。`ReflectService` 的构造器（reflect.ts L213-223）顺手做了四组 mixin（L219-222）：`get/set/provide/accessor/mixin`、`runtime/effect`、`inject/plugin`、`on/once/parallel/emit/serial/bail/waterfall`——你在 DSH 里写的每个 `ctx.xxx`，要么是某服务的键，要么是这四行转发出来的方法。

**第 1 站：`ctx.plugin()`。** `RegistryService.plugin`（registry.ts L316-336）先用 `resolve()`（L222-228）把三种插件形态归一成回调函数（类插件按 `isConstructor` 区分，见 fiber.ts L250-261 的 `execute`：`new callback(ctx, config)` 或直接调用）；然后为该回调取/建 runtime 记录 `{ fibers: DisposableList }`（L322-328）——同一插件类的所有 fiber 都登记在这里，这就是 `registry.delete()`（L258-267）能一次 dispose 该插件全部实例的原因；最后 `new Fiber(...)`（L330）并返回一个可 `await` 的 fiber 包装（L331-335）。

**第 2 站：Fiber 构造。** fiber.ts L222-333：为这次装载创建子 context `parent.extend({ fiber: this })`（L236）；把对象形态 inject 的 config 写进 intercept 原型链（L238-245）；把"销毁我"注册成父 fiber 上的一个 effect（L265-297）——3.5 加固点之三就在这行；对外发布 `internal/plugin`（L302）；最后对每个依赖 `_checkImpl` 并 `_refresh`（L314-319）。依赖不全时，一切到此为止，fiber 停在 PENDING。

**第 3 站：激活。** 依赖齐备或发生变化时，`_refresh`（L611-623）算出新 epoch，`_setEpoch`（L625-639）从 INACTIVE 切换到激活，`_reload`（L646-673）拍下依赖快照、经 `internal/config` waterfall 解析 config（L642-643，校验函数 `resolveConfig` 在 L50-62，失败抛 `ValidationError` L19），然后 `_execute(this._runner)`（L656）真正跑插件体：返回的函数/迭代器产出全部登记为 disposer；类插件还要跑 `[Service.init]`（L253-257）。

**第 4 站：运行期。** 插件体里的一切注册经 `ctx.effect`（fiber.ts L415-561）或 `ctx.on`（events.ts L256-259）挂上 fiber；读服务走 Proxy（reflect.ts L136-171，见 3.2）。任何一个 provide 或其撤销都会触发 `ReflectService.notify`（reflect.ts L314-336），所有声明了该名字的 fiber 重新 `_checkImpl` + `_refresh`——依赖方随之卸载或重载。

**第 5 站：卸载。** `fiber.dispose()`（fiber.ts L196，实现在 L265-297 的 effect 里）置空 uid、从 runtime 的 fibers 列表移除（L266-275），然后 `_unload`（L675-696）：取出全部 disposers（`DisposableList.clear()` 返回反转列表，utils.ts L27-31），**逐条 await，每条独立兜错**（L676-686，一条 disposer 抛错只记日志，不饿死兄弟清理），清空依赖快照（L687），最后按新 epoch 决定回到 PENDING 还是立即重载（L688-695）——服务恢复时插件自动复活，就是这条边。

**第 6 站：诊断。** `fiber.getEffects()`（L568-572）返回当前 fiber 的效应树（带 label，L96-101 的 `EffectMeta`），`Fiber.update()`（L736-753）与 `restart()`（L718-723）支撑配置热更新。[docs/cordis-tutorial/06-composition-and-hmr.md](../../docs/cordis-tutorial/06-composition-and-hmr.md) 会带你诊断"插件为什么一直没加载"。

把六站旅程压成一张速查表：

| 阶段 | 发生了什么 | 关键代码 |
|---|---|---|
| 装载 | 插件形态归一、建 runtime 与 fiber | registry.ts L316-336 |
| 依赖初判 | 逐项 `_checkImpl` + `_refresh`，缺则停在 PENDING | fiber.ts L314-319 |
| 激活 | 依赖快照、config 校验、执行插件体与 `[Service.init]` | fiber.ts L646-673 |
| 运行 | 注册走 `ctx.effect` / `ctx.on`；读服务走 Proxy | fiber.ts L415-561 / reflect.ts L136-171 |
| 依赖变化 | `notify` 让依赖方先卸载再重载 | reflect.ts L314-336 |
| 卸载 | 逆序逐条 await disposer，每条独立兜错 | fiber.ts L675-696 |

## 5. 常见误区

**误区一：以为 `ctx` 是全局单例。** 它看起来像（到处都是 `ctx`），但每个 fiber 有自己的 context：`parent.extend({ fiber })`（fiber.ts L236）用原型链造出子层，服务可见性还要过 isolate 标签关（reflect.ts L237-243、L286-292）——同一服务名在不同 scope 是两个互不可见的槽位。preset realm（[第12章](第12章-扩展系统-配置即组合一切皆插件.md)）正是靠这一点让每个 agent 会话拥有自己的 `ctx.tools`。正确心智：**你手里的 ctx 是"从根到当前 fiber 这条链"的视图**。

**误区二：以为 `inject` 是可选的优化。** 它不是锦上添花的提示，而是激活的**前置条件**：缺服务，插件一行都不执行，永远 PENDING，且没有任何报错（PENDING fiber 甚至不维持进程存活，进程静默退出，见教程 03-services L72）。另注意这份 vendored Cordis **没有 optional inject**——可缺的依赖别写 inject，用 `ctx.get(name)` 探测（reflect.ts L233-235）：

```ts
export const inject = ['llm']         // 硬依赖：缺了整个插件不激活
const settings = ctx.get('settings')  // 可选依赖：没有提供者时是 undefined
```

DSH 里真有这种用法：preset 注册表就以 type-only 形式持有一个可选的 `settings` 服务（[packages/preset/agent-preset-registry/src/index.ts:9](../../packages/preset/agent-preset-registry/src/index.ts) 的源码注释明确写着 "the optional `settings` service"）。调试"插件为什么没跑"的第一件事就是核对它 inject 的每个键是否真的有人提供。

**误区三：以为 `Service.init` 是构造函数。** 它是构造**之后**由框架调用的 `async*` generator（service.ts L13；fiber.ts L253-257 调用），`yield` 出的每个值都是 disposer。把异步初始化塞进构造器在 JS 里不可能（构造器不能 await），所以异步启动 + 对应清理被拆到 init 里。对照真实样本 [packages/preset/agent-preset/src/index.ts:27-29](../../packages/preset/agent-preset/src/index.ts)。

**误区四：以为 `declare module` 块就是注册。** 声明合并只产生类型，不产生一行运行时代码（教程 03-services L40 原文：It generates no code）。只写 `declare module` 不 provide，运行时读 `ctx.greeter` 会得到 3.2 的那句 `cannot get property ... without inject`；只 provide 不写 `declare module`，功能正常但失去类型检查。两者是同一服务的两个半边。

**误区五：以为注册完就完事。** 凡是安装，必须有人负责拆除——这正是 [AGENTS.md](../../AGENTS.md) "Registrations are effects" 纪律的含义：`ctx.effect()` / `ctx.on()` / 注册表 `register()` 返回的 disposer，一个都不能丢。忘记返回 disposer 的插件在第一次 HMR 重载后就泄漏一份注册；第 12 章会看到扩展系统如何在每次配置变化时依赖这条纪律安全重装整个插件树。

## 6. 动手环节

**练习 1：跟官方教程敲前三章。** 按 [docs/cordis-tutorial/index.md](../../docs/cordis-tutorial/index.md) 的 Setup（L13-34）建好 `tmp/cordis-tutorial`，依次完成：

1. [01-first-plugin.md](../../docs/cordis-tutorial/01-first-plugin.md)：第一个函数插件；
2. [02-lifecycle-and-effects.md](../../docs/cordis-tutorial/02-lifecycle-and-effects.md)：卸载时效应如何被撤销；
3. [03-services.md](../../docs/cordis-tutorial/03-services.md)：本章 3.3/3.4/3.6 的所有断言——PENDING、顺序无关、依赖跟踪——都在这一章有可运行的验证步骤，尤其要做 L72 的"删掉提供者"实验。

**练习 2：在 DSH 里找 `static inject`。** 在 `packages/` 下全库搜索 `static inject = [`，找三个例子并记录：插件类、声明的依赖、猜测"谁提供这些服务"。参考答案（你应能找到至少 25 处）：

| 插件 | 位置 | 声明的依赖 |
|---|---|---|
| 压缩引擎 | [packages/compaction/compaction-basic/src/index.ts:114](../../packages/compaction/compaction-basic/src/index.ts) | `['llm', 'tokenMeter', 'sessions']` |
| agent 循环 | [packages/core/agent-loop/src/index.ts:331](../../packages/core/agent-loop/src/index.ts) | `['agents', 'sessions', 'llm', 'tools', 'systemPrompt', 'sessionProjections']` |
| 设置服务 | [packages/settings/settings/src/index.ts:224](../../packages/settings/settings/src/index.ts) | `['configEditor', 'profileContext']` |
| 沙箱 fs | [packages/fs/fs-sandbox/src/index.ts:56](../../packages/fs/fs-sandbox/src/index.ts) | `['sandboxPolicy']` |
| preset 行 | [packages/preset/agent-preset/src/index.ts:13](../../packages/preset/agent-preset/src/index.ts) | `['agentPresets']` |

挑其中一个，顺着依赖键反查提供者（在 `packages/` 搜 `provide = '...'` 或对应 `declare module` 块），画出一条"消费者 → 服务 → 提供者"的三节点边。

**练习 3：亲眼看到三条错误与一条静默。** 在练习 1 的环境里：

1. 让 consumer 不声明 `inject` 却读 `ctx.greeter`——观察 3.2 的 `cannot get property "greeter" without inject`；
2. 声明 `inject` 但删掉提供者那行——观察进程静默退出（没有任何输出）；
3. 在同一插件里 provide 又 inject 同一个名字——验证 3.6 的自等死锁：永远 PENDING；
4. 给 consumer 的 effect 注册两个东西（一个监听器 + 一个定时器），卸载时在两个 disposer 里各打一行日志——验证逆序。

## 7. 本章小结

1. Cordis 以源码 vendoring 进入 DSH（[vendor/](../../vendor/README.md)），五个核心概念：插件、服务仓库、inject、类型化事件、可逆效应（[docs/cordis-primer.md](../../docs/cordis-primer.md) L9-13）。
2. 插件有三种形态：函数、类、带 `apply` 的对象（registry.ts L92-95）；每次装载产生一个 fiber（fiber.ts L184），fiber 状态机为 PENDING/LOADING/ACTIVE/FAILED/DISPOSED/UNLOADING（L147-154）。
3. `interface Context`（context.ts L16）靠声明合并开放扩展；运行时的 `ctx` 是 Proxy（L74），普通属性读取经 `ReflectService.handler`（reflect.ts L136-171）沿 fiber 链解析服务。
4. `ctx.on`、`ctx.plugin`、`ctx.effect`、`ctx.get` 这些方法都是 ReflectService 构造时的 mixin 转发（reflect.ts L219-222）。
5. `Service` 子类 `super(ctx, name)` 构造即注册、随 fiber 自动注销（service.ts L57 → reflect.ts L277-305）；声明合并负责类型，两者缺一不可。
6. `Service.init` 是构造后的 async* generator，yield 出 disposer（service.ts L13）；`Service.check` 表达"已注册但暂不可用"；`resolveConfig` 沿祖先链合并 intercept config（L86-102）。
7. `inject` 决定激活：缺服务永远 PENDING（fiber.ts L611-623）；epoch 记录提供者 fiber 的 uid（L620），提供者替换会令依赖方先卸载再重载；本版本没有 optional inject，可缺依赖用 `ctx.get(name)` 探测。
8. 加载顺序不是写出来的，是"服务需求"自动表达出来的——`cordis.yml` 的行序无关紧要。
9. 注册必须是可逆效应：`Fiber.effect()` 立即执行、disposer 逆序执行、先到先触发、重复 no-op、UNLOADING 期拒绝新 effect（fiber.ts L415-561）。
10. `ctx.on()` 本身也是 effect（events.ts L256-259）；仓库纪律"Registrations are effects"（[AGENTS.md](../../AGENTS.md)）是 HMR 与 preset 切换不泄漏状态的根基。
11. DSH 对 fiber 生命周期打过本地加固补丁（[vendor/README.md](../../vendor/README.md) 修改 #6）：effect wrapper 先登记、UNLOADING 拒新、子 fiber 先拿 parent-owned disposer。

## 8. 必读源码回顾与延伸阅读

**必读源码回顾**（都在 [vendor/cordis/src/](../../vendor/cordis/src/)，按此顺序读）：

1. [context.ts](../../vendor/cordis/src/context.ts)——L16-33 接口形态，L71-84 Proxy 构造，L99/L121/L141 三个子层方法；
2. [service.ts](../../vendor/cordis/src/service.ts)——L42-59 构造即注册，L13 静态 symbol 一览，L86-102 config 合并；
3. [registry.ts](../../vendor/cordis/src/registry.ts)——L8-10 形态判定，L19 Inject 类型，L92-146 三种插件形态，L316-336 `ctx.plugin` 的完整实现；
4. [reflect.ts](../../vendor/cordis/src/reflect.ts)——L136-171 Proxy handler（本章 3.2 的主角），L277-305 provide，L314-336 notify；
5. [fiber.ts](../../vendor/cordis/src/fiber.ts)——L147-154 状态机，L222-333 构造，L415-561 effect，L597-696 依赖刷新与卸载；配合 [vendor/README.md](../../vendor/README.md) L38 的加固说明 #6。

**延伸阅读：**

- [第2章 分层架构](第2章-分层架构-vendor-packages-apps的骨骼.md)：vendoring 的动机与 vendor 层在仓库中的位置——为什么 DSH 要连框架一起拥有；
- [第4章 事件驱动](第4章-事件驱动-五路派发与三个事件域.md)：本章只打开了事件总线的 mixin（`ctx.on`/`ctx.emit`），五路派发模式、waterfall 语义与 `internal/service`、`internal/status` 这些本章略过的内部事件都在那里；
- [第12章 扩展系统](第12章-扩展系统-配置即组合一切皆插件.md)：本章的 isolate/intercept/effect 三块积木如何搭出 `cordis.yml` 组合、HMR 热替换与 preset 隔离 realm；
- [docs/cordis-primer.md](../../docs/cordis-primer.md)：五个概念 + 派发模式 + 实用规则的官方速查版；
- [docs/cordis-tutorial/](../../docs/cordis-tutorial/index.md)：七章动手教程，第 5 章（配置校验）与第 6 章（组合与 HMR）正好补齐本章没展开的两块；
- [vendor/README.md](../../vendor/README.md)：vendoring 清单与 23 条本地修改记录——读它，你会看到"拥有自己的框架层"具体是什么样子。

下一章我们给这座仓库装上神经系统：[第4章 事件驱动——五路派发与三个事件域](第4章-事件驱动-五路派发与三个事件域.md)。
