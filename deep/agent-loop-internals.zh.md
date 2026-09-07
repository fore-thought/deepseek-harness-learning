---
title: "Agent 主干内幕：turn/step 状态机、受保护流水线与作用域链"
tags: [dsh, agent-loop, session, tools, scope, event-sourcing]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/core/{agent,agent-loop,agent-default-model,agent-tool-presentation,scope,session,system-prompt,tools}/src 直读（43 文件 / 12,842 行 [MEASURED]）+ docs/{agent-lifecycle,event-producer-consumer,defensive-patterns}.zh.md + docs/subsystems/{core,tools,session,scope,system-prompt}.zh.md + 本会话真机旁证"
updated: 2026-09-05
---

# Agent 主干内幕：turn/step 状态机、受保护流水线与作用域链

[English](agent-loop-internals.md) | [中文](agent-loop-internals.zh.md)

> 本篇是主干概览三篇——[会话事件日志](../agent-runtime/session-event-log.zh.md)、
> [turn/step 主循环](../agent-runtime/turn-step-loop.zh.md)、
> [工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)——的 deep 档收口：
> `packages/core/` 全组 8 包源码级走读。概览篇讲"对外是什么"，本篇补"内部靠什么成立"：
> 相位状态机的私有迁移、创建事务的三方取消融合、调度器四段式、作用域链（含对
> 概览"两级扁平"口径的修正）、`…Map → derived-union` 类型模式与品牌化 id。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| （tools-pipeline）"两级扁平作用域：全局或恰好一个 scope key" | **修正**：现行 `dsh-scope` 支持作用域链（`bindScopeParent`/`scopeChainOf`/`ScopedLayers.chainLayers`）。注册视图沿链**向下继承**（远祖先铺、最近者改名胜出），事件准入沿链**向上延伸**（祖先作用域的监听者收得到后代派发）；"不向下继承给 subagent"仍成立——子代理是另起链根，不是链上后代 |
| （session-event-log）"resume 时 `interruptedTurnClosers` 在写句柄内合成补齐" | 归因说破：函数住在 `core/session/repair.ts:29-135`（纯函数，无 I/O），**调用点是 agent-loop** `resumeWith`：`open('write')` → `read(0)` → closers → 同一 `handle.append`（index.ts:876-879）。持久化篇说的"写侧首 mutation 消费"修物理撕裂尾，本篇的 closers 修语义配平——两层正交 |
| （turn-step-loop）"变了才 append `request/header`" | 细化：`reason` 四值 `initial/resume/change/series`；**surface 代际变了即使 header 不变也写 `series` 头**（`startsRequestSeries` = 压缩改写后要开新的模型消息系列）；第 2 个请求起，seed 配置从持久 header 剥离 `adapterDefaults` 标记的字段后再进 waterfall（agent.ts:60-67） |
| （turn-step-loop）"`cancel` 默认清 inbox" | 补三点：`keepInbox` 保留队列且**不记 canceled splice**；wake 输入的 `wakingAfterAbort` 分类**先于**插入捕获，重入 cancel 不能改判（agent.ts:126-131）；abort 后到达的 wake 一律重分类到 `next-turn`，靠 `wakeRequested` 闩在收敛点重放 |
| （tools-pipeline）"①append tool/call 持久事实先落盘" | 补全谱系：PTC 坍缩（collapse）拒绝的调用**也记 call+result 对**，错误里带"改从 `run_code` 内调用"的指路文案（index.ts:1427-1434）；取消时未启动的模型调用逐个合成 `ABORTED_BEFORE_DISPATCH` 配对（tool-calls.ts:250-260）——日志里每个 `tool/call` 必有结果，provider 才收得下历史 |
| （session-event-log）`TurnEndReason` 六值 | 补 `consumed-work.ts`：无步 turn 的"形状歧义"（reject 的 blocked 与被砍的 aborted 都无 step）靠 inbox 的 `removedCount/outcome:'canceled'` 记账区分，`foldConsumedWork` 单遍折叠出"最后一笔有账目的 turn + 是否有工作被丢" |
| （概览三篇未提） | 新机制两处：`runMaintenance`（维护相位对外仍是 idle，见下）；`agent-loop/config-start-failed` 瞬态事件——声明式 agent 启动失败要有名字地失败，不能让等它的消费方永远挂起 |

## 分组解剖与依赖方向

8 包按依赖自底向上：`scope`（零依赖，508 行）→ `session`（2,690）→
`agent`（接口+注册表+词汇，1,626）/ `system-prompt`（619）/ `tools`（5,294，含 PTC
与 schema 层）→ `agent-loop`（1,943 驱动器）；两个 60-96 行小件
`agent-default-model`（默认模型 settings 持有者）、`agent-tool-presentation`
（preset 携带的 `presentAs(mode)` 声明行）。合计 43 文件 / 12,842 行 [MEASURED]。
Provider 反转的物理证据：`agent` 包只定义 `AgentFactory` 接口与 `setFactory`
注册槽（index.ts:367-383，重复注册抛错），**不 import 任何循环实现**；
`ctx.agents.create/resume` 经 `getTraceable + Reflect.apply` 把调用方上下文重新
trace 给工厂——所有权跟随调用者 fiber，不叠 Cordis 代理层。

## 相位状态机：驱动器的私有世界

`ReactLoopAgent.phase` 三态（agent.ts:39-47）：`idle{lastTurn}`、
`running{abort,turn,step,wakeRequested}`、`maintenance{abort,lastTurn,wakeRequested}`。
对外只露两值（`idle|running`），maintenance 归 idle——"对外状态抹掉维护相"是
`status` getter 的一行映射。要点：

- **kick = `while (await turn()) {}`**；错误在驱动边界被静默收纳（已 emit
  `agent/error`），`finally` 里若仍是 running 相就落回 idle 并消费 wake 闩。
- **turn 间换 AbortController**（agent.ts:337-339）：续跑轮持新 controller，
  旧 controller 上设过的闩自动失效——"活驱动的闩是陈旧的"由对象身份保证。
- `whenIdle()` 用 do/while 对照 `activityDone` 接力替换后的驱动——对应防御性
  模式「异步状态不是同步状态」：一个 running 区间可能罩着多条排队消息，任何
  "等某条消息的结果"都是误用。
- `runMaintenance(job)` 只允许从真空闲进入；维护期间到达的 wake 留在 inbox，
  任务落定后才放闸（agent.ts:154-174）。压缩等"不算轮次的活"由此获得
  不与驱动并发的席位。
- 空批次也有轮界：step0 且消息为空 → `completed` 收轮但**不花模型调用**；
  pre-step reject → `blocked`。`turn/start` 先落盘再领取——被拒绝/被清空的轮
  在日志里留下完整形状（防御性模式「正交结果独立上报」）。
- `agent/turn-stopping` 只在 **turnEnds 已定 && nextStep 空** 时 serial 触发
  （goal driver 的续跑钩子）；listener steer 后机器重读 inbox——决定权在数据不在顺序。

## 步结算：attempt、interrupted 与粘性的 max-tokens

一次模型调用的持久化有四种结算（`AssistantStreamAttempt`，assistant-stream.ts）：

1. 正常成功 → `assistant/message{message,usage,stream}`（嵌入精确紧凑流）；
2. 失败/重试到顶（`finish.kind === 'error'|'aborted'`）→ `assistant/attempt{stream}`
   ——不伪造模型可见历史，但流的原始事实保住；`agent/request-error` waterfall
   返回 `{kind:'retry'}` 则**同一 step 内重开 while(true) 重建请求**（重读 surface 代际）；
3. 取消且有已交付内容 → `assistant/message{interrupted:true}`——`interruptedBlocks()`
   取安全前缀；
4. 取消且无内容 → `attempt`。

settle 的次序纪律：**先 durable append 成功、后 emit 终帧**
（`outcome.committed{seq}`）；append 被拒则 `abandon()`——实时的
`agent/assistant-stream` 帧是进程内瞬态，UI 不能把没落盘的说成落盘的。settle
失败与流失败合成 `AggregateError`（agent.ts:416-422）——两个独立事实各报各的。
max-tokens 粘性在 turn() 层实现：后来的 completed 不许覆盖 `max-tokens`
（agent.ts:302）。

## 请求纪元：header 折叠、剥离与不变量断言

`buildRequest`（agent.ts:488-588）每步执行同一程序：读 `session.requestHeader()`
（增量折叠缓存）→ 组 seed（首请求 = options 声明；后续 = `requestProposal(持久header)`
剥离 `adapterDefaults` 标记字段）→ `agent/request` waterfall → `prepareCall`
（middleware 可服务未注册路由，`NO_ADAPTER` 降级为直接配置）→ `canonicalHeader`
归一（空 system/tools 一律缺省字段）→ 与 baseline `headerEquals` → 按四 reason 追加
`request/header` → `request/context` 三元组 diff → `markAgentLoopRequest(deepFreeze(...))`
出闸。

出站前 `llm/stream` 上还压着**包属不变量**（agent-loop/invariant.ts，`prepend`
防被短路监听器静音）：被标记的 loop 请求必须 frozen、带活 session id、日志里
存在 `step/start` 与 `request/header`、`options.messages` 与 `session.deriveMessages()`
**逐字节 JSON 相等**、config 与折叠 header 字段一致——这些断言就是"模型可见即已
记录"的运行时守夜人，漂移即 fail loud。

## Session.append：先快照、再入带、后派发

热路径纪律（session/index.ts:699-749）：`snapshotJsonValue` **单遍迭代**读取-校验-
拷贝嵌套值（有状态 getter 不能给校验和存储各供一值）；非 lossless JSON 即拒，
**在 append 现场失败**而不是等后端 flush；`appending` 哨兵禁重入；回调集**先解析
后入带**（Cordis 派发校验的拒绝发生在日志改变之前），`log.push` 之后才带
per-listener 容错地 invoke——同步 `session/event` 观察者看到已提交状态，监听器
异常永不改变返回值。`deriveMessages/headerFold/contextFold` 三个增量缓存共享
同一水位思想：每个节点只投影一次，调用成本 O(new)。

创建/登记的通用编排（agents 与 sessions 两注册表同构）：`enter`（入带，不宣告）+
`announce`（`created` 同步抛=否决发布、异步拒=只报告）拆两步，让工厂能把会话与
agent 折进**同一个 generator effect** 按序拆迁（并发 sibling 效果会先摘发布钩子、
丢掉驱动最后几条事件）；`detachRequested` 延迟闩保证创建派发未展开时不拆自己
正在宣告的带。`AgentRegistry` 的 initiator 双槽 AsyncLocalStorage（值槽+链槽）配
`activeInitiatorRuns` 计数排水：fiber UNLOADING 时先 close（拒绝新边界）再 await
已返回 Promise 的链落定，然后才 disable——dispose 达到完全停稳，而不只是请求停止。

## 受保护流水线：调度器四段与单调 guard

`TOOL_RUNTIME_SCHEDULER`（内部 symbol，唯一消费方是 agent-loop 调度器）把一次
调用切成 `prepare → dispatch → finalize/finish` 四段，让"有序 pre-execute 可
await、只有工具体重叠"成为可能：

- `prepare`：造 token/callId/rootCallId、坍缩预检（collapse 直接 `final-result`，
  **不进策略管道**——只会被拒的调用不该惊动审批与 guard）、参数快照冻结、
  `tools/pre-execute` waterfall → `ask` 走 `ctx.get('approval')`（服务缺席/无
  agent/unavailable 三种都 deny 且**理由各异**，模型能分辨"人说了不"和"没通道"）
  → `guardReason`。`ToolGuard` 无 allow 返回值：**deny 单调只增**，监听器顺序
  无法翻案（index.ts:696-704）。
- `dispatch`：`tools/execute` around 包装（可换 signal 不可去 signal；
  `fuseToolSignals` 把调用方原始 signal 融回来，包装者劫持不了取消）。
- `finalize`（带 post）/`finish`（免 post）：`tools/post-execute` → `block` 把纠正
  反馈变 isError / `accept` 可换 content 或（仅成功时）换 value；最后过定义属主的
  `finalizeContent`（入参前快照捕获，防 arguments getter 换掉回调——
  index.ts:1389-1401 的注释是一场事故复盘）。
- 结果可信性用 **token 品牌 + WeakMap** 把关：`canonicalResults` 只认"本注册表本次
  dispatch 造出的结果"，around 包装者自造的对象一律重新归一化。

取消契约：body 已启动 → 排空到 quiescence 再改判 `ABORTED`；未启动 →
`ABORTED_BEFORE_DISPATCH`；两者都保留工具自产结构化错误（prior 链进
`cancellationResult`）。步骤内调度（`tool-calls.ts`）：每组开始前**重读**
`executionMode`（注册表热变化能制造新屏障），`commitReady` 只在 contiguous 槽
推进——派发可重叠、日志永远模型序。`deferContext` 与 `additionalContexts` 汇入
next-step inbox 走 FIFO；`concludeTurn()` 经 WeakSet 记入 `concludesTurn`，整批
提交完才结束轮。

## 作用域链与呈现坍缩（对概览的修正展开）

`dsh-scope` 三原语：`createScope`（mint 打标签的 Cordis fiber）、`scopeTarget`
（路由 carrier：保留 base 过滤器 + 沿 parent 链上溯匹配；**事件向上流、不向下流**）、
`bindScopeParent`（一次性绑链、防环；只有绑链者持 `rebind` 特权句柄）。
`tools.view(scope)` 的解析顺序是 per-agent 工具集的钥匙：**继承面**（global ⊕ 各祖
layer，近者胜）→ restriction 交集**只作用于继承面**（own layer 的注册不受过滤——
子代理的 structured-output 工具注册在自己 layer 才没被子过滤器误杀，注释明说这是
一次真实 bug 的修正，index.ts:1121-1142）→ 本层注册覆盖同名 → 预留 `run_code`
transport 最后插入（不进任何可过滤层）。`presentAs(mode)` 最近声明胜出沿链解析
（preset 的常驻声明覆盖其下全部 agent），`collapseSection` 与执行期 `collapses()`
用**同一谓词**——提示词里"只能直呼 run_code"的铁律与注册表拒绝行为不可能漂移。
scope 不变量（generated resolvers，28 个 scoped 事件 [MEASURED]）双向钉死：必须带
carrier 派发、carrier key 必须与 payload 里的 subject 同一对象。

## 类型模式：`…Map → derived-union` 与四种日志数字

`docs/subsystems/core.zh.md` 把它列为全仓公约：接口 `ThingMap`（判别标签→变体），
`type Thing = ThingMap[keyof ThingMap]`，插件**声明合并**加键即加变体——
`SessionEventMap`（append 词汇）与 `TurnEndReasonMap`（封闭联合）都在其列。
`SessionEvent<T>` 本体是 mapped-type 判别联合：`surfaceOp`/`sourceEventSeqs`
**只存在于三个 surface 类型分支**（`K extends SurfaceEventType ? {...} : object`），
给 `turn/start` 传 `surfaceOp` 是编译错误；运行期 `surfaceOpOf` 再双向钉（非
surface 类型带标记=抛，surface 类型缺标记=抛）。品牌化数字四种各司其职：
`SessionSeq`（存在的位）、`SessionLogOffset`（可为事件数的间隙/偏移）、
`SessionSeqCursor`（`SessionSeq | -1` 水位）、`OptionalSessionSeq`（`| null`）——
`seq = log.length` 连续性契约下它们可互换的场合恰是 bug 温床，品牌让编译器代劳。
`ignorable?: true` 的缺省方向值得记：**未标记者默认 required、读方拒收**——遗忘
标记的代价是过度拒绝（麻烦），而不是静默 resume 一个被掏空的会话。

## 真机旁证与数字速览

- 本篇作者即一次 subagent 投递：任务书中的 parent session id、共享工作区、seed
  平衡前缀继承，对应 `createAgent` 的 seed 通道与 `session/end-seed{inherited:true}`
  构造器独占写入；`fork` 拒绝开放轮内边界（`SessionForkError OPEN_TURN`）。
- 本会话运行时快照的固定前缀 "Current runtime context. This snapshot supersedes
  earlier runtime-context snapshots." 逐字出自 `joinContextSections`
  （system-prompt/index.ts:290）；`RuntimeContextProjection` 只在**内容与上次 retained
  不同**才造候选注入消息，且从后往前扫 surface 恢复水位——动态上下文"变化才可见"。
- 计数 [MEASURED]：`SessionEventMap` 核心 12 类；`KNOWN_SESSION_EVENT_TYPES` 51 型
  （与[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)同 build 一致）；
  `SESSION_FORMAT_VERSION = 2`（types.ts:86——迁移包 `session-format-v0-to-v1`/`v1-to-v2`
  已在仓，概览/平台篇成文时的 v0 口径属历史快照）；`turnBoundary` 投影
  `stateVersion: 2`；SECTION_ORDERS 30 位、CONTEXT_ORDERS 3 位；
  `DEFAULT_MAX_PARALLEL_TOOL_CALLS = 10`（settings `agent-loop.maxParallelToolCalls`
  热改：getter 每**组**读穿、在飞组不动——"配置改动不半途换轨"的样板）。

## 诚实边界

- 待核实：`tools/schema.ts`/`json-schema.ts`/`ptc.ts`/`py-types.ts` 未逐行——schema DSL
  与 PTC 桥的展开归 [PTC 与 code-runtime 内幕](./ptc-code-runtime.zh.md)，本篇只引用其
  在流水线上的挂点。
- 待核实：Cordis `serial/waterfall/emit` 的调度内幕在 `vendor/cordis`，归框架地基篇
  （[Cordis 内核](../cordis/cordis-kernel.zh.md)）与 [Cordis 热重启内幕](./cordis-hot-reload.zh.md)；
  本篇按公开语义使用。
- 待核实：`core/session/surface.ts` 的区间替换事务已走读校验/plan-apply 两层，但
  pruner 逐节点改写的调用侧全量语义以 [表层改写与压缩](./surface-compaction.zh.md) 为准。
- 待核实：真机旁证来自本会话自身 prompt 的观察，属间接证据；未在本机执行 core 组
  测试套件（`agent-loop-testkit` 在场未跑）。
- 待核实：`AgentLoop` 声明式 agents 的 launcher identity
  （`CONFIGURED_AGENT_IDENTITIES_KEY`）只读实现、未追 boot/cli 消费端。

## 相关

- 概览三篇（本篇是其收口）：[会话事件日志](../agent-runtime/session-event-log.zh.md) ·
  [turn/step 主循环](../agent-runtime/turn-step-loop.zh.md) ·
  [工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)
- 日志怎么落盘与物理修复：[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)
- 表层 replace 的全细节：[表层改写与压缩](./surface-compaction.zh.md)
- PTC 执行面坍缩：[PTC 与 code-runtime 内幕](./ptc-code-runtime.zh.md)
- 事件总线与 fiber 的框架语义：[Cordis 内核](../cordis/cordis-kernel.zh.md) ·
  [插件组装与启动](../cordis/plugin-composition.zh.md)
- 投递/取消/审批归因的下游消费：[委派与编排](../augmentation/subagent-orchestration.zh.md) ·
  [人机问答与反馈](../augmentation/questions-and-answers.zh.md)
