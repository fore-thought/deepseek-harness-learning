---
title: "会话事件日志：唯一真源"
tags: [dsh, session, event-sourcing]
status: active
license: CC-BY-SA-4.0
evidence: "packages/core/session/src 直读 + docs/subsystems/session.zh.md + 子代理B/E调研"
updated: 2026-09-04
---

# 会话事件日志：整个 harness 的唯一真源

> DSH 把"对话历史"做成**仅追加事件日志**（event sourcing）：`ctx.sessions`
> （`SessionStore`）持有内存会话，每条事实是一个带序号的 `SessionEvent`。
> 模型看到的历史、UI 渲染、持久化、fork/恢复、压缩改写、遥测——全部从这条流**投影**派生。

## 事件词汇表（`SessionEventMap`，可声明合并扩展）

核心 12 类（`packages/core/session/src/types.ts`）：

| 事件 | 含义 |
|---|---|
| `turn/start · turn/end` | 轮次开闭；`turn/end` 带 `TurnEndReasonMap`（completed/aborted/blocked/error/max-tokens/interrupted） |
| `step/start · step/end` | 步骤（一次模型请求+其工具调用） |
| `user/message` | 用户/注入消息（唯一"模型可见新输入"通道） |
| `assistant/chunk` | 原始流式分块（UI 保真与回放的原料） |
| `assistant/message` | 折叠后的助手消息（usage 同条持久） |
| `tool/call · tool/result` | 工具调用与结果（`sourceEventSeqs` 回指 call） |
| `request/header` | 请求纪元头：渲染后的 system + tools + 模型配置（EpochHeader） |
| `request/context` | 路由/容量变化声明 |
| `session/end-seed` | fork 种子边界 |

信封公共字段：`seq`（会话内单调）、`time`、`ignorable?`（前向兼容标记：
老读取方可安全跳过）、`surfaceOp?`、`sourceEventSeqs?`（事实间因果回指）。
各子系统经 **声明合并** 往里加事件（`agent/inbox/spliced`、`compaction/*`、
`llm/retry`、`hook/*`…），**扩展事件类型 = 扩展持久化协议**，无需改 session 包。

## Surface（表层）：模型实际看到什么

- 只有 3 类事件"产消息"（`SurfaceEventType`）：`user/message`、
  `assistant/message`、`tool/result`；`deriveMessages()` 从日志投影出模型历史。
- `SurfaceOp = 'append' | { op:'replace', start, end }`：**压缩改写历史不是删日志，
  而是追加一个带 replace 的检查点事件**——日志仍完整，表层被区间替换；
  `replaceGeneration` 递增标识代际。
- 铁律"**模型可见即已记录**"：任何抵达模型请求的输入都必须能从日志重建，
  由运行时不变量断言（见 [插件组装与启动](../cordis/plugin-composition.md) 的 invariants 机制）。

## 投影框架（读侧的通用原语）

`ctx.sessionProjections`：注册 `ProjectionDefinition { key, stateVersion, init, apply, view? }`
纯同步 fold 单元；一次订阅 `session/event` 全家共享。host 侧用 `stateOf()` 读类型化状态，
客户端经 `snapshot()` 拿裁剪视图。agent-loop 的 `turnBoundary` 投影（轮/步边界水位）
是读方标杆用法；重试计数、会话统计、标题都走同一框架。
`ctx.sessionProjectionCache`（storage 域 `session_projcache`）提供
checkpoint→tail replay→full refold 的**读阶梯**，冷启动不必全量重放。

## 崩溃恢复语义

- 写失败**尽力而为**、不阻断对话（日志可缺尾）；`flush` 才是持久性屏障（见 [会话持久化与存储](../platform/session-persistence.md)）。
- resume 时 `interruptedTurnClosers`：对未闭合轮在**写句柄内**合成补齐
  （缺果的 `tool/result`、未闭合 `step/end`、`turn/end{interrupted}`）——
  修复回写只允许发生在写所有权之下；冷读者只做内存配平、不改盘。

## 为什么这样设计

- fork/transcript/回放/遥测/多设备续读 = 同一事件流的不同消费方；
- "追加一切 + 表层可区间替换"让压缩、注入、修复都保持可审计；
- `ignorable` 与前向兼容让**旧 UI 读新日志、新 UI 读旧日志**都不炸。

## 相关

- 谁在写这些事件：[turn/step 主循环](./turn-step-loop.md) · [会话持久化与存储](../platform/session-persistence.md)（怎么落盘）
- 投影怎么送到浏览器：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
