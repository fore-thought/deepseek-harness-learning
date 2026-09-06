---
title: "委派内幕：命名注册表、Activation 驻留与四路远程传输"
tags: [dsh, subagent, delegation, continuable]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{subagent/*,acp/acp} 源码直读+真机见证；docs/subsystems/subagent.zh.md"
updated: 2026-09-05
---

# 委派内幕：命名注册表、Activation 驻留与四路远程传输

> 概览篇[委派与编排](../augmentation/subagent-orchestration.md)停在接口词汇层；本篇下潜到
> `packages/subagent/` 全组 + `packages/acp/` 的实现层：能力闸如何把关、续执行管理器靠什么
> 推导状态、描述符怎么当"可恢复身份的唯一持久化"、四路进程外传输各交付什么协议边界。
> 证据 = 源码直读（行号当前 checkout）+ 一条特殊来源：本篇汇编者本身是一个 continuable
> fork 子会话，多处机制属**一手见证**（见「真机见证」节）。

## 包拓扑（实测行数）

| 包 | 行数 | 角色 |
|---|---|---|
| subagent/subagent | 19 文件 / 4959 行 [MEASURED] | Service Definition（`ctx.subagents`） |
| subagent-in-process-driver | 352 | 一次性共享驱动（spawn/fork 复用） |
| subagent-{spawn,fork}-in-process | 61 / 85 | 两个进程内 provider（全能力旗标） |
| subagent-{acp,codex,claude-code,dsh-sdk} | 762/1206/857/523 | 四路进程外传输 |
| tool-subagent | 1168 | 面向模型的委派工具（绑定单提供方） |
| tool-subagent-control | 295 | 全局 send_message / interrupt_agent / 可分立 list_agents |
| acp/acp | 1705 | **反向面孔**：automation-only ACP server（harness 被委派为 agent） |

## 能力双闸门：五面旗标 vs 方法存在性

`SubagentCapabilities` 五面启动时旗标（agentOptions/outputSchema/depthLimit/toolFilter/
persona）由**服务**在 `start()` 前核对——缺能力即 `UNSUPPORTED_CAPABILITY` 拒，绝无
"接受了再静默降级"。可继续路径不走旗标：**`prepareContinuable` 方法的存在本身就是能力**，
TypeScript 类型收窄即发现机制。六提供方对照：

| 提供方 | 五面启动旗标 | prepareContinuable | inheritsParentContext |
|---|---|---|---|
| spawn | 全 ✅ | ✅ | false |
| fork | 全 ✅ | ✅ | **true** |
| acp / codex / claude-code | 全 ❌（`NO_START_CAPABILITIES`） | ❌ | false |
| dsh-sdk | 仅 agentOptions（NO+路由例外） | ❌ | false |

dsh-sdk 是唯一的例外档：`SDK_START_CAPABILITIES` = NO + `agentOptions:true`，并公布
`agentRouteDefaults`（provider/model 静态路由，供消费方预检前合并）——因为
outputSchema/深度/过滤/人格的所有权"不过进程边界"（包注释原话）。

## 注册表与生命周期事件（index.ts:190/509）

`SubagentRuntime extends TypertRemoteService`（'subagents'）：`registerProvider` 是
effect-scoped 生成器——重名 `DUPLICATE_PROVIDER` 抛、卸载发 `subagent/provider-removed`；
**移除提供方只挡新 start，不撤销已交付的 run**。生命周期对 `subagent/start|end` 按
"委派方 parent 作 scope carrier"过滤派发，每个 listener 独立异常收容（抛/reject 只
warn，不饿死同伴）。一次性 run 与每个 Activation 驻留纪元发**同一词汇对**：冷恢复 = 一个
新 `runId` 的纪元。观察器先挂 promise 再同步 emit start，保证 start→end 定序
（lifecycle.ts `observeRun`）。

## 续执行管理器：Activation 是驻留单位，不是任务

`continuation.ts` 1649 行 [MEASURED]，核心是 `SubagentContinuationManager`：

- **Activation**（L187）= 一个驻留纪元的记录：`handle`（持有的 AgentHandle）、
  `ancestry: WeakSet<Agent>`（物化时刻的精确在线祖先链——Weak 成员在中间祖先离册后仍保身份）、
  `ownedChildren: Set<SessionId>`（非空即挡住本级 settle）、`disposal`（memoized 拆卸
  事务，**存在即准入截止**）、`accepted: Set<MessageId>`、`announced`、`poke`。
- **状态是推导的不是登记的**（`stateOf` L1054）：running = Agent.status running **或**
  accepted 非空；waiting = 已静默但 ownedChildren 未清空；settled = 双空。accepted 集合
  的存在理由很具体：`Agent.status` 在"waking send 已受理"与"微任务准入"之间仍是 idle，
  纯 status 观察会把排队中的纪元误判为 settled。
- 每子一把 `ChildLock`（L356）线性化 交付/释放/拆卸；`watchSettlement`（L1464）把
  "settled 判定 + 开启拆卸"放**同一个临界区**——区外判定会撕裂：投递可能观察到一个
  watcher 正要拆掉的句柄。
- 拆卸三形态：全局 `drain`、按精确 host 拥有的 `drainDescendants`（`closingScopes`
  表记录 root→成员，作用域关闭持续到该 root 离册）、按选择子的 `drainChildren`。
  **取消自顶向下、释放心 child-first**；分支失败逐个记录、全数尝试后才聚合成
  `ACTIVATION_TEARDOWN_FAILED`。最后 flush 是尽力而为：rejection 只 warn——
  钉住子会话会把整条祖先永久 pin 在 waiting。
- `holdOwnership`：空闲父级在"子正在建立/恢复"期间不得 settle；调用方 signal 只管到
  inbox 受理，此后管理器独立持有纪元。

## 描述符：可恢复身份的持久化（descriptor.ts）

`SUBAGENT_DESCRIPTOR_VERSION = 3`（L48）。要点是实现层比文档层更狠：

- **显式字段白名单**（`assertKnownKeys`）：one-shot 只许 version/mode/provider/label；
  continuable 另加 agentProvider/agentModel/reasoningEffort/persona/toolFilter。
  永不快照可合并扩展的 `AgentOptions` 整对象——无关扩展值不该因"不是 JSON"而毁掉继续执行。
- **快照在任何 await 之前完成**（startContinuable 内），非无损 JSON 的载荷在子存在前就
  reject 整次调用；`maxTokens`/`outputSchema` 刻意出局（每次激活的预算，非持久身份）——
  代价是**冷恢复不恢复预算**，重放路由的默认值接管。
- "权威"有两种且互洽：`foldSubagentDescriptor`（L317）**取第一条** descriptor 即终局
  （建立方恰好只追加一条）；列表身份投影则是"见 descriptor 即整体重置"的 **last-wins**
  （projection.ts:169）——因为 fork 种子会重放**祖先**的 descriptor，`inheritedEventCount`
  之前的都不算数。畸形当前版本 → 投影折叠成 `null` 哨兵（不抛），恢复路径判损坏。
- 事件仅日志可见：无 `surfaceOp`、永不进模型历史、跨压缩存活。

## 冷恢复：提供方缺席的重建（L1067）

`send_message` 打到无 Activation 的直接子 → `coldResume`：`sessionQuery.observeSession`
→ **先按持久 header 授权**（精确在线父级）→ 只折叠 `inheritedEventCount` 之后的自身后缀
→ descriptor 提供 provider/model/effort/persona/toolFilter → `ctx.agents.resume()`
经私有 activation-owner 作用域 → 提交等待轮次。**冷恢复不经任何子提供方分发**——持久
Session 已含种子前缀，descriptor 就是全部重建输入。强依赖两个服务：缺
`sessionPersistence` → `PERSISTENCE_UNAVAILABLE`；缺 `sessionQuery` →
`CONTINUATION_UNAVAILABLE`（fail-closed，可继续是"持久会话"的语法学）。

## 结算通知：为什么必须由管理器来发（L1631）

模块注释给出动机：外部 `subagent/end` listener 做不这件事——payload 不指认 parent、
子句柄彼时已 dispose、唤醒父级 settle 观察的 release 已经跑过。实现细节三连：

1. **三态投递**：父级所在树已在拆卸 → `inject`（不唤醒：waking 一个静息 Agent 是烧一次
   模型请求）；父级 idle → `followup`（一个普通轮次）；父级忙 → `steer`——
   `Inbox.claim()` 一次拿走整个 next-step 批次，**N 个孩子同时结算只花一个 step 而非 N 个轮次**。
2. **无条件投递**：每个拿到过 id 的调用方必得通知——token 上限、模型故障、取消、拆卸
   这些最需要通知的场景，恰是孩子没机会自己开口的场景；`announced=false` 的
   （受理前回滚）保持沉默，"调用方被告知不存在的孩子不欠账"。
3. **先送达再放权**：`notifySettlement` 排在 `releaseOwnership` 之前——放权后父级
   watcher 晚一个微任务就发现自己"无子且静"而开拆，`keepInbox:false` 的 cancel 会清掉
   还没领的通知书。来源 kind 是 `subagent-settled`（form=notice）而非 `agent-message`：
   这是运行时在陈情，不是孩子自己在说话，混进 transcript 会把没写过的话记在孩子头上。

## 策略继承：委派边界上定格的东西（child-agent.ts）

- `captureDelegatedPolicyOverrides`（L242）：只取**父会话显式** sandbox override（从不
  取部署默认值或一次性 grant）+ approval 针为 `'never'`（approval 已 compose 时）。
  必须在首个 await **之前**同步捕获——"父级之后的切换属于父级的未来，不属于这个孩子"。
- 落盘为 `source:'delegation'` 的 `sandbox/mode`、`approval/policy` 事件，追加在 fork
  种子之后（fresh 压过种子里的 stale），子自己后续切换再压过它们。
- `SUBAGENT_DELEGATION_CONTEXT`（"你的权限范围在启动时固定、不可从内部放宽、被拒不要
  换路重试"）以 **runtime-context 贡献**注入而非 system-prompt section——部署提示词
  父子保持均质。
- `applyChildComposition`（L199）：先 `composeFrom(parent)` 加入父 preset、后注册子级
  persona shadow 与 `tools.restrict()`——"不 join 的孩子看到空注册表"正是此函数要防的缺陷。
- 子选项继承（`resolveChildAgentOptions`）：provider/model/effort 取**最新请求头**
  （`requestHeader()?.config`，创建期选项仅作首轮前兜底、保留 maxTokens）；换路由而
  未点名 effort → **删掉**父级 effort 让新模型解析自己的默认（这条与工具文档措辞同源）。
- 深度：`delegationDepthOf = max(header, runtime)` 单调下界，冷恢复洗不浅；
  `SubagentDepthError` 拒超 `maxDepth`（工具层默认 3，tool-subagent index.ts:128）。

## one-shot 驱动与两种 stop 语义

驱动（in-process-driver）单轮单果：descriptor 经 `agent/pre-step` 在**首次 enter 之后**
追加（"初始轮次内、首次请求前"，L81）；cancel 走 flags.cancelled + `child.cancel
({kind:'parent'})`。结果映射 `toStopReason`（L50）：blocked→refusal、**interrupted→error**。
而 Activation 纪元的 `epochStopReason`（lifecycle.ts）：interrupted→**aborted**、
blocked→refusal，且 `foldConsumedWork` 的 `droppedUnrun`（有已受理却被丢弃的工作）把
"completed" 改判 aborted——**清理成功不等于工作完成**，终因以子日志为准。两处对
interrupted 的不对称映射属实读；语义差异未逐案实测（见诚实边界）。结构化输出 = 强制
capture 工具（structured.ts），spawn/fork 两包都刻意**不注入** `tools` 以免改变自身
apply 时机。fork 种子 = `completedTurnPrefix`（fork index.ts:48）：`findLast(turn/end)`
之后全截，进行中轮永不入种；空种子**不落** seed 字段（会话保持 unseeded 形）。

## 四路进程外传输（协议边界速览）

- **subagent-acp**：spawn 任意 ACP agent，ndjson over stdio，client 侧驱动。子进程的
  `session/request_permission` **无人类中转**：reject（默认）或取首个
  `allow_once/allow_always` 选项。失败诊断只由**封闭词汇**拼装（stage∈{initialize,
  new-session,prompt,process,teardown} × category∈{protocol,configuration,transport,
  process-start,process-exit,remote-limit,unknown} + 退出码），子级的工具标题与选项文字
  **永不进入** diagnostic。dispose 阶梯：EOF 静默窗 6000ms → 信号升级，POSIX
  SIGTERM→SIGKILL 3000ms；Windows 直接强杀 [MEASURED]（run.ts 常量 L83/L86）。
- **subagent-claude-code**：官方 Agent SDK 拉起真实 CLI，SDK 子进程挂进共享 subprocess
  owner；permissionMode 默认 `dontAsk`，可枚举 acceptEdits/auto/plan/bypassPermissions。
- **subagent-codex**：包内 `app-server --stdio`，**线程存在后才发布** run。
- **subagent-dsh-sdk**：整副 DSH 子运行时（profile `sdk` + patch 列表 + **独立 dshHome**
  隔离嵌套会话），SDK client stdio JSON-RPC，dispose 走 `shutdown` 协议握手限时。
- 公共骨架在 out-of-process.ts：cwd 解析（config 覆盖 else 父会话 workspace，**绝不回落
  进程 cwd**——一个 server 进程服务多会话）；`X_OK` 可入性探测（mode-600 目录过得了
  `statSync` 过不了 spawn）；`settleRunResult` 发布后永不 reject；`diagnostic` 4096
  UTF-8 字节截断且不劈字符。三产品包 inject `subprocess`（复用脱敏 env 与树级拆卸），
  dsh-sdk 只注入 `subagents`（进程由 SDK 自拉）。env 策略：父环境**凭据脱敏**后叠加
  显式 config env——显式键可穿透 scrub（子是自带 key 的另一副 harness）。
  命名导出纪律（禁 default export）直引 postmortem 0001。

## ACP 的另一副面孔（packages/acp）

`packages/acp` 是**被委派方向**：automation-only 的 ACP server 桥，把 harness 的持久会话
暴露给受信程序化客户端（initialize/new/resume/close/list(分页 100)/prompt/cancel/
一次性 permission 裁决/config option），展示与人类交互特性刻意留在 UI 模块。两个包各引
`@agentclientprotocol/sdk` 的 client/server 半区——于是 **DSH 既能委派任意 ACP agent，
也能被任意 ACP client 委派**，配上 dsh-sdk 即"harness 套 harness"两条正交通道。

## 模型工具面

`tool-subagent` 按配置绑定**单一提供方**（多实例须各起 `toolName`）；
`backgroundMode` 决定缺省方向：one-shot（默认，前台 await 全果）| continuable（后台缺省，
回 `{subagentId}`）。模型路由可选开`subagentModelSelection`（用户设置采样 + 会话投影
记录每子所选）。控制面三工具全局唯一：`send_message`/`interrupt_agent`（根插件）与
`list_agents`（**可分立加载**——部署可以只给交付不给发现）。工具层零路由逻辑，鉴权全在
服务侧（`exec.agent` 必须恰为在线 sender）；`list_agents` 用注册表把服务态细化为
`running|idle|ready`（ready=仅存于盘上、可恢复、非终态），one-shot 子**从模型列表省略**
（send_message 够不到它们，但 descendants 遍历仍路过）。

## 真机见证 [MEASURED]

本篇汇编会话就是一个 continuable **fork** 子会话（父级 id 逐字可见），一手核对：
任务尾部确实被 `continuableInitialPrompt`（L305）追加了含 `agent_id` 的返回指引；
`SUBAGENT_DELEGATION_CONTEXT` 全文逐字节一致（child-agent.ts）；fork 种子的父级前缀
就在自身历史里；"被拒操作勿换路重试、写进回复交还委派方"与策略继承一节的 `never`
针一致。settle 通知机制正在为父级服务——它收到的是 kind=notice 的陈情，不是本文。

## 诚实边界

- 待核实：structured.ts（capture 工具全机制）、control.ts / list-children.ts 的三级供值
  阶梯实现细节——本篇仅到文档口径 + 调用点。
- 待核实：codex wire.ts（660 行）与 claude-code run.ts 的协议帧序未逐行读；dsh-sdk
  run.ts 握手消息序读了前半。
- 待核实：acp/acp/session.ts（496 行）resume/config-option 内部逐段未读。
- 待核实：one-shot `interrupted→error` 与纪元 `interrupted→aborted` 的不对称，代码
  属实、生产语义未实测。
- 行号 = 当前 checkout；上游快速迭代会漂。

## 相关

- 概览篇（本篇是其深读展开）：[委派与编排](../augmentation/subagent-orchestration.md)
- AgentHandle 所有权与投递/取消：[Agent 主干深读](./agent-loop-internals.md)
- fork 种子的日志契约：[会话事件日志](../agent-runtime/session-event-log.md)、
  [持久化格式与崩溃恢复](./persistence-crash-recovery.md)
- 身份/计时投影与缓存阶梯：[投影、遥测与格式迁移](./session-projection-telemetry.md)
- 工作流如何复用本 seam 起子：[工作流与 agent team](./workflow-agent-team.md)
- ACP 网关面（host↔client）：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
