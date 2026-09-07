# DeepSeek Harness 源码架构手册

[English](README.md) | 中文

本手册严格依据 **DeepSeek Harness**（`dsh`，开源 LLM agent harness）源码，讲清它的
架构设计——写给完全没摸过电脑的人读，读完你脑中能装下一张完整架构图，
并有能力、有冲动设计自己的 harness。

- 第三方独立作品，与 DeepSeek 官方无关。
- 基于 2026-09 源码 checkout；每条结论都带源码仓库相对路径证据，可复核。
- 每个页面都有两份：`foo.md`（英文）与 `foo.zh.md`（中文），同目录、排版与内容
  严格一致。仍标 `status: draft` 且带 i18n-stub 标签的英文页为占位页，待翻译。

## 状态

| 线 | 内容 | 位置 |
|---|---|---|
| `v1.0.0`（tag，封板） | 完整中文版：22 概览 + 29 源码级深读，覆盖全部 51 包组 |
  仓库 Tags 页；勘误走 `v1.x` |
| `main`（2.0 手术线） | 手册宪章下的双语重写：图先于字、背景页、零门槛行文 | 当前视图 |

## 结构

- 全景：[总览](./overview.zh.md) · [主题导航](./dsh-harness-index.zh.md)
- 框架地基：[Cordis 内核](./cordis/cordis-kernel.zh.md) ·
  [插件组装与启动](./cordis/plugin-composition.zh.md)
- Agent 主干：[会话事件日志](./agent-runtime/session-event-log.zh.md) ·
  [turn/step 主循环](./agent-runtime/turn-step-loop.zh.md) ·
  [工具注册表与执行流水线](./agent-runtime/tools-pipeline.zh.md)
- 模型层：[LLM 层](./llm-layer/llm-vocabulary.zh.md) ·
  [上下文工程](./llm-layer/context-engineering.zh.md)
- 平台：[会话持久化与存储](./platform/session-persistence.zh.md) ·
  [host/client 分层与 API 网关](./platform/host-client-boundary.zh.md) ·
  [应用壳](./platform/web-cli-boot.zh.md)
- 执行世界：[文件系统与沙箱](./execution/filesystem-and-sandbox.zh.md) ·
  [进程、shell 与终端](./execution/shell-process-terminal.zh.md) ·
  [代码运行、LSP 与远程世界](./execution/remote-and-code-runtime.zh.md)
- 增强层：[委派与编排](./augmentation/subagent-orchestration.zh.md) ·
  [能力供给](./augmentation/skills-mcp-hooks.zh.md) ·
  [自组织](./augmentation/goal-plan-todo.zh.md) ·
  [人机问答与反馈](./augmentation/questions-and-answers.zh.md) ·
  [后台任务与外部触发](./augmentation/background-and-triggers.zh.md) ·
  [配置面](./augmentation/settings-and-credentials.zh.md)
- 源码级深读（29 篇，按包组簇承接）：
  [主题导航承接表](./dsh-harness-index.zh.md#专题深读deep29-篇--组级全覆盖)

## 许可

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — 全文见仓库根目录
[LICENSE](./LICENSE)。
