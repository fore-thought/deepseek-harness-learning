---
title: "typert：从类型声明到类型化远程"
tags: [dsh, typert, rpc, codegen, websocket]
status: active
license: CC-BY-SA-4.0
evidence: "读 DSH 仓库 typert/*、api/gateway/src/*、core/{agent,session}/* 与 docs/api-gateway.zh.md；Node v22 真机实测已构建 dist"
updated: 2026-09-05
---

# typert：从类型声明到类型化远程

> 本篇回答：TS 类型如何变成跨进程 RPC——发现机制、构建期生成、分发顺序、帧协议、事件回推
> 各有什么硬约束，以及哪些广为流传的说法其实不成立。衔接概览页
> [host/client 分层与 API 网关](../platform/host-client-boundary.md)。
> 证据基准：DSH 仓库源码直读 + 本机真机实测。

typert 是 DSH 的类型化远程调用层：`protocol`/`registry`/`loader`/`generator` 四包加 `api/gateway`
（host+client）。设计主线一句话：**类型只在构建期存在，运行期只读 descriptor**。

```text
@Remote 声明 → analyzer（ts.Program → FaceModel/TypeGraph）
  → emitter（Zod + descriptor + d.ts 四表）→ typert.host.js / typert.remote-client.js
  → loader 经 exports["./typert"] 发现 → registry → gateway 分发 → ctx.remote.<ns>.<m>
```

## @Remote 的真实标记机制

概览篇曾记"运行时靠 `Symbol.for('cordis.remote.*')` 标记发现 `@Remote`"——**深读实测推翻**：
全仓 grep `cordis.remote` 零命中。真实发现 = 两条可见数据 + 一次 `ctx.reflect.props` 全扫：

- 原型上的**字符串键** `'@deepseek-ai/dsh-typert-protocol/remote-methods'`，值为 frozen
  `{version:1, methods:[...]}`（decorator addInitializer 写入，
  `typert/protocol/src/index.ts:134,267-314`）。用字符串键而非 symbol 是刻意的：让协议包的
  **第二个已安装副本**也能读到第一份写的标记（`docs/api-gateway.zh.md:133`）。
- 服务实例上的可见字段 `typertRemote`（`TypertRemoteService` 构造器或 `bindTypertRemote`，
  `protocol/src/index.ts:143-169`）。

**@Remote 三形态**（`index.ts:176-203`）：裸装饰器（direct）；`@Remote('alias')` 换 endpoint 名
（实现体名 ≠ endpoint 名时记 `descriptor.implementation`）；`@Remote({ mode: 'stream' })`
（选项对象必须**恰好只有** `mode` 一个键）。同一方法只许一个 `@Remote`。装饰器 =
**零运行时包装**：不改方法、不注册拦截，只在 fiber 初始化追加 frozen marker；重复同配置幂等、异配置 throw；
强语义全在构建期与分发期（发现：`api/gateway/src/index.ts:275-290`）。

生产 lookup/context 全集小得出人意料：lookup 仅 `agent` + `session`（wire `agentId`/`sessionId`，
类型均 `SessionId`），context 仅 `agent`——**整个远程对象协议目前就这两个实体**；wire id 即领域
id，不存在 id minting 或映射表（`core/agent/src/types.ts:12-14`）。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| 运行时靠 `Symbol.for('cordis.remote.*')` 发现 `@Remote` | **推翻**：全仓零命中；真实=原型字符串键 + `typertRemote` 字段 + `ctx.reflect.props` 全扫（见上节） |
| 文档"校验请求值和返回值"→ 返回值同样校验 | **推翻（运行时层面）**：`descriptor.result` 的 strict codec **任何运行路径不执行**（client 侧同样不 parse）；实测：result 声明 `z.number()`、返回 string → 调用原样通过。**输入**才双向校验（client `parseInput` + host decode）；result schema 真实作用=生成期约束 + 编译期类型 + JSON Schema 投影原料 |
| `TypertLookupMap` 有对象↔id 引用计数/GC 语义 | **推翻**：typert 四包无 id minting/计数/回收；lookup provider=无状态 store resolver（`resolve: sessionId => this.get(sessionId)`）。真实设计=**声明持久性**：provider 卸载后 `definitions` 表永不删（registry `service.ts:219,311-320`），让弱回退合成继续把该参数名归类为 lookup 而非放行成普通 JSON |
| seq 校验在哪侧？ | **证实：全在 client**。journal client 拥有 cursor 代数 + 缺口修补；host `stream-server` pump 对 seq 零感知；WS 帧本身无序号字段，seq 藏在业务 payload 里 |
| 背压与重连细节？ | **证实**：背压=**无应用层窗口**，全连接共享一条写 promise 链串行化 + client 无界 Deque；重连=每 connection generation **至多自动重试 1 次** |
| `@RemoteScope` / context-receiver 用量？ | **新发现：生产 0 使用**（src 全域 grep 与已构建产物双确认为零），三层完整实现但只有 fixture 在用；生产 agent 定向全部走 **direct + 自动 scope 投影** 变体 |
| 分发是否缓存业务对象？ | **证实：零缓存**——每次调用现解析注册表（见「gateway 分发」） |
| JSON-safe 值判定全栈一致？ | **新发现不对称**：`-0` 在宿主 decode 路径放行、在事件/结果路径拒绝（见诚实边界） |

## 发现与装配：loader、registry、构建强制

**loader**（插件 `typert-loader`，inject `['typert','loader']`，解析锚 = `ctx.baseUrl` 而非自身包
URL——pnpm 隔离布局下才能看见兄弟包）：

- 发现链：Loader entry 名 → `<pkg>/package.json` → **`exports["./typert"]` 锚**（字符串或带
  `default` 的条件对象）；`packages` 配置项显式点名嵌套包，缺失/坏 = 大声 throw。
- 增量：`internal/plugin` 事件标脏 entry 名 → `queueMicrotask` **合并 flush**；activation 首扫 =
  同一增量路径跑全部 entries，失败聚合成 `AggregateError` 让插件 FAILED；稳态下单包坏只 log
  不毒化其余。缓存永久（`artifactPath` 含负判断 + manifests 不过期）——**插件集变更要重启**。
- `validateTypertManifest` 把生成物当**不可信跨边界输入**做全字段深检（自述包名相符、
  `face='host'`、schema 实例带 `_zod`、invocation 约束复刻 registry 版、sourceLocation 正整数）。

**registry**（键格式：schema=`<package>#<name>`、face=`<package>#<face>`、endpoint=`<ns>/<method>`）：
`DescriptorStore` 双索引（endpoint + id），批内与存量冲突**整批拒绝**；commit 写 `history`（Set，
只进不出）→ `hasSeen` 支撑"撤回过的 strict endpoint 禁止弱回退降级"；变更通知观察者逐个
try/catch 不毒化。LookupStore **wire 声明终身制**（同 key 改声明 = throw）；`configure()` 组合层可
覆盖 resolver 但声明仍归提供方；`identifyHost` 两个 kind 同时认出同一 Context = throw。

**validateInvocation 约束精选**（`registry/src/service.ts:655-712`）：id/service/namespace/method
字符合法；wire 字段禁重复；lookup 参数禁 `acceptsUndefined` 且必须带 lookup key；json 参数禁带
lookup；`cancellation.parameter` 只能是 `'signal'`；scope 只许 direct 且必须**恰好一个 lookup
参数**且其 wire==scope.wire 且 lookup==scope.context；strict codec 必须有 `parse`。

## 生成管线：analyzer 是生成器 + lint + codemod

- 每 face 单独 `ts.createProgram`（composite:false, noEmit:true），分批 8 个合并模型，共享解析缓存。
- 两种绑定语法（字符串字面量硬要求）：类 extends `TypertRemoteService` 且构造器**首句直接**
  `super(ctx,'<key>'[, opts])`；或 `readonly typertRemote = bindTypertRemote(this,'<key>')`。
- 方法规则：public 实例、非 abstract、有实现体、**非泛型**、参数全具名 identifier、无
  rest/默认值/显式 `this`。取消参数名必须 `signal`（全局 `AbortSignal`）、必须末位，出现即剔除并记 `cancellation`。
- **边界双轨制**：`type` = 作者语法投影（进 d.ts 与类型图），`codecType` = checker 解析后的
  具体投影（只喂 Zod）；optional 参数 codec 自动并 `undefined`，顶层显式 `T|undefined` →
  descriptor `acceptsUndefined:true`。
- **拒绝表**（`assertRemoteJsonType`）：拒 any/unknown、bigint、symbol、class 实例、
  callable/constructable、symbol 键与 index signature、未解析类型参数、纯 symbol 键交集；递归
  类型必须可命名；wire 类型必须具名、从**非根 type subpath** 导出（选择=字典序最小保确定性）。
  放行字面量家族、never、union/intersection/tuple/array（递归）与普通对象。**branded 原语靠
  phantom 交集过滤通过**：品牌被剥掉、本体 string 留下，Zod 侧生成
  `z.intersection(z.string(), z.unknown())`（真机产物实测）。
- **缺显式注解**：check 模式 fail；write 模式**把推断结果自动注回源码**再整轮重跑——生成器兼做
  lint + codemod，把"约定"沉进构建闸而非 code review。
- **自动 scope 投影**：direct 方法、恰好一个 lookup 参数、其 key 同时存在于 ContextMap 且 wire
  typeSymbol 一致 → 加 `scope:{context,wire}`，同一 descriptor 由此在 client 长出双签名。

**产物与构建强制**：host face `typert.host.js` 导出 `TYPERT={package,face,schemas,invocations,model}`；
remote face 导出 `TYPERT_REMOTE={package,descriptors}` + default。`.d.ts` 一次合并 **4 张表**：
`TypertRemoteNamespace$<hex>`（裸方法名）、`TypertRemoteMap['<ns>/<m>']`（flat，**带 lookup 首参**）、
`TypertRemoteNamespaceMap['<ns>']`、`TypertRemoteScopeMap['<ctx>:<ns>/<m>']`（**去掉 lookup 参**）——
实测 fixture：direct `(agentId, request, signal?)` vs scoped `(request, signal?)`，stream 返回
`AsyncIterable<T>`；`dtsMap` 把生成属性映射回源方法（编辑器直跳实现）。构建强制（`workspace.ts:90-148`）：
`exports["./typert"]` 必须精确等于 host 产物对；有 remote 必配
`exports["./remote"]` + files 清单；没 Remote 方法却留着 remote 产物 = throw；writeBundle 对
无 remote 的包**主动删陈旧三件套**。

## gateway 分发：一次调用的完整顺序

每次调用**现解析注册表，零缓存业务对象**。调用序（`api/gateway/src/index.ts`）：

```text
endpoint 两段 → resolveDescriptor：strict 表 →（hasSeen 但缺席 = definition-unavailable，
  禁降级）→ SRC 现场合成（多服务同 endpoint = ambiguous-endpoint）
→ args 精确匹配（extra/missing；可缺 = json 且 (acceptsUndefined 或 src-json)；
  **lookup id 永不可缺**）
→ receiver context：direct=网关 ctx；context=provider.resolve(wire 值) 得活的 agent 级
  Context，再在该 ctx 上 get(service)
→ binding 一致性：binding.service === originalOf(receiver)（经 cordis symbols.original 拆追踪代理）
→ 逐参 decode（strict: zod.parse → assertJsonValue；src-json: 仅 assertJsonValue）
→ provider-mismatch 三查 → resolve undefined = lookup-not-found；RemoteError 原样重抛（保留业务码）
→ 追加 signal → Reflect.apply
```

**SRC 弱回退合成**（`index.ts:672-765`）：参数名取自 `Function.prototype.toString` 截括号内文本
（要求纯标识符、无重复）；`signal` 末位=取消；参数名命中**保留的 lookup definitions**（声明持久性
的消费点）→ 归类 lookup；无 zod、codec 全 `src-json`；id=`src:<serviceKey>#<endpoint>`。认领集 =
strict（local 表 **或 hasSeen**，撤回过的也占名防降级）+ `$events/result` + SRC 集；mux 路由注册前先过
`requestRejection(req)` 信任闸，未信任裸 HTTP 401/403。错误出口：结构识别的 `RemoteError` →
{code,message,details} 原样上 wire；其它折 `gateway/internal`；业务 abort → `gateway/cancelled`。

## 事件回推：双向远程的另半场

- `registerRemoteEvents` 要求**唯一 source**（重复注册 throw）；每 client 连接分配 uuid `clientId`（冲突重铸）；
  **新连接先补投全部 in-flight waterfall 再发 ready 帧**（`{type:'ready',clientId,host}`）——晚到 client 也能参与仲裁。
- waterfall 要求 subject `identifyHost` 得 **kind==='agent' 且非空** identity，且 request 上 `agent`
  字段 === subject；pending 事件挂 **Context effect** 观察者——agent 上下文释放即 cancel，请求自带 signal 并入。
- 结果仲裁 **first-response-wins + all-next-then-next**：某 client 交 `result` 即 settle 并向其余
  广播 cancel 帧；交 `rejected` 整次 reject（name/code/details 恢复）；交 `next` 只撤该 client、
  **全部交还才** `resolve(next())`。
- 可转发事件有**编译期过滤器**（`TypertForwardingMode` 条件类型）：只放行无 this 的 void `emit`
  与签名 `(request,next)`、`request.agent` 同时在 LookupMap 与 ContextMap 的 Promise 结果
  `waterfall`。装配侧 allowlist 实测 = 18 项、其中 2 项 waterfall（`approval/request`、
  `user-questions/request`）。

## 三流与帧协议：一条 WS，N 个逻辑流

- 帧协议（`stream-protocol.ts`，WS 路径 `/api/remote.mux`）：单 WS、**文本 JSON 帧**（binary →
  close 1003）。上行 `{type:'open'|'cancel',streamId,...}`；下行 `item`（`value` 键**整个缺席**=void
  项）/ `error` / `end`；键集精确匹配否则 close 1008；重复 streamId=1008；client 无效帧 close 4002。
- **背压 = 无应用层窗口**：server 每帧 await 进**全连接共享的一条写 promise 链**（保序即限速），
  无优先级；client 侧 `StreamInbox` 无界 Deque。赌的是消息小、本地/局域网、心跳判死兜底；
  终帧发不出去 = 判死物理代（close 1011）。心跳 = server `ws.ping()` 默认 **2000ms**，missed≥2
  （`MAX_MISSED_HEARTBEATS`）先 `setImmediate` 复查再 terminate；timer 均 `unref()`。
- **重连纪律**：`RemoteStream` supervisor 每个 connection generation **至多自动重试 1 次**（连接活着
  立即重开，断了等 generation 恢复再开 1 次）；非 carrier 错误一律终止并标记 `gateway/internal`。
  `accept()` 由领域层确认 opening baseline/cursor 合法后调用才重置 attempt；单消费者硬闸（taken）。
- 快照流：每代**恰好一个** opening snapshot（重复或 delta 先行 = violation）；retry 期间旧快照保持已发布。
- journal 流（抽象 = cursor 代数）：live entry 三分支——≤last 跳过、部分重叠 = violation、有 gap →
  **repair：`readPage(through=newEntry.last)` 与 follow 流赛跑**（期间新 entry 入队；generation
  更替 = superseded 让位新 opening）；**页尾必须恰好落在 `through`**；`mergeReplacement` 拒重叠、
  遇新缺口二次补页、还差 = violation。session 实例：cursor=number(seq)、`emptyCursor=-1`。

## 错误词汇与边界数字

- `RemoteError` = 真 Error + 结构标记 `isDSHRemoteError:true`，跨 realm 判别**不用 instanceof**；
  `RemoteFailure` 按 code 判别联合；`RemoteResult<T>={ok:true,value}|{ok:false,error}`——
  **carrier 失败一律折叠进 error 分支**，只有装配故障（arity/未挂载/无 adapter）才 reject。
- 码表：基码 3（`gateway/bad-request|cancelled|internal`）+ 17 个 `gateway/*` infra 码
  （detail 统一 `{endpoint,field?}`）+ 领域码（`session/not-found`、`session/agent-busy` 等）。
  endpoint 语法 `/^[A-Za-z0-9_$.-]+$/` 且非 `.`/`..`；registry segment 禁 `#`。
- 类型词汇：6 个可合并空 declaration map（Lookup/Context/Remote/RemoteScope/RemoteNamespace/
  RemoteEventSelection）+ `RemoteErrorDetailsMap`；Lookup/Context 用 phantom symbol 携带类型对。

## 诚实边界

- 待核实：`-0` 判定不对称（decode 放行、事件/结果路径 `Object.is` 拒绝）系静态读码所得，未实测。
- 待核实：analyzer.ts 约 40% 精读（96 个 fail 点全部枚举 + 关键段精读），`convertType` 全函数、
  merged interface 方差合并未逐行；renderer/cordis-catalog 只读头部。
- 待核实：session-controller 的 `@Remote` 方法体（prompt/cancel 冷恢复/ownership fence）未读；
  `lookups.configure('agent',...)` 生产调用点未定位。
- 待核实：rpc-host HTTP bridge 与 journal page-error + signal 竞态分支只到静态读通深度。
- 待核实：generator `mode:'workspace'` emit 与 package 模式并发语义未实测；`@typert service`
  标签全仓用量未清点。

## Java 移植观察

1. **@Remote 是零运行时语义纯标记**：Java 注解天然同构——RUNTIME 保留供 SRC 式回退，
   APT/javaparser 供严格产物；"强语义放构建期与分发期"的分工可直接照搬。
2. **SRC 回退有现成更强等价**：`javac -parameters` + `Parameter#getName()` 替代 `Function.toString`
   截参数名；但弱回退边界值得照抄——弱模式**保留 lookup 声明分类**，client **永远拒收弱 codec**。
3. **hasSeen 历史**（只增 Set）= 便宜易漏的防降级机制：撤回的 strict endpoint 占名、禁静默降回弱校验；JVM 平迁零成本。
4. **结构判别优先于 instanceof/异常类型捕获**——跨 realm 教训对应 JVM 跨 classloader 场景，同样成立。
5. **双轨投影动机可弃，导出纪律可留**：Java 走 APT 已解析模型可单轨；但"wire 类型必须是 public、
   非根导出的 API 面类型"这条纪律可平迁。
6. **lookup 参数命名即协议**（name==key、wire==`key+"Id"`）：改参数名=breaking change。Java 用
   `-parameters` 照抄，或注解显式化（更稳）。
7. **取消树是横切最大成本**：`AbortSignal.any` 组合取消贯穿全部流/事件/页/仲裁；JVM 无现成等价，
   需取消树/结构化并发/虚拟线程方案。
8. **背压现状 = 无协议**（单写链 + 无界 inbox）；JVM 传输层（如 Netty writability 反馈）能做得
   更强，属可选差异点。
9. **生成器兼 codemod**（write 模式自动注回类型再重跑）：约定沉进构建闸而非 review；Java 对应 Error Prone / AutoFix。
10. **产物命名纪律代码级强制**（两 face 共包、exports 逐字段比对、陈旧产物主动删）：Java 多模块
    用构建脚本锁 artifact 命名，防"产物在但约定漂移"。

## 相关

- host/client 分层与网关全貌（本篇所属概览页）：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
- 三流的消费方与会话历史投影：[持久化格式与崩溃恢复](./persistence-crash-recovery.md)
- PTC 代码运行时（另一条跨边界通道）：[PTC 与 code-runtime](./ptc-code-runtime.md)
- 事件语义基座：[Cordis 内核](../cordis/cordis-kernel.md)
- Java 移植总图：[Java 移植地图](../java-porting-map.md)