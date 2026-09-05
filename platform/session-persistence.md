---
title: "会话持久化与存储：日志落盘的工程细节"
tags: [dsh, persistence, storage, jsonl]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{session,storage}/* + docs/subsystems/{persistence,storage}.zh.md；子代理E调研"
updated: 2026-09-05
---

# 会话持久化与存储：日志落盘的工程细节

## SessionPersistence seam（`ctx.sessionPersistence`）

- 接口：`create/open/stat/list` + 逐会话 `SessionHandle`
  （`read/append/flush/close`）；后端 JSONL 实现
  （`session-persistence-jsonl`：`session.jsonl.zstd` 帧）。
- **进程内单写者**：句柄是唯一写门；agent-loop 是句柄获取/发布点——
  **不经 loop 发布的会话不持久化**（fork 只在"loop 拥有的会话"上做，防止半路会话）。
  跨进程租约是**已记录的限制**（persistence.zh.md），不是缺失。
- **写路径**：每条 `session/event` 按会话 id 路由进活跃句柄的
  **有界 write-behind 批窗口**（200ms，`LIVE_WRITE_BATCH_MAX_DELAY_MS` [MEASURED]；
  首事件开窗、后续加入不重置截止）→ 到期批量 append；
  `session/flush` parallel 事件 = 取消等待排空 + 错误观察检查点；
  `session/disposed` = 最终排空 + close。"append 尽力而为、flush 才是屏障"是
  **接缝契约的上限**——JSONL 实现的显式 append 每帧同步 fsync（失败截回回滚），
  真正的批缓冲只发生在事件路由窗口；全链见
  [持久化格式与崩溃恢复](../deep/persistence-crash-recovery.md)。
- **撕裂尾**（crash 半行）永不到达读取方（读侧修复归 reader；写所有权下由
  `interruptedTurnClosers` 补齐回写，见 [会话事件日志](../agent-runtime/session-event-log.md)）；实体化延迟到
  首次 append/flush（不产生空文件垃圾）。
- 事件对 UI/远程是实时流（`session/event` 广播），持久化失败**不回滚内存事实**——
  "日志可缺尾"是明示的可靠性等级。

## storage 枢纽（`ctx.storage`）：非会话数据的 KV 领域

- `StorageBackend { kv? }` → `KvUnit`；提供方
  `storage-json`/`storage-sqlite` 共存注册；
- `ctx.storageDomain`：**领域路由 + schema 校验 + 写链**
  （`defineDomain` 声明；全库仅 3 域：workspace、message-feedback、session_projcache
  ——领域是稀缺登记，不是随手建表）；`domain/changed` 事件驱动镜像。
- 变更令牌：**不透明 revision**（变更代际不透明化是跨版本兼容的关键）。
- `ctx.sessionTelemetry`（OTLP 提供方可选，一 context 单实现）+
  `session-record` waterfall 脱敏——遥测是投影消费者。

## 读侧工具箱

- `ctx.sessionQuery`：live 优先合并 `persistence stat/list`；revision 令牌做
  冷缓存键；`coldBlankProbeMaxEvents/Bytes` 用元数据限工（冷会话探查有预算上限）；
  全文检索 = sqlite FTS 提供方。**读永远不阻塞在解析上**。
- `ctx.sessionTitle`：标题提供方（首 prompt LLM / 全 prompt LLM 两种），
  log-backed 可重放。
- `ctx.sessionFileReferences / sessionReferenceResolver`：@ 引用文件的
  session 寻址适配与跨会话快照准备（[委派与编排](../augmentation/subagent-orchestration.md) 的 fork 与 @ 引用靠它）。

## 相关

- 投影框架与读阶梯：[会话事件日志](../agent-runtime/session-event-log.md)
- 事件怎么从这些读侧原语摆到浏览器：[host/client 分层与 API 网关](./host-client-boundary.md)
