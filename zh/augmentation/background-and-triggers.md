---
title: "后台任务与外部触发：jobs 与 webhook"
tags: [dsh, jobs, webhook, background]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{jobs,webhook}/* + docs/subsystems/{jobs,webhook}.zh.md；子代理F调研"
updated: 2026-09-04
---

# 后台任务与外部触发：jobs 与 webhook

## `ctx.jobs`：一切"跑了不止一拍"的东西

- `JobRegistry` 抽象 seam + `LocalJobRegistry` provider；生产方是**四个工具**
  （tool-bash / tool-pwsh / tool-subagent / tool-terminal 的 pty-send）声明
  `JobStart{kind, label, hooks}`；**workflow 不走 jobs**——其可观察性归
  `workflow/*` 事件对（早先"workflow……"列入生产方系误载，深读 grep 全库
  `jobs.start(` 仅四调用点 [MEASURED]）。`JobKindMap` 声明合并扩展 kind 词汇
  （在盘实况四家：bash·pwsh·subagent·pty-send；官方页只列前两）。
- **准入预检**：owner 并发上限（默认 ≤10）+ 无 owner 共享桶 + **必须有已 attach 的
  controller 服务该 owner 才准入**（防孤儿任务）；
- 生命周期钩子 `JobHooks{cancel, done, readOutput}`——注册表不关心任务本体语义，
  **bash 进程与子 agent 会话在同一生命周期协议下并存**——后台任务、委派子代理
  与模型轮询同走一条 `job_output` 线；
- 结算 **first-wins 终态、完成通知最后发**（reporter 可能同步开模型轮次——通知顺序
  即因果顺序）；`read`：流式=上次读后增量、final=终态幂等；`wait` 有界；
- **owner 会话 id = 授权边界**（id 可预测，安全靠鉴权不靠藏）。
  `job_list/job_output/job_kill` 工具是模型侧薄壳。

## `ctx.webhookRuntime`：外部世界进 inbox 的另一扇门

- 提供方适配器（`dsh-webhook-github`）在注入的 `ctx.webServer` 注册路由：
  **验未改动 raw body 签名** → 内存 `dispatch()` → 立即 202；
- `dispatch`：快照匹配规则、回调彼此独立、任何结算前返回、异常按规则隔离；
- 回调返回 `WebhookSessionRequest` → 校验 permission/agent preset →
  resolve/创建规范 Workspace → 建 Agent（cwd=workspace 路径）挂 preset →
  follow-up `user/message{source.kind:'webhook'}` 进 inbox 获准后提交
  （[turn/step 主循环](../agent-runtime/turn-step-loop.md) 的 inbox 唯一输入闸在此再次兑现）；
- **无队列/无重试/无去重/无崩溃重放**——文档明说的克制：webhook 只负责"把外部事件
  变成一条会话输入"，可靠增值是消费方的事。`WebhookEventMap` 按 kind 声明合并扩展。

## 相关

- 子代理后台运行同一注册表：[委派与编排](./subagent-orchestration.md)
- 外部输入的合流点（inbox）：[turn/step 主循环](../agent-runtime/turn-step-loop.md)
- HTTP 载体：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
- 两缝内幕（准入三闸/first-wins 结算全序/webhook 会话创建事务/签名纪律）：
  [作业注册表与 Webhook 运行时](../deep/jobs-and-webhooks.md)
