---
title: "工具注册表与执行流水线"
tags: [dsh, tools, pipeline, guard]
status: active
license: CC-BY-SA-4.0
evidence: "packages/core/tools/src；docs: tool-execution-pipeline / tools / scope；子代理B"
updated: 2026-09-05
---

# 工具注册表与执行流水线

> `ctx.tools`（`ToolRuntime`，`packages/core/tools`）管两件事：
> **注册表**（工具 schema 供提示词组装）与**带把关的执行流水线**（每次工具调用过六道闸）。
> 工具是面向模型的能力的统一注册面——加一项能力 = 在 `ctx.tools` 上注册。

## 工具定义与 schema 纪律

- `defineTool` DSL；输入/输出都走**被强制的 JSON Schema 子集**
  （`tools/src/json-schema.ts`），保证跨提供方与代码生成的稳定性；
  输出类型推断 `InferValue/InferArgs` 纯编译期。
- `executionMode`：`parallel`（仅显式 `isConcurrencySafe === true`）/
  `exclusive`（屏障）；**异常/未知一律按 exclusive（fail-closed）**。

## 每次调用的六段流水线

```text
① append tool/call（持久事实先落盘）
② tools/pre-execute  waterfall → PreToolDecision
     'ask' → ctx.get('approval') 一次性询问（审批服务缺席/无 agent/取消 = deny，fail-closed）
     'deny' → 直接产 error 结果
     [单调 ToolGuard：同一调用内 deny 只增不减，防重入翻案]
③ tools/execute      waterfall（around 包装：可换 signal 不可去 signal；可整体接管）
④ 工具体执行（拿 ToolRunContext：deferContext/concludeTurn/agent 句柄）
⑤ tools/post-execute waterfall → PostToolDecision（可改写输出、附加 additionalContexts）
⑥ append tool/result（sourceEventSeqs=[callSeq]，meta 持久）
```

- `additionalContexts` 进 next-step inbox——工具可以合法地给下一拍注入模型可见上下文；
- `concludeTurn()`：工具可宣布"整批提交后本轮结束"（如 `exit_plan_mode`、
  Ralph 到限的止损语义都走这条正路，不是循环内特判）。

## 步骤内调度：并行派发、按序提交（`agent-loop/src/tool-calls.ts`）

- 模型一次响应可含多个 tool call：按 `executionMode` 分组，exclusive=屏障、
  parallel=有界滚动池（默认并发 10，settings 可热改）；
- **派发可重叠，结果提交严格按模型序**（`commitReady` 只在 contiguous 槽推进）——
  日志顺序永远是模型看到调用的顺序，恢复/回放无歧义；
- 每次 start 前重读 mode：注册表热变化可以制造新屏障；已启动的 abort 后排空按序提交，
  未启动的记合成错误 `ABORTED_BEFORE_DISPATCH`。

## 作用域层（scope）：per-agent 工具集

`packages/core/scope`（零服务、零依赖的底层库）实现两级扁平作用域：

- 注册项要么全局，要么归属恰好一个 **scope key**（活跃 agent 对象自身即 key，身份比较）；
- **shadowing**：作用域内同名覆盖全局（最具体者胜出）——按 agent 定制 persona/工具变体的机制；
- **restriction**（`tools.restrict`）：为单个 scope 过滤全局工具集，多个 restriction 取交集；
  被过滤的工具**既不进提示词、也拒绝执行，与不存在不可区分**；
- 作用域注册**不向下继承**给 subagent；子树行为靠 lineage 数据表达。
  带 `this: Scoped<T>` 的监听器参与作用域过滤派发（carrier `thisArg` + filter），
  注册表主体事件（如 `tools/change`）故意不过滤——全局变化关乎所有作用域。

## 审批与提权通道（approval seam 速览，详见执行世界篇）

`sandbox_permissions` 这类"一次性重试带提权请求"的语义 = `PreToolDecision 'ask'` +
`ctx.userQuestions`/approval waterfall 的组合产物；应答方可以是人（GUI 卡片）、
钩子、或自动策略——同一个挂点，不同 provider。

## 相关

- 流水线在循环中的位置：[turn/step 主循环](./turn-step-loop.md)
- schema 怎么进提示词：提示词组装（`ctx.systemPrompt`：分段 section + order +
  `complete` 独占语义 + tools provider，按作用域链合并渲染）
- 审批应答的传输：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
- ptc/both 执行面坍缩与 run_code 桥内幕：[PTC 与 code-runtime 内幕](../deep/ptc-code-runtime.md)
