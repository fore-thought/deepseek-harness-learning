---
title: "dsh-harness 主题导航"
tags: [dsh, moc]
status: active
license: CC-BY-SA-4.0
updated: 2026-09-04
---

# DSH 源码架构 · 主题导航

> 目的：为"Java 版大模型 harness"打前站，把 DSH（TypeScript 实现）的架构吃透。
> 从 [DSH 源码架构总览](./overview.md) 读全景；按层进入分篇。

## 阅读顺序建议

1. [DSH 源码架构总览](./overview.md) —— 全景与设计决定
2. 框架地基：[Cordis 内核](./cordis/cordis-kernel.md) → [插件组装与启动](./cordis/plugin-composition.md)
3. Agent 主干：[会话事件日志](./agent-runtime/session-event-log.md) → [turn/step 主循环](./agent-runtime/turn-step-loop.md) → [工具注册表与执行流水线](./agent-runtime/tools-pipeline.md)
4. 模型层：[LLM 层](./llm-layer/llm-vocabulary.md) → [上下文工程](./llm-layer/context-engineering.md)
5. 平台层：[会话持久化与存储](./platform/session-persistence.md) → [host/client 分层与 API 网关](./platform/host-client-boundary.md) → [应用壳](./platform/web-cli-boot.md)
6. 执行世界：[文件系统与沙箱](./execution/filesystem-and-sandbox.md) → [进程、shell 与终端](./execution/shell-process-terminal.md) → [代码运行、LSP 与远程世界](./execution/remote-and-code-runtime.md)
7. 增强层：[委派与编排](./augmentation/subagent-orchestration.md) → [能力供给](./augmentation/skills-mcp-hooks.md) → [自组织](./augmentation/goal-plan-todo.md) →
   [人机问答与反馈](./augmentation/questions-and-answers.md) → [后台任务与外部触发](./augmentation/background-and-triggers.md) → [配置面](./augmentation/settings-and-credentials.md)
8. 横切收束：[Java 移植观察地图](./java-porting-map.md)

## 证据基线

全部结论来自 DSH 源码直读（本机源码 checkout），证据一律写 DSH 仓库相对路径，
可按路径复核；未闭环处内联标 `待核实`。
