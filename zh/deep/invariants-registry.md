---
title: "运行时不变量：包自有断言与影子重建"
tags: [dsh, invariants, runtime-diagnostics, defensive]
status: active
license: CC-BY-SA-4.0
evidence: "packages/runtime-diagnostics/invariants/src/index.ts 全读 + 全仓 ./invariant 伴生清点
  + core/session、compaction、goal-round-driver、llm-retry 伴生精读 + docs/subsystems/invariants.zh.md
  + packages/AGENTS.md 约定 + invariants/README.zh.md；均为 DSH 仓库相对路径"
updated: 2026-09-05
---

# 运行时不变量：包自有断言与影子重建

> 总览设计决定 2「**模型可见即已记录**」说"并有运行时不变量断言"——本篇就是那个
> 断言层的执法机制：一个 180 行、零产品导入的注册表服务（`ctx.invariants`），
> 加上 **39 个包自有 `./invariant` 伴生插件**（合计 ≈3,131 行 [MEASURED，
> 2026-09 checkout]）——每个包在旁边守自己的持久关系，违规抛归因到包名的错误。

## 注册表侧：只管三件事

`packages/runtime-diagnostics/invariants/src/index.ts`（180 行，tests 276 行）
不导入任何 session/agent/tools 包，注册表只拥有**选择、名称保留、子 fiber 生命周期**：

- **`Config` 选择配置**：`enabled`（默认 true）+ `package_allowlist` /
  `package_blocklist`（区分大小写的 JS 正则源，blocklist 在 allowlist 之后优先）；
  条目启动期**明确报错不静默**——空白、带边空白、重复、非法正则全部抛出；
  有效模式可以不匹配任何当前已加载包（为未来 HMR 保持确定性）；
  过滤器在服务生命周期内固定，改它=重载插件。
- **`InvariantInstaller` 形状**：`(ctx, fail) => void | Promise<void>` 加可选
  `inject`；`fail` 是 `InvariantFailure = (message) => never`——报告即抛出，
  不存在"记一笔继续跑"。抛出的 `InvariantError` 携带稳定 `code: 'INVARIANT'`、
  完整 npm `packageName` 与 `invariant violated by "<package>": …` 前缀：
  归因不需要注册表认识任何产品代码。
- **注册即保留归属**：`register(name, installer)` 即使被过滤器判为不启用也**保留
  包名**（两个插件永远无法静默认领同名）；启用的 installer 在专属子 fiber 运行、
  同步/异步完成都会在注册成功前被 join；失败**原子回滚**（dispose 子 fiber +
  释放保留），损坏的检查不留半截监听器。返回的 disposer 双向绑定：卸载配套或
  服务任一侧都清干净，配套可重载后重注册同名的——HMR 安全。

## 伴生约定的反转：从"空伴生"到"省略即合规"

三处官方文本有时间差，读源码要按最新口径：

| 出处 | 口径 |
|---|---|
| `docs/subsystems/invariants.zh.md` | 无检查项的包导出**空安装器**，起始注释 `No runtime invariant:` 解释 |
| `packages/AGENTS.md`（现行） | **只为会发散的观察发布 `./invariant`**；否则省略接线并在包 README 写理由；空伴生/忽略 reporter 会被 `verify-package-invariants` 拒绝 |
| 实测（2026-09 checkout） | src 内空安装器 **0 个**；README 理由行 **122 处** [MEASURED]——反转已落地 |

即：子系统页那段"空安装器"描述是 2026-08-28 简化案（Agent Note
`omit-unneeded-invariant-companions`）之前的口径，现行约定是"省略接线 +
README 归置理由"。伴生数与 `exports "./invariant"` 声明**恰好相等（39=39）**
[MEASURED]，接线由机械门保证没有半挂状态。

## 39 个伴生在守什么

统一形态是**影子重建**：伴生持一份与权威事实同构的折叠器，对每个新事件先验后改，
不一致即 `fail`。最大的四份（`compaction` 341 行、`core/session` 231 行、
`time-context` 185 行、`llm-retry` 163 行 [MEASURED]）都是这个骨架：

| 包（行数） | 守的关系 | 内幕所在 |
|---|---|---|
| core/session(231) | seq 单调、step 级事件指名开闭中的 turn/step、tool/call↔result 配对（deferred transition 验证不改已提交 trace） | [事件日志](../agent-runtime/session-event-log.md) · [主干内幕](./agent-loop-internals.md) |
| core/agent(27) · core/agent-loop(56) | agent 状态转移；**loop 构建请求**可从日志重建（仅 loop 显式构建者，一次性直调 LLM 不在此约定） | [主干内幕](./agent-loop-internals.md) |
| core/scope(36) · core/tools(120) | 作用域过滤派发主体；流水线阶段配对与冻结结果 | [工具流水线](../agent-runtime/tools-pipeline.md) |
| core/system-prompt(51) · preset/agent-presets(74) · time-context(185) | 组装 section 名；preset 挂载位置；持久时钟读数（"持久钟面"防回放漂移） | [提示词组装](./prompt-assembly-context.md) |
| llm(104) · llm-retry(163) | 流式语法终界；`llm/retry` 的 provider 中立 payload 与退避帽（`MAX_TIMER_DELAY_MS` 边界复用） | [LLM 内幕](./llm-adapters-metering.md) |
| compaction(341，最大) | 括号状态机 start/summary/end/**end-seed 豁免**，并 import `SurfaceManager` 影子重折表层 | [表层改写](./surface-compaction.md) |
| goal(70) · goal-round-driver(76) | 持久 goal 流折叠；**续轮提示词逐字节重建**（`foldGoal`→视图→`renderGoalRoundPrompt`→深比较） | [自组织四域](./goal-plan-todo-schedule.md) |
| tool-todo(93) · schedule(47) · plan-mode(42) | 整表形状与开放轮次；schedule 流；plan-mode 载荷 | [自组织四域](./goal-plan-todo-schedule.md) |
| user-approval(101) · commands(60) · permission-presets(33) | 审批 asked/decided 配对；命令 run/done 配对；preset 引用指向活树 | [交互与审批](./interaction-approval-feedback.md) |
| hook-protocol(110) | 钩子调用/结果配对 | [技能·MCP·hooks](./skill-mcp-hooks-internals.md) |
| fs(42) · sandbox-policy(36) | 文件系统事件身份；mode 值合法域 | [文件系统深读](./filesystem-observation.md) · [沙箱内幕](./sandbox-execution.md) |
| subagent(84) · tool-subagent(45) · agent-team(31) | 提供方与 start/end 配对；团队记录形状 | [委派内幕](./subagent-deep.md) · [工作流与 Teams](./workflow-agent-team.md) |
| workflow(125) · tool-workflow(151) | 生命周期身份；workflow 记录形状 | [工作流与 Teams](./workflow-agent-team.md) |
| jobs(49) · webhook(43) | 快照字段关系；外部触发记录 | [后台与触发](../augmentation/background-and-triggers.md) |
| session-title(74) · session-log-deepseek(66) | 标题来源消息 seq；收据水位 | [派生侧深读](./session-projection-telemetry.md) |
| settings(43) · credentials(33) · authorization(40) · storage-domain(62) · workspace(52) | 提交事件对照活动服务/内存镜像（**实体缓存镜像**一族） | [配置面深读](./settings-credentials-workspace.md) |
| client/hmr(53) · modules(40) · ui-renderer(42) | stat-watcher 生命周期；启动入口图；slot 变更版本化 | [Web Client 架构](./web-client-architecture.md) |

细节两处值得单独点名：`core/session` 伴生 import 了 `./repair.ts` 的
`TOOL_NOT_STARTED`——**崩溃修复的产物本身也受不变量复查**，修复与断言共享同一
常量词表；`compaction` 伴生直接实例化 `SurfaceManager` 做平行表层——它不是检查
"字段像不像"，而是**把折叠器原样再跑一遍对答案**，这就是"真实关系而非人为断言"。

## 挂载面：生产组合默认不跑任何不变量

全仓 bundle/app 接线里，注册表只出现在 **`sdk-minimal`** 一处
（`cordis.patch.yml` L110-123：注册表 + session/agent/scope/agent-loop 四核心
伴生 [MEASURED]）；`base`、`web-app` 等出厂组合零挂载。包 README 首段"标准
agent 组合已挂载"指的正是这个**最简诊断组合**，别读成"产品默认开检查"——
挂载即 `verify` 级观察面，产品组合用**测试与 verify 脚本**承担同等纪律。
作为观察者，伴生只读请求与持久状态、从不改写内容，提供方缓存复用与不挂时
完全一致（README「模型体验」同款声明）。

## 官方口径对读与邻居谱系

设计出处是两篇 Agent Note（`2026-07-19-package-owned-invariant-service` 定
注册表为何不碰产品代码；`2026-07-19-package-invariant-runtime-contracts` 定
断言面与机械门）；`docs/defensive-patterns.zh.md` 是另一谱系——"血泪缺陷
规则"文库（正交结果独立上报、dispose 必须停稳、环境清洗等），**规则给作者，
不变量给运行时**，两者同源不同执法方式，互补不互替。

## 与"模型可见即已记录"的三角关系

该设计决定其实是三层结构：**约定**（事件日志仅追加，[事件日志篇](../agent-runtime/session-event-log.md)）
→ **消费**（投影/回放/压缩全从日志派生，[派生侧深读](./session-projection-telemetry.md)）
→ **执法**（本篇：包自有影子重建，违约即抛归因错误）。错误码
`INVARIANT` 还有一条跨包豁免线：llm 注册表的监听者故障包住只 log、
唯独 `code==='INVARIANT'` **重抛不吞**（`llm/llm/src/index.ts:360`，
机制在[LLM 内幕](./llm-adapters-metering.md)已讲）——不变量违例是全局唯一
"不许被观察者故障掩埋"的信号。另注意两词勿混：本篇的包自有不变量与
`llm-retry` payload 校验器同族不同人——后者的 175 行校验器归重试事件链
（[LLM 内幕](./llm-adapters-metering.md)）。

## 相关

- 设计决定原文：[总览](../overview.md) 关键设计决定 2
- 被断言的日志本体：[会话事件日志](../agent-runtime/session-event-log.md)
- 括号锁/表层代际（被最大伴生复查的机制）：[表层改写与压缩](./surface-compaction.md)
- 注册即逆序释放的框架底座：[Cordis 内核](../cordis/cordis-kernel.md)
- 挂载方 bundle 解剖：[Profile 组装与启动](./boot-bundles.md)

## 诚实边界

- 39 伴生 / 各文件行数 / 122 README 理由行 / sdk-minimal 五挂载行均为
  2026-09 checkout 快照 [MEASURED]，上游迭代会漂移。
- 全量精读 4 份伴生（session/compaction/goal-round-driver/llm-retry 头部），
  其余 35 份按 name/inject/校验函数签名结构式走读——"守什么关系"列为口径
  自 README 表格 + 签名证据，未逐行。
- `verify-package-invariants` 脚本未本机执行（越出只读 checkout 边界）；
  README 表格 34 行与实测 39 包存在列举滞后（含一行 `dsh-client-runtime` 在
  本 checkout 无对应 src 伴生），按"README 自称目录非穷尽"理解，不判官方错误。
- 子系统页/AGENTS/README 三口径中"空伴生"问题的最终裁决以
  `verify-package-invariants` 实际断言为准，本篇只报告文本差异与 src 实测。
