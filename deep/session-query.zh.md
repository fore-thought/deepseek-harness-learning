---
title: "会话查询内幕：逻辑语料、FTS5 派生索引与授权工具面"
tags: [dsh, session-query, fts5, sqlite, lineage, full-text-search]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/session-query/{session-query,session-query-sqlite,tool-session-query,session-log-export}/src 直读（5,888 行）+ 四包 tests 契约标题 + docs/subsystems/session-query.zh.md + bundle/{base,web-app} cordis.patch.yml + 本机 node:sqlite FTS5 分词实测"
updated: 2026-09-05
---

# 会话查询内幕：逻辑语料、FTS5 派生索引与授权工具面

[English](session-query.md) | [中文](session-query.zh.md)

> 会话事件日志是唯一真源（[会话事件日志](../agent-runtime/session-event-log.zh.md)）；本篇讲**怎么把它读回来**：
> `packages/session-query/` 四包 = 查询接缝（session-query）、SQLite FTS5 提供方（session-query-sqlite）、
> 模型工具面（tool-session-query）、ZIP 导出（session-log-export）。投影/标题/遥测的供给侧内部机制
> 归 [投影与遥测内幕](./session-projection-telemetry.zh.md)，本篇只写到查询侧怎么消费它们。

## 读者直觉 → 深读结论

| 直觉 | 深读结论 |
|---|---|
| 搜索索引是常驻加速层 | **发布组合默认全关**：`bundle/base` 与 `bundle/web-app` 两层 patch 都写 `path: ':memory:'` + `openAt: never`；`ctx.sessionQuery` 仍挂载（精确读/标题/谱系可用），全文调用抛 `SESSION_QUERY_SEARCH_DISABLED` 且 `node:sqlite` 永不 import |
| 中文全文检索开箱好用 | `unicode61` **不分词**：连续中文串 = 单一 token。实测查「压缩」0 命中、查整串「热重启与压缩算法设计」1 命中 [MEASURED]；中文子串检索须走 `filterEvents` 的 `text` 字面扫描 |
| 搜索结果有相关度分数 | 无浮点分数。确定性排名键：命中数↓、文档长度↑、时间↓、（会话内）seq↓ /（跨会话）session_id↑——浏览器 fixture 镜像同一组键，源码注释要求两处同改 |
| 游标 = 偏移量 | Base64url 包裹 `{version, instance(UUID), scope, 请求指纹, generation, offset}` 六元组校验；**相关语料一变即 `STALE_CURSOR`**，语义是「重发整个搜索」而非静默移页 |
| 索引靠事件实时维护 | **惰性拉平**：每次检索在串行队列里先跑一遍完整 reconciliation；live 会话索引放 TEMP 表（连接私有，随重开蒸发），持久索引放主库 |
| 导出 = 拷原始日志文件 | 导出**重序列化**：逐逻辑事件写 canonical JSONL，与磁盘上的 zstd 帧/chunk 打包形态无关——任何持久化后端导出结果同构 |
| 模型能搜到同机其他工作区 | 硬边界 = `header.cwd` 全等 + 授权前后**双校验**（观测 header 复查防同 id 换主）；越界谱系只渲染边界标记，不泄露隐藏 id |

## 接缝形状：一个服务键、两层实现

`SessionQueryEngine`（Service 基类，键 `ctx.sessionQuery`，`inject = ['sessions']`）把
**精确读/过滤/追踪做成基类具体方法**，只把 `searchSessions`/`searchEvents` 两个抽象方法留给后端。
妙处在 `openAt: never` 下其余能力照常工作：装配的就是 SQLite 子类，但 never 只拒这两个抽象
入口，其余全靠基类里不依赖后端的实现。错误面是封闭 17 码的 `SessionQueryError`
（config.ts:29-47，继承 `HarnessError`）——码即路由协议。

## 逻辑语料：live-preferred 的三层语义

`SessionCorpus`（corpus.ts）把「同一个会话」的内存态与磁盘态折成一份视图：

- `listSessions`：先铺持久化 listing，再让 `ctx.sessions.list()` 的 live 会话**覆盖同 id 记录**；
  两侧 header 六字段（id/createdAt/cwd/parentSession/isSeeded/delegationDepth）不一致 →
  `SESSION_QUERY_SOURCE_CONFLICT`（sources.ts:11-25）。排序确定性：createdAt 降序、id 升序裁决。
- `load`：**已知 live 目标永不触碰持久化**——可选后端坏了也不能让内存里的事实不可读
  （corpus.ts:91-120 注释明示）；冷读期间会话 attach → 直接改读 live。
- `projectMany`（批量标题等）：按首次出现序去重；4 路 worker 从游标自取；
  投影器**同步**执行且逐条借用事件数组（批量绝不保留整日志）；单会话失败隔离成
  `rejected` 结果，取消才整体拒绝。
- 冷读底座 `readColdSessionLog`（cold-read.ts）：read handle 读完整校验日志 +
  `interruptedTurnClosers` **只内存配平、不回写**——与
  [持久化深读](./persistence-crash-recovery.zh.md)「冷读者」一节同文互证。

## 观测租约与 prepared 缓存

`observeSession` 返回 Disposable 的 `SessionObservation`（observation.ts）：一份不可变
会话切面 + `retain()` 引用计数租约。冷路径用 `ctx.sessions.prepare`（**不 publish**、不进
store）恢复会话，缓存键 =（persistence 实例身份, stat revision）——revision 令牌即持久化篇
的 stat 五元组；实例换代 = 缓存作废（不同实例的 revision 不可比）。LRU 上限默认 **5**，
被租约 pin 住的条目不受容量驱逐、释放时补扫（evict sweep）——8 个用例钉死这套 pin/驱逐语义。
`projectionMode: all/none` 决定是否顺带水合投影（registry 直hydrate 或 projection-cache
两路皆备）；投影侧机制见 [投影与遥测内幕](./session-projection-telemetry.zh.md)。
接缝之外的消费方：subagent 的 list-children/continuation（`observeSession`）、
session-reference（`listSessions` + `readSurface`，归
[提示词组装深读](./prompt-assembly-context.zh.md) 的注入线）。

## 三态表层与关系追踪

`analyzeEventLog` 对整条日志跑**一次** `foldSurface`——与模型历史推导同一状态机，
每个事件落入 `current | shadowed | log-only` 三态；这就是「模型可见即已记录」的查询侧推论：
surface 不是查询自己发明的标签，而是重放同一个折叠。折叠失败 = `INVALID_SURFACE`。

- `traceEvent` 五元组：`replacedBy`（位置直接后继替换者）、`replacementChain`
  （沿直接替换者走到链尾）、`replacedEventSeqs`（目标自己遮蔽的节点）、
  `sourceEventSeqs`（目标引用的因果回指）、`derivedEventSeqs`（正向扫描：谁引用了目标）。
  目标存在性检查**先于**表层校验（tracing.spec:408 钉住）。
- `traceSession` 谱系：ancestors 线性外推（成环 → `INVALID_LINEAGE`；父链走出可见语料 →
  `complete: false` + `unresolvedParentId`，与 root **互斥可判**）；descendants 森林用显式栈
  迭代构建（深嵌套不吃 JS 调用栈），兄弟序 createdAt 升 / id 升。

## 语义文本：什么可搜，什么永不可搜

`extractSessionEventText`（extraction.ts）是唯一的首方语义文本口径，同时喂 `filterEvents`
扫描与 FTS 索引。收录：用户/助手消息正文、工具名+参数、工具结果正文+错误 name/code、
`todo/write` 状态+内容、`turn/end` 的 error 消息与 aborted/max-tokens/interrupted
（completed 无文本）。**拒收**：reasoning 块、`request/header`、step 结构事件、流分片、
一切未知声明合并事件——新事件类型不因为 payload 恰好含字符串就自动可搜，语义须由属主
显式登记。测试钉住「排除私有链标记、保留可见回答」。`text` 过滤器
（`compileSessionTextFilter`）= 字面量正则转义 + 空白弹性 + Unicode 大小写不敏感，
注入安全；它与提供方无关，SQLite 缺席时照样工作。

## SQLite 派生索引：所有权与重置纪律

schema.ts 的开局三拒：application_id 非 `0x44534851`（ASCII 'DSHQ'）拒、无标记但非空文件拒、
出现未知用户表拒——**绝不 reset 别人的库**；只有「认识的派生索引 + 版本不符（现 v8）」
才原地 DROP 重建。缺路径以 0700 目录 + 0600 文件 owner-only 创建，已有文件保留模式。
双层布局：主库 `persisted_sessions` + `persisted_docs`（FTS5，可丢弃、跨重启存活）；
TEMP 库 `live_sessions` + `live_docs`（连接私有 = 天然进程隔离，重开即散）。
FTS 表除 `text` 外全列 UNINDEXED，`seed_length` 忠实携带继承切面数。
`journalMode` 是封闭联合、直接内插 PRAGMA（不绑参）——「验证在前、突变在后」的
注释纪律把 SQL 注入面钉死在闭集上。

## 拉平：串行队列、稳定观测、双指纹

每次检索在 `_serialized` promise 链上独占执行（精确读不受此锁），入口先跑 `_reconcile`：
diff 索引行 vs 当前观测 → 只读**新增/变更**的冷日志 → 单个 `BEGIN IMMEDIATE` 事务提交；
失败全回滚、下次检索重试。变更检测两套指纹：持久侧比对 stat revision；live 比对
sha256(header+inheritedEventCount+events) 的 base64url 指纹。观测本身还须**稳定**：
前 list、读日志、后 list，两次快照或 live 成员集不同 = 整轮重来，只给 **2** 次机会
（`STABLE_OBSERVATION_ATTEMPTS`）——反复抖动宁可失败，不把半个世界写进索引。
生成号三条线：global（持久写批次，落 `search_state`）、local（每条 live 变更递增）、
persistenceEpoch（后端换绑）——游标按 scope 绑对应那条。

## 查询执行：排名、片段、游标

候选 CTE 用 `UNION ALL` 合并两表，持久分支带 `NOT EXISTS(temp.live_sessions)`——
有 live 所有者的会话只从 TEMP 侧出，不双计。`match_count` =（带标记文本字节长 − 去掉
起始标记后的字节长）/ 标记字节长，即高亮区间数。片段以非字符码位 U+FDD0/U+FDD1 为
哨兵标记（`sanitizeFtsText` 预先把文档与查询里的这两个码位及 NUL 折成 U+FFFD，杜绝碰撞），
按 Unicode 码点截 **240**、命中点置于约 1/3 处、补省略号。预算三道闸：外层谓词 ≤ **14**、
绑定参数 ≤ **32,766**（SQLite portable 上限）、页大小 ≤ **100**（默认 20）——全部以
`INVALID_FILTER`/`INVALID_LIMIT` 快速失败在语句准备之前。游标解码六项校验
（version/instance/scope/指纹/offset/generation）；会话内游标只被**目标会话**的变更作废，
跨会话游标被任何拓扑变更作废（sqlite.spec:563 钉住）。

## 模型工具面：五个只读工具与授权边界

`tool-session-query` 注册 `session_search` / `session_event_search` / `session_trace` /
`session_event_trace` / `session_event_read` 五个只读工具（后三个 `isConcurrencySafe`），
一段 systemPrompt 引导「搜索→追踪→精读」的用法链。授权模型：

- 调用者 = 绑定 agent 的会话；跨会话检索要求 `header.cwd` 存在且**全等**，预授权用
  `filterSessions(id+cwd)`，拿到结果后再对**观测 header** 复查
  （`assertObservedTargetAuthorized`）——防「预授权后会话被同 id 换主」；
  `session_search` 连调用者自己的会话都排除（自己的上下文本来看得见）。
- 搜本会话历史时，seq 上界裁到 `turnBoundary` 投影的 `lastStepStartSeq - 1`：
  **看不到正在执行本调用的这个 step**（防自指环）；无 step 边界 = 类型化失败；
  用户给的区间整个落进当前 step → 不调 FTS 直接空页。
- 谱系渲染脱敏：越界祖先收成一行边界标记、被剪子树留 `null` 占位渲染成
  `[outside workspace subtree]`、未解析父链不泄露 id（tool spec 799/820/1107 三连钉）。
- `service-boundary` 把 17 个服务码翻译成模型安全文案；`INVALID_CONFIG` 与
  `SOURCE_CONFLICT` 降级为泛化 `TOOL_FAILED`——运维性失败不向模型解释自己；
  完整 cause 链（含循环防护）进 logger.warn。内部翻页对模型隐形：`collectPages`
  抽干 provider 分页凑满授权后 ≤100 条，**只剩被拒命中时不报 capped**；
  提供方重复游标 = 直接 `INVALID_CURSOR` 拒绝而非死循环。
- 标题富化失败隔离：单条标题读炸只标 `untitled (title unavailable: CODE)`，不毁一次搜索。

## 导出：/export、HEAD 预检与流式 ZIP

`session-log-export` = Web 专属 `/export` 命令 + GET/HEAD `/api/session.export` 路由
+ 浏览器下载对话框（`command/executed` 事件触发，同一 controller）。响应前
`flushLiveSessionLog` 先过 store 的 flush 屏障再开读——「append 尽力而为、flush 才是
屏障」契约的直接消费。归档成员：根日志 `session.v2.jsonl`（v0 名 `session.jsonl`，
代次命名由 `sessionFormatLogFilename` 裁决，与持久化篇的 `session.v2.jsonl.zstd`
落盘更名同一条格式代次线）、`subagents/<安全id>/…` 按谱系序、
`media/<attachmentId>.<ext>` 内容寻址（共享图只存一份）、`files/<2位>/<digest>/<名>`；
无 manifest。流式纪律：fflate Zip + 64KiB 响应高水位门（生产端等消费者 pull）、
65,536 单元分块、**代理对跨界回退一码位**（防 astral 字符被 U+FFFD 悄悄吃掉，archive
spec 483 钉住）、中途任何成员失败 = 整个流 error（宁失败不静默少导）、500 不回显后端
错误（防绝对路径进浏览器错误条）、HEAD 预检走同一条准备链但不产 body。

## 实测数字速览

| 指标 | 数值 | 注 |
|---|---|---|
| 四包源码行 | **5,888**（核 2,000 / SQLite 1,663 / 工具 1,306 / 导出 919） | [MEASURED] 逐文件行数合计 |
| 四包测试行 | **7,950**（≈源码 1.35 倍） | [MEASURED] 测试即合同 |
| 错误码 | 服务层 17 + 工具层 4 | config.ts 封闭联合 [MEASURED] |
| 分词实测（unicode61） | fts5vocab 直读词表：连续中文整串成**单一 token**（`帧撕裂尾`、`热重启与压缩算法设计` 各一条）；「压缩」0 命中；`AI` 不中 `BRAID` | node:sqlite v24.20.0 [MEASURED] |
| 默认参数 | 窗口 50 / 并发 4 / 冷缓存 5 / 页 20-100 / 片段 240 码点 / 谓词 14 / 绑定 32,766 / 稳定观测 2 次 / 工具 cap 100 / 超时 30s / DEFLATE 6 / 高水位 64KiB | 源码常量 [MEASURED] |

## 诚实边界

- 待核实：分词实测是同款 FTS5 定义的独立 `node:sqlite` 复刻，非经 DSH 引擎端到端；
  SQLite 内嵌版本随 Node 发行漂移，中文 token 行为应在目标 Node 版本复测（方向可信）。
- 待核实：四包测试套件未在本机执行（源码走读 + 用例标题/关键节取证；sqlite.spec 57 例、
  tool spec 69 例是行为合同的主要依据）。
- 待核实：`launcherSessionQueryPath` 声明了启动槽位契约，但当前 checkout 的
  packages/apps 内无任何写入方——外部 launcher 供值还是纯留白，未考证。
- 待核实：「发布默认关」只查了 base 与 web-app 两层 patch；sdk/headless/acp 层是否
  重述该行的值未逐层翻验。
- 待核实：`openAt: never` 下 reconciliation 与代际竞态无法实机触发，并发语义结论
  全部来自代码走读 + 测试标题。

## 相关

- 真源与折叠状态机：[会话事件日志](../agent-runtime/session-event-log.zh.md)
- revision 令牌、flush 屏障、撕裂尾配平：[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)
- 表层改写（shadowed 的成因方）：[表层改写与压缩](./surface-compaction.zh.md)
- 投影/标题/遥测供给侧：[投影与遥测内幕](./session-projection-telemetry.zh.md)
- 会话引用注入线：[提示词组装与运行时上下文](./prompt-assembly-context.zh.md)
- 委派树的谱系来源：[委派与编排](../augmentation/subagent-orchestration.zh.md)
- 持久化概览（接缝归属层）：[会话持久化与存储](../platform/session-persistence.zh.md)
