---
title: "Cordis 热重启与热重载内幕"
tags: [dsh, cordis, hot-reload, hmr, lifecycle]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库源码直读：vendor/{cordis,loader,include,hmr}/src、packages/boot/app-boot、apps/cli/src/profile-boot；关键语义均以已构建 dist 真机最小复现实验确认（非 mock）"
updated: 2026-09-05
---

# Cordis 热重启与热重载内幕
> 概览篇说过“依赖变化 → 自动重启”“patch 一编辑 → 热重启该子系统”（见 [Cordis 内核](../cordis/cordis-kernel.md)
> 与 [插件组装与启动](../cordis/plugin-composition.md)）。本篇内幕：epoch 回卷重放、disposer 真实并发语义、shadow 重绑、
> Group 回滚形态、HMR 三级决策与双缓存失效、`!!js` 求值时机、boot → watchUserPatches → 重启链；证据基准：源码直读 + 真机实测。

## 内核：epoch 指纹驱动的回卷重放（`vendor/cordis/src/fiber.ts`，754 行）
- `_refresh()`（611 行 [MEASURED]）把依赖实现快照 `_store` 拼成 `':' + impl.fiber.uid` 串；任一依赖换 fiber 实例 → 串变 →
  `_setEpoch()`（625）：旧 epoch 为 INACTIVE → 直接 `_reload()`，否则先 `_unload()`。但 `_setEpoch` **先写入**
  `_runner.epoch` 再触发迁移，`_unload` 尾部（690-694）查得 epoch 非 INACTIVE 时**就地 `_reload`**——构成“换提供方 → 回卷 → 就
  地重放”，**不经过 PENDING 停靠**；PENDING 只在依赖彻底缺席时出现。真机实测：V1 注销 → 消费方回卷进 PENDING；V2 挂载 notify → 同一 tick 重放完
  成（+47ms）。
- `_reload`（646）：`await Promise.resolve()` 检查点——epoch 已过期即放弃执行插件代码（防 stale load）；`_resolveConfig`（
  641）走 `internal/config` waterfall + Standard Schema 同步校验；回调抛错 → 记 `_error` + epoch 置 INACTIVE（
  FAILED），`await()`（703）时重抛；`_reload` 期间 epoch 又变 → 迁移块（664-671）不结算而是再次 `_unload`（防丢更新）。
- `update(config, noSave)`（736）：非 ACTIVE → 只更新 `_config` 并 defer；ACTIVE → `_resolveConfig` 后走
  `internal/update` waterfall，**默认 `next()` = `restart()`**（718 行 = `_setEpoch(INACTIVE)` +
  `_refresh()`，同一配置标准回卷重放）。监听器不调 `next` = 否决重启；loader 的写回钩子靠 `await next()` 排在重启完成之后执行。
- **PENDING 期预注册 effect 的回卷**（构造函数 disposer，283-296）：fiber 创建时先 emit `internal/plugin`，观察者可能在
  PENDING 期就往 fiber 挂 effect；此时 epoch 本就 INACTIVE、`_setEpoch` 无迁移可驱动——disposer 显式 `_unload()` 回卷预注册工
  作，再 `while (this.inertia) await`。注释（289-293）言明：`inertia` 永不 reject，真 reject 只剩 logger 自身故障，**进程崩掉是
  诚实结果**。
- **唤醒链**：`provide()`（reflect.ts:277）的 disposer = `delete store[key]` → `notify([name])`（314）→ 遍历
  registry 全部 runtime 的全部 fiber，`name in fiber.inject` + isolate 过滤命中者 `_checkImpl` + `_refresh`。
  provide disposer 还 `await Promise.allSettled(fibers.map(f => f.await()))`——**提供方注销要等所有依赖方回卷完成才算完**
  ；实测顺序：provider 自身 effect 逆序先走（close2→close1）、consumer 回卷、provide disposer 压底收账；notify 尾部按 isolate
  filter 广播 `internal/service`（诊断面）。root fiber 的 `dispose()` = `restart()`（context.ts:107）。

## 异步 disposer 的真实语义：逆序启动、并发完成
- `_unload`（675）：`DisposableList.clear()` 返回**注册序数组再 reverse**（utils.ts:27-31），随后
  `Promise.all(map(async d => { await Promise.resolve(); runDisposable(d) }))` ⇒ 语义 = **逆序启动、并发跑完**，
  错误逐个 `logger.error` 吞掉、**永不 reject**。
- 对照：单个 effect 自身的 `dispose()`（fiber.ts:449-463）把内部 disposables 逆序**串成链**（`task = task.then(...)`）严格
  顺序完成——两套语义并存 [MEASURED]。`runDisposable`（114）+ `effectInertia` WeakMap（112）：已被别处开始释放的 effect，第二入口**
  加入在途任务**（join）而非重跑——双入口幂等合流。

## shadow 重绑：跨 ctx 消费的归属规则（`vendor/cordis/src/utils.ts`，287 行）

带 `tracker={associate?, property?, noShadow?}` 的对象（Service 实例、mixin 后的服务）被别的 ctx 读出时，`getTraceable`（
117）包一层 Proxy（`createTraceable`，165 行）：
- 读 `tracker.property`（即 `svc.ctx`）→ **返回读取方的 ctx**，不是服务创建方的；读 `associate.name` 点号属性（如
  `tools.execute`）→ 转读 ctx 上注册的访问器 → **任何插件可按名遮蔽服务的某个成员**（dotted accessor 机制）；
- 方法调用（非 `noShadow`）→ `createShadowMethod`（156）把 this 换成 shadow 对象（
  `withProp(receiver, property, ctx.extend({[shadow符号]: origin}))`，149 行）——方法体内
  `this.ctx.effect(...)` **落在调用方 fiber 上**；
- 嵌套 tracker 递归重绑；`noShadow`（logger/reflect 等身份敏感服务）保留 shadow 层供其解析 origin；set trap 禁写 `svc.ctx` 与
  original；可调用服务（`[Service.invoke]`）由 `createCallable`（226）每次调用现场 trace 再 dispatch。

真机归属实测：consumer 调 provider 服务的方法注册 effect → `ownerFiber=consumer`；重启 consumer 时该 effect 回卷重放，
provider 不动。这就是“注册即可逆 + 归属正确”的粘合层；Java 侧没有隐式 this 重绑的等价物（见 Java 移植观察）。

## Loader 的三个全局钩子（`vendor/loader/src/index.ts`）
- **await 闸**：provide `loader` 带 `[Service.check]`——intercept `{await:true}` 的消费方在 `getTasks()` 非空时
  保持 PENDING（EntryTree.await 46-63 循环 + init finally 的 `notify(['loader'])`，entry.ts:263-266）：“配置树未结
  算 → 依赖方不许激活”。
- **`internal/config` 树载体豁免**（92 行）：entry 根 fiber 激活时对 config 递归 `interpolate`；带 `EntryGroup.key` 的
  Group/Include **保持字面**——嵌套行的表达式归各行自己的 fiber。
- **`internal/update` 双钩子**：prepend 的（103 行）在 `await next()`（重启完成）后把新 config 写回
  `entry.options.config` + `tree.write()`（noSave 或子 fiber 跳过）；普通注册的（111 行）打 reload 日志。
- **`internal/plugin` 自我卸载七 case**（117-157）：created/未被跟踪/子插件/registry 删除路径/树在销毁/`_disposing` 在途
  /disabled 已知，七重过滤幸存的 `ctx.fiber.dispose()` ⇒ `entry.options.disabled = true; tree.write()`。真机证实：自我
  卸载后 YAML 出现 `disabled: true`，且其余 `!!js` 表达式原样保留。

## Group 事务回滚的精确形态与 `Entry.update` 四路径

`EntryGroup.update(config[])`（config/group.ts:59，全文 129 行）：新行/变更行**全部先行**（`Promise.allSettled` 并行
create）→ 任一失败即抛（单错直抛、多错 AggregateError）→ 回滚 = **逆序 remove 新增行**（`remove(id, true)`，isDispose 不
unlink）→ **重建全部旧行** → `data = oldConfig`；回滚再失败 → AggregateError 首位夹原错。护栏：`ctx.fiber.uid === null`（树已
没）直接 return 不回滚（75 行）；Group 的 `[Service.init]` 先 `yield () => this.stop()`（首个注册 = 最后执行）。

`Entry.update(options, create, force)`（config/entry.ts:142，全文 303 行）四路径：
| diff 情形 | 动作 | 失败处理 |
|---|---|---|
| 无 fiber | init | 还原 options |
| 目标 disabled | dispose | — |
| 仅 config | `_patchContext`（114）：重设 ctx 原型 + `fiber.update(config, noSave=true)` | 用旧 options 再 patch 一次=回滚，双错 AggregateError |
| 含 name/inject/group | import 新模块 → dispose → `_start` | 旧 options + 旧插件回滚重启 |

id 寻址：`EntryTree.sep=':'` 嵌套 id，`store[id]` 全局唯一复用 Entry 对象（换 parent 原地搬）。`getOuterStack`（248）把“哪个配置
文件哪一行”拼成伪栈帧 `at <baseUrl>#<entryId>` 注入错误堆栈——启动错误可归因到配置行，排错设计亮点。

## HMR：三级决策与模块失效（`vendor/hmr/src/index.ts`，576 行）

**前置**：`ctx.loader.internal` 必须存在（否则构造抛 `--expose-internals is required`，120 行）；它 == Node 内部级联
ModuleLoader，**按 API 形态探测 v1/v2**（看 `getOrCreateModuleJob`/`getModuleJobForImport` 谁存在，明确不看
`process.versions`——注释记载 24.0~24.11 误判教训，loader/internal.ts:148-166）。**externals 预计算**（217-224）：
watcher 开启前，从 `process.argv[1]` 的 loadCache job 递归 `linked`（跳过 `node:` 与 node_modules）取 CLI 入口依赖树 =
框架本体。

onChange 决策表（246-268，判定顺序自上而下）[MEASURED]：
| 序 | 条件 | 动作 |
|---|---|---|
| 1 | 文件 == 某 Include 的 filename | `refreshConfig`（串行化 refresh） |
| 2 | kind ≠ change 或非 config | add/unlink 忽略 |
| 3 | url ∈ externals | `loader.exit()` 整进程重启 |
| 4 | url ∈ loadCache | stash + debounce `partialReload`（400） |
| 5 | 其它 | 仅广播 `hmr/change`（watch-only 回退，交由消费方） |
- **ignoreInitial 死锁教训**：主 watcher `ignoreInitial: true`（239 行）——源码注释记录真实事故：初始扫描的 add 事件会 refresh 一个
  仍在 in-flight 初始 apply 的 Include，失败回滚与 refresh 互等 = **启动死锁**（`built-bin.e2e.ts:661` 有回归测试）。而
  `registerConfig`（134）的精确路径监视**保留初始扫描**（注册时已存在的 patch 层必须先 apply 一次），`findWatchRoot`（64）向上找现存祖先目录做
  depth 限制监视——**文件不存在也能 watch**。`refreshConfig`（297）按 per-key dirty/running 双态循环合并，错误 → warn + 广播
  `hmr/config-update-failed`。
- `partialReload`（400）：`analyzeChanges`（345）从 stash 起沿 linked 依赖图传播 accepted（未决态收敛 declined）→ 按
  baseUrl 解析各 entry 的 name 为 URL（v1 `resolve`/v2 `resolveSync` **参数序相反**，216-222）→ 对每个 plugin entry
  job 做 `loadDependencies`（37，含根自身），依赖树命中 accepted 才入 reloads。**双缓存清理**（461-475）：ESM loadCache 用
  `Map.prototype.delete.call`——**Node 24 的 `loadCache.delete` 只把键置 undefined 不真删**（注释明示的坑）；再清 CJS
  `require.cache`（Node 24 `import()` 双缓存都会命中）；先备份、可回滚。
- 应用：`registry.delete(plugin)`（dispose 旧 runtime 全部 fiber）→
  `oldFiber.parent.registry.plugin(newPlugin, oldFiber._config)` 在**原父 ctx** 用**原始未解析配置**重挂，回填
  `fiber.entry` 与 `entry.fiber`（loader 跟踪不断链）。import 失败 → esbuild BuildFailure 走 @babel/code-frame 代
  码帧诊断（error.ts），回滚 = 还原双缓存 + 旧插件对象重挂；成功广播 `hmr/reload`。

## `!!js` 求值时机表：表达式在谁的 ctx 求值
| 表达式位 | 何时求值 | 在谁的 ctx | 证据 |
|---|---|---|---|
| 条目 `disabled: !!js` | 每次挂载决策（update/refresh 的 `_disabled`） | **loader ctx**（entry.ctx，非插件 ctx） | entry.ts:94-104 `disabledOf`；表达式节点原样留在 options，**写回 YAML 保留表达式形态**（真机证实） |
| 条目 config 内 `!!js` | fiber 激活算 config 时（`_reload` → `_resolveConfig` → `internal/config`） | **插件 ctx**（inject 已就绪，可读被注入服务） | loader/index.ts:92-101 + fiber.ts:641；实测 `!!js ctx.dep+'-interp'` → `V1-interp` |
| Include/Group 自身 config | **永不自动插值**（树载体豁免） | — | `EntryGroup.key` 标记，loader/index.ts:99；嵌套行表达式延迟到该行自己的 fiber 激活 |
| Include patches 内 `!!js` | 逐条目走各自 `internal/config` | 各条目 ctx | include 只做 `applyEntryPatches` 结构化替换 |
| watchUserPatches 重算 | `entry.update` → `fiber.update(noSave)` → `internal/update` 链 | — | 见下一节 |

求值器本体：`new Function('ctx', 'expr', 'with(ctx){return eval(expr)}')`（loader/config/utils.ts:5-8）——
**with + eval 双料**；ctx Proxy 的 get trap 顺带把 `process`、`dshHomePath`（boot 里 provide 进 root，
app-boot/index.ts:807）变成表达式命名空间。`interpolate`（12）递归替换 `__jsExpr` 节点（YAML tag `tag:yaml.org,2002:js`，
predicate 往返序列化见 include/src/index.ts:9-14）。

## 完整链：boot → watchUserPatches → epoch 重启（live profile）

`boot()`（packages/boot/app-boot/index.ts:790）：new Context → `provide('dshHomePath')` →
`plugin(Loader)` → `prepare(ctx)`（启动环境快照 + cmdline 三服务，**先于任何配置树条目挂载**）→ `mountRootInclude`（519：内置
include/group 钉死 id 'include'，patches = **所有层一次性 flatten**）→ `loader.await()` 结算循环（每 await 后
re-check `ctx.get('loader')`——树可在途中被整棵 dispose，返回裸 ctx 不抛）→ `assertEntriesActivated`（616）审计：FAILED 点
名 + 原始栈、PENDING 点名**缺哪些服务**、rejection checkpoint 与 failLoud 去重；失败路径 `ctx.fiber.dispose()`（root 重启式释放
整树）+ 沿 cause 链折叠最深栈。`installFailLoud`（642）：unhandledRejection → 先写诊断再 `release()`（终端恢复，2s 超时上限，611 行
[MEASURED]；计时器保持 referenced 防事件循环空转 exit 0）→ exit(1)；handler 在 release 期间**不摘除**。

`patchReload` 消费点（apps/cli/src/profile-boot.ts:283-313）：live 且树仍 ACTIVE 时，若 profile 未自行启用 hmr → 注入
timer + 挂 **watch-only HMR**（`root: []` 零模块根、纯 config 监视；源码模块热替换默认关闭，由 base bundle 决定是否开）；随后对
profile/home patch 各挂一个 `watchUserPatches`（index.ts:253）。startup profile（
headless/sdk/acp/sdk-minimal，profile.ts:137-170 模板表；自定义 profile **默认 live** =
DEFAULT_PROFILE_PATCH_RELOAD，169 行）一个钩子都不挂——“一次性生命周期换依赖会自毁”（web 常驻可换、headless 跑完即走不值得换）。

refresh 全链（文件编辑 → 热重启）[MEASURED]：chokidar change → `refreshConfig` 串行 → `loadOptionalPatches`（
present-but-broken 抛，ENOENT=无层；`anchorInsertedPluginNames` 把 insert 行相对名钉成 file URL）→ `composeLive`（
bundle 层在下、用户层居中、overlay 在顶重排）→ `entry.update` → diff=config → `fiber.update(noSave=true)` →
`internal/update` → Include 自己的钩子（include/src/index.ts:206：path 未变 →
`enqueue(applyPatches(cached.data, newPatches) → root.update)`，**不重读文件、不重挂 Include 本身**）→ EntryGroup
事务 diff → 仅被 patch 命中的行走 config 路径 → fiber 回卷重放 → **未变行/未受影响子系统零扰动**。所有 `root.update` 经 `enqueue`（
225）单飞串行化（组事务不可重入，注释 218-224 记载 init apply 撞 watcher 初始扫描的竞态教训）；`INACTIVE_EFFECT` 在 register 期 = 应用在
退出 → 返回 no-op disposer 不崩。**写盘**：`Include.write()`（371）发 `loader/config-update` + `setTimeout(0)` 合批
→ writeQueue 严格串行 → tmp+rename 原子替换，EACCES/EBUSY/EPERM 退避重试 10 次 × `(n+1)*50ms`（35-42 [MEASURED]，
Windows 抗锁）；W_OK 只读探测 → readonly 拒写。

## patch 语义：整行替换不深合并；审计面 = 装载面

`applyEntryPatches`（include/src/index.ts:58）：`structuredClone` 输入（**绝不别名共享**——注释明说热重算需要能撤销，别名会烤死旧值）；
id 递归索引（group 的 config 数组入索引，**insert 的行当场补索引** → 同列表后条 patch 可打前条 insert 的行）；非 insert patch =
config **整行替换、不深合并**；可选 `name` 守卫（不匹配 warn skip）；无目标 warn skip。启动装载、`--dump-config`（
`renderConfigDump`，412：逐层 diff 生成 `# == 来源, patched by …` 注释行）与 profile 组合 `composeEntries`（
profile.ts:854）**共用同一函数**——杜绝“dump 说的和跑的不一样”。附带模块解析双锚：bundle 解析安装锚点**先**、profile 目录后；
`$DSH_HOME/profiles/node_modules` 镜像安装依赖闭包（BFS 含 peerDependencies——Service Definition 是实现的 peer，外置插件
必须共享同一 cordis 实例，profile.ts:565-600）。

## 概览篇说法 → 深读结论
| 概览篇说法 | 深读结论 |
|---|---|
| “fiber 卸载时 effect 逆序释放”（内核篇） | 精确化：批量释放 = **逆序启动、并发完成**、错误吞掉永不 reject；严格逆序串链只属于单 effect 的 `dispose()` |
| “Group 对一批 create 先全量执行、失败逆序回滚”（组装篇） | 证实但不完整：回滚 = 逆序 remove 新增行 + **重建全部旧行** + 还原 `data`；回滚再失败时 AggregateError 首位夹原错 |
| “条目自我卸载把 `disabled: true` 写回 YAML”（组装篇） | 真机证实，且文件其余 `!!js` 表达式原样保留；触发判定 = `internal/plugin` 七 case 过滤链幸存 |
| “`!!js` 在插件上下文插值”（组装篇） | 细化为时机表：`disabled` 在 **loader ctx** 求值、config 在**插件 ctx** 求值、树载体永不自动插值 |
| “注销会唤醒依赖方 → 自动重启”（内核篇） | 证实全链无 PENDING 空档；且提供方注销**等所有依赖方回卷完成**（allSettled）才算完 |
| 组装篇 HMR 速写与 patchReload 边界留白 | 回填：`analyzeChanges` → 双缓存失效 → 原父重挂全链；profile.ts:137-170 模板表 + profile-boot.ts:283-313 消费点；外加 PENDING fiber 预注册 effect 的显式回卷（fiber.ts:283-296） |

## 精选常量与数据结构字典
| 常量/结构 | 值/位置 | 语义 |
|---|---|---|
| HMR 默认配置 | debounce 100ms；root=`['.']`；ignored=`node_modules`/`.*`/`cache`/`data` [MEASURED] | hmr/index.ts:563-571（chokidar）；事件 `hmr/{change,reload,config-update-failed}` 见 24-31，后者显式标 @mode parallel |
| 写盘退避 | 10 次 × `(n+1)*50ms` [MEASURED] | include/src/index.ts:35-42；`ConfigFileError` 分阶段（read/parse/validate）见 126，ENOENT 首读有 initial 兜底 |
| 内建事件表 `internal/{plugin,status,config,update,get,set,listener,dispatch,service}` | events.ts:305-372 | 全部带 @mode 注解，gen-cordis-catalog 交叉校验；loader 侧 `loader/{config-update,entry-init,partial-dispose,patch-context}`（loader/index.ts:22-26，patch-context=waterfall 钩子挂载点） |
| `Realm`/`LocalRealm('#id')`/`GlobalRealm('@label')` + `EntryOptions{+intercept,+isolate}` | isolate.ts:36-72,8-12 | isolate 符号命名空间（realm GC 在 partial-dispose）；概览篇记的字段集其实来自模块扩充（core 版只有 6 字段） |
| `ModuleLoaderV1/V2` 形态探测 + `JsExpr{__jsExpr}` | internal.ts:60-120；utils.ts:26 | v2=Node ≥24.12 才有、两版 resolve 参数序相反；`!!js` 的 JSON 中性载体（可 structuredClone/JSON 往返） |

## Java 移植观察
**epoch 重启无需运行时黑魔法**：uid 指纹 + notify 全表扫描 + 逆序回卷重放是纯数据结构逻辑（真机复现在未开 `--expose-internals` 的普通 node 进程里
  完整跑通），JVM 完全可复刻。
**真正的分叉是模块级 HMR——JS 私有技巧**：loadCache/require.cache 双失效 + `ModuleJob.linked` 图遍历 + 原生 addon 取内部
  loader；Java 对应物（热替换 ClassLoader/Instrumentation/OSGi）语义不同且更重。DSH 自己也只留 watch-only 默认档（live 用
  `root: []` 跑 config-only HMR，源码热替换是 base bundle 显式开启的 dev 特性）——**config 热重启 = 产品必需件、模块热替换 =
  开发者便利件**，Java 版可先只做前者。
**shadow 重绑需要显式化**：JS 靠 Proxy + thisArg 替换实现“服务方法里的注册落在调用方 fiber”；Java 无 this 重绑，等价设计 = 方法签名显式携带
  caller-scoped 上下文（如 `register(CordisContext ctx, ...)`）或每次调用注入 `ResourceScope`；tracker/noShadow 二元
  （身份敏感服务保留 origin）也要在类型面上表达。
**`!!js` = with + eval，是安全面**：config 表达式可触碰 ctx 上的一切（`process.env` 等）。Java 等价物（SpEL/Groovy）同样有注入面，
  Spring 生态对 SpEL 沙箱有成熟讨论；移植时应把“表达式在谁的 ctx、什么时机求值”的时机表记成**行为契约**，不只是实现细节。
**异步 disposer 的并发语义是文档债**（JSDoc 写“逆序执行”、实现是“逆序启动并发完成”，实测澄清）。Java 移植若改为严格顺序 await，行为兼容但启动/卸载时延变长——两种选
  择都要写进契约测试。
**环境引导防护可直接继承**：`BOOTSTRAP_NAMES` 禁改名单已含 `JAVA_TOOL_OPTIONS`/`_JAVA_OPTIONS`/`JDK_JAVA_OPTIONS`（
  app-boot/index.ts:96-127）——上游已把 JVM 启动钩子列入“.env 禁改名单”，Java 版 harness 沿用这套白名单纪律即可。

## 诚实边界
- 待核实：`fiber.await()` 在“load 进行中又被 dispose”的多轮 inertia 链收敛性，只有代码走读，未做对抗性时序实验。
- 待核实：`analyzeChanges` 的 accepted 沿依赖方向传播、pending 段存在价值（推测=给共享库文件预分类；正确性由 partialReload 二阶段检查兜底）。
- 待核实：`internal/get/set` waterfall 在真实 dsh 里的消费方（只读到钩子本体，未 grep 生产端挂载点）。
- 待核实：`--expose-internals` 由 apps/cli launcher 哪个环节注入（execArgv 拼装处未读）。
- 待核实：`vendor/timer` 实现（`ctx.debounce` 来源）、logger 门面、sdk-minimal 的完整 patch 树。

## 相关
- 内核语义（epoch/effect/事件的地基）：[Cordis 内核](../cordis/cordis-kernel.md)
- 配置树与启动序的概览：[插件组装与启动](../cordis/plugin-composition.md)
- 启动链的应用壳半边：[应用壳](../platform/web-cli-boot.md)
- 热重启“产品状态零扰动”的持久化侧印证：[持久化与崩溃恢复](./persistence-crash-recovery.md)
- 移植决策总览：[Java 移植观察地图](../java-porting-map.md)
