---
title: "作业注册表与 Webhook 运行时：first-wins 结算与 fire-and-forget 的完整边界"
tags: [dsh, jobs, webhook, background, registry]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{jobs,jobs-local,tool-jobs,webhook,webhook-github} src 全读 + docs/subsystems/{jobs,webhook}.zh.md"
updated: 2026-09-05
---

# 作业注册表与 Webhook 运行时：first-wins 结算与 fire-and-forget 的完整边界

> 两组 seam 一篇收口：`ctx.jobs`（一切"跑了不止一拍"的东西的注册表）与
> `ctx.webhookRuntime`（外部世界进 inbox 的免提门）。它们是
> [后台任务与外部触发](../augmentation/background-and-triggers.zh.md) 两节的源码级展开。
> 共同气质：状态机小、顺序纪律重——每处"谁先谁后"都有可指认的理由。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| 生产方"bash 后台、subagent 后台、workflow……" | **workflow 不走 jobs**：全库 `jobs.start(` 实测仅 4 处调用点——tool-bash / tool-pwsh / tool-subagent / tool-terminal（`pty-send`）[MEASURED]；workflow 的可观察性是 `workflow/*` 事件对（见 [工作流与 Agent Teams](./workflow-agent-team.zh.md)），不进作业注册表 |
| kind 词汇"bash/subagent/…" | 声明合并实际在盘词汇 = `bash · pwsh · subagent · pty-send` 四家 [MEASURED]；核心 `JobKindMap` 只预置两条语义位，其余由各包自行合并进表 |
| "完成通知最后发" | 全序可指认：`settle()` 先记终态 → 释放全部 waiter → `markSettled` → `notifyChanged` → 才轮到完成监听者（见「结算」节）；reporter 同步开轮次也不抢跑 |
| webhook "验未改动 raw body 签名" | 顺序钉死：**verify 在 JSON.parse 之前**（octokit 对字节算 HMAC，解析后重序列化即失真）；验不过连 payload 形状都不回显 |
| "无队列/无重试/无去重/无崩溃重放" | 属实且更进一步：**无执行状态表**——活动调用集合是私有的、随进程消失；规则 disposer 的"先隐身再 abort 后排空"是唯一的生命周期账本（见「卸载」节） |

## JobRegistry seam：抽象类自己站岗

- `packages/jobs/jobs` 只有契约没有实现：`JobRegistry extends Service` 的构造器
  `if (new.target === JobRegistry) throw`——抽象在运行时是 erased 的，误把本包写进
  组合行会装出一个"无方法的 ctx.jobs"、远在事故点才爆雷，所以**装载当场 fail-loud**
  （index.ts:74-77）。
- `JobId` 单独住 `brand.ts` 叶子模块：包根与 types.ts 都经 owner/listener 签名够到
  `dsh-agent`，浏览器端 Client 程序连类型都解不开；品牌 leaf 让 wire 消费方零依赖
  （brand.ts 头注）。品牌化 id 生成式 `<kind>-N`：**可预测是设计**——
  "访问控制依赖拥有者授权，而非 id 的保密性"（types.ts JSDoc）。
- 快照跨字段不变量不靠自觉：`jobs-invariant` 伴生插件（invariant.ts 49 行）对每条
  终态快照断言 id 前缀=kind+正整数、label 非空、`finishedAt 恰在终态存在`、
  `ownerSession` 与回调 owner 一致——注册表 bug 会被不变量当场点名。
  `webhook-invariant` 同款（见会话创建节末）。

## 准入：闸门在 run() 之前

`LocalJobRegistry.start`（jobs-local/index.ts:131-190）在 `run()` 之前依次过闸，
任一拒绝**不留 id 不留资源**：

1. **controller 服务闸**：`servesOwner` 问的不是"进程里有没有 controller"，而是
   "该 owner 的 scope 链上有没有"。注册表全进程一个实例，controller/listener/changed
   三张表都按注册 ctx 落进 `ScopedLayers`：全局层（宿主组合自带）服务一切 owner；
   agent 组合 scope 下挂的只服务其下合成的 agent。摊平表会出两种事故：一个 preset 的
   controller 替另一个 agent 的 `start()` 开绿灯；或一次结算惊动所有 preset 的
   通知器（index.ts:104-116 注释原话）。
2. **配额闸**：`maxConcurrentJobsPerOwner` 默认 10 [MEASURED]，按**精确 owner**
   分桶（无 owner 共享一桶），计数含 `running`+`stopping`；报错文案自带
   补救路径（kill 或等待后重试）。
3. `owner` 必须是**当前注册的活实例**（`agents.get(ownerId) !== owner` 即抛，
   防"同 id 换人"后复用旧句柄）；owner cleanup effect 挂进 owner 自己的 scope——
   agent 拆卸时取消其作业并 await 停稳，跨越 producer 热重载存活。

`run()` 同步返回 hooks 后注册即原子（counters 铸 id → store.set → 挂 done 监听），
"registration cannot fail from here"。producer 的 `done` 被约定**不许 reject**，
真 reject 了运行时兜底转 `failed` 并 warn——防清理与 waiter 挂死。

## 结算：first-wins 的全序

`settle()`（index.ts:416-440）是本篇最密的顺序纪律：

```text
若已终态 → return          ← first-wins：teardown 强制 failed 后，迟到的
                              producer 结算不翻案
记 status/detail/output/finishedAt
waiters>0 → reported=true   ← 有人在等的结算，通知已被消费方"预定"
同步释放全部 waiters → markSettled → notifyChanged(owner)
最后才跑 onJobDone 监听者   ← reporter 可能同步开模型轮次：别的观察者必须
                              先见到已承诺的记录，通知顺序=因果顺序
```

监听者逐个 containment（throw/reject 都只 warn）；服务拆卸后 `listenersClosed`
闸死。**拆卸取消也计入 reported**：owner 正在消失时开轮次通知等于白烧一次模型请求。

其余三动词同样"先做不可逆的那半"：

- `kill`：先 producer cancel（抛出则状态与通知位都不动）再记 `stopping`+reported；
  已终态则只补 reported 并回 `already-finished`。
- `read`：流式 job 消费增量、final-output job 幂等返终稿；**终态读取顺手置
  reported**（读过=通知过）。
- `wait`：`using deadline(signal, timeout, TASK_WAIT_TIMEOUT)` 区分"等到超时"
  与"被取消"——超时是**好结果**（返回当时快照、不杀 job），取消才 reject；abort
  同步摘除 waiter，防同 tick 结算把"已经欠它的等待"误记为已通知。

## tool-jobs：薄壳不薄

插件装载即 `attachController('tool-jobs')`（index.ts:259）——三个控制工具的存在本身
就是给 `servesOwner` 供闸；不装载该包的组合里 `start()` 一律拒。

- **输出帽是 producer 定价的**：`outputLimitBytes` 随 `JobStart` 声明，
  `tools/pre-execute` 以 `prepend` 抢在压缩类监听者前把帽挂到 exec 上、
  `finalizeContent` 兑现——先核"渲染还是不是自己产的形状"（canonical value
  校验），是才保 output/status 分栏裁剪，否则整段兜底截断。
- **完成通知的送达经济学**（index.ts:278-299）：owner 忙 → inject 进 next-step inbox
  （N 个 job 同结算只花一步）；owner 闲且 `completionDelivery='wakeup'`（默认）→
  `followup` 开轮；预算 `maxConsecutiveWakes=3` [MEASURED] 封顶"叫醒→开工→
  完工→再叫醒"的自激链，**用户亲手输入即回血**（claimed 事件里 `source.kind === 'user'`
  才清计数——本插件自己排的队不回血）。
- 提示词席位 `tool:jobs` 把使用纪律写给模型：不轮询不 sleep、终答前收齐在途作业、
  `wait:true` 只用于真被阻塞——模型侧三工具是注册表语义的薄壳，不发明状态。

## WebhookRuntime：验证→冻结→分发的三步纪律

`dispatch()`（webhook/index.ts:129-140）同步段内完成：closing 拒 →
`snapshotDelivery`（五字段校验 + `snapshotJsonValue` 无损 JSON 化 +
`deepFreeze`）→ 逐规则起调用。**规则拿到的是冻结快照**，篡改不了别人的输入；
每条调用独立 containment、任何回调结算前返回。

- 注册是泛型面 + 擦除存储：公开 `register<K>` 保住提供方作者类型，表内只存校验
  共享 kind 后的 `AnyWebhookRule`。
- **卸载 = hide → abort → drain**：`disposeRegistration` 先删表（后续交付进不来）、
  abort 该规则自己的 controller、再排空 `active` 集合；`disposal` memoize 防
  并发双拆。运行时整体 unload 用 effect 兜底排全部规则。
- 三枚品牌 id（rule/source/delivery）各管一事：**deliveryId 只是出处字段**——
  runtime 不存储它、不拿它去重；重复投递可以创建重复会话，克制写进文档。

### 会话创建事务（session.ts）

非 `null` 结果走一段镜像插件启动序的编排，每步 await 后 `throwIfAborted`：

```text
resolveRequest      同步快照+全字段校验（跨 await 前把规则结果与宿主脱钩）
presets 预检        permissionPresets.resolve + agentPresets.resolve + standingKeyFor
workspace create    规范 Workspace（并发下别人已建=复用）
agents.create       sessionId 铸 webhook-<uuid>；cwd=workspace 路径；
                    setup 挂 agentPreset + installInitialModelSelection
workspace.attachSession   ← attached 位置起后失败才走 detach 回滚
permission set / title rename / followup(prompt, source.kind='webhook')
```

回滚分两段且**不取代原始错误**：attach 前失败只 dispose agent；attach 后失败
detach+dispose（各自失败只 warn）。预检期自动建的 Workspace **保留**——并发调用者
可能已在使用。`installInitialModelSelection` 给"显式 model 选择"上临时钩：仅在该
会话**尚无持久 request/header** 且路由撞上继承值时改写 `reasoningEffort` 等字段——
首个请求落盘纪元头后钩子自动失效，选择权交回正常链路。`webhook-invariant` 在
`agent/inbox/spliced` 事件上核对：webhook 来源消息准入时其会话必须**恰好属于一个
Workspace** 且路径=cwd——创建事务的后置断言有人站岗。

### GitHub 适配器：错误码面就是文档

`webhook-github`（handler.ts 123 行 + body.ts 63 行）的顺序与码表：
POST-only(405 + allow 头) → content-type 严格 JSON+至多一个 charset 参数(415) →
`readBoundedUtf8Body`（Content-Length 声明先拒 413、流式计数再拒、声明过大时
`request.resume()` 放干水流、非法 UTF-8 拒 400）→ 三个头必恰一值
（`headersDistinct` 防头拼接走私，400）→ credentials.resolve(503) →
**octokit verify 先于 parse**(401) → `parsePayload`（仅 `JSON.parse` 在 try 内
——其他异常不冒充格式错，400）→ dispatch（runtime 缺席 503）→ 202。
secret 每请求按 ref 现解，配置热轮换免费获得。官方评审指南把这条路由挂在
**隔离的第二台 WebServer** 上：暴露 webhook 入口不等于暴露浏览器 API。

## 相关

- 概览浅层：[后台任务与外部触发](../augmentation/background-and-triggers.zh.md)
- 三家 producer 的执行体内幕：[Shell 与终端内幕](./shell-terminal-internals.zh.md) ·
  [委派内幕](./subagent-deep.zh.md)
- 不走 jobs 的编排通道：[工作流与 Agent Teams](./workflow-agent-team.zh.md)
- followup/inbox 的合流闸：[turn/step 主循环](../agent-runtime/turn-step-loop.zh.md)
- 品牌化 id 与 ScopedLayers 底座：[工程基建](./engineering-base.zh.md) ·
  [主干循环内幕](./agent-loop-internals.zh.md)

## 诚实边界

- `tool-jobs` 的 `job_kill` 输出 schema 尾段（index.ts:372 之后）未逐行；
  三工具的 presentCall 细节以读到的公共段为准。
- 数字（默认 10 并发 / 30s wait 默认 / 10min wait 帽 / 3 连唤预算）为本 checkout
  源码常量 [MEASURED]，patch 层可覆盖，出厂值≠现场值。
- GitHub 签名验证端到端未实测（无凭据与外发通道）；码表与顺序取自源码直读，
  octokit `Webhooks.verify` 内部行为以其实现为准。
- `pty-send` 与 terminal 会话生命周期的耦合细节归
  [Shell 与终端内幕](./shell-terminal-internals.zh.md) 口径，本篇只在 producer 映射里点名。
- `WebhookModelSelection` 省略时"快照当前部署默认直到首个持久 header"的竞态窗口
  未做多 agent 并发实测。
