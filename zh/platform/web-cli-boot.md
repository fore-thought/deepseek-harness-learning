---
title: "应用壳：CLI 启动、浏览器 boot、SDK 与 ACP"
tags: [dsh, apps, cli, web, sdk, acp]
status: active
license: CC-BY-SA-4.0
evidence: "apps/{cli,web}、packages/{sdk,acp,host,client/web} + docs；子代理E调研"
updated: 2026-09-04
---

# 应用壳：CLI 启动、浏览器 boot、SDK 与 ACP

## 一个 CLI，四个孔

`apps/cli` 是**唯一 bin**（包名 `dsh`）：launcher 实为**三模式分发**
（profile / plugin / dump-config）× 5 个 profile 模板；`dsh web` =
`--profile web` 的别名；headless/sdk/acp 是 profile 不是可执行文件
（入口纪律由 launcher 校验，见 [插件组装与启动](../cordis/plugin-composition.md)；
"孔"指应用面而非代码分支——六 bundle 装配差异见 [Profile 组装与启动](../deep/boot-bundles.md)）。

## 浏览器端 boot（Web 版的"第二次启动"）

```text
web-app 粘合插件：解析前端 dist 位置 → 打印带 token URL → 开浏览器
frontend-static：认领 webserver 回退席位；authorizeIndex：token→host-only cookie→302
index 渲染：clientModules(ctx.clientModules 注册表) 经 webserver/index-inject
  贡献 window.__DSH_BOOT__ row + combo 脚本
浏览器 AppWebEntry.run()：建 lazy-CJS 模块表 → 预取 immediately 行
  → new Context + Loader 激活全部 client entry → uiRenderer.mount 一次 renderSlot('root')
```

（Web GUI 里"客户端插件改动需要 dev:web watcher 才热更新"的既有事实，根源就是这条线：
浏览器侧 Cordis 树独立于 Host 树热载。）

## SDK：进程外正门

`packages/sdk/`：`JsonRpcLineTransport`（换行分帧 JSON-RPC 2.0）+
方法 `initialize/session/prompt/shutdown`、通知
`session.event/session.status/subagent.{started,finished}`。
server 端只是 stdio profile 里的一个插件；TS/Python SDK client 以**子进程 spawn**
同版本 `dsh --profile sdk`（Python wheel 打包 dsh CLI）——
**SDK 不内嵌 harness，而是买一张进程外的票**：版本（client×dsh 版本号**逐字相等**硬锁）、
沙箱、配置全部归 launcher 纪律管。协议面与回收阶梯内幕：
[SDK 三件套深读](../deep/sdk-embedding.md)。

## ACP：给编辑器/自动化的标准孔

`packages/acp/` = 标准 ACP v1 **仅自动化**服务器：
`session/new|list|resume|close、prompt、cancel、update、request_permission`；
无 fork/删除/load（标准能力边界）。同时 ACP 也是 **subagent 提供方**的目标形态
（把轮次委派给另一个 ACP 产品，见 [委派与编排](../augmentation/subagent-orchestration.md)；
`packages/acp`=对外 server 与 `subagent-acp`=对外驱动是**两包两面孔**，见
[委派内幕](../deep/subagent-deep.md)）。

## headless：一次性运行器

`dsh-headless`：不带服务器跑完任务即退（`config.task:
!!js ctx.headlessStartup.task`——命令行 flag 值经服务注入门控流入插件配置，
[插件组装与启动](../cordis/plugin-composition.md) 的表达式机制用例）。CI 与脚本集成的孔。

## 相关

- 启动装配全链：[插件组装与启动](../cordis/plugin-composition.md)
- 浏览器对话的传输线：[host/client 分层与 API 网关](./host-client-boundary.md)
