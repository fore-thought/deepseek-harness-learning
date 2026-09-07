---
title: "人机问答与反馈：userQuestions 与 feedback"
tags: [dsh, interaction, user-questions, feedback]
status: active
license: CC-BY-SA-4.0
evidence: "packages/interaction/*、feedback/*；docs: user-questions / feedback；子代理F/E"
updated: 2026-09-05
---

# 人机问答与反馈：userQuestions 与 feedback

> `packages/interaction/` 是"工具/policy 与 UI 的**中立词汇层**"：提问、审批、
> 命令、权限预设四件套都在这个组里，宿主与前端各持半边。

## `ctx.userQuestions`：agent-scoped waterfall 应答

- 模型侧工具 `ask_user_question` → `UserQuestionService.ask(request)`
  → 以 **agent 为 scope key 派发 `user-questions/request` waterfall**；
- **应答方是可插拔的 waterfall 监听者**：GUI 卡片（经 Remote 线到浏览器，见
  [host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)——这就是"请示一律卡片"能跨进程工作的原因）、钩子、
  或自动策略。**waterfall 顺序认领、先应答者胜**（非并发竞速，深读纠正旧口径）；
无应答者 → `NO_PROVIDER` 显式失败，
  **绝不静默挂起**；取消 → `ASK_ABORTED`。
- 回答是"选择+自定义文本"的并集，带 `multi_select` 语义；`Intent` 现行**唯一变体**
  `{kind:'plan-review', approve}`（生产者=plan-mode），旧"确认/选择/补充信息"三值
  词汇表已不存在（深读纠正）。

## approval：fail-closed 的一次性询问

`ctx.approval`（user-approval 包）：工具流水线 `'ask'` 决策的消费端
（[工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)）；**服务缺席=deny**。审批与提问共用 Remote waterfall 传输，
但领域上是两个 seam（一个改变执行许可、一个采集人类输入）。

## `ctx.messageFeedback`：生命周期绑定的消息反馈

storage-domain sidecar（全库 3 域之一——workspace、message-feedback、
`session_projcache`——与持久化篇口径统一，见
[配置面](./settings-and-credentials.zh.md)）：opaque version **乐观并发**；反馈绑"生命周期中的
消息"（assistant `messageId`）而非渲染产物——UI 重建不丢标注。

## `ctx.permissionPresets`：策略档位

注意归属：read-only / workspace-write / danger-full-access 的**模式词汇在
`sandbox/sandbox-policy`**，本 seam 只是"沙箱+审批两旋钮"的**捆绑表**
（derive 平局数学与 pinInitial 见深读）。你当前会话顶部的"file policy"提示
即这个 seam 的输出。

## 相关

- 传输线：[host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)
- 执行侧消费：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)
- 沙箱档位执行：[文件系统与沙箱](../execution/filesystem-and-sandbox.zh.md)
- 内幕深读（四域纠偏全表/真机 never 政策样本）：
  [交互、审批与权限档位](../deep/interaction-approval-feedback.zh.md)
