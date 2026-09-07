---
title: "派生侧深读：投影、标题、遥测与格式代次"
tags: [dsh, projection, telemetry, session-title, format-generation]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/session/* 源码直读 + docs/subsystems/{session-projection,session-telemetry,session-title}.zh.md
  + docs/persistence-catalog.zh.md + 真机实测（盘上多代次日志只读解码、projcache 文档剖析）"
updated: 2026-09-05
---

# 派生侧深读：投影、标题、遥测与格式代次

> [持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)讲的是日志的**写侧**（帧、写门、
> 撕裂尾）；本篇讲**派生侧**：从"唯一真源日志"往外的每一条产出通道——投影 seam 与它的
> 持久缓存、标题/统计/大纲三个读侧单元、遥测脱敏 waterfall、增量上报，以及把这一切
> 身份都绑上去的**格式版本代次机制**。证据基准 = 源码直读 + 本机 harness home 只读实测。

## 概览篇说法 → 深读结论

| 概览篇/姊妹篇说法 | 深读结论 |
|---|---|
| "跨进程租约是已记录的限制，不是缺失" | **已过时**：本 build 已实现（lease.ts，POSIX flock + Win32 命名信号量），见「写租约」节 |
| "跨进程租约已声明未实现"（姊妹篇） | 修正：租约本 build 落地——`SessionAlreadyOwnedError` 有了真实抛点（lease.ts）；唯 `SessionOwnershipLostError` 仍无抛点：**设计上无到期，故无"失权"** |
| `SESSION_FORMAT_VERSION = 0`（姊妹篇快照） | 现为 **2** [MEASURED]（core/session/src/types.ts:86）；v0/v1/v2 三代并存于盘 |
| `assistant/chunk` 是核心事件（主干篇） | **代次敏感**：v2 持久词表顶层无 chunk 行（released 目录删籍），流式分块折进 `assistant/attempt`/`message` 的内嵌 stream（实测活跃 v2 日志 chunk 行 = 0 [MEASURED]） |
| 投影"从日志投影派生"一句带过 | 实为一整套读阶梯 + 持久缓存 + 身份守卫；本主题 18 个包中的 4 个只为这条链路服务 |

## 投影 seam：纯函数单元 + 急切驱动

`ctx.sessionProjections`（session-projection/src/index.ts，677 行）是"框架负责
驱动、领域负责计算"的三方分离：领域贡献一个 `ProjectionDefinition`——`init/apply/view`
三个**纯同步**函数 + `stateSchema/viewSchema` 两道校验 + `stateVersion` 失效版本号。
红线全部写在类型注释里：异步单元会撕破载体的一致切面；state 必须是 plain JSON
（持久缓存的前提）；对不关心的事件 `apply` **必须返回同一引用**——`Object.is` 就是
变更判据，引用不变 = 零下游工作。

驱动订阅 `session/event` 一次，把每个已提交事件即时推给每个单元（eager）。每会话
每单元一个水位 cell（state + observedSeq + 最近两个原始 view 的双槽缓冲）：state 变才算
view、view 再变才发变更流——对象型 view 想抑制"仅内部 state 变化"的发布就得复用引用。
注册是随 fiber 走的 effect：域插件卸载后其 key 从快照消失，客户端读作**能力缺失**；同
key 多注册方（同一工具包挂进 N 个 preset）计数共享，最后一个 disposer 才删键。**全量值
事件规则**是承重结构：携带状态的日志事件必须携带变更后的完整状态、绝不含裸增量——
每次转移因此恒廉价、每个被供给的值自描述（对消费方即 last-wins）。

## 读阶梯：从"绝不回头"到"零 I/O"

同一个配方（缓存 state + 前向尾带 replay + view）按 I/O 递减排成阶梯：

- `snapshot(session)`：**全同步一致切面**，所有值共享同一 `asOfSeq` 水位；载体在切
  页面切片的同一 tick 读它，两次读取因此能用同一个序号。
- `stateOf/cachedSnapshot`：只读已物化的活 cell，落后于实时——是 hint 不是基线。
- `viewCheckpoint`：**零 I/O 档**，直接 view 持久行中 ver 匹配者；"与最后一次持久
  checkpoint 一样旧，但永不会错"。
- `restore`：冷读配方。取 `ver` 匹配、且 `baseSeq-1 <= seq <= endSeq` 的行作种子，
  从种子水位继续 fold 尾部事件；不可用的行丢弃并从 `init` 重折。
- `coldSnapshot/hydratePrepared`（缓存侧）：完整日志冷读 + 回写刷新（fail-soft、
  fire-and-forget），首读建行、后读吃行。

`restoreFloor` 的**锚下移一格**设计是全局最精妙处：尾读起点取"最低可用水位减一"，
于是尾读本身证明了日志延伸到哪——若崩溃修复把日志截到某行水位之下（shrunk log），
空尾读让 end 低于所有水位，restore 拒绝并强制全量重读；`baseSeq > 0` 时丢弃行也直接
throw（从 seq 0 重折才 sound）。缓存行**永远只是折叠捷径，永不是权威**。

## 持久投影缓存：写序即正确性

`session_projcache` 域实测规格 [MEASURED]（spec.ts）：`version: 7`、
`compatibleVersions: [3,4,5,6]`、`layout: 'per-record'`（每会话一个 JSON 文档，
写一行不换全表）、`invalidRecords: 'backup-and-skip'`（坏记录挪成 `.bak.<stamp>` 继续
开机——派生数据不配炸启动）。per-record 的版本作用域让"某会话文档过期"只报废那一个
会话的捷径。写触发 = 三个必写点（会话创建、`turn/end`、dispose 的 live-to-cold 时刻）
+ 节流双通道，出厂配置 `writeEveryEvents: 200` / `writeIntervalMs: 5000` [MEASURED]
（bundle/base/cordis.patch.yml:165-166）。

- **durability barrier 写序**：`write()` 先取 registry 切面、**先 flush 会话日志**、
  再落缓存行——崩溃只能让缓存落后于日志（尾带重放稍长），绝不领先（凭空造出日志里
  没有的值的折叠）。
- **身份守卫**：每条记录绑 `{formatVersion, createdAt, cwd?, isSeeded, inheritedEventCount}`。
  "会话 id 命名的是槽位，不是生命周期"——删后重建的同 id、或 persistence root 被整体
  换掉，旧记录过不了身份匹配，绝无可能给无关日志播种。`formatVersion` 缺失的旧记录
  **永不能当折叠捷径**（证不了行语义）；但生命周期匹配的前代记录可以经
  `cachedPredecessorTitle` 只透出标题一个 key（listing 提示、`asOfSeq: -1` 哨兵值，
  刻意不复用可能被基数变更重映射的序号）——因为格式归一化会改变其他 key 的当前含义。
- 实测本机：缓存文档 183 个 > 盘上日志目录 141 个 [MEASURED]——存在随日志消失而遗留的
  孤儿记录（未见清理路径），而身份守卫正是它们绝不污染新生命周期的防线；
  本会话记录 21 个 key 同以 `seq=830` 一个切面落盘、identity.formatVersion=2 [MEASURED]。

## 读侧单元巡礼：title / sessionStats / turnOutline

**title（stateVersion 1，host+wire）**：wire 值就是纯字符串或 null——列表行的形状。
输入侧另注册 host-only 的 `titleInput`（v3）：只折 `{first, count, lastSeq}` 有界聚合，
O(1) 状态；全文消息列表在真要跑 provider 时才从日志现扫。

**sessionStats（v1）**：计步权威是 `step/end` 而非 `assistant/message`——loop 在 finally
里对每个进入过的 step 恰记一条，取消/失败/max-tokens 全有；数消息会漏算取消步、
多算 usage 宿主空消息。墙钟四件套（模型时长/首 token/解码/tool 配对）逐字段镜像
客户端窗口折叠，让"无此单元时整窗回退"成立。两处防污染细节：`tool/result` 用
`Object.hasOwn` 查 pendingCalls——callId 是模型 JSON 边界进来的，`"constructor"` 这种
键必须读作不匹配而不是继承函数（否则 toolMs 中毒 NaN）；`turn/end` 清空未配对残留，
防持久 state 无界生长。

**turnOutline（v2）**：锚在 `turn/start` 的 seq 而非提示消息——它是**回页目标**：把窗口
翻到该 seq 必含整轮。预览预算按栏卡字号量出（提示 50 字符/响应 120 字符 [MEASURED]，
投影注释含 13px/276px 推导）；逐块截断防单块多 MB 文本为一条预览做全量正则归一。
响应走 draft 缓冲、`turn/end` 才提交，且 draft-only 的 apply 保持 turns 数组引用不变——
变更流每轮最多推 3 次（边界/提示/响应）。

实测本会话盘上 21 个投影单元同表 [MEASURED]：除本篇四键外还有 llmRetry、goal、
tokenUsage、contextPressure、contextBreakdown、turnBoundary、plan、todos 等——它们
散布在其他包组、共吃同一条 seam；`title` 服务自己就靠 `stateOf(session, 'turnBoundary')` 读步边界。

## 标题服务：钉住、修订与等路由

持久事实只有一种：`session/title` 事件（latest-wins，log-only——永不进模型面）。
`source` 三态里 `'user'`（显式改名）**钉住**：在途自动生成立即被 supersede、后续
用户消息不再排任何程；解钉只有显式 `refresh`（它甚至会在没有 provider 时把 fallback
重新盖过钉住的用户标题——"unpin 即覆写"是故意的）。确定性 fallback（首条合格消息
前 5 词/40 字节，接受上限 80 字节 [MEASURED] base yml:51-53）同步派生自日志，所以
**没有 provider 的部署也有标题**。

自动调度的克制点在**等路由**：pending 工作不立刻跑，直到 `request/header` 把本轮主请求
路由记进日志（或 `llm/stream` 拦截点核对"路由未变 + 步边界已过提示 seq"），provider
因此总能拿到与主请求一致的 route 出处；`revision` 计数器 + AbortController 保证任何完成
落盘前四事自证（服务活、注册方对、会话对、修订号对）。provider 结果被服务验收：
规范化、≥1 个源消息 seq、seq 必须来自本次快照且严格保序。归一化是一整套终端安全
工程：OSC/CSI/ESC 转义、C0/C1 控制符、**双向算法不可见控制符**（防标题显示欺骗）
逐类剥除 + UTF-8 码点边界截断 [MEASURED]（normalize.ts 全文）。不变式伴侣守持久等式：
`messageSeqs` 为空 ⟺ `source.kind==='user'`，且每个引用 seq 必须指向更早的真人消息。
LLM 辅助（title-llm 278 行 + 两个 30 行薄壳）在调用**前**把完整可见请求（system/
messages/route/maxTokens）记成 `session/title-llm-request`——生成失败后载荷仍复现模型
输入；用户文本以 JSON 数组成框，防结构分隔符被戳穿。

## 遥测：账本镜像、脱敏 waterfall 与 SDK 边界

seam 契约把职责线画在 `emit()`：harness 负责**完整权威捕获**，批处理/重试/排队/丢失
全归上报 SDK（OTel 后端原样组装 LoggerProvider + BatchLogRecordProcessor + OTLP/HTTP
exporter，两个 SDK 选项对象原封透传——"重挑字段会静默丢掉其余"）。记录两通道：
ledger 一比一镜像日志事件（含插件合并进来的、seam 从未听说过的类型，fall-through 为
info 而非 assertNever）；ops 只装两个无日志之家的信号（`agent-error`、`shutdown`）并
刻意省略 seq 标识——它们是告警信号不是累加条目，重复被容忍。接收端按
`(session.id, format_version, event.seq)` 去重。

handoff 游标是**模块级 WeakMap**——源码自注的"对 registrations-are-effects 纪律的
狭窄例外"：cordis 无 HMR 状态移交 API，按 Session 对象键控是唯一能让热重载后重新收养
的 fiber 从游标续传、而非重交历史的存活周期。游标语义是"已交接"非"已送达"。脱敏 =
`session-telemetry/record` waterfall，**自带零规则**：导出数据恰好干净到部署挂了
什么规则的程度；监听器抛错 = 扣留该条（fail-closed），永不外溢进循环。

OTel 后端三态模式里 `FEEDBACK_ONLY`（出厂默认 [MEASURED] base yml:192）最讲究：
`feedback/record` 事件到达时才整话回补，且先做**正典复核**——`session.eventAt(seq)` 必须
=== 该事件（"同意是被提交的记录，不是可独立发射的总线值"），否则拒绝。`flush` 提示
**故意不实现**：转发给 forceFlush 会制造与 shutdown 内部 drain 的并发未定义交互而
静默丢尾。外层 3s shutdown 截止也是故意的——SDK 的 export timeout 不包 `forceFlush()` 的
等待，socket 从未拿到的传输可以永远悬着。用户身份 = harness home 下的匿名随机 UUID，
挂在 Resource 而非逐条记录上（collector 按 Resource 聚合）。

## session-log-deepseek：日志即收据

与 telemetry 平行的另一条出口：官方 API 请求字段 `dsh_session_log` 的增量贡献
（默认关 [MEASURED] `enabled: false`）。设计点只有一个但很响：**送达水位写回日志本身**
——`session-log-deepseek/delivery-accepted` 事件。重启恢复靠保守重发不确定尾带即可，
无需另一个 store；`acceptedThrough` 折带扫（WeakMap 记住扫描位）。折叠按
`sessionFormatVersion` 匹配代次（缺省=0），跨代次收据不误吃。不变式伴侣守"水位必须
早于自身 seq、非继承事件必须点名所在会话"。实测本机（非官方 API 路由）该事件
计数 = 0 [MEASURED]，与默认关一致。

## 格式版本代次：不可变发布 + 相邻迁移链

`session-format` 五件套把"格式"升格为一等机制：

- **命名即代次**：v0 保留原 `session.jsonl(.zstd)`；此后每代 `session.vN.jsonl`。正则只认
  小写、无前导零、无 `.v0`、非临时名——**文件名不可信，头内 version 必须一致**
  （`ensureCurrent` 首查两值不符即 throw）。
- **目录内仲裁**：取数值最高的规范代次为 current；同目录混进另一压缩后缀 = 硬错误；
  旧平铺布局 = 硬错误。较旧的可读代次**原地保留不动**。
- **构建期静态目录**（session-format-catalog/generated.ts）：currentVersion=2 + 每版本
  一个冻结 codec + 完整相邻链，编译期即验"无重复、无缺口、无指向未来"。
- **相邻迁移边**：v0→v1 是**身份/归一边**——51 个 released-v0 事件类型逐个冻结成员
  清单 [MEASURED]；退役类型（`request/header-delta`、`mode/set`）直接拒迁；
  `steering/message`→`user/message`、裸 content 包成带 id/role 的 message、
  `legacy-message:<id>:<seq>` 合成缺省消息 id、turn reason 规范化。v1→v2 是**基数变更边**：
  把 `assistant/chunk` 事件族按 turn:step 归组成 attempt、流内嵌进
  `assistant/attempt` 与 `assistant/message`，seq 因此压缩——重映射表贯穿
  `sourceEventSeqs`/`surfaceOp.start/end`/`compaction/*` 影子区间/`session/title` 的 messageSeqs；
  inherited 切点若劈开一个 attempt 则整会话拒绝迁移；fork 边界物化为
  `session/end-seed` 的 `data{inherited:true}`（缺标记者合成补全）。下游插件合并的
  当前版本事件**不属于目录**，未来迁移边须显式登记 disposition。

**独占发布**（generation.ts 724 行）：迁移读源文件稳定快照 → 内存 migrate → 编码为
恰好 2 帧（头帧独立 + 单主体帧）→ 写 `session.migration.<hex>.tmp`（wx 0o600 + fsync）→
复核暂存件 → **重读源指纹**（stat 五元组 + sha256），变了就整轮 continue →
`link()`（Windows `MoveFileExW` 不覆盖）独占发布 → reopen 验证（防符号链接/非常规
文件/大小写非规范名/目标字节不以前缀开头）→ 清暂存。读侧 `requireStoredLog` 对 current
代次复用姊妹篇讲过的 coldLogMemo；`ensureJsonlGenerationCurrent` 对 current 输入
**零写入零迁移回调**。方向感知拒绝保持姊妹篇描述的措辞，另加
`JsonlGenerationNewerVersionError`：最高代次是未来版本时，即使旁边躺着可读旧代也拒绝。

实测同会话两代并置 [MEASURED]（本机盘上、只读解码）：

| 观察项 | v0 冻结源 | v2 迁移产物 |
|---|---|---|
| 行数 | 6254 | 1822（−70.9%） |
| 顶层 chunk 行 / 打包 chunk 行 | 1736 / 2700 | **0 / 0**（折入内嵌 stream） |
| zstd 帧数 | 2538 | 2 |
| 明文 | 6.75 MB | 6.33 MB |
| `session/title` / `title-llm-request` / `end-seed` | 4 / 1 / 2 | 4 / 1 / 2（标题事实无损过迁移） |

## 写租约：内核当仲裁

姊妹篇"已声明未实现"的跨进程租约在本 build 落地（lease.ts + win32.ts）：POSIX 对
`session.lock` 非阻塞 `flock(2)`，**锁后复核 inode**（锁的是 inode 不是路径；被删后
重建的锁文件携带新 inode，孤儿 inode 上的锁证明不了任何事；≤3 次重试）；Win32 更
彻底——`Local\\dsh-session-lock-<sha256(小写绝对路径)>` 命名信号量，**不碰文件系统**，
读者/搜索/目录删除全程无感，零超时 wait 撞 `EBUSY` → `SessionAlreadyOwnedError`。
两平台共同点：内核在描述符/最后句柄关闭时放锁（**含进程暴毙**），崩溃持有者永不
阻塞继任者；而活着的卡死持有者**保锁到进程退出——刻意无到期**，任何到期机制都可能
没收一个停摆写者、其复活追加会撕日志。获取时机 = 对既有产物的写开，或新建会话的
**首次实体化写**（未实体化 = 无盘上足迹）；POSIX 释放**不删**锁文件（保留稳定 inode
供后来者复核）；浏览器 worker 把 fs-ext 打桩为立即成功（单进程，进程内写声明已足够）。
实测本机 141 个会话目录 `session.lock` 文件 = 0 [MEASURED]——Windows 分支无锁文件的活证。

## 诚实边界

- 待核实：`assistant/attempt` 行在**活写入**侧的精确产生时机（实测活跃会话
  attempt=14 对 step/end=67，比例未解释；迁移侧语义已读透，写侧归 core/agent-loop）。
- 待核实：帧游走为手写（Node `zstdDecompressSync` 只吃单帧、流式解码停在首帧——
  这本身印证了"帧是独立解码单位"）；块边界解析未校验块类型 3（保留值）。
- 待核实：迁移中途 SIGKILL 的 e2e 未跑；`generation.spec.ts` 962 行的 barrier 注入
  测试仅通读断言目标。
- 待核实：session-query 包组的观测路径只见一处 `hydratePrepared` 调用点（observation.ts），
  未展开——它是派生侧的另一个消费者，归后续批次。
- 快照口径：141/120/23/2/183 等为本机盘上瞬时计数，复跑会漂 [MEASURED]。
- 未读：session-persistence-jsonl/index.ts 中与代次仲裁无关的句柄细节（姊妹篇已覆盖
  其写路径与恢复；本篇只取代次接点）。

## 相关

- 写侧姊妹篇（帧格式在彼处，本篇不重复）：[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)
- 概览篇（投影/租约说法的出处）：[会话持久化与存储](../platform/session-persistence.zh.md)
- 事件本体（v2 词表修正的落点）：[会话事件日志](../agent-runtime/session-event-log.zh.md)
- 投影消费者（历史尾页/推送帧）：[host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)
- 反馈事件（FEEDBACK_ONLY 的触发源）：[人机问答与反馈](../augmentation/questions-and-answers.zh.md)
- 配置面（patch 层写触发/模式值的读法）：[配置面](../augmentation/settings-and-credentials.zh.md)
