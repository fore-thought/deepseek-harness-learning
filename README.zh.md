# DeepSeek Harness 源码架构手册

[English](README.md) | 中文

本手册严格依据 **DeepSeek Harness**（`dsh`，开源 LLM agent harness）源码，讲清它的
架构设计——写给完全没摸过电脑的人读，读完你脑中能装下一张完整架构图，
并有能力、有冲动设计自己的 harness。

- 第三方独立作品，与 DeepSeek 官方无关。
- 基于 2026-09 源码 checkout；每条结论都带源码仓库相对路径证据，可复核。
- 语言配对：每个页面有英文文件（`foo.md`，主语言）与中文文件（`foo.zh.md`），
  排版与内容严格一致；开关行固定在标题正下方。中文单语页（英文翻译未到）暂不
  放开关，英文版落地当批补齐。

## 状态

| 线 | 内容 | 位置 |
|---|---|---|
| `v1.0.0`（tag，封板） | 完整中文版：22 概览 + 29 源码级深读，覆盖全部 51 包组 |
  仓库 Tags 页；勘误走 `v1.x` |
| `main`（2.0 手术线） | 手册宪章下的双语重写：图先于字、背景页、零门槛行文 | 当前视图 |

## 结构

| 分区 | 英文 | 中文 |
|---|---|---|
| 全景与导航 | `overview.md` | [overview.zh.md](./overview.zh.md) |
| 框架地基（插件内核） | `cordis/` | 同名 `.zh.md` |
| Agent 主干（循环·工具·提示词） | `agent-runtime/` | 同上 |
| 模型层（适配·上下文） | `llm-layer/` | 同上 |
| 平台（网关·持久化·客户端） | `platform/` | 同上 |
| 执行世界（fs·沙箱·shell） | `execution/` | 同上 |
| 增强层（子代理·技能·目标） | `augmentation/` | 同上 |
| 源码级深读（29 篇） | `deep/` | 同上 |

## 许可

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — 全文见仓库根目录
[LICENSE](./LICENSE)。
