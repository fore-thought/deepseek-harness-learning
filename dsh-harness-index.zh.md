---
title: "dsh-harness 主题导航"
tags: [dsh, moc]
status: active
license: CC-BY-SA-4.0
updated: 2026-09-05
---

# DSH 源码架构 · 主题导航

[English](dsh-harness-index.md) | 中文

> 目的：把 DSH（TypeScript 实现的开源 LLM agent harness）的架构吃透——
> 纯对象知识；如何用它再造什么，属其他主题。
> 从 [DSH 源码架构总览](./overview.zh.md) 读全景；按层进入分篇。

## 阅读顺序建议

1. [DSH 源码架构总览](./overview.zh.md) —— 全景与设计决定
2. 框架地基：[Cordis 内核](./cordis/cordis-kernel.zh.md) → [插件组装与启动](./cordis/plugin-composition.zh.md)
3. Agent 主干：[会话事件日志](./agent-runtime/session-event-log.zh.md) →
   [turn/step 主循环](./agent-runtime/turn-step-loop.zh.md) →
   [工具注册表与执行流水线](./agent-runtime/tools-pipeline.zh.md)
4. 模型层：[LLM 层](./llm-layer/llm-vocabulary.zh.md) → [上下文工程](./llm-layer/context-engineering.zh.md)
5. 平台层：[会话持久化与存储](./platform/session-persistence.zh.md) →
   [host/client 分层与 API 网关](./platform/host-client-boundary.zh.md) →
   [应用壳](./platform/web-cli-boot.zh.md)
6. 执行世界：[文件系统与沙箱](./execution/filesystem-and-sandbox.zh.md) →
   [进程、shell 与终端](./execution/shell-process-terminal.zh.md) →
   [代码运行、LSP 与远程世界](./execution/remote-and-code-runtime.zh.md)
7. 增强层：[委派与编排](./augmentation/subagent-orchestration.zh.md) →
   [能力供给](./augmentation/skills-mcp-hooks.zh.md) →
   [自组织](./augmentation/goal-plan-todo.zh.md) →
   [人机问答与反馈](./augmentation/questions-and-answers.zh.md) →
   [后台任务与外部触发](./augmentation/background-and-triggers.zh.md) →
   [配置面](./augmentation/settings-and-credentials.zh.md)

## 专题深读（deep/，29 篇 · 组级全覆盖）

源码级走读 + 真机实测，对 `packages/` **全部 51 个包组**每组至少一篇主承接
（承接表=本页下表，兼作"谁管什么"的速查索引）。用法建议：首轮按上方 1→7 顺读
概览建骨架，deep/ 作为第二轮按需深读入口；每篇自证、证据路径可复核。
下表按层分组，「主承接包组」列即覆盖账：

| 专题深读篇 | 主承接包组 | 衔接概览篇 |
|---|---|---|
| [主干循环内幕](./deep/agent-loop-internals.zh.md) | core（8 包全组） | [turn/step 主循环](./agent-runtime/turn-step-loop.zh.md) · [工具流水线](./agent-runtime/tools-pipeline.zh.md) |
| [提示词组装与运行时上下文](./deep/prompt-assembly-context.zh.md) | system-prompt · context（6 包） · preset | [turn/step 主循环](./agent-runtime/turn-step-loop.zh.md) · [上下文工程](./llm-layer/context-engineering.zh.md) |
| [派生侧深读：投影/标题/遥测/格式代次](./deep/session-projection-telemetry.zh.md) | session（派生侧 18 子包） | [会话持久化与存储](./platform/session-persistence.zh.md) · [会话事件日志](./agent-runtime/session-event-log.zh.md) |
| [会话查询内幕](./deep/session-query.zh.md) | session-query（4 包） | [会话持久化与存储](./platform/session-persistence.zh.md) |
| [表层改写与压缩](./deep/surface-compaction.zh.md) | compaction（4 包） | [会话事件日志](./agent-runtime/session-event-log.zh.md) · [上下文工程](./llm-layer/context-engineering.zh.md) |
| [LLM 适配器与计量内幕](./deep/llm-adapters-metering.zh.md) | llm（7 包含 token-meter） | [LLM 层](./llm-layer/llm-vocabulary.zh.md) |
| [附件与溢出](./deep/attachment-spill.zh.md) | attachment · spill（5 包） | [上下文工程](./llm-layer/context-engineering.zh.md) |
| [持久化格式与崩溃恢复](./deep/persistence-crash-recovery.zh.md) | session（持久化侧） · storage（seam/后端） | [会话持久化与存储](./platform/session-persistence.zh.md) |
| [HTTP 载体与控制器分层](./deep/host-gateway-webserver.zh.md) | host（5 包） · api（5 包） | [host/client 分层与 API 网关](./platform/host-client-boundary.zh.md) · [应用壳](./platform/web-cli-boot.zh.md) |
| [Web Client 浏览器架构](./deep/web-client-architecture.zh.md) | client（45 包全组） | [host/client 分层与 API 网关](./platform/host-client-boundary.zh.md) |
| [typert 远程协议与代码生成](./deep/typert-remote-protocol.zh.md) | typert（4 包） | [host/client 分层与 API 网关](./platform/host-client-boundary.zh.md) |
| [SDK 三件套深读](./deep/sdk-embedding.zh.md) | sdk（3 包） · examples | [应用壳](./platform/web-cli-boot.zh.md) |
| [Profile 组装与启动六 bundle](./deep/boot-bundles.zh.md) | boot（2 包） · bundle（6 包） · apps | [插件组装与启动](./cordis/plugin-composition.zh.md) · [应用壳](./platform/web-cli-boot.zh.md) |
| [Cordis 热重启与热重载内幕](./deep/cordis-hot-reload.zh.md) | extensions（4 包） · vendor | [插件组装与启动](./cordis/plugin-composition.zh.md) |
| [文件系统深读](./deep/filesystem-observation.zh.md) | fs（7 包全组） | [文件系统与沙箱](./execution/filesystem-and-sandbox.zh.md) |
| [沙箱执行内幕](./deep/sandbox-execution.zh.md) | sandbox（4 包） | [文件系统与沙箱](./execution/filesystem-and-sandbox.zh.md) |
| [Shell 与终端内幕](./deep/shell-terminal-internals.zh.md) | shell（10 包） · subprocess · terminal · guard | [进程、shell 与终端](./execution/shell-process-terminal.zh.md) |
| [LSP 与 E2B 远程世界](./deep/lsp-e2b-remote.zh.md) | lsp（3 包） · e2b（3 包） | [代码运行、LSP 与远程世界](./execution/remote-and-code-runtime.zh.md) |
| [PTC 与 code-runtime 内幕](./deep/ptc-code-runtime.zh.md) | code-runtime（2 包） | [代码运行、LSP 与远程世界](./execution/remote-and-code-runtime.zh.md) · [工具流水线](./agent-runtime/tools-pipeline.zh.md) |
| [Web 工具深读](./deep/web-tools.zh.md) | web（6 包全组） | [工具注册表与执行流水线](./agent-runtime/tools-pipeline.zh.md) |
| [委派内幕](./deep/subagent-deep.zh.md) | subagent（9 包） · acp | [委派与编排](./augmentation/subagent-orchestration.zh.md) |
| [工作流引擎与 Agent Teams](./deep/workflow-agent-team.zh.md) | workflow（4 包） · experimental（8 包） | [委派与编排](./augmentation/subagent-orchestration.zh.md) |
| [自组织四域深读](./deep/goal-plan-todo-schedule.zh.md) | goal · plan · todo · schedule | [自组织](./augmentation/goal-plan-todo.zh.md) |
| [交互、审批与权限档位](./deep/interaction-approval-feedback.zh.md) | interaction（5 包） · feedback · identity | [人机问答与反馈](./augmentation/questions-and-answers.zh.md) |
| [配置面深读](./deep/settings-credentials-workspace.zh.md) | settings · credentials · workspace · storage-domain | [配置面](./augmentation/settings-and-credentials.zh.md) |
| [技能·MCP·hooks 内幕](./deep/skill-mcp-hooks-internals.zh.md) | skill（4 包） · mcp · hooks（3 包） | [能力供给](./augmentation/skills-mcp-hooks.zh.md) |
| [后台任务与外部触发深读](./deep/jobs-and-webhooks.zh.md) | jobs（3 包） · webhook（2 包） | [后台任务与外部触发](./augmentation/background-and-triggers.zh.md) |
| [不变量注册表](./deep/invariants-registry.zh.md) | runtime-diagnostics · 全仓不变量横切 | [会话事件日志](./agent-runtime/session-event-log.zh.md) |
| [工程基建深读](./deep/engineering-base.zh.md) | util（13 包） · test-support · native · python | （新辟主题，无概览对应） |

## 证据基线

全部结论来自 DSH 源码直读（本机源码 checkout），证据一律写 DSH 仓库相对路径，
可按路径复核；未闭环处内联标 `待核实`。基线快照 2026-09；深读篇中带日期的
"现行/已修正"标注即复核时点。