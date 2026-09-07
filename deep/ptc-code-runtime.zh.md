---
title: "PTC 与 code-runtime：把工具面编成 SDK"
tags: [dsh, ptc, run-code, code-runtime, worker-thread]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/core/tools/src/{ptc.ts,index.ts}、packages/agent-tool-presentation/src/index.ts、packages/code-runtime/{code-runtime,code-runtime-worker-thread}/src/*、packages/core/agent-loop/src/tool-calls.ts 源码直读 + 本机 Node v24 对已构建产物真机实测（精选 16 项）"
updated: 2026-09-05
---

# PTC 与 code-runtime：把工具面编成 SDK

[English](ptc-code-runtime.md) | [中文](ptc-code-runtime.zh.md)

> 本篇回答：模型从"逐个点名工具"切换为"写一段程序批量调工具"时，工具面如何坍缩成
> `run_code` 一个入口、SDK 面怎么生成、程序在哪个盒子里跑、预算怎么算。衔接概览篇
> [代码运行、LSP 与远程世界](../execution/remote-and-code-runtime.zh.md) 与[工具注册表与执行
> 流水线](../agent-runtime/tools-pipeline.zh.md)。证据基准：DSH 仓库源码直读 + 本机真机实测。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| 堆上限溢出 → worker-exit | **收窄修正**：仅 V8 old-space 溢出触发；TypedArray/external memory 不计入，狂堆只撞 compute 预算（实测 #11 A/B） |
| isolation 仅诊断标签、不构成安全承诺 | **证实并强化**：程序可动态 import node:fs / node:child_process / node:worker_threads，甚至伪造入站消息（实测 #15/#16）。诚实声明写在实现处（worker 模块头、`CodeRuntime.isolation` JSDoc："not a security claim"），SAFETY.md 只有泛化声明 |
| tools/ptc-dispatch-log waterfall 记录派发 | **精化**：waterfall 只重塑持久日志副本；真实事件是 `tool/code-dispatch-start` + `tool/code-dispatch` 成对，log-only，永不重入模型上下文 |
| binding 消息桥协议、字节预算、25ms 轮询 = 文档级印象 | **逐行重建**：wire 协议、位置保持剥离、双账本预算、ELU 轮询全部有行号证据与实测支撑 |
| PTC 调度器"与 native 类似" | **证实**：桥内每-run 迷你调度器完整复刻 native 循环契约，注释逐条互指 |

## run_code 传输桥（packages/core/tools/src/ptc.ts，678 行）

- **语言风味惰性解析**：`RunCodeFlavor{description, codeDescription}` 按 `CodeRuntime.language`
  键控（`satisfies` 编译期钉住与 `SDK_RENDERERS` 同步）；工具定义在注册期铸造、早于 runtime
  已知，故风味经 `defineProperty` getter 到 schema 发射时刻才解析（ptc.ts:660-677）：无 runtime
  → 降级 TS 风味（只来自文档目录收割，模型面永不见）、未知语言 → 响亮抛错。
- **每-run 迷你调度器镜像 native 契约**（ptc.ts:341-456）：每个子派发一个 `PendingDispatch`
  （start/classify/abandon/commit/flight/settled/mode）；单一有序 driver lane——先按提交序
  commit 队头、再按提交序 start 队头；容量闸 `!exclusiveActive && (exclusive ? inFlight==0 :
  inFlight<maxParallel)`；exclusive 屏障持到 commit 完成（含 post-execute）。分类在 **start 前
  重读** `registry.executionMode(input).kind`（ptc.ts:529，fail-closed）。有序阶段（start 事件、
  prepare、finalize/finish、context deferral）都在 driver lane 内，只有 body 阶段重叠。
- **binding 双重快照**（ptc.ts:150-166/463-599）：派发与日志的参数各做一次 `snapshotJsonValue`
  互相独立，工具改参污染不了日志；空参报错文案即 "call the tool with an arguments object,
  e.g. {}"。子调用 id = `parentCallId + ":code:" + n`（提交序编号、确定性）；settle 先 resolve
  程序、再排队 logWork——spill 等慢后端不得占用派发槽。
- **images 搭外层管道（deferContext）**：子调用结果含 image 时，commit 阶段把
  `createUserMessage{source:{kind:'plugin',plugin:'tools-code-mode'}}` 经 `exec.deferContext`
  挂到 run_code 自己的结果上（ptc.ts:561-566）；`additionalContexts` 逐条转发；`concludesTurn`
  仅成功嵌套结果可转发——政策化失败不能终止恢复中的轮次。实现侧按调用 WeakMap 键控，在外层
  tool/result 之后、下一步边界之前，deferred 前置合并进 additionalContexts 按调用序出（tools
  index.ts:1569-1580；agent-loop tool-calls.ts:147-161 消费）。
- **派发事件 log-only + 绑定面 = 可见集去 run_code**：`tool/code-dispatch-start` 与
  `tool/code-dispatch`（types.ts:10-57）带 rootCallId/parentCallId/subCallId 三级身份，UI 用
  subCallId 配对、time 差算耗时，`deriveMessages()` 忽略它们。三条通道各司其职：程序完成值 →
  模型文本；images → 下步 user 消息；dispatch 事件 → UI/回放。可绑函数面是
  `registry.schemas(exec.agent)` 去掉 `run_code`（ptc.ts:606-614）：程序可绑的恰是提示词承诺的
  （scoped 加入、restricted 消失）；命名空间 null-proto + defineProperty，`__proto__` 工具名也是普通 own key。
- **迭代式 JSON 渲染缩进封顶**：模型可见输出用迭代渲染器，两空格缩进、总缩进封顶
  `MAX_JSON_INDENT_CHARS=10`，深子树自动 compact，输出尺寸对 canonical JSON 线性。程序面失败包
  成 `CodeRunFailedError`（code=`CODE_RUN_FAILED`），文本带失败 kind + 捕获日志供模型自纠。
  run 级 `runController` 任何原因结算都 abort——在飞子派发被杀而非孤儿化，finally 里
  `drainDispatches()` 无痕放弃排队未启动者。`presentCall` 借 description 当卡片标题，刻意**无**
  presentResult——防大结果重复进 host 视图。

## 三模式工具面：发射面与执行面分家

- `ToolPresentationMode = native | ptc | both`；config `mode` 默认 native、
  `maxParallelSubCalls` 默认 10 [MEASURED]（tools index.ts:644/783-786）。
- **发射面 wireSchemas**（index.ts:972-993）：native → 全部可见 schema；ptc → 只留 `run_code` 且
  `knownNames=[run_code]`（native 工具名变非法）；both → 全部 + run_code。投影前
  `requireCodeRuntime(mode)` 验语言，渲染器表缺项 = 装配期错误。
- **执行面坍缩 collapses（index.ts:1315-1317）——安全相关谓词的唯一之家**：`!nested &&
  modeFor(scope)==='ptc' && name!=='run_code'`。模型直调只准点名 run_code；嵌套子派发（parent
  token 置位）全可见。mode 经 `modeFor`（链上最近 scope）解析、不读部署默认——否则 native 部署
  下 preset 给的 ptc agent 会绕过坍缩，"announce 一面、执行另一面"。**坍缩在
  pre-execute/审批/guard 之前确定性拒绝**（index.ts:1364-1372）：策略监听器永远不会"批准"一个
  必败调用；错误 = `UNKNOWN_TOOL` + 教程式文案（实测 #13）："only run_code is callable directly
  — call demo from inside a run_code program instead"。
- **两段提示词**：`tools:ptc-only`（坍缩规则段，text 与执行面共用同一谓词）与 `tools:sdk`
  （SDK 段，order `TOOLS_SDK=5000`，system-prompt/src/index.ts:149）；native 渲染空串 → 段被丢弃。
- **presentAs 与 config 分工**：`tool-presentation` 插件行（agent-tool-presentation，全文 72 行）
  只 inject tools、不 inject codeRuntime——native 行必须能在无 runtime 的部署挂载；
  `Config.mode` required，缺省即白组一行。仅 ptc/both 才 `ctx.inject(['codeRuntime'],
  presentAs(mode))`（L69-71）：runtime 缺失 = 挂载即响亮失败。presentAs 要求 scoped context、一
  scope 一声明，layer.mode + 两段同一 effect、卸载精确恢复（index.ts:938-966）。
  `DSH_TOOLS_MODE` env 只是 demo/headless 的部署级切换（标 TEMPORARY workaround），正式机制 = preset 行。
- **view() 解析序**（index.ts:1143-1184）：继承层合并 → 链上 restrictions 求交 → 本 scope own 注册
  最后遮蔽 → transport 在能力过滤**之外**最后插入（per-scope：native agent 的分发表不因他处 ptc
  混入 run_code）；注册期无条件保留 run_code 名。schema 两层：defineTool DSL 里 `type:'json'`
  合法、投影为省略 type 的任意 JSON 节点，`register()` 只认投影后 7 值封闭子集；run_code 自己的
  transport 不经 register()。

## SDK 代码生成（ts-types.ts）

- `renderToolsSdk` = 固定使用契约文案（与本库宿主系统提示 "Program-only SDK bindings:" 逐字同源）
  + `JsonValue` 别名 + `ToolArgsMap/ToolOutputMap/ToolName/ToolCallError/declare const tools`；
  `ToolSdkSchema = ToolSchema + output`；`sdkSchemas` 排除 run_code 并附 canonical output schema。
- **字典序排序 → 字节确定性 → prompt 缓存稳定**（L288-292 注释点明动机）。JSDoc 压行 + 转义星斜杠
  序列——schema description 不得截断生成注释；bash 示例**条件发射**：仅当 schema 真的接受示例字面
  量。Python 镜像：ptc 下生成的 SDK 是模型唯一形状来源 → 每对象渲染具名 TypedDict。

## code-runtime seam：跨语言可移植契约

- 词汇表（code-runtime/src/types.ts）：`CodeBindingFunction = (args: unknown) => Promise<CodeJsonValue>`，
  两端 lossless JSON、**无** seam 级字节帽；`CodeRunRequest{program, bindings, signal?}` 遵循
  "explicit-over-implicit"——请求不带调参、预算全归实现 config；abort 语义"runtime 只停止发问，在飞 binding 归调用方结算"。
- **六失败正交 kind**（L91-108）：`exception`（含 parse/transform 失败）、`timeout`（compute/wall）、
  `abort`、`worker-exit`（substrate 死亡，如 OOM）、`invalid-output`、`output-limit`。
  **error-as-field**：error 是 resolved 结果的字段、永不 reject；程序面错误只有 message
  过境（内部 metadata 留在契约外），worker 侧再包成可 catch 的 `ToolCallError`。
- **四保留集 = 跨语言可移植契约**（code-runtime/src/index.ts:40-87）：`RESERVED_BINDING_GLOBALS`
  = {console, __dsh_main__, __builtins__, __name__, __debug__}（console = worker 日志槽；dunder 系
  = Python bootstrap/CPython 编译期常量注入槽）；`RESERVED_ERROR_MEMBERS` = JS Error 排除项 +
  CPython 异常协议成员；`DUNDER_MEMBER = /^__.+__$/` 整族拒绝；`PORTABLE_RESERVED_WORDS` = ES ∪
  Python 并集——**加一门语言 = 对存量绑定名的 breaking review**。标识符规则
  `[A-Za-z_][A-Za-z0-9_]*` 排斥 `$tools` 这类 JS-only 拼写（实测 #5）。
- language/isolation 是只读串："informational, not gating"；isolation "not a security claim"。
  已知值 typescript/python；worker-thread / process / container。

## worker-thread 后端：防事故，不防作恶

- **strip-only 壳与位置保持 → 诊断坐标不漂移**（host index.ts:84/302-309）：模型代码包进 `async
  function __dsh_program__() { … }` 壳再过 `stripTypeScriptTypes`，壳与真实执行语法位一致
  （async body 允许顶层 return/await，裸模块 parse 会拒）；剥离保位（删除语法转空白）→ 按壳长度
  slice 回 body 后模型行列号原样。语法错或非可擦除语法（enum 等）= 程序失败、**不 spawn worker**
  （实测 #2）。
- **每 run 新建 Worker**：`env:{}`（比 subprocess 纪律的 scrub 更强）、`execArgv:[]`（不继承
  host loader 钩子）、`resourceLimits.maxOldGenerationSizeMb`（默认 512）；stdout/stderr 兜底捕获
  （JS 层 write 已被 patch，native 层漏网字节以 strayLogs 附在 done 日志之后）。开发世界直载
  src/worker.ts，发布走独立 CJS bundle lib/worker.cjs——实测 #7 的异常栈帧证实构建世界确走 CJS。
- **入站敌意闸 = 形状重建 + 幂等，不防说谎**（parseWorkerMessage，L133-165）：对端跑模型代码，
  编译期类型无意义——逐字段校验+重建（伪造多余字段不随行、非 number call id 永不被回显）；junk
  静默丢弃（host 监听器抛错 = 宿主进程崩）；answered 集每 id 至多一答；binding 查找限 own-property
  （伪造 constructor 走不上原型链）；完成值 host 端再解码再验。实测 #15/#16 实弹：程序经
  `import('node:worker_threads')` 拿真 port 可伪造 log 行（正常收录）与 done（抢先结算）——
  **防事故不防作恶**，真实安全边界在审批/沙箱 seam（见 [沙箱执行](./sandbox-execution.zh.md)）。
- **双账本字节预算、编码前计数**：worker 侧 LogBuffer 与 host 侧 OutputLedger 同一算法：起始 2
  字节、条目间逗号 1、`jsonStringBytesUpTo` 不物化转义形态直接数 UTF-8（代理对=4、引号反斜杠=2、
  孤儿代理=6、控制符 2 或 6）；超限发"放得下的前缀" + 一次性 output-limit 信号、诊断句也计入；logs
  与完成值共享 `maxOutputBytes`（默认 67,108,864=64MiB，下限 4）；实测 #10：120B 帽留 78 字符前缀。
- **busy-time 与 wall 双型 timeout**：busy 预算 = host 每 25ms（`ELU_POLL_INTERVAL_MS`，非配置项
  ——粒度是唯一效果）轮询 worker 的 `eventLoopUtilization().active`，超 `computeMs`（默认
  60,000）判 "compute budget exhausted (Nms busy)"——**等慢工具的不计费、热循环不可耍赖**（实测
  #8）；`maxWallMs`（默认 600,000）纯兜底 await 永挂（实测 #9）。**堆 = 仅 V8 old-space**：溢出 →
  worker-exit "JS heap out of memory"；TypedArray/external 不计入（实测 #11，即收窄修正）。
- **结算首到赢 + drain 序**：`finish()` 首到赢：清计时器/监听 → setImmediate 让已排队 pipe 字节先
  落 → terminate + await 双管道 drain + await exit → 再 resolve——日志在超时/中止/异常路径全部保留。
  dispose = 标记不可用 + 在飞全判 `abort('runtime disposed')` + await 到静默。
- **AsyncFunction 形参注入的撞名副作用**（bootstrap.ts:405-412）：程序体交给
  `(async(){}).constructor`（AsyncFunction 非全局，经实例拿构造器），形参表 = namespace globals
  + errorClass 名 + console。代价：注入名占用形参位——实测 #16 活证：程序顶层 `const tools =
  …` → `SyntaxError: Identifier 'tools' has already been declared`。console 替身仅 5 级方法、
  inspect 限深；stdout/stderr write 被 patch 进同一缓冲且保 Node 回调契约（queueMicrotask）。
- **intrinsic 捕获纪律 + 迭代遍历 + 扁平 token wire**：桥内部全部走预捕获原语（Reflect.apply 调
  Array.prototype 系、defineProperty 写槽、append/takeLast 不碰 `Array.prototype.push`），跨
  realm 对象经"构造器原生代码串 + 原型链"验真；循环检测 = active Set + 迭代遍历，三处零递归——栈深
  与数据深无关（实测 #3/#14）。`WorkerJsonWire = Token[]` 前序展开（容器 =
  `{kind:'array',length}` / `{kind:'object',keys}`、标量直接是元素）——structured clone 只见
  一层；解码端严格：稠密数组、字段精确（多一少一皆拒）、键去重、非有限与 -0 拒、根唯一、尾帧不完整拒。

> 一个有趣的互证：深读实测期间一段真实失败路径的栈帧 `src/bootstrap.ts:259/296/343`，与源码直读
> 定位的 bindingFailure、wireReplies reject、makeNamespaces reject 三处行号逐一吻合——源码运行时互证。

## 真机实测精选

方法：直接 import 已构建产物（code-runtime-worker-thread 的 lib/index.js）在裸 Context 上 new
runtime（绕 Loader，config 手填全量）；端到端另装 SystemPrompt + ToolRuntime(mode:'ptc') + demo 工具。全部 [MEASURED]：

| # | 实测内容 | 结果 |
|---|---|---|
| 1 | happy path：binding echo 往返 | `Object.keys(tools)`=声明名、`getPrototypeOf(tools)=null`、`Object.keys(process.env).length=0`（env 清空证实）；`require` 不可见、`fetch` 可用 |
| 2 | 含 enum 的程序 | exception "TypeScript enum is not supported in strip-only mode"，不 spawn worker |
| 3 | 20 万层嵌套数组 | 往返成功（扁平 token + 迭代遍历生效） |
| 4 | 完成值为 function；完成值含 undefined 字段 | 均 invalid-output（无损 JSON 严格） |
| 5 | binding global `$tools` / `console` | 均 REJECT："not a usable identifier" / "reserved binding global" |
| 6 | 零参调用 binding；binding resolve undefined | 分别拒 "binding arguments must be lossless JSON" / "binding resolution must be lossless JSON" |
| 7 | 程序 throw | exception 带 stack（帧指 lib/worker.cjs eval 帧），throw 前 logs 保留 |
| 8 | 热循环 computeMs=300 | timeout "compute budget exhausted (300ms busy)"，先前 logs 保留 |
| 9 | 永挂 await，maxWallMs=3000 | timeout "wall-clock ceiling reached (3000ms)" |
| 10 | maxOutputBytes=120 + 刷屏 | output-limit：留 78 字符前缀 + 定长诊断 |
| 11 | 堆 A/B | (A) push 4MB TypedArray 不死于 OOM、只撞 compute 预算；(B) 纯对象泄漏 heap=64MB → worker-exit "JS heap out of memory" |
| 12 | binding 抛错 | 程序 catch 得 ToolCallError（toolName='broken'、instanceof true、msg 原样） |
| 13 | PTC 端到端 | wireSchemas 只含 run_code；程序 `tools.demo()` 嵌套成功 isError=false；模型直调 demo → UNKNOWN_TOOL + 教程式文案；`tools.run_code` 不是函数（绑定面去自身） |
| 14 | 原型污染实验 | 换 Array.prototype.push / Object.prototype / JSON.stringify / Object.keys 后，binding 往返与完成值快照照常 |
| 15 | 伪造入站 log | `{type:'log', text:'FORGED-LINE', extraPoison:1}` 被收下、多余字段剥掉、进正常 logs |
| 16 | 伪造 done；顶层撞名 | 伪造 done 抢先结算 'HACKED'、真 done 被 settled 闸丢弃；程序顶层 `const tools` → SyntaxError（形参注入活证） |

## 诚实边界

- 待核实：py-types.ts 只读了头部结构注释，Python 端 TypedDict 生成细则未逐行。
- 待核实：prepare/guards/approval 在 run_code 子派发中的完整重入路径只粗读接口；子派发触发审批 ask 的 UI 闭环未实测。
- 待核实：CPython 后端源码不在仓库（私有、不发布），其 RESERVED 集执行与 `__dsh_main__` 包装细节无法对照。
- 待核实：abort 后"在飞 binding 归调用方结算"只有代码路径为证，竞态未实弹；image 经 deferContext 的端到端也只读了代码路径。
- 待核实：生产装配路径（dsh-base tools 行 + preset 行）以源码/补丁文件为据，未起真 dsh 进程验证。

## 相关

- [代码运行、LSP 与远程世界](../execution/remote-and-code-runtime.zh.md) —— 本篇的概览篇
- [工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md) —— 被复刻的 native 调度器与坍缩挂点
- [会话事件日志：唯一真源](../agent-runtime/session-event-log.zh.md) —— log-only 派发事件的持久化去处
- [沙箱执行](./sandbox-execution.zh.md) —— run_code 代码真正受制的另一半边界
- [turn/step 主循环](../agent-runtime/turn-step-loop.zh.md) —— commit 与 deferContext 的消费侧
