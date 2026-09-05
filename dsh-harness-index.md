---
title: "dsh-harness 主题导航"
tags: [dsh, moc]
status: active
license: CC-BY-SA-4.0
updated: 2026-09-05
---

# DSH 源码架构 · 主题导航

> 目的：把 DSH（TypeScript 实现的开源 LLM agent harness）的架构吃透——
> 纯对象知识；如何用它再造什么，属其他主题。
> 从 [DSH 源码架构总览](./overview.md) 读全景；按层进入分篇。

## 阅读顺序建议

1. [DSH 源码架构总览](./overview.md) —— 全景与设计决定
2. 框架地基：[Cordis 内核](./cordis/cordis-kernel.md) → [插件组装与启动](./cordis/plugin-composition.md)
3. Agent 主干：[会话事件日志](./agent-runtime/session-event-log.md) →
   [turn/step 主循环](./agent-runtime/turn-step-loop.md) →
   [工具注册表与执行流水线](./agent-runtime/tools-pipeline.md)
4. 模型层：[LLM 层](./llm-layer/llm-vocabulary.md) → [上下文工程](./llm-layer/context-engineering.md)
5. 平台层：[会话持久化与存储](./platform/session-persistence.md) →
   [host/client 分层与 API 网关](./platform/host-client-boundary.md) → [应用壳](./platform/web-cli-boot.md)
6. 执行世界：[文件系统与沙箱](./execution/filesystem-and-sandbox.md) →
   [进程、shell 与终端](./execution/shell-process-terminal.md) →
   [代码运行、LSP 与远程世界](./execution/remote-and-code-runtime.md)
7. 增强层：[委派与编排](./augmentation/subagent-orchestration.md) →
   [能力供给](./augmentation/skills-mcp-hooks.md) →
   [自组织](./augmentation/goal-plan-todo.md) →
   [人机问答与反馈](./augmentation/questions-and-answers.md) →
   [后台任务与外部触发](./augmentation/background-and-triggers.md) →
   [配置面](./augmentation/settings-and-credentials.md)

## 专题深读（deep/）

源码级走读 + 真机实测六篇，回填并修正概览篇留下的开放问题（含推翻性修正）：

| 专题 | 衔接概览篇 | 深读篇 |
|---|---|---|
| Cordis 热重启与 HMR 内幕 | [插件组装与启动](./cordis/plugin-composition.md) | [cordis-hot-reload](./deep/cordis-hot-reload.md) |
| typert 协议与代码生成 | [host/client 分层与 API 网关](./platform/host-client-boundary.md) | [typert-remote-protocol](./deep/typert-remote-protocol.md) |
| 沙箱执行（仲裁/令牌/拒绝分类） | [文件系统与沙箱](./execution/filesystem-and-sandbox.md) | [sandbox-execution](./deep/sandbox-execution.md) |
| 持久化格式与崩溃恢复 | [会话持久化与存储](./platform/session-persistence.md) | [persistence-crash-recovery](./deep/persistence-crash-recovery.md) |
| 表层改写与压缩 | [会话事件日志](./agent-runtime/session-event-log.md) · [上下文工程](./llm-layer/context-engineering.md) | [surface-compaction](./deep/surface-compaction.md) |
| PTC 与 code-runtime | [工具注册表与执行流水线](./agent-runtime/tools-pipeline.md) · [代码运行、LSP 与远程世界](./execution/remote-and-code-runtime.md) | [ptc-code-runtime](./deep/ptc-code-runtime.md) |

## 证据基线

全部结论来自 DSH 源码直读（本机源码 checkout），证据一律写 DSH 仓库相对路径，
可按路径复核；未闭环处内联标 `待核实`。
