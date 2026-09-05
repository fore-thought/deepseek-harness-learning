---
title: "持久化格式与崩溃恢复：zstd 帧、撕裂尾与写门"
tags: [dsh, persistence, zstd, crash-recovery, atomic-write]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/{session,storage,checkpoint-policy,jsonl} 源码直读 + 真机实测（真实落盘只读解析、截断副本恢复实验）"
updated: 2026-09-05
---

# 持久化格式与崩溃恢复：zstd 帧、撕裂尾与写门

> 本篇回答三个问题：会话日志在磁盘上到底是什么容器、每一帧怎么写出去、崩溃留下的
> 撕裂尾如何被修成语义配平的日志。它是[会话持久化概览](../platform/session-persistence.md)
> 中 SessionPersistence seam 一节的深读展开。证据基准 = DSH 仓库源码直读 + 本机真机实测：
> 对 harness home 下 `sessions/<projectKey>/<sessionId>/session.jsonl.zstd` 的真实
> 落盘全程只读解析，恢复实验用截断副本。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| write-behind 批窗口"参数未给出" | 已证实 **200ms**：`LIVE_WRITE_BATCH_MAX_DELAY_MS = 200`（storage.ts:34）；首事件开窗、后续事件**不重置**截止 [MEASURED] |
| "append 尽力而为、flush 才是屏障" | 这是**接缝契约的上限**；JSONL 后端的显式 append 本身即同步 fsync 持久（index.ts:822-856），flush 只补实体化。真正的批缓冲只发生在事件路由通道（`session/event` → `enqueueLive` 的 200ms 窗） |
| 实体化 = "临时文件→rename" 直觉 | **修正**：POSIX 用 **link()+unlink() 而非 rename()**——link 对已存在目标失败 = 防两进程同 id 互相覆盖（index.ts:704-761）；Windows 用 `MoveFileExW` + `MOVEFILE_WRITE_THROUGH` |
| "撕裂尾永不到达读取方" | 定量闭环：读侧双层修复（帧级+行级）+ 写侧首 mutation 消费；恢复能力是**压缩块粒度全有或全无**（见「崩溃恢复」节） |
| "跨进程租约是已记录的限制" | 更精确：**已声明未实现**——`SessionOwnershipLostError` 存在（errors.ts:52-67 明示）但现有后端从无抛点 |
| zstd 帧格式（概览篇未展开） | 拼接帧容器：1 帧 = 1 次落盘批次，checksum 开启，非 single-segment（zstd.ts:111-120） |
| revision 令牌（概览篇未展开） | stat 五元组拼接，**不是内容哈希**（index.ts:108-121） |
| withFileLock（概览篇未展开） | 会话日志后端**不使用** atomic-write 包；`.lock` 兄弟文件锁留给 settings/credentials 等消费方 |

## 接缝契约：四条红线

`SessionPersistence`（Service 基类，ctx 键 `sessionPersistence`）提供
`create/open/stat/list` + 服务级 `flush()`。契约注释（index.ts:122-229）的语义红线：

- 事件 `seq` 从 0 连续、**永不重写**；撕裂物理尾**永不返回给读方**；写方在首次 append 前负责截尾修复。
- `create` 即**进程内可见**（pending 表），但**跨进程可见性推迟到实体化**；
  崩溃时未实体化 = 从未存在，不产生空文件垃圾。
- 新鲜度定义：append/flush 的 promise resolve 之后，同一后端实例上**之后开始**的读至少看到该前缀。
- `SessionHandle`（handle.ts:17-119）= 单所有者通道：`read|write` 两访问态，
  write = 进程内独占声明；read 不回头（`observedLength` 单调，越界抛 "stored log
  shrank"）；close 幂等不可取消，关闭后一切操作 → `SessionHandleClosedError`。

校验单一来源 `storage-contract.ts`：`assertVersion` 版本方向感拒绝（新格式 →
"upgrade the harness"，旧格式 → "no upgrade path"）；`validateStoredEvents` 对未知
事件类型 **fail-closed**（除非 `ignorable:true`）；`materializeAppendBatch` 深快照
——校验值即落盘值，防 accessor 双读漂移；`assertContiguous` 守住连续性。

## 物理容器：拼接的 zstd 帧

- 容器 = **拼接帧**（concatenated frames），帧边界 = 落盘批次边界：头部帧与每个事件批
  帧相互独立，压缩从不跨批合并；头帧实测 178B→209B 明文、恰 1 行。
- `JsonlCompression = 'zstd' | 'none'`；同一会话 root 混用两种编码 = 硬错误，
  旧平铺布局 = 硬错误——都是显式拒绝，没有静默兼容。
- **存储行 ≠ 逻辑事件**：`ChunkRow`（`text-chunks` / `reasoning-chunks` /
  `tool-call-chunks`，chunk-rows.ts:67-152）不注册进 `SessionEventMap`：`seq0/time0`
  锚点 + `data{turn,step,index,dt[],texts[]|{id,name?,args[]}}`；`MIN_RUN = 3`、
  精确键白名单 classify、分数 `time` 拒打包；实测最大 run = 18。
- `sourceEventSeqs` 区间编码（seq-ranges.ts）：≥3 个连续段折成闭区间，实测样本
  `[[29,86]]` = 58 个 chunk seq 折成 1 条；单引用原样存数字（`[88]`）。
- 头部行（`type:'session'`）字段：version/id/createdAt/cwd?/parentSession?/
  seedLength?/origin?/delegationDepth/agentPreset?；退役字段出现即拒。实测子会话头部带
  `parentSession`+`seedLength`+`origin:"subagent"`+`delegationDepth:1`——
  fork 的继承链一行说尽。

### 信封样本（字段值已脱敏）

```json
{"type":"reasoning-chunks","seq0":31,"time0":1757000000000,
 "data":{"turn":1,"step":1,"index":0,"dt":[0,1,0],"texts":["…"]}}
{"type":"user/message","seq":21,"source":{"kind":"user","rpcId":"<rpc>"},"surfaceOp":"append"}
{"type":"session","version":0,"id":"<sessionId>",
 "cwd":"<workspace>","delegationDepth":0,"agentPreset":"standard"}
```

实测出现率（6049 帧快照）：`ignorable` = 0/33279；`surfaceOp:"append"` 的事件数与
surface 投影统计精确相等（276）；`sourceEventSeqs` 248 条、其中区间编码 124 条。

## 写门：200ms 批窗与单写者

- `writers: Map<SessionId, JsonlSessionHandle|null>`（storage.ts:349）：
  `registerCreated`/`claimWrite` 先以 null 占位 → `adopt` 绑定句柄 → `release` 删除
  ——进程内"一会话一写者"的物理强制点。
- 路由安装（storage.ts:464-499）：`session/event` 按会话 id 查 writers →
  `enqueueLive`；`session/flush` = drainLive()+flush() 屏障；`session/disposed` = close() 终排空。
- `enqueueLive`（storage.ts:218-227）：`structuredClone` 隔离副本入 buffered；
  **无定时器且未暂停**才 `setTimeout(200ms)`——开窗由首事件决定，窗口内后续事件只入队、
  不重置截止。
- `drainBuffered`（storage.ts:231-252）：单飞（`draining` 合流）；链上 splice 整批
  `persistContiguous`；**失败回滚** = 批前置拼回 buffered + `drainPaused = true`——
  自动定时器静音，此后只有显式 `session/flush` 会再试并大声抛错。
- 突变串行 = `chain` promise 链：公开 append/flush 与内部 drain 同链排队；close 循环
  drain 直到"一轮排空后 buffered 仍为空"，再等链、释放所有权、聚合 drain 失败。
- 每帧落盘 = `open('a')` → 整帧 `writeFile` → `handle.sync()`；失败 →
  `rollbackAppend` 截回原 size + fsync（index.ts:857-866）——杜绝半帧留下的重复 seq。

## 原子实体化：link 而非 rename

1. `encodeMaterialization`：头部行**独立 1 帧** + 首批事件**独立 1 帧**，帧边界不与压缩合并。
2. 临时文件 `${final}.${hex12}.tmp`，`open('wx', 0o600)` + 写入 + **fsync**。
3. POSIX：root→project→dir 逐级 **0o700 + 先 fsync 父目录再建子目录** →
   `link(tmp, final)` → dir fsync → `rm(tmp)`。源码注释明示 link-not-rename 的理由：
   **两进程同 id 并发实体化不可互相覆盖**。
4. Windows：`ensureDurableDirectoryWin32` 以 mkdtemp 造暂存兄弟 `.dsh-mkdir-*` →
   `MoveFileExW(..., MOVEFILE_WRITE_THROUGH)` 逐级发布；`publishNewFileWin32` 不覆盖、
   不跨卷；`\\?\` 命名空间全程；koffi 懒加载 kernel32。
5. `rejectExistingLog`：目标已存在 = 大声拒绝（TOCTOU 兜底）。卫生实测：全库 124 个
   会话目录扫描，`*.tmp` / `*.lock` 残留 = **0** [MEASURED]。

## revision、稳定读与冷路径缓存

- fileRevision = `dev:ino:size:mtimeNs:ctimeNs`（bigint stat 五元组拼接）——
  **不是内容哈希**；契约口径"等值可当未变，不等值不承诺"；所有权换手不改 revision。
  未实体化会话用 `memory:<backend>:<counter>` 占位令牌。
- `readStableFile`（index.ts:522-552）：stat→readFile→stat；变了只重试**一次**；
  再变则按"本轮 pre-stat size"截断——append-only 之下该尺寸前缀就是已提交前缀。
- 实测旁证：首次解析时对活跃日志用朴素 `readFile`，恰在并发大帧追加中途取快照，
  自实现帧游走报 `bad magic`（内容撕裂）；换 stat-read-stat 协议后 6234 帧严丝合缝
  对齐 EOF。这就是该函数存在的理由。
- `coldLogMemo`：LRU 上限 **2**（index.ts:48）——"冷观测→紧随 resume"的交接复用窗口；
  键 = 会话 id，值带 revision 守卫；任何本地突变即失效该条。

## 崩溃恢复：撕裂尾按压缩块全有或全无

- 读路径（index.ts:555-625）：`scanZstdFrames`（含保留位/保留块型校验）逐帧解码完整
  帧；末帧结构不完整（tornStart）→ `decompressZstdPrefix`（`ZSTD_e_flush`）让扫描器
  继续吃前缀明文 → 产出 `tornTruncateTo` + `recoveredTail`。
- 不变量：**完整帧内出现撕裂 JSONL 行 = 直接判损坏**——帧是原子写单位，帧完整而行撕裂
  只可能是真损坏。
- 写侧首 mutation 消费（storage.ts:265-281）：先 `truncateTornTail`（truncate +
  fsync，warn "recovered from a torn tail"），再把 `recoveredTail` 作为普通批**重写
  持久**，然后才写新批；cursor 从未计入撕裂批 ⇒ **无双写**。

恢复能力实测定量（截断副本实验）[MEASURED]：

| 撕裂帧 | 截断点 | 恢复结果 |
|---|---|---|
| 1133B 单批帧 | 70% | 0 事件——整批（1 事件）丢 |
| 50136B / 2 压缩块帧（块边界 37279B） | 30% / 50% / 70% | 0 行 |
| 同上 | ≥85% | 93 行 / 263 事件平台期 = 第 1 块全量 |

明文块 = 131072B（zstd 默认 128KB），前缀恢复是**压缩块粒度**的全有或全无。本仓默认
帧尺寸（中位 229B）几乎总是单块 ⇒ **撕裂帧 ≈ 丢那一整批**；前缀恢复主要在超大批
（fork 种子、大 tool-result）时起作用。

- `interruptedTurnClosers`（repair.ts:29-138）：`pendingCalls` 按 Map 插入序关单；
  合成 result 的 id = `interrupted-tool-result-${callId}-${seq}`；已见 `tool/call` →
  `TOOL_OUTCOME_UNKNOWN` + "Do not retry blindly." 指引文案，未见 → `TOOL_NOT_STARTED`；
  `time` 复用最后真实事件时间戳（不造未来时间）；**结果先、step/end 中、turn/end 末**
  固定序。活体实测：对进行中的会话尾部真实合成出 `[step/end, turn/end{interrupted}]`；
  冷读者（cold-read.ts）对同一函数**只内存配平、不回写**。
- 三语义检查点（checkpoint-policy/src/index.ts:63-91，fail-closed）：`llm/stream`
  用可迭代包装把 `ctx.sessions.flush` 卡在**首个下游 chunk 之前**（抛错即不派发到
  模型）；`tools/execute` 仅顶层 checkpoint 通过后才放行副作用，嵌套工具复用外层持久
  call；`agent/pre-step` flush 上一步的一切。
- e2e 契约（tests/crash-recovery.e2e.ts）：SIGKILL 于 request-dispatched /
  tool-side-effect 两 failpoint，固化"请求前缀先持久""tool 意图先持久、结果补
  UNKNOWN"。该套件 `describe.skipIf(win32)`，Windows 原生不执行。

## 实测数字速览

数字出自真实落盘文件的 stat-read-stat 稳定快照（分析期间文件持续增长：5805→6234 帧）。

| 指标 | 数值 | 注 |
|---|---|---|
| packed 行 / 数据行 | **65.3%**（5182/7940） | [MEASURED，5912 帧稳定快照] |
| packed 行承载事件 / 全部事件 | **91.5%**（29808/32566） | 同上快照 |
| zstd 压缩比 | **≈2.0**（4.06MB 明文 → 2.03MB 物理） | 5912 帧快照；信封 JSON 主导体积 |
| 帧字节 min / 中位 / max | 87 / 229 / 16858 | [MEASURED] |
| 6234 帧稳定快照 | 存储行 8305 / 逻辑事件 34180 / seq 0..34179 全连续 | 官方解码 **0 失败** |
| `request/header` | 全程仅 1 行、35230B = 最大单行 | 125 步未再写纪元头 ⇒ 头按"渲染后 prompt 纪元变化"才追加 |
| 事件类型词汇表 | `KNOWN_SESSION_EVENT_TYPES` = 51 | 本 build 快照计数 [MEASURED] |
| 格式版本 | `SESSION_FORMAT_VERSION` = 0 | [MEASURED] |

类型分布（5912 帧快照）：未打包 `assistant/chunk` 行 1692（block-start/end、usage、
finish 不打包）；`step` 123×3、`tool` 123×2、`agent/inbox/spliced` 42、
`tool/code-dispatch(-start)` 173×2；`dt` 时延零占 61.8%、无负值。

## Java 移植观察

1. 持久层可在 JVM 上等价复刻且更顺：link-vs-rename、`MoveFileExW`、目录 fsync、
   `wx` 独占创建都有直接对应（`Files.createLink`、`FileChannel.force`、
   `CREATE_NEW`；`ATOMIC_MOVE` 恰是 DSH 刻意避开的语义）。zstd 帧语义与语言无关，
   但 **Node 默认 128KB 块边界正是实测恢复粒度的来源**——换 zstd 库（JNI/airlift）
   必须重测块大小与恢复平台期，同参数不可假设。
2. 200ms 批窗 + 失败回序缓冲 + `drainPaused` 静音是纯策略代码，无框架依赖；
   promise 链突变串行在 Java 对应单线程执行器 / 有锁队列，语义等价。
3. 撕裂恢复的契约本质 = "批为原子单位"不变量。移植可整删 zstd 而保留行协议测试，
   但 Scanner 的 issue 闩锁与 `committedBytes` 语义必须随读路径保留。
4. revision 用文件系统元组而非内容哈希：JVM `BasicFileAttributes` 同样能拿
   fileKey/size/mtime/ctime（ctime 语义跨平台不一致，但只用作缓存键，无害）。
5. `withFileLock`（`.lock` 兄弟文件 + `wx` 独占 + 指数退避 20→200ms、默认
   waitMs=2000）刻意留给 settings/credentials；会话侧跨进程写租约 = 已声明未实现
   ——Java 版若先做多进程会话共享，需要**发明**这一层而非照抄。
6. koffi FFI 对应 JEP 454 FFM API；但 `MoveFileExW` 的 WRITE_THROUGH 语义在
   `java.nio.file.Files` 无直接开关，需 FileChannel 层配 FlushFileBuffers，或接受弱化并记录差异。
7. 崩溃修复的归属原则原样继承：持久层只给物理有效前缀；语义配平（合成 closer）归
   agent 层、且**只在写所有权下回写**，冷读者只内存配平——两调用点共用同一纯函数，天然可移植。

## 诚实边界

- 待核实：`SessionPreparation` 与 `appendUnstoredSuffix`（恢复后再排干未持久种子尾）
  全文未读，仅在 resume 走读中见调用式。
- 待核实：session-projection-cache 落盘阶梯与 telemetry `session-record` 持久内容未读（超接缝）。
- 待核实：crash-recovery e2e 未在本机执行（`describe.skipIf(win32)`）；"多进程并发
  实体化 link 竞争"仅代码证据，无实测。
- 待核实：帧断言基于本仓样本 + 自实现游走与官方实现逐位一致比对；非默认压缩级别 /
  字典压缩未测兼容（代码路径不支持字典）。
- 待核实：实测数字与活跃文件赛跑——字节/事件数为快照值，复跑会漂移；比例型数字
  （压缩比 ≈2.0、packed 占比、块恢复平台期）预期同类负载下稳定 [ESTIMATED]。
- 待核实：解析内存方法为 ≤2.1MB 快照全量入内存；>100MB 超大会话的压力观察未做。

## 相关

- 概览篇（本篇是其深读展开）：[会话持久化与存储](../platform/session-persistence.md)
- 事件本体与投影消费：[会话事件日志](../agent-runtime/session-event-log.md)
- 写所有权在循环里的位置：[turn/step 主循环](../agent-runtime/turn-step-loop.md)
- 文件系统执行世界的姊妹深读：[沙箱执行](./sandbox-execution.md)
- 子会话头部继承与 fork 种子大批：[委派与编排](../augmentation/subagent-orchestration.md)
- 本篇 Java 观察的总映射：[Java 移植总图](../java-porting-map.md)
