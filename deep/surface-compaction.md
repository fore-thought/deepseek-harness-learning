---
title: "表层改写与压缩：replace 语义、影子价与括号锁"
tags: [dsh, compaction, surface, projection, token-pricing]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{core/session,core/agent-loop,compaction/*,session/*,llm/token-meter,goal/goal-round-driver} 源码直读 + 本机 dsh home 真实会话日志只读解析与已构建 dist 合成实验真机实测"
updated: 2026-09-05
---

# 表层改写与压缩：replace 语义、影子价与括号锁

> 本篇回答：模型可见的"表层"如何被 `replace` 合法改写；压缩如何以日志括号对为锁按事务全序落地；改写后的
> token 价如何让纯消费方免状态扣减；存储层用什么行压缩扛住 96% 的 chunk 洪水。衔接概览篇[上下文工程：计量、
> 压缩、附件与溢出](../llm-layer/context-engineering.md) 的留白；证据基准 = DSH 仓库源码直读 + 本机真机实测。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| `replace{start,end}` 只知"闭区间" | 双端 inclusive；按**表层位置** `indexOf` 解析；替换节点 splice **占据**区间起点；`start` 数值可大于 `end` |
| "surface 折叠细节"未读实现 | plan/commit 两阶段 `validateNext`；`seq === log.length` 全局断言；`surfaceOp` 双向强制（缺抛、多也抛）；唯一投影规则 `deriveEventMessage`，代际不变时调用 O(新节点) |
| "chunk 行压缩存储"只知概念 | 三种 packed row；`MIN_RUN=3` 是格式常量非可调项；**本方快照（5912 帧口径）行级压缩比 4.1× 实测** |
| "锁=日志上不配对的事件对" | unmatched `compaction/start` 阻塞新事务，**除非有更新的 `session/end-seed` 证明它属于继承历史**（豁免） |
| `retainRatio 0.16` 校准细节 | 默认常量全部实读（0.8/0.16/8192/1/1 [MEASURED]）；加载期 + 容量期两层校验 |
| 头部锚定与配对平衡未读实现 | `start` 恒=表层首节点；N+1 切点模型，`inProgress<0`=损坏表层即抛 |
| pruner、影子价、`canonicalHeader`、`series` 未读实现 | 8192/4096/1024 码点切片不拆代理对；"只许改 content"由折叠器强制；空字段缺席、tools 顺序敏感；表层代际变化**自动**开新 series 且全仓唯一生产者=goal 续跑（概览篇未写的**新增机制**） |
| 投影帧/checkpoint 只读文档 | `restoreFloor` 的 "one-below anchor" 是承重设计；identity 防"同 id 不同生命周期" |
| 真实 compaction 事件样本缺失 | 仍未见：13 份日志零 compaction=**测量环境事实**（1M 窗口未达阈值）；replace 语义已用已构建 dist 合成实验全量验证。概览篇说法**无被推翻项**，以上全为"补深/钉死/新增" |

## 表层资格、replace 区间语义与投影缓存

- 表层资格仅三类：`SURFACE_EVENT_TYPES = {user/message, assistant/message, tool/result}`；折叠逐事件跑
  `surfaceOpOf`：非资格类型带 `surfaceOp`/`sourceEventSeqs`→抛，资格类型缺失→也抛；`Session.append`
  再以条件参数焊一层类型：`T extends SurfaceEventType ? [opts: SurfaceIntent] : []`。
- `isReplaceOp` 恰好 3 键 `{op,start,end}`、双端非负安全整数（`-0` 拒绝）；人读 transcript 只准读
  append-origin（`isAppendSurfaceEvent`/`isReplacementSurfaceEvent` 分流）——替换副本 model-only，
  落进人读视图会擦掉用户已看过的对话（注释明说）。
- `start/end` 是表层节点 seq，在**当前表层**里 `indexOf` 定位，找不到或 `startIdx>endIdx`→抛；折叠动作
  `nodes.splice(startIdx, endIdx-startIdx+1, replacementSeq)` 使替换节点**占据** shadow 区间起点、区间
  整体从表层消失（日志全保留）；`start===end`=单节点改写（pruner 用）。位置有序非数值有序：合成实验实测
  第一次 replace 后表层呈 `[3,2]`，`shadowedRange` "start 可大于 end" 由此解释；positional replace 使
  `replaceGeneration += 1`，append 不增代。
- 因果回指 `assertProvenance` 四条：`sourceEventSeqs` ①严格早于自身 seq ②去重 ③非空（唯一豁免：
  `assistant/message` 允许 present-empty）④覆盖全部 shadowed 节点，缺一即抛。
- `tool/result` 单点改写须 shadow 恰好 1 个当前 tool/result、除 `content[0].content` 外全字段深相等
  （`assertToolResultRewrite` + 手写 `isDeepEqualJson` 保 browser-safe、禁 node: import）——**"pruner
  只能改结果正文"是折叠器强制而非约定**。
- `SurfaceManager.validateNext` 两阶段：先 plan（不改状态）后 commit，`Session.append` 推日志前调用，
  坏事件在源头抛而非落盘后爆炸。逐事件断言 `event.seq === expectedSeq`（"seq = log.length" 全系统契约）；
  seed 路径走同一 transition——seed 与 live append 同规则，杜绝"内存里能活、存储写不下"。
- 投影缓存三元组 `derived`/`derivedNodes`/`derivedGeneration`：代际变化→全清重投影，否则只 fold 新表层
  节点→每调用 O(新节点)；返回新数组、元素共享事件里的深冻结 data（零拷贝投影缓存）。
- 唯一规则 `deriveEventMessage`：`user/message`=data 逐字、`tool/result`=`data.message`、
  `assistant/message`=`data.message` 且 **content 为空→null**——max-tokens 步骤的空 assistant 轮只是
  usage 宿主，不得注入模型可见序列；其余 null。投影层不得重新加框（如 `<context>`），framing 归生产者
  （`<system-reminder>` 由 agent-instructions 烤进 content）。

## chunk packed row：存储层的行压缩（`chunk-rows.ts`）

- 动机注释自带实测：JSON envelope 约 56× 于 payload。packed row 是**存储编码词汇、不是会话事件**：无
  `SessionEventMap` 条目、不进 `snapshotEvents()`，tag 用无斜杠裸名（`text-chunks` 等三种）。
- 编码白名单极严：envelope 恰好 `{type,seq,time,data}`、data 恰好 `{turn,step,chunk}`、chunk 精确键形
  +原始类型+整数 time；**不认识的形状逐字透传**（丢压缩不丢数据）。run 连续条件=seq+1 且同 turn/step/index
  （tool-call 另要求同 id、name 在场性+值一致；dt 间隙须安全整数、负值合法=时钟回拨）；`MIN_RUN=3` 原注释
  "格式常量非可调项"（两种布局互解等价）。
- 解码行 tag 命中→全量校验后展开（dt 累加越界即 "malformed" 抛——校验 encoder 值域，越界=损坏存储非可猜
  数据）；未命中→普通事件。**整批损坏必炸、绝不静默丢 run**。压缩比实测见实测记录节。

## 范围选择与 tool-pairing 平衡

- `selectCompactableRange`（`compaction-basic/src/region.ts`）四步：①断言 token-meter 表层与会话表层
  逐位一致（防陈旧 measurement）；②自尾向前累计**路由价**达 `retainTokens`→`keepFromIdx`，为 0→null
  （没东西可压）；③向前退到最近的 `toolPairingBalancedBefore` 切点，仍到 0→null；④返回
  `{start: surfaceNodes[0], end: surfaceNodes[keepFromIdx-1]}`——**头部锚定**：永远从表层首节点压到
  截止点、保留尾部。溢出/手动路径传 `retainTokens=0`＝尽量整段压掉、只留最后一段平衡尾巴。
- 配对模型（`compaction/src/tool-pairing.ts`）：表层 N 节点→**N+1 个切点**；delta=`assistant/message`
  计 tool-call 块数、`tool/result` 计 -1；平衡 ⟺ `inProgress==0`，**`inProgress<0`=result 无配对
  call=损坏表层抛**；`WeakMap` 缓存 `{generation,cutBalanced,indexBySeq,inProgress}`，失配整体重折。

## 压缩事务全序：括号锁与 end-seed 豁免

`compactSurfaceRegion`——"同步相邻"就是锁语义：

```text
validateSurfaceRegion（位置 + 双边界切点平衡）
→ inspectCompactionEntryState 自尾向头单遍扫（open turn / 最近 unmatched compaction/start / 最近
  session/end-seed）：unmatched start 且无更新 end-seed → busy。**end-seed 豁免**=恢复/fork 继承的
  孤儿括号不锁死新生命周期
→ append compaction/start（锁先于一切 await 落日志；owner 自动=turn 号、手动=null standalone）
→ prepareCompaction 双轨取价：Σ启发价（影子价协议用）、Σ路由价（收缩比较用）
→ await summarize（唯一长耗时步）→ shrink 闸门：加框摘要 tokens ≥ Σ路由价→抛（"replacement 必须降压
  力"）→ 稳定性复检两式：自动=whole-surface 重测全等；手动=selected-span（范围仍在表层 +
  shadowedSeqs 全等 + 重新等值计价）——区间外注入/新消息不否决手动
→ commitCompactionBody（同步不喘息）：compaction/summary → user/message(checkpoint, surfaceOp=replace,
  sourceEventSeqs=[startSeq, summarySeq, ...shadowedSeqs]) → compaction/end
→ catch：任一失败对 compaction/end 做且仅做一次尝试；关闭失败=故意留可检测的 unmatched start；
  flush 失败单独归类 'persistence'（数据已提交）
```

- checkpoint 回指锁事件与摘要事件本身——替换节点自声明"这一段由这次事务压出"；手动失败分类：commit 阶段
  'commit' / `SurfaceChangedError` 'changed' / 其余 'summary' / 中止 'cancelled' / 同步抛 'busy'。
- 摘要输入=`requestHeader()` system+tools 原文 + shadow 区间逐节点投影→摘要调用是上一请求真前缀（KV
  cache 复用）。不变式伴侣（`invariant.ts`）每分发前校验：括号配平/同 id 贯穿/turn 归属精确（numbered 在
  turn 内、standalone 在 turn 外）/至多一次 summary/`shadowedSeqs` 恰等于折叠时刻表层区间/成功 end 必配 summary。

## 触发策略与常量

| 常量 | 默认 [MEASURED] | 说明 |
|---|---|---|
| `DEFAULT_THRESHOLD_RATIO` / `DEFAULT_RETAIN_RATIO` | 0.8 / 0.16 | 触发线与保留尾（窗口占比）；与 `retainTokens` 互斥；加载期校验 ratio 冲突、容量期拒 `retainTokens>=thresholdTokens` |
| `maxTokens` | 8192 | 摘要调用输出上限 |
| `compactionRetries` / `maxOverflowRetries` | 1 / 1 | 正常与溢出恢复预算；`auto` 默认 true |
| pruner `thresholdChars/headChars/tailChars` | 8192/4096/1024 | 加载校验 head+marker+tail ≤ threshold（产物必更短且低于阈值） |

- 自动三监听：`agent/pre-step` waterfall 压测 `compactIfNeeded('pressure')`——**永不 reject/改写消息**、失败
  记日志后 `next()`（压缩失败绝不卡轮次）；`agent/request-error` 遇 `CONTEXT_WINDOW_EXCEEDED` 走溢出恢复
  （跳过阈值与保留尾、`retainTokens=0` 强制平衡压缩），**表层代际前进过即记一次 retry**（pruner 的 model-free
  落盘也算进展）、返回 `{kind:'retry'}` 重建请求；idle 与 `assistant/message` 清 `overflowRetries`。
- pressure 流程：缺 contextWindow=`TargetPressureConfigError`（按 targetKey 去重告警）→异步决策后锁复查→低于
  阈值 return→有 pruner 先剪并重测（能免则免 LLM）→循环 `compactionRetries+1` 次每轮重测→仍超阈**抛不静默**；
  溢出压后代际未前进→放弃 retry。摘要路由：`summarization{Provider,Model}`（须成对）→最新 header config→
  `agent.options` 皆空才抛；输出只留 text 块，finish=error/aborted/max-tokens 全 fail-closed（宁可失败不落）。
- checkpoint 消息框架：`<compacted-summary>` 包裹 + 前导 `CHECKPOINT_PREAMBLE`（"This is an
  automatically generated checkpoint condensing an earlier span …"）+ `COMPACTION_INSTRUCTION` 为**最后
  一条 user 消息**（8 段固定 Markdown、遇旧摘要块合并重写而非搬运）；`/compact` 无参数、六码人话文案。

## model-free 剪枝与影子价协议

- pruner（`compaction-tool-result-pruner`）度量按 **Unicode 码点**（`Array.from`）：不拆代理对、可能拆
  字素簇；`PRUNE_MARKER='\n\n[... tool result middle pruned ...]\n\n'` 只插第一个相交文本块、非文本块原序
  保留、防御双闸（marker 未落位/结果不降→抛）。`pruneSession` 对快照表层逐 `tool/result` 处理，每个替换
  同步相邻两条：`compaction/prune`（log-only，影子价=`estimateMessage` 原消息）→`tool/result` 替换
  （`start=end=`原 seq、`sourceEventSeqs=[原 seq]`）；中途抛错已落替换保持 durable、注释明示不回滚。
- **影子价协议总纲**（types.ts 注释）：表层 replace 的价格=**紧邻其前的一条计量事件**（summary 或 prune），
  纯消费方（前端/CLI 显示节省量）无需留每节点价态即可扣减；replacement 必须同步紧随；其间的
  `compaction/start|end` 是 log-only 不在表层，"紧邻"按表层读序成立。

## request/header：canonical 表示与 series

- `canonicalHeader`：空 system/空 tools→**字段整体缺席**；`adapterDefaults` 仅当
  `reasoningEffort===true` 或 `maxTokens===true` 保留（全 false 无信息）；日志、折叠、比较三方共用此
  表示。`headerEquals`=config 深等 ∧ 两旗标逐等 ∧ system 逐等 ∧ tools 逐位 `JSON.stringify`（**schema
  顺序敏感**）；`Session.requestHeader()` 是折叠增量态（`headerFoldSeq` 水位）冻结返回。
- loop 侧每步（`core/agent-loop/src/agent.ts`）：`startsSeries = startsRequestSeries ||
  requestSurfaceGeneration !== surfaceGeneration`——**表层代际变了自动开新 series**（压缩落地后前缀已变）。三分
  支：首请求→`initial`/`resume`；header 变→`change`；header 未变但新 series→**再落一条同 header 事件
  `reason:'series'`**（"相同 header 的新起点"声明，不是 diff）。遗留防御：`request/header-delta` 与
  `reason:'fallback'` 在 seed/append 双路径直接抛（index.ts:215,364）——无兼容层、未发布期斩版。
- `startsRequestSeries` 全仓唯一生产者=goal 续跑（`goal-round-driver/src/index.ts:425`）。实测
  [MEASURED]：深读实测会话仅 1 条 header（seq 25、`initial`、整行 35142B=全日志最大行）；12 份兄弟会话
  日志均 `initial`+`resume`=「实例换、header 未换」实证；全库无 `change`/`series` 样本。

## 投影帧、读阶梯与语义检查点

- 契约 `ProjectionDefinition{key,stateSchema(zod),init,apply,wire?,stateVersion}`：**apply 不感兴趣必须
  返回同引用**（`Object.is` 短路下游工作）、view 同理复用引用抑制伪发布；drive eager 逐提交事件过所有单元
  （cell 落后→逐 seq 补折、缺 seq 即抛），状态引用变才重算 view、`views[0]/views[1]` 双缓冲
  `Object.is` 比较、变才发 `onChanged`；注册按 key 引用计数。
- 读阶梯（冷启动不必全量重放）：①`viewCheckpoint` 零 I/O 直读 view 存储行（"stale as rows, never wrong"，
  `asOfSeq` 取最低被服务水位——低报安全）；②`restore` 尾部重放：`restoreFloor` 给"最低可用水位再 -1"作
  读起点——**锚点 -1 是承重设计**：尾部读同时证明日志真实长度，崩溃修复截短日志后行水印>日志末端即拒用该行
  （`baseSeq>0` 且行不可用→抛 "re-read from seq 0"），行可用=ver 匹配 ∧ `seq>=baseSeq-1` ∧
  `seq<=endSeq`；③full refold 从 seq 0。
- 持久行 `(sessionId → {identity, rows:{key→{ver,seq,val}}})`（domain `session_projcache` v6，坏文档挪 .bak
  跳过不炸启动）。`identity={createdAt,cwd,isSeeded,inheritedEventCount}`——**会话 id 是槽位不是生命周期**，读
  前先对 header 验身份，防删后重建/换存储根折进无关日志。写路径全 fail-soft；`write()` 先 checkpoint→**先
  flush 会话日志再写缓存行**——崩溃只可能"cache 落后于日志"（重放补偿）、绝不可能超前（幻影值不可能）。
- `session-checkpoint-policy` 三挂点：`llm/stream`（带 sessionId）先 `await sessions.flush` 再开流——
  **已记录的请求前缀不 durable 就不发出请求**；顶层非嵌套 `tools/execute` 先 flush 再放行工具体、flush 后
  复查 abort→合成 `TOOL_ABORTED_BEFORE_DISPATCH`（不落半句），嵌套工具复用外层 durable call；
  `agent/pre-step` 请求前 flush 上一步已提交内容（首步≈no-op）。
- 轮内自动压缩的 pre-step 发生在 flush 屏障之前→压缩产物随请求前缀被该屏障一起落盘；手动 `compactNow`
  自带 flush 选项；loop 不在轮边界 await flush——durability 归这个 policy 包。

## 深读实测记录 [MEASURED]

- 样本=本机 dsh home 一份真实会话日志（`sessions/<workspace-slug>/<sessionId>/session.jsonl.zstd`，压缩态约
  2.0MB，只读）：**每写一帧**的 zstd 级联、5912 帧全部独立解码成功（魔数 `28 B5 2F FD` 帧边扫描）、解压
  3.52MB/8019 行（含裸 tag `session` 的 header 行）；逻辑事件 32883（seq 0..32882 **零空隙连续**）、
  `assistant/chunk` 31500 条（96.7%）、packed 行 5182 → **行级压缩比 4.1×（5912 帧快照口径）**；重新 pack 仅
  得 2206 条——差值实证"批间 flush 切断 run"。
- `sourceEventSeqs` 存储形态：seq 87→`[[29,86]]`、seq 140→`[[94,139]]`（range 对）；折叠器对密文数组报
  "must densely contain non-negative safe integers" 实测复现，钉死职责链：**range 编解码只属于 jsonl 存储
  边界**（`session-persistence-jsonl/src/format.ts:269,287`，decode 传 `maxEntries=record.seq`——引用不
  可能≥自身）与回放工具，内部消费方（surface/token-meter）只见展开态。
- 真实 dist 端到端：`foldSurface(全日志)`→276 节点、replacements 0、投影 276 条消息（user 28/assistant 124/
  tool-result 124、null 0）；合成实验 replace 全通过（连压收成单 summary 节点；缺 shadow 引用、表层缺标记、
  引用未来 seq 等六条错误路径逐一复现）。`ignorable` 0 条；全 home 13 份日志 `compaction/*`=0、`replace`=0。

## 诚实边界

待核实：真实落盘的 compaction 字节样本仍未获得（本机无发生条件），语义证据止于合成实验、不变式断言与子系统文档
三方对齐；token-meter 表层折价内部（`surface-fold.ts`/`route-pricing.ts`）未逐行读，只钉死其消费的接口面。
待核实：`scanLog` 帧级容错细节；影子价"紧邻"的消费端具体读点；`agent/inbox/spliced` 与压缩保留消息的交互。
待核实：多进程同开一会话是否另有防线——括号锁并发结论基于代码注释自述（"tolerating concurrent writers
needs a signal beyond the log"），未见测试复现。

## 相关

- 概览篇的压缩与计量分工：[上下文工程：计量、压缩、附件与溢出](../llm-layer/context-engineering.md)
- 表层寄主的事件日志与投影框架：[会话事件日志：唯一真源](../agent-runtime/session-event-log.md)
- 压缩挂点的轮次机器：[turn/step 主循环：agent loop 的心脏](../agent-runtime/turn-step-loop.md)
- jsonl 写路径与 flush 屏障：[会话持久化与存储](../platform/session-persistence.md)
- 括号锁 / end-seed 豁免的崩溃面姊妹篇：[持久化与崩溃恢复](./persistence-crash-recovery.md)
- `startsRequestSeries` 的唯一生产者：[自组织：goal、plan mode、todo、schedule](../augmentation/goal-plan-todo.md)
