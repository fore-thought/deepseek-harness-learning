---
title: "LLM 适配器与计量内幕：注册表、流协议、重试与 token 锚点"
tags: [dsh, llm, adapter, streaming, token-meter, retry]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/llm/* 七包 src 源码直读 + docs/subsystems/{llm-streaming,token-meter}.zh.md + docs/deepseek-llm-api-wire-extensions.zh.md + pi-ai@0.84.2 dist 面核对；常量计数均 [MEASURED] 自源码"
updated: 2026-09-05
---

# LLM 适配器与计量内幕：注册表、流协议、重试与 token 锚点

[English](llm-adapters-metering.md) | [中文](llm-adapters-metering.zh.md)

> 本篇是 [LLM 层概览](../llm-layer/llm-vocabulary.zh.md) 的源码级深读展开：`ctx.llm` 注册表
> 如何做到"换轨无缝隙"、流协议两条错误路径在哪个边界汇合、一次调用如何被钉死在同一个
> 适配器代次上、重试为什么"先持久再等待"、token 计量如何用"provider usage 锚点 + 有符号
> 表面增量"绕开启发式对 CJK 的系统性低估。证据基准 = 七包源码直读；无凭据流量，网络侧
> 行为以代码证据为限（诚实边界）。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| 注册 all-or-nothing、`replace()` 原子换轨 | 机制量化：候选集先全量校验（`prepareRoutes` 不写一字），再 `commitRoutes` 单同步段删除+重挂并发 `llm/adapters-updated`；禁 dispose-then-register——中间空窗会被观察者看见"提供方消失又回来" |
| max-tokens/中断时同步裁剪 tool 调用与 replay blocks | 精确化：`max-tokens` 结局**全部** tool-call 块被丢（含已闭合的），replay 逐块条目按同一保留位图裁剪；`interruptedBlocks()` 更严——只留非纯空白 text/reasoning，未闭合 unknown 块也丢 |
| 一次适配器调用 = 一次提供方尝试 | 双保险：loop 层重试归 `llm-retry` 插件，pi-ai 侧再把 SDK `maxRetries: 0` 硬钉；DeepSeek 是裸 fetch 本无库重试 |
| 重试计数存 session-projection（step/start、turn/end 清零） | 深化：状态桶键 = `[provider, policyKey]` 复合（同提供方不同策略各算各的）；一次重试是**两条**事件——`llm/retry`（可取消等待**之前**先持久）与 `llm/retry-started`（等待完成后），配 175 行不变量校验器把链条钉死 |
| idle watchdog 默认 5 分钟 | 证实：两适配器 `DEFAULT_STREAM_IDLE_TIMEOUT_MS = 300_000` [MEASURED]；且 watchdog **只在 `next()` 未完成时计时**——消费方停顿不算提供方停顿 |
| TokenUsage 互斥语义、DeepSeek 扣缓存 | 证实并补全：`totalTokens` 仅在"三计数 safe 且与 wire 总值一致"时保留否则**省略**；pi-ai 侧 cache 桶为 0 时同样省略（pi-ai 报 0 而非缺席） |
| agent loop 从不依赖具体提供方包 | 生成侧证实：`llm` 包全部 import 均为基础包（cordis/typert-protocol/brand/util-{values,crypto}/timeout/attachment/schemastery），提供方包零依赖 |

## `LlmRuntime`：三本账与一个变异点

`ctx.llm`（`llm/src/index.ts`，1134 行）维护三本独立的账：

- **adapters**：route → `{adapter, provider, retryPolicy}`。注册经 `ctx.effect` 包裹
  （fiber 消亡即释放）；`retryPolicy` 在注册瞬间解析冻结——这是**唯一**不能被每请求
  解析刷新的字段，两个插件都靠 `replace()` 原地换轨来跟踪它的变化。
- **directory**（configurable providers）："可以配什么"与"正在服务什么"分离。pi-ai
  插件把 **41 个内置提供方**（`pi-ai@0.84.2` dist/providers/all.js 的 provider 模块
  import 计数 [MEASURED]）全量声明为可配置目录——休眠路由也有配置地址，Models 页面
  能在任何路由注册前渲染。手工声明的网关路由带 `declared: true`，随 profile 来去。
- **discoveries**：按 **settingsNs** 键控（不是按 provider）——"正在新增的提供方还
  没有路由名可指"；草稿态凭据是一次性的，harness 从不存储（`LlmModelDiscoveryRequest`
  的 apiKey 只服务本次探询）。

`emitAdaptersUpdated` 的容错值得一读：Cordis emit 用 Array.map，一个同步抛错会饿死
后续监听器；注册表通知是**非否决**的，所以逐个包 try/catch、异步 rejection 也接住，
唯独 `code === 'INVARIANT'` 的错误收集后重抛——观测性故障不能否决已提交的事实变更，
但不变量违例必须炸穿。

## 一次调用的代次锁定：`prepareCall` 与终界归一

loop 不走裸 `stream()`，而是 `llm.prepareCall(config, signal)` → `PreparedLlmCall`：

- 一次查询同时拿到模型元数据（contextWindow、defaultMaxTokens、reasoning efforts）、
  冻结的 retryPolicy 与 **one-shot** 派发入口——header 记录与真正 dispatch 共用同一
  registration，HMR 换轨不可能把"A 代次的能力答案"配上"B 代次的端点"。
- 派发前 `callConfigEquals` 逐字段核对；复用或对不上 → `INVALID_PREPARED_CALL`。
- `adapterDefaults` 标记哪些字段是适配器默认值填的——loop 在 `agent/request`
  waterfall **之前**剥掉带标记的字段，让最终选择的确切模型解析重新填当次值。
- 终界 `adapterStream`：适配器选择、dispatch、iterator 构造、迭代抛错统一收敛成终止
  `error`/`aborted` finish chunk（`adapterFailureChunk`）；而**中间件与消费方错误
  保持抛穿**——generator 在 `yield` 前先结束适配器侧的 try 块，就是这个切分的实现
  手段。两种错误投递风格（throw / 带内 finish）协议同时承认：pi-ai 恰好只用后者
  （它从不流中抛），DeepSeek 两者都用。
- loop 构建的请求被 `markAgentLoopRequest`（进程内 WeakSet）标记并**深冻结**——到达
  `llm/stream` 的监听器只能读不能改（可重建性不变量的实现前提）；手工调用无标记、
  不冻结。
- 投影最后一步在`运行中`完成（`forAdapter` + content.ts）：file 块对所有路由无条件
  折成 handle 文本（名字+字节数+只读路径+跨委派指引）；image 块仅当路由**显式声明**
  不含 image modality（负能力）才折成占位文本。未编目的端点一律按 text-only 处理——
  高估能力会让主机把图片持久进历史、之后每一轮都被拒。

### 纪元头与代次差量：`request/header` / `request/context`

loop 每步对头部三种裁决（core/agent-loop/src/agent.ts:543-575）：首轮/恢复 →
`reason: 'initial'|'resume'`；与已记录 canonical header 不等 → `'change'`（可带
`startsSeries`，压缩重写表面后的续命）；相等但表面代次变了 → `'series'`。125 步不
重写纪元头是常态（[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md) 的 6049 帧
快照里 `request/header` 仅 1 行）。`request/context` 则是**三元组差量事件**：
provider/model/contextWindow 任一变化才 append——token-meter 占用投影拿它当容量
分母的 last-wins 槽。

## 折叠与紧凑流：一个 fold、两种打包

`BlockAssembler` 是唯一的 chunk→Message fold，深读补三点：

- **delta-only 宽容**：没有 block-start/end 的流也能组；已闭合 index 再来的杂散 delta
  直接忽略——坏适配器既撑不爆内存也污染不了已完成块。
- **唯一保留/丢弃决定**在私有方法 `assembled()` 一次算完：`blocks()` 与
  `replayState.blocks` 从同一位图派生——"存储的元数据永远描述存储的内容"不是口号，
  是这一个函数。`ReplayEnvelope` 逐块条目数与块数不符 → 整个信封作废。
- 未闭合块按累积 delta 组装；finish 缺失默认 `{kind:'stop'}`。

`AssistantStreamAccumulator`（assistant-stream.ts，297 行）是日志侧的镜像词汇：同块
连续的 text/reasoning/args delta 折成一条 `AssistantStreamRecord`（`time0` + 精确
`dt[]` + 每 delta 一成员），`expandAssistantStream` 严格校验成员数/时间戳/工具身份后
**无损**重建。持久回放、遥测、计量、UI 组装都展开嵌入式结算流，而不把进程本地
`agent/assistant-stream` frame 当持久事实。物理存储行的打包（`MIN_RUN = 3` 等）见
持久化深读篇——两处是同一词汇的接缝两侧。

## 重试：先持久、再等待、全程可证

`llm-retry` 插件**自身无配置**（`retryPolicy` 放错位置直接报错并点名去处）。语义：

- normal：`retryableCodes` 不命中 → 让位下游；命中且未超 `maxRetries` → 调度。
  always：先 `settleDownstream`——下游恢复器的决定是 retry 就采纳、抛错只 warn 吞掉
  （"永远重试"的路由不该被别的恢复器否决），随后无条件调度。
- 延迟裁决：`providerRetryAfterMs` 优先于本地指数退避（`initial·2^(retry-1)` 封顶
  `maxDelayMs`，jitter 乘子在 [1-r, 1+r] 对称）；normal 下提供方要的延迟超本地上限 →
  放弃让位；always 下退回本地延迟。`Retry-After` 头兼容秒数与 HTTP-date 两种拼写。
- **顺序是契约**：`llm/retry` append 在可取消等待之前——崩溃后读日志就知道"第 N 次
  重试已被调度"；等待完成才补 `llm/retry-started`。投影在 retry-started 处才释放该
  step 的计数槽（tokenUsage 单元同理：重试 attempt 由此获得独立请求的计数资格）。
- 配套 `llm-retry-invariant`（175 行）：每条 `llm/retry` 校验"开着的 turn/step 号
  匹配、provider 等于失败请求的 provider、retry 号 = 同链上一条 +1、retryId 全链复用
  且不与他链碰撞、normal 的 retry ≤ maxRetries"；`llm/retry-started` 必须配对且
  不重复；加载旧会话时全量重演。退避指数钳在 1024 防溢出。

默认策略（retry-policy.ts:14-21）[MEASURED]：normal / 5 次 / 首延迟 500ms / 上限
10000ms / jitter 0.1 / codes = EMPTY_RESPONSE、RATE_LIMIT、SERVER、TIMEOUT、
TRANSPORT。

## DeepSeek 适配器：端点与密钥同代次

- 连接事实**每操作解析**（options() thunk + 按 raw 快照身份 memoize；坏快照保留上一
  代好配置并一次性告警）。baseURL 兜底链 = 显式配置 → 受信任启动环境的
  `DEEPSEEK_BASE_URL` → 公网端点——checkout 可以把自己的 agent 指向本仓网关。
- `apiKeyEnv` 是**凭据引用**、随行于端点快照：一个被拒的 settings 代次不可能把自己的
  key 配给上一代的端点。`assertUsableApiKey` 只静默修空白（trim），字符非法时**不回显
  密钥任何部分**、按 ref 指路修复处。引用了却解析不到 → `MISSING_CREDENTIAL`
  fail-loud；**只有完全不引用凭据的路由**才落到环境探测。
- `httpErrorCode`：401/403→`AUTH`；413→`INVALID_REQUEST`；quota 文案族→`QUOTA`；
  429→`RATE_LIMIT`；400 且溢出分类器命中→`CONTEXT_WINDOW_EXCEEDED`；≥500→`SERVER`；
  其余 `HTTP_<status>`。上下文溢出**只有一个规范码**（error.ts 的三条正则吃遍各家
  措辞），消费方按码路由、绝不解析文本。
- SSE：eventsource-parser 管帧（含跨 chunk 的 UTF-8 半序）；本模块只管协议——
  `[DONE]` 是**唯一**合法终点，EOF 缺它就 `STREAM_CLOSED`。块折叠、usage、finish 全部
  **押后**到 [DONE] 统一冲刷（usage 有两种到达形状：贴 finish 或尾随 usage-only chunk，
  取最新）；thinking 模式 reasoning 在前、空串首 delta 不开块；`acceptIdentity` 把
  续传 delta 里的空串/null 一律读成"不更新"而非"清空"；stop 且零块 = `EMPTY_RESPONSE`
  错误而非空成功——空消息会静默终结轮次。
- **图片是重工程**：路由像素预算（`DEFAULT_MAX_TOKENS = 256_000`、
  `DEFAULT_CONTEXT_WINDOW = 1_000_000`、内联 20MiB / 请求 600 张上限 [MEASURED]）→
  Files API 上传复用（7 天过期、余 1 小时主动换、配额错误先清 100 个最旧本方文件）→
  上传解析失败**降级 base64 重拼一次** → 提供方点名 stale file-id 则先失效映射再重试
  一次 → 归一化字节仍被拒时产出指名图片名/尺寸/色彩空间 sRGB/sRGBA 的诊断。请求级
  offload 是**确定性量子**算法：超预算按整字节量子（64MiB）/整数量子（20 张）从最旧
  替换为占位文本；源码例：129 张 1MiB 图在 128MiB 限下移除最旧 65 张，且该前缀到
  总历史 192MiB 前恒定——逐请求重演不抖动。
- `purpose === 'session-title'` 强制 `thinking: disabled`（serialize.ts 第一行裁决：
  标题辅助调用不为思考付费）。`x-deepseek-harness-user-id/session-id/compact` 三个
  归属头在 fetch 处拼装。
- 模型能力来自**配置目录**而非探测：未编目的 model 安全地按 text-only 处理；
  `thinking: disabled` 的部署只暴露 `off` 一档 effort——能力面与部署事实一致。

## pi-ai 适配器：不可变快照与目录复用

pi-ai（`@earendil-works/pi-ai@0.84.2`，承载多协议实现的第三方库）路径全部设计围绕
一个事实：**`Models.streamSimple()` 是惰性的**——提供方在流首次消费时才解析，发生在
凭据 await 之后。所以配置变更必须**重建集合**而非原地改：每次解析产出
`{profiles, models}` 不可变快照（按 profiles 身份 memoize），进行中的操作抱着自己的
快照跑完——"切换模型在下一步生效、绝不在流中途生效"。

- 目录路由**复用 catalog provider 本体**（`reuseCatalogProvider` 把 stream/streamSimple
  委托回去）：Bedrock 的 Smithy 模块经独立入口加载，拆件重建会静默砍掉能用的提供方；
  被拒的只有 catalog 的动态刷新——这条路由的目录是 settings 文档说了算。
- 手工路由只能走**三协议表**（openai-completions / openai-responses /
  anthropic-messages）：Bedrock 的 SigV4、Vertex 的 ADC、Codex 的 OAuth 都不是
  "端点+key+headers"能表达的形状，注释明说"宁缺毋滥，来一个消费者补一行"。
- auth 三面桥：CredentialStore（登录/轮换落 `credentials` 记录，`RECORD_SCOPE =
  'llm-pi-ai'`，OAuth grant 原样存、只把"JSON 化不了的显式 undefined 成员"归一）+
  AuthContext（环境探测）+ `authorization` 登录流（seam 缺席的组合就没有登录面，
  其余照常）。store/context 每次调用读 ctx——**集合重建不弄丢登录身份**。
- `maxRetries: 0` 硬钉 SDK 自重试；`stop` 参数直接 `UNSUPPORTED_OPTION` 拒绝。
- 错误分类 `classifyPiAiError` 是**文本模式匹配**，`XXX` 注释点名原因：pi-ai 把
  caught error 拍扁成 message，undici 的可操作细节留在被丢弃的 cause 上——只能按
  措辞判 `TRANSPORT`（`terminated`、`Premature close` 都在名单里）；
  `mapStopReason` 兼检测上下文溢出（pi-ai 的 isContextOverflow 按 usage 对
  contextWindow）与 `pending/deferred` 终止态（映射为不可重试错误）。
- `assertServiceable` 把"不可服务的配置"**在写入处拒绝**（settings 层回
  `settings-rejected` 并点名路由/模型）；注册表才能看见的冲突（profile 声称了别家
  适配器族的路由）在 onChange 换轨时被拒下、告警、保留旧路由集继续服务——两段拒收
  各有名字、各有诊断。空 `providers` = **休眠挂载**：零路由注册，settings 一给
  profile 立刻挂轨。
- 能力描述的宽容是刻意的：`describableReasoningLevel` 在**目录构建**路径上把"部署
  配了但模型不支持"的档位降为不可见而不抛错——一个错配字段不该把整条路由的所有
  模型从选择器里抹掉；请求路径照旧硬拒。`reasoningInfo` 对"只有 off 一档"的模型
  干脆**不报** reasoning 能力：off 的语义是"省略参数"，对这类模型等于撒谎。
- 默认容量与预算 [MEASURED]（config.ts）：contextWindow 262_144、maxTokens 32_768、
  请求图基线 20MiB/2048²/1MB。

## token-meter：两轨表面与一个锚点

固定启发式（estimate.ts）：`CHARS_PER_TOKEN = 4`、块结构开销 4、角色封装 4
[MEASURED]——它系统性低估 CJK 与 JSON schema，**所以它从不单独回答"占了多少
context"**。两条轨各司其职：

- **测量轨**（`TokenMeter.measure()`，O(表面)）：positional fold 给每条表面事件一个
  定价节点（含嵌套 tool-result 里的 image/file 出现位）；`request/header` 锚定的
  **提供方 usage** 是 baseline——仅当锚点 canonical header 与当前一致、且 usage 总量
  ≥ 锚点全启发式价（保守性前提）才复用，之后只按**有符号增量**修正表面移动：
  `totalTokens = usage锚 + Δsurface`。锚后追加一条消息，压力立刻动、提供方数字不动；
  压缩遮蔽一段，立刻缩。route image pricing 在此轨重定价图片节点（`priceImages`
  应答数量与出现次数不符 → fail-loud），file 节点按请求期 handle 文本重定价。
- **投影轨**（三个 ProjectionDefinition，状态 O(1)）：持久缓存的检查点必须恒定大小，
  所以压缩的"被遮蔽价"经 **shadow-price claim** 协议进来——`compaction/summary` /
  `compaction/prune` 申报被替换区间的启发式价，**紧邻的下一条** replace 消费它；
  claim 范围对不上 = 当场抛（活体违约要响）；前协议时代的旧日志无 claim → 中性折叠
  （重放退化为漂移而非失败）。
- `contextPressure` 三槽刻意 **last-wins 非原子**（分子=最新 usage 样本、分母=最新
  `request/context`、表面=运行折叠）：文档承认换模型瞬间会新旧配对——它自称
  "用户界面参考值，不是计费或门禁输入"。usage 样本在**该事件自己上表面之前**锚定，
  所以 `assistant/message` 对齐的是它自己那次请求看到的表面。
- `tokenUsage` 单元：四互斥桶累计；同 turn/step 的后续结算**替换**上一份（重试链共享
  槽位），`llm/retry-started` 释放槽让下一次 attempt 改走累加。`assistant/attempt`
  也带 usage——被拒的请求照样花了钱。
- `deriveTurnTokenUsage`（turn-usage.ts）是给 UI 的**可证明聚合**：idle/open/
  finishClosed/settled 状态机严格重演 step 生命周期，任何边界缺失、计数溢出、total
  矛盾 → 整个 turn 的披露返回 undefined（宁缺毋滥）。
- `contextBreakdown`（system/tools/message 三格构成图）与测量轨共享同一个估计器，
  文档明说"三者相加不等于 projectedTokens——把构成当近似展示，别当总数"。
- `logRevision` = 已消费事件数，消费者游标；`measure()` 返回深冻结 detached 快照。

## 扩展表与清单：两条 wire 顶层字段

`deepseek-llm-api-extensions` 是一张**认领制**表：字段名 → 唯一 provider（重复注册
抛错、字段名必须 trimmed 非空）。`prepare()` 并行取全部字段（structuredClone + 递归
冻结，贡献方不持有出站请求的可变别名）、可被取消打断（`abortable` race）；
`accept()` 只在 HTTP 2xx 之后跑（适配器侧）、幂等合流、allSettled 后聚合抛错。冲突
检查在适配器：扩展字段撞基础请求字段 → `REQUEST_EXTENSION` 直接失败这次模型调用。

官方组合贡献两个字段：`dsh_session_log`（session 侧贡献者，见持久化深读篇）与
`dsh_plugin_packages`（plugin-package-inventory-deepseek）。后者口径苛刻得值得学：
**当前活跃 agent 请求代**的全 Loader 包集合——只数 fiber ACTIVE、非 group、非
disabled 的 entry；裸包名经 `createRequire(...).resolve.paths` 从四个锚点解析
manifest（解析不到就抛，不静默缺报），loose 文件模块向上找最近 manifest（无名清单
视为匿名模块跳过）、`cordis:*` 内部项出局；请求带 sessionId 时**并入该 agent 的
standing preset 树**（动态 import 可选对等包 `agent-presets`，不碰 Loader 内部）；
`name+version` 去重后按 **ICU 无关**的逐字符比较排序——清单要跨主机逐字节稳定；
进程级 manifest 缓存自带 TODO（版本原位替换不是支持中的升级路径，故不失效）。
`enabled` 默认 true——这是一份**要出境到提供方**的清单，显式开关留在配置面。

## 诚实边界

- 待核实：无真机网络实测（无凭据流量）；SSE/翻译/重试延迟均为代码证据与官方测试面，
  `llm-deepseek/tests` 未逐读。
- 待核实：`file-store.ts`/`upload-index.ts`/`files-api.ts` 的上传复用键与配额回收
  细节只走接口面；`image-tokens.ts`/`request-pricing.ts` 的 v4 视觉计量公式未逐行
  对照官方定价页。
- 待核实：`llm-pi-ai` 的 discovery.ts/login.ts/context.ts 与 catalog.ts 的 22 项
  compat 开关只读接口与关键注释，未逐项展开；`models.generated.js` 目录模型总数
  未统计。41 内置提供方计数取自 pnpm store 内 0.84.2 dist 快照，随上游漂移。
- 待核实：`resolveImageAttachmentAccess` 与 e2b/远程 fs 组合下的路径映射行为未实测
  （fs 侧见沙箱深读篇）。
- 范围说明：模型选择 UI（client 组）与 settings/credentials 接缝本体不在本篇——见
  [host/client 分层](../platform/host-client-boundary.zh.md) 与
  [配置面](../augmentation/settings-and-credentials.zh.md)。

## 相关

- 概览篇（本篇是其深读展开）：[LLM 层](../llm-layer/llm-vocabulary.zh.md)
- 谁驱动 stream 与 header：[turn/step 主循环](../agent-runtime/turn-step-loop.zh.md)、
  [会话事件日志](../agent-runtime/session-event-log.zh.md)
- system/tools 槽位的来源：[提示词组装与上下文注入](./prompt-assembly-context.zh.md)
  （同期产出）
- 压缩消费计量数字：[表层改写与压缩](./surface-compaction.zh.md)、
  [上下文工程](../llm-layer/context-engineering.zh.md)
- 紧凑流行的物理侧与 usage 遥测：[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)
- 凭据/设置的接缝语义：[配置面](../augmentation/settings-and-credentials.zh.md)
- 投影单元的注册与缓存：[投影、遥测与格式迁移](./session-projection-telemetry.zh.md)
  （同期产出）
