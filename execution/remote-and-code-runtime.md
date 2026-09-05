---
title: "代码运行、LSP 与远程世界"
tags: [dsh, code-runtime, lsp, e2b, ptc]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{code-runtime,lsp,e2b}/* + docs/subsystems/{code-runtime,lsp}.zh.md；子代理D调研"
updated: 2026-09-05
---

# 代码运行、LSP 与远程世界

## code-runtime（`ctx.codeRuntime`）：模型写代码、harness 跑代码

run_code 之外的一切 JS 执行都收敛到这个 seam（PTC/"程序化工具调用"的宿主）：

- `run({program, bindings, signal}) → CodeRunResult{value?, logs, error?}`；
  `CodeRunFailure.kind` 六路正交：exception/timeout/abort/worker-exit/
  invalid-output/output-limit；
- 本地实现（`code-runtime-worker-thread`）：**每次运行新建 Worker**；
  host 先 `stripTypeScriptTypes`（node:module；异步函数壳保持位置）；
  worker 内捕获 console/stdout/stderr → LogBuffer **按外层 JSON 字节预算即时传回**；
  binding 调用走关联 id 消息桥（**host 视入站为敌意**）；
  预算=实测 busy-time（`eventLoopUtilization` 25ms 轮询）+ wall 上限 +
  堆上限（**仅 V8 old-space 溢出触发 worker-exit**；TypedArray 等 external memory
  不计入，只撞 compute 预算——概览篇口径的收窄修正 [MEASURED]）+ 输出上限；
- **诚实的隔离声明**：`isolation` 只读描述符**仅是诊断标签不构成安全承诺**
  （containment 非安全边界）——与 fs-sandbox 的自述同一口径。这类"文档不夸大"
  的纪律贯穿全库。

## LSP（`ctx.lsp`）：刻意闭合的小词汇表

- **4 个操作封顶**：`goToDefinition · findReferences · goToImplementation · hover`；
  结果联合也闭合；扩展名→语言 provider 注册（`lsp-stdio` 起真语言服务器）；
- 消费方=面向模型的 `tool-lsp`；**没有让模型直接操纵 LSP 会话状态**——
  把 IDE 能力压成四个稳定查询，是"能力词汇表保持小"的又一例
  （对照 [能力供给](../augmentation/skills-mcp-hooks.md) 的 MCP 只桥 Tools）。

## E2B：远程执行世界的完整标本

`packages/e2b/`：`ctx.e2b` 共享单一远程 SDK 句柄；
**fs-e2b + subprocess-e2b 成对替换本地世界**（[文件系统与沙箱](./filesystem-and-sandbox.md)）；
Bash/PTY/LSP/工具全部自动搬到远程，无需逐工具改造——"seam 换提供方=换产品形态"
的教科书演示（architecture.zh.md 能力 seam 一节的实例）。

## PTC（programmatic tool calling）速记

`tools/src/ptc.ts` + `agent-tool-presentation`（native/ptc/both 展示模式
preset 行）：工具既能被模型逐个调用，也能以编程面暴露给 code-runtime 里的脚本——
`tools/ptc-dispatch-log` waterfall 记录派发。多轮工具编排从"消息往返"
降为"一段程序"。（本会话用 run_code 调 read/write 正是这个形态。）

## 相关

- 执行世界配对总论：[文件系统与沙箱](./filesystem-and-sandbox.md)
- 工具流水线：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.md)
- 远程化后 UI 如何跟进：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
- run_code 桥/调度器/敌意闸与 21 项真机实测：[PTC 与 code-runtime 内幕](../deep/ptc-code-runtime.md)
- `ctx.remote` 背后的协议与生成器：[typert 远程协议与代码生成](../deep/typert-remote-protocol.md)
