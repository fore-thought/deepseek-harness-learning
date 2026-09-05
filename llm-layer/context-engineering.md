---
title: "上下文工程：计量、压缩、附件与溢出"
tags: [dsh, compaction, token-meter, attachment, spill]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{token-meter,compaction,attachment,spill} + docs 同名子系统页；子代理C调研"
updated: 2026-09-05
---

# 上下文工程：计量、压缩、附件与溢出

> 都是**能力 seam**（可选能力，不属于循环主干）：计量器、压缩引擎、附件仓、溢出仓。
> 循环与插件把它们当服务消费，全部可被 patch 行替换。

## token 计量（`ctx.tokenMeter`）

- `TokenMeasurement`：以 **surface（表层）节点**为单位的 `TokenSurfaceNode` 树：
  `logRevision`、baseline（真实 usage 锚点 | 估算）、`surfaceDeltaTokens`（带符号）。
- 双轨估算：图片按**路由定价**（visualTokens+模型可见文本），文本用固定启发式
  （`CHARS_PER_TOKEN=4`、块/角色 overhead 4 —— 内置默认，可调）；测量轨/投影轨两轨表面
  与 shadow-price 协议展开见 [LLM 适配器与计量内幕](../deep/llm-adapters-metering.md)；
- usage 锚点复用条件严格：同 envelope 且 ≥ 完整路由重定价才复用 → **缓存的是事实，
  不是猜测**。压缩/改写后按 delta 修正而非全量重算。

## 压缩（`ctx.compaction` = `CompactionEngine`，一 context 一实现）

触发两路：**压力**（agent/pre-step 里 `measure ≥ contextWindow × thresholdRatio`，
默认 0.8）与**溢出恢复**（`agent/request-error` 里规范 code
`CONTEXT_WINDOW_EXCEEDED` → `compactIfNeeded('context-overflow')`）。

`compaction-basic` 的算法（`packages/compaction/compaction-basic/`）：

1. 先调可选 `ctx.toolResultPruner`（无模型剪枝：head/middle/tail，产
   `compaction/prune` 影子计价事件）→ 重测，能免则免摘要调用；
2. `selectCompactableRange`：头部锚定、保留尾部（默认 retainRatio 0.16），
   边界必须 **tool-pairing 平衡**（`toolPairingBalancedBefore/After`）——
   绝不把 tool-call 和它的 result 拆进不同侧；
3. append `compaction/start`（**锁=日志上不配对的事件对**，崩溃可检测、语义清晰；
   恢复/fork 继承的孤儿括号被更新的 `session/end-seed` 豁免，不锁死新生命周期）；
4. 摘要调用**复用会话自身 system/tools 前缀**（`purpose:'compaction'`）——
   最大化提供方前缀缓存命中；指令作为最后一条 user 消息；产
   `<compacted-summary>` 文本；
5. append `compaction/summary` + 带 `surfaceOp:{op:'replace'}` 的
   `user/message` 检查点（**改写表层不动日志**，见 [会话事件日志](../agent-runtime/session-event-log.md)）→
   `compaction/end` 释锁。
6. 结果 `CompactionResult.shadowedRange` 是表层位置跨度（不是数值 seq）；
   仅当代际真的前进才返回 retry（溢出恢复防死循环）。

## 附件（`ctx.attachments` = `AttachmentStore`）

- **先持久化、后发事件**；`sha256 内容寻址`不透明 id（`ImageAttachmentRef`）；
- 两层分离：normalized 存储版（解码校验/EXIF 方向/长边 2048 规范化/质量梯度）
  vs **request variant**（按模型像素预算策略生成；`variantId = 附件+策略+固定编码器参数`
  → 同策略重放字节一致）；并发合并、每等待方独立取消、实例级限流默认 2；
- `resolveImageAttachmentAccess`：附件宿主路径 → **工具执行世界里的只读路径**
  ——附件对模型和工具同一扇窗（见执行世界篇）。
- 三级漏斗（admission→normalization→request variant）与落盘耐久全解：
  [附件与溢出](../deep/attachment-spill.md)。

## 溢出（`ctx.spillStore` = `SpillStore`）

工具纯文本结果超 `maxInlineBytes` → 存溢出仓、结果替换为
`SpillRef { locator, bytes, retrievalHint }`（不透明定位符+取回提示）；
保存失败**尽力而为**：保留内联，绝不因仓库故障毁掉本轮。fork 继承定位符不复制内容。
两武装（post-execute 三跳过 + ptc-dispatch-log）与预算代数见[附件与溢出](../deep/attachment-spill.md)。
（你现在看到 grep 大结果"存盘并给路径"，就是这个机制。）

## 相关

- 计量喂给谁：[turn/step 主循环](../agent-runtime/turn-step-loop.md) 的压力检查；
- provider 复用意图：[LLM 层](./llm-vocabulary.md)；
- 表层 replace 语义：[会话事件日志](../agent-runtime/session-event-log.md)
- 事务全序、常量组、影子价协议与真机验证：[表层改写与压缩](../deep/surface-compaction.md)
- 分工澄清：本篇管**容量面**（计量/压缩/仓储），"抵达模型的内容如何合成"归
  [提示词组装与运行时上下文](../deep/prompt-assembly-context.md)
