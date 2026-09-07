---
title: "LLM 层：统一词汇与适配器"
tags: [dsh, llm, adapter, streaming]
status: active
license: CC-BY-SA-4.0
evidence: "packages/llm/* + docs/subsystems/llm-streaming.zh.md；子代理C调研"
updated: 2026-09-04
---

# LLM 层：统一词汇与适配器

[English](llm-vocabulary.md) | [中文](llm-vocabulary.zh.md)

> `packages/llm/llm` 定义"对话词汇表"（Message/ContentBlock/StreamChunk）与
> 适配器 seam（`ctx.llm` = `LlmRuntime`）；提供方适配器（`llm-deepseek`、
> `llm-pi-ai`）与重试（`llm-retry`）都是插件。**agent loop 依赖的是这个包，
> 从不依赖任何具体提供方包**。

## 统一词汇（`llm/src/types.ts`、`message.ts`）

- `ContentBlockMap`：text/reasoning/image/tool-call/tool-result——**可扩展联合**
  （声明合并加块类型）；
- `StreamChunk`：封闭七事件：`block-start · text-delta · reasoning-delta ·
  tool-call-delta · block-end · usage · finish`，以 `index` 关联块；
  契约：**usage 先于 finish，finish 后不再出块**。
- `Message`（不可变）+ `MessageSourceMap`（user/plugin/model/tool——谁"造成"了这条
  消息，注入可归因）+ `ContextForm`（instructions/catalog/snapshot/notice/relay/recall：
  注入上下文的**体裁**，进 prompt 时可差异化渲染）。
- `TokenUsage`：input/output 互斥语义、cacheRead/Write 单列（DeepSeek 要从
  `prompt_tokens` 扣缓存——提供方方言在适配器层消化）。
- 失败词汇：`LlmFailure` + `FinishReasonMap`（含 `aborted/error` 带 failure）；
  `CONTEXT_WINDOW_EXCEEDED` 等规范 code 是压缩恢复的触发词。

## 适配器 seam（`llm/src/index.ts`）

- `LlmAdapter` 抽象类：必实现 `stream()`；可选
  `providerInfo/providerRetryPolicy/listModels/resolveModel/prepareCall/imageRequestPricing`。
- `LlmRuntime.registerAdapter` → `AdapterRegistrationHandle`（disposer +
  **原子 replace**）：注册 all-or-nothing、重复 provider 报 `DUPLICATE_ADAPTER`；
  route 集变化用 `replace()` 原子换轨，在途流保持旧事实。
- 一次适配器调用 = 一次提供方尝试（库内重试禁用，重试归 loop 层）。
- `ReplayEnvelope`：适配器私有回放态（response 级 + 与发射块**逐块对齐**条目；
  "内容与元数据同一保留/丢弃决定"是不变式）。私有回放态**仅当同一适配器实例
  同时拥有历史与目标提供方时才回传**——跨提供方不搬运方言。
- `BlockAssembler`：**唯一共享 fold 点**（chunk → Message），max-tokens/中断时
  同步裁剪 tool 调用与 replay blocks。所有消费方（落日志/UI/派生历史）共用同一折叠，
  杜绝"UI 看到了日志没有的文本"。
- `AppIdentity + attributionHeaders()`：每个提供方 HTTP 请求必带 User-Agent 归属。

## DeepSeek 适配器一帧（`packages/llm/llm-deepseek/src/`）

```text
serialize.ts   Message[] → OpenAI 兼容 wire（tool_calls/reasoning；图片按模型像素预算
               生成 request variant：Files API file_id 或 base64 内联，超限最旧优先降级）
fetch POST /chat/completions → sse.ts eventsource-parser 解帧（缺 [DONE] 即抛 STREAM_CLOSED）
→ translate.ts usage/finish → StreamChunk 七事件
idle watchdog（默认 5 分钟；提供方 TIMEOUT 与调用方 ABORTED 严格区分）
```

连接参数（baseURL/key）**每请求经 `ctx.settings`/`ctx.credentials` 解析** →
改 key 换 baseURL 不重启（见 [配置面](../augmentation/settings-and-credentials.zh.md)）；
`deepseek-llm-api-extensions` 把 `dsh_plugin_packages`/`dsh_session_log`
这类顶层请求字段做成**认领制扩展表**：各包注册字段贡献者，`prepare()` HTTP 前合并、
`accept()` 事务 2xx 后提交。

## 重试：循环侧、可持久（`packages/llm/llm-retry`）

失败步骤进 `agent/request-error` waterfall → llm-retry 读注册时冻结的
`ResolvedRetryPolicy`（normal{maxRetries,retryableCodes,退避+jitter} / always 两模式）→
**先 append `llm/retry`（可取消等待之前先持久化）再等待** → 返回
`{kind:'retry'}` 不调 next → 同轮换新步骤号续跑。提供方 `Retry-After` 优先于
本地指数退避；重试计数存 session-projection（`step/start`/`turn/end` 清零）。两事件链
（`llm/retry` 先持久→可取消等待→`llm/retry-started`）与 175 行不变量校验器见深读。

## 相关

- 谁驱动 stream：[turn/step 主循环](../agent-runtime/turn-step-loop.zh.md)
- token 计量/压缩/附件/spill：[上下文工程](./context-engineering.zh.md)
- 内幕深读（注册表三本账、流协议裁决、双适配器对照）：
  [LLM 适配器与计量内幕](../deep/llm-adapters-metering.zh.md)
- 附件→模型的字节旅程：[附件与溢出](../deep/attachment-spill.zh.md)
- 提供方界面（模型选择/定价）怎么到浏览器：[host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)
