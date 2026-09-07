---
title: "交互、审批与权限档位：命令注册表与带审计的三问"
tags: [dsh, interaction, approval, commands, permission-presets, feedback]
status: active
license: CC-BY-SA-4.0
evidence: "packages/interaction/{commands,permission-presets,tool-ask-user,user-approval,
  user-questions}/src、packages/feedback/*/src、packages/identity/anonymous-user-id/src 全读
  + 真机实测（本汇编会话自身）；口径 docs/subsystems/{commands,user-questions,approval,
  permission-presets,feedback}.zh.md"
updated: 2026-09-05
---

# 交互、审批与权限档位：命令注册表与带审计的三问

[English](interaction-approval-feedback.md) | [中文](interaction-approval-feedback.zh.md)

> 概览篇 [人机问答与反馈](../augmentation/questions-and-answers.zh.md) 的深读承接：
> `packages/interaction/` 5 包 + `packages/feedback/` 2 包 + `packages/identity/` 1 包
> （合计 2711 行 [MEASURED]）。三个 seam（提问/审批/预设）共享同一套安全语法：
> **封闭词汇 + fail-closed + 落日志审计**；本文按"审计与 turn 的关系"这条暗线组织。

## 概览篇说法 → 深读结论（先纠偏）

| 概览篇说法 | 深读结论 |
|---|---|
| "多应答者取先完成者" | **不准确**：`user-questions/request` 与 `approval/request` 都是
  waterfall——**顺序认领**（返回即认领、`next()` 下传），不存在竞速语义。
| "数据结构带 Intent（确认/选择/补充信息）" | 现行 `AskUserQuestionIntent` **只有**
  `{ kind: 'plan-review', approve: string }` 一个变体（types.ts），且 `approve` 必须点名
  本题自有选项、必须带 `detail`，否则 `ask()` 直接 `BAD_INTENT`——在提问者处拦截，
  而不是每个 UI 各自防御。生产者见 `plan/plan-mode`（exit_plan_mode 审阅）。
| "read-only/workspace-write/danger-full-access 档位词汇**在此**定义" | 模式词汇在
  `sandbox/sandbox-policy`（`SANDBOX_MODES`/`setSandboxMode`）；permission-presets
  定义的是**模式×政策的捆绑表**与派生读出。
| approval "服务缺席=deny" | 精确化：政策先行、无应答者落 `'unavailable'`、
  失控返回值归一 `'unavailable'`——**归一发生在 seam 内部**，调用方只需处理封闭枚举。
| messageFeedback "两域之一" | 与持久化篇统一口径：全库 storage-domain **3 域**
  （workspace、message_feedback、session_projcache），见
  [持久化深读](./persistence-crash-recovery.zh.md)。

## approval：封闭四态与两级政策

`ctx.approval`（`user-approval`，278+101+84 行）只管一件事：**只读同进程的许可问题**。

- 结果词汇是闭集：`'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'`——
  唯一授权形态是**一次性**（grants apply only to the requested action）。
- 政策两级：`ApprovalPolicy = 'ask' | 'never'`；`effectivePolicy` = 会话日志里
  **最后一条** `approval/policy` 事件 ?? 部署配置默认（schema 已默认 `'ask'`）。
  `'never'` 是"不问而确定拒绝"的无人值守姿态（CI/headless）。
- **`'never'` 在服务自身 `decide()` 路径判、先于任何派发**（index.ts 注释原文）：
  若做成 waterfall 前置监听器，任何 `prepend: true` 注册的应答者都能插到它前面，
  "无论注册顺序 deterministically 拒绝"的承诺就破了——**只有服务自己的请求路径守得住**。
  这是"门禁不该长成监听器形状"的样板论证。
- 三道路由兜底进 promise 链（`Promise.resolve().then(...)`）：同步 throw 的监听者
  与异步 throw 走同一条 containment 路径；answerer 抛错 → 问题 fail closed
  （`'unavailable'`），**而不是让调用方的工具调用漏网放行**。
- abort 竞速：信号先赢则 `'cancelled'`，迟到答案被 settled-promise 结构性丢弃。

### 审计对必须 turn 封闭（本篇暗线）

每次 `request()` 落一对事件：`approval/asked`（id+toolName+?callId+?reason）→
`approval/decided`（同 id+outcome），且 **`hasOpenTurn` 前置检查**——无开放 turn 直接
throw。理由写在源码注释里：turn 是持久日志的 commit/replay 边界，**夹在两个 turn
之间的裸事件重放时与崩溃尾巴不可区分，会被静默丢弃**。配套 invariant 伴生插件
（`invariant.ts`）经 `internal/dispatch` 预提交暂存（staged WeakMap）校验配对、
重复 open id、未知 outcome——提交前拦，不是提交后报警。

政策状态对模型的传达走 **systemPrompt context 注入**（`approval:policy` 段，
order=`APPROVAL_POLICY`）：`'never'` → "Approval prompts are disabled in this session…"、
`'ask'` → "Approval policy: ask…fails closed."；完整现值随**保留历史之后**注入，
政策切换不重写稳定 system-prompt 缓存前缀。运行中切换另走 `setPolicy`：落事件 +
以 `source: { kind: 'plugin' }` 注入 user message 通知模型（初始化期则直接
`setApprovalPolicy`，因为没有"先前可见值"可通知）。

**委派种子**：`subagent/subagent/src/child-agent.ts:263-266` 在"未发布创建窗口"内给
子会话追加 `sandbox/mode` 与 `approval/policy`，`source: 'delegation'` 标记来源；
`session-format-v0-to-v1` 的 payload-validation 把 `source` 白名单钉死为
`'delegation'`。

### 真机实测（本汇编运行环境即样本）[MEASURED]

- 用户主会话的运行时上下文逐字命中 ASK_SENTENCE；本**子代理会话**逐字命中
  NEVER_SENTENCE——与"子代理无人间应答者、政策随委派播种"的机制陈述互为印证。
- 在 `never` 子会话内发起一次 `sandbox_permissions: danger-full-access` 提权请求：
  **不弹任何卡**、即时得到拒绝（"the user rejected escalating this command…"），
  即 `'never'` 短路 `'rejected'` 的确定性路径。

## user-questions：模型侧提问工具与归属校验

- 模型工具 `ask_user_question`（`tool-ask-user`，97 行）是 `ctx.userQuestions` 的薄消费者：
  入参 schema 宽容（`additionalProperties: true`）、出参 schema 严格、渲染即
  `JSON.stringify`——答案作为**普通工具结果**回流 agent loop。
- `ask()` 的三重归属校验（错误词汇 `EMPTY_QUESTIONS / CALLER_NOT_LIVE /
  DELEGATED_CALLER`）：supplied agent 必须是注册表**精确活实例**（防伪造/陈旧句柄），
  且必须是 `agents.roots()` 成员——**被拥有的子代理没有人类应答者，问了就是永远挂起**。
  边界由**运行时所有权**而非持久谱系决定：谱系再深，resume 成新运行时根就能正常问。
  错误消息直接把出路写进正文（"include the unresolved question … in the child's
  final result"）——fail-closed 还要 teach-the-caller。
- 派发经 `scopeTarget(agent, agent)` 过滤：agent-scoped 监听者只收到自己 agent 的请求；
  无 agent 时走非 scope waterfall。错误跨线回来可能只剩 plain object，
  `restoreUserQuestionError` 按 name/message/code 三字段**重建类实例**——闭集词汇的
  wire 往返韧性样板。

## commands：人类命令注册表与"无 turn 生命周期"

`interaction/commands`（491 行，全组最大包）是**插件自持命令**的宿主侧真源：

- 语法：`parseCommand` 精确正则（小写名 + 分隔符先行断言），
  `rawInput` 逐字保留（含分隔空白），注册名同规则；描述非空、handler 必须是函数，
  全部在注册边界 `normalizeDefinition` 拦下。
- 作用域：**ScopedLayers 遮蔽**——普通 ctx 注册=全局；command-injected 插件挂在
  `agent.ctx` 下=该 agent 专属影子。重名错误消息直接给出修法提示。
- 生命周期：`command/run` → handler → `command/done`（`commandId` 配对，仿
  `tool/call↔tool/result`），但两者是**直挂 log-only、无 turn 包裹**——与 approval
  的 turn 封闭恰好相反：命令不在任何模型的 turn 里发生，持久化走普通 checkpoint
  排空。同一仓库里"审计对必须 turn 封闭"与"独立插件事件 eager 不 flush"两种语义并存，
  分界就是**是否属于某个 turn 的事实**。
- `commandId = cmd-<instanceToken-8位><monotonic>`：实例令牌保证同一份 resume
  日志上永不重复（invariant 伴生同时查重 id、孤儿 done、`sourceEventSeq` 越界/指向
  另一对生命周期事件）。
- `recordInput: false`：输入载荷由**领域事件自持**的命令（如 `/feedback`）不再把
  rawInput 重复进日志——一处事实一个真源。
- 附件在注册表边界素材化：未声明 `input.attachments` 即拒；文件走**唯一**的
  staged-receipt resolver（重复注册 throw）；素材化 await 存储后、进 handler 前
  **再查一次取消**（重试会复写状态）。
- `commands/change` 是 emit 通知：每个监听者独立 contain（同步 throw 与 rejection 都
  只 log.warn）——注释点名 Cordis emit 用 `Array.map`、一个同步 throw 饿死后继，
  非否决通知必须逐个包隔离。
- 远程面：服务继承 `TypertRemoteService`，`list`/`execute` 标 `@Remote`——浏览器端
  命令面板经 [typert 协议](./typert-remote-protocol.zh.md) 直调。

## permission-presets：捆绑写路径、投影读路径、派生 custom

`ctx.permissionPresets`（398 行）把两个独立旋钮（沙箱模式×审批政策）包成用户可选项：

- 默认表 [MEASURED]：`workspace-write`（workspace-write+ask）、`danger-full-access`
  （danger-full-access+never）；`custom` 是**保留字**——派生的"不匹配任何档"状态，
  不是表项、不是切换目标、不进事件载荷。
- 写路径三步曲（`apply()`）：先记 `permission/preset`（用户**意图**，log-only、
  不进模型转录），再逐个"变化了的旋钮"走各自的 canonical setter（`setSandboxMode`、
  `setApprovalPolicy`/`setPolicy`）。选当前档=完全 no-op。执行与重放**永远读旋钮折叠**，
  预设事件只在"两档捆绑相同"时保住用户选过哪档的意图。
- 读路径 = `permissions` 会话投影（stateVersion 2）：KnobState 三字段 null=未压过
  （组合默认值在 view 时补），`session/end-seed` 置 `seeded`。**平局数学**共享一份
  `derive()`：上次选择仍匹配则它赢，否则表序首个匹配，都不中=custom。
- 初始化钉选（`pinInitialPermission`）：全新会话钉默认三事件；seeded/半初始化只补缺失
  事实、不覆盖已有值。构造期守卫两条：shell 执行方必须 confine（否则预设无意义）、
  默认档必须可解析（否则要求显式 `defaultPreset`）。
- 写入口只有一条：**`/permission` 命令**（web 弹层把选中值提交成这一行）；
  `settings` 命名空间 `permission.defaultPreset` 管未来会话。
- 消费方实测分布：`sandbox/mode` 字面量散布 12 个包、49 处引用（fs/shell/terminal/
  subagent/core.session/client/api，v0→v1 迁移的 payload-validation 也在守它）——
  旋钮事件已是持久协议公民 [MEASURED]。

## feedback 与 identity：日志之外的旁挂

**message-feedback**（365+127+80 行）是 storage-domain 第 3 域的旁挂服务，
"永不创建/恢复 Agent 或 Session"（自述 sidecar）：

- **为什么不进会话日志**：反馈是人类对消息的评价，属模型世界之外——写进唯一真源
  等于注入非事实。方案=每会话一行 `row{ session: 生命周期身份, items[] }`；
  身份=createdAt+cwd，**给 SessionId 复用场景上栅栏**（id 复用时的陈旧行整行不可见）。
- 乐观并发：item 带 opaque `version`（每实质变更换新 UUID）；`ifVersion` 不中→
  `version-conflict` 并回带权威现值供 reconcile；等值 put 命中**原样返回不 bump**；
  delete 幂等（absent 也是成功）。每会话一条 mutation 尾巴队列串行化读-比-写。
- 目标必须命中 **derived append-origin** 的 `assistant/message`
  （`isAppendSurfaceEvent` + `deriveEventMessage` 复核 messageId）——表层改写出来的
  非 append 源消息不收反馈。
- 旁挂写入前先过**主日志持久化屏障**（`ensureTargetDurable`：live owner 走
  `sessions.flush`，再按 freshness 保证重读 durable 前缀复验目标）——反馈永远骑在
  一段已落盘的对话事实上。
- 全 Remote：list/put/delete 标 `@Remote('list'|'put'|'delete')`，浏览器直调
  （[host/client 边界](../platform/host-client-boundary.zh.md)）。

**command-feedback**（98 行）：`/feedback <text>` → `recordFeedback()` 落
`feedback/record`（log-only、与触发无关、永不进模型上下文；`recordInput: false`
同型）→ 回执拼匿名 id 与**共享披露语句**（telemetry 状态 closed-union ×
assertNever）。"eager 但 unflushed"：回执说的是"已记日志"，不是"已落盘"。

**anonymous-user-id**（91 行）：harness home 作用域（`$DSH_HOME` > `/.dsh`）的
`.anonymous-user-id` 裸 UUID 行。设计声明：绝不从主机名/网络/git remote 等**可识别源**
派生；`wx` 独占创建裁决并发首启、读回败者采纳胜者；写失败（只读 home）**尽力而为**
——本进程仍持一致 id，遥测与反馈永不被持久化失败阻断。

## 诚实边界

- UI 应答者实现（`client/ui-user-questions`、Remote 线到浏览器的投递细节）未走读，
  仅按 seam 词汇+消费方清单陈述。
- `DELEGATED_CALLER` 分支未真机触发（避免为用户会话弹卡），依据是校验代码与
  本会话 NEVER_SENTENCE 的旁证。
- ACP 侧 approval 消费（`packages/acp` 出现在消费方 grep）具体接线未走读。
- 官方 subsystem 五页未逐字比对（其为 JSDoc 生成面，本文以源码为准）。

## 相关

- 概览篇：[人机问答与反馈](../augmentation/questions-and-answers.zh.md)
- 审批的调用方：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)
- 旋钮执行侧：[沙箱执行内幕](./sandbox-execution.zh.md) ·
  [文件系统与沙箱](../execution/filesystem-and-sandbox.zh.md)
- 日志语义暗线的另一端：[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)、
  投影机制 [会话投影与遥测](./session-projection-telemetry.zh.md)
- 跨进程传输：[typert 远程协议](./typert-remote-protocol.zh.md)
