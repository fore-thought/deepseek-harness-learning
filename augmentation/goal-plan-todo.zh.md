---
title: "自组织：goal、plan mode、todo、schedule"
tags: [dsh, goal, plan, todo, schedule]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{goal,plan,todo,schedule}/* + docs/subsystems 同名页；子代理F调研"
updated: 2026-09-04
---

# 自组织：goal、plan mode、todo、schedule

[English](goal-plan-todo.md) | [中文](goal-plan-todo.zh.md)

> 四种"跨轮次维持的工作状态"，共同的实现姿势：**持久领域事件 + 投影 + CAS revision**，
> 调度语义与状态本体严格分离。

## goal：同会话完成目标（`ctx.goals`）

- `GoalService`：目标附着在**现有会话**上，`GoalRef{id, revision}` **CAS**
  演进；phase = `active|paused|blocked|complete`（blocked 带策略 code+说明；驱动器另有
三个自动 block 机器码：round-limit / queue-failed / prompt-rejected）。
- 持久事件 `goal/change` = **全量快照**（revision+1）或墓碑；实时
  `goal/changed` 走 Scoped 派发（按 agent 过滤）。
- **Goal Round**：驱动器 `goal-round-driver` 是 `agent/turn-stopping`
  serial 终检的消费方——该续则续（获准轮次的 `user/message` 标注
  `GoalMessageSource{goalId, revision, round}`，回放拒绝缺口/陈旧 revision/超上限）。
- **激活（armed/disarmed）是进程本地权限、刻意不入回放**：重启/fork 后自动续跑
  必须经一次**人类授权的 resume 变更**才能重新上膛——"长任务续跑"默认需要人。
- `/goal` 人类命令与模型工具 `create_goal/get_goal/update_goal` 操作同一领域。

## plan mode：协作状态位（`ctx.planMode`）

`PlanModeController + plan/mode{active}` 仅日志整值事件。一个包贡献三种挂点：
`plan:policy` 提示词段（order 50，遮蔽全局 persona）+ `exit_plan_mode` 工具
（**注册恒在、模式切换只改其执行决策**）+ `/plan` 命令。评审通道复用
`ctx.userQuestions`（见 [人机问答与反馈](./questions-and-answers.zh.md)）。

## todo：代理自用清单

`todo/write{todos}` 整列表替换仅日志事件（无服务端 item 状态机——
"完整列表替换"语义把复杂度推给协议而非存储）。

## schedule：定时器只造"普通消息"

- `ScheduleRecord = After|At|Every`；`schedule/change` 是唯一持久权威
  （严格 decoder：拒未知版本/复用 id）；
- 到期动作=把提醒作为**普通 follow-up `user/message`** 回注 live session
  （dispatch=入队即持久，无回执）——**不为定时器发明第二种输入通道**；
- cold/busy 期间 `Every` 只补最新一次到期并直推下一个（错过不枚举补偿）；
  多 Every 到期合一批 follow-up（限模型轮数）；存储一律 UTC RFC3339（4 位年），
  时区只在工具边界显式；
- fork 按 `inheritedEventCount` 折叠：**保留历史、不接管父的活动提醒**。

## 相关

- 四域内幕（CAS 转移表/激活边沿缴械/驱动器全链/DST 算法）：
  [自组织四域深读](../deep/goal-plan-todo-schedule.zh.md)

- serial 终检与 inbox 准入：[turn/step 主循环](../agent-runtime/turn-step-loop.zh.md)
- 提问/批准的应答线：[人机问答与反馈](./questions-and-answers.zh.md)
- 外部触发进同一 inbox 的另一路：[后台任务与外部触发](./background-and-triggers.zh.md)
