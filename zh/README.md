# DSH 源码架构知识库

> 语言开关：English track → [../en/README.md](../en/README.md)；1.0 封板版 = tag
> `v1.0.0`（本树在 main 上进入 2.0 手术期，行文逐步遵守手册宪章）。

> 对 DeepSeek Harness（`dsh`，开源 LLM agent harness）的**中文源码架构梳理**。
> 第三方独立作品，与 DeepSeek 官方无关。基于 2026-09 的源码 checkout；
> 上游快速迭代，与最新源码不符时以上游为准（各篇证据均为源码仓库相对路径，可复核）。

## 怎么读

- 第一次来：打开 `overview.md`（全景 + 设计决定 + 分层导航）。
- 用 Obsidian 打开本文件夹作为 vault 体验最佳（相对路径双链、mermaid、反链图谱；
  GitHub 网页端链接同样可点）。

## 内容结构

```text
overview.md / dsh-harness-index.md   全景与导航
cordis/          插件框架地基（内核 / 组装启动 / 热重启内幕）
agent-runtime/   主干（会话事件日志 / turn-step 循环 / 工具流水线）
llm-layer/       模型层（词汇与适配器 / 计量压缩附件）
platform/        平台（持久化 / host-client 网关 / 应用壳）
execution/       执行世界（fs 沙箱 / 进程终端 / code-runtime·LSP·E2B）
augmentation/    增强层（委派编排 / 技能挂载 / 自组织 / 问答 / 后台 / 配置）
deep/            专题深读（29 篇：源码级走读与真机实测，组级全覆盖 packages/ 51 组）
```

## 许可与署名（Notice）

<div align="left">

This work is licensed under
<a rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">CC BY-SA 4.0</a>
<img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="CC" width="20" height="20">
<img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="BY" width="20" height="20">
<img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" alt="SA" width="20" height="20">

</div>

（图标用 width/height 属性而非 style 内联样式——GitHub 渲染会剥离后者。纯文本转载请使用：
"This work is licensed under CC BY-SA 4.0. To view a copy of this license, visit
https://creativecommons.org/licenses/by-sa/4.0/"）

- 本仓库文字内容的著作权归其作者，以 **CC BY-SA 4.0**
  （[署名-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-sa/4.0/)）发布，
  全文见根目录 LICENSE。
- 转载/改编请：①保留本许可声明并指向本仓库（作者不另行要求具名，指向仓库即满足署名）；
  ②标注修改；③衍生作品使用相同许可。
- 文中引用的 DSH 源码片段 © DeepSeek AI（DSH 本体为
  [MIT License](https://github.com/deepseek-ai/deepseek-harness/blob/main/LICENSE)），
  仅作架构说明用途并标注仓库相对路径。
- 本作品为第三方独立梳理，与 DeepSeek 官方无关；非担保性文档，以上游源码为准。
