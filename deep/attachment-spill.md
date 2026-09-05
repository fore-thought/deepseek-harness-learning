---
title: "附件与溢出：大字节进事件日志的两条仓储通道"
tags: [dsh, attachment, spill, image, tool-output]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{attachment,spill}/* src 直读 + fs/tool-fs(read_image) + llm/{llm,llm-deepseek} 消费侧 + docs/subsystems/{attachment,spill}.zh.md"
updated: 2026-09-05
---

# 附件与溢出：大字节进事件日志的两条仓储通道

> 会话事件日志是唯一真源，但**不能被字节压垮**：图片二进制与超长工具文本都以
> "引用/定位符"入日志。attachment 仓管**不可变内容寻址二进制**（先持久化、后发事件），
> spill 仓管**一次性的工具超大文本**（全文落仓、内联换预览）。本篇是
> [上下文工程](../llm-layer/context-engineering.md) 附件/溢出两节的源码级展开。

## 两仓分工

| | attachment | spill |
|---|---|---|
| 内容 | 栅格图片（4 种 mediaType）+ 逐字节文件 | 纯文本工具结果 |
| 身份 | `sha256:<digest>` 内容寻址（品牌 id，消费方禁解析/推路径） | 不透明 `SpillLocator`（本地实现=文件路径） |
| 入日志形态 | 结构化引用块（`ImageBlock`/`FileBlock`） | 原文被 preview+notice 替换，定位符在文本里 |
| 生命周期 | 不可变对象、被检查点长期引用 | 保留天数 + 启动一次性清扫 |
| 存储失败语义 | 拒绝（无引用则无事件） | 尽力而为：保内联，绝不把成功调用变错误 |

## AttachmentStore seam（`ctx.attachments`）

- 契约核心：**对象完整持久化之后才发布引用**，引用里永远没有文件系统路径、
  bearer URL 或 base64（`packages/attachment/attachment/src/index.ts`）。
- 批量语义（`saveImages`）：先验全部成员再提交任何成员——校验失败**零写入**；
  存储失败不留半成品引用，已发布的内容寻址对象成为不可达孤儿、等未来保留策略回收。
- `admitPromptContent` 是 Host prompt 端点的安全不变量：浏览器 image part 在
  **任何消息创建前**升为持久引用，wire 调用方永远无法引用一个自己没上传过的附件。
- `AttachmentError` 故意**不继承** `HarnessError`：基类在 `@deepseek-ai/dsh-llm`，
  而 llm 反过来依赖 attachment（`ImageBlock` 引用 `ImageAttachmentRef`），共享基类
  成环。消费方按稳定 `code` 集合路由（`isAttachmentError` 查 code 不查原型链），
  重复包安装下依然可判别——"形状重复、协议不锁原型"的又一例。

## 内容寻址仓与落盘耐久（`attachment-local`）

根为 `<DSH_HOME>/attachments/v1`；一切对象走同一条发布管线
（`publishImmutableObject`/`publishImmutableObjectStream`，src/store.ts）：

1. 暂存写入 `tmp/`（`O_CREAT|O_EXCL`、0600），流式边写边算 sha256 → 文件 fsync；
2. 硬链接到 `objects/<2hex>/<sha256>`；EEXIST 幂等去重，但去重前**重算既有对象
   摘要**，不符即 `ATTACHMENT_CORRUPT`——宁可失败不让脏对象冒充身份；
3. chmod 0400 转只读；Windows 分支顺序相反（先 unlink 暂存名再 chmod）：只读属性
   经硬链接共享，先设了就两个名都删不掉；
4. POSIX 逐级 fsync 目标父目录直至 root——**文件 fsync 不够**，目录项未落盘时
   检查点会引用不存在的对象；Windows 靠 NTFS 日志（源码注明不开目录句柄）。

- `ensureDurableHome` 处理一个并发陷阱："目录已存在"不等于"目录项已耐久"——并发
  创建者可能还没 fsync 它的父目录；每个进程独立把 DSH_HOME 证到文件系统根，每进程一次。
- 读回 `readImageFile` 三级校验：摘要、`probeImage` 头字段对照 ref 的
  mediaType/bytes/宽高。头探针不解码像素——admission 曾对同一批字节做过全解码，
  摘要证明字节未变，历史重放因此不承担像素放大成本（src/store.ts L431-458）。
- 文件（`file-store.ts`）逐字节存、零 admission 限额：规范对象在
  `file-objects/<2hex>/<sha>`，再挂一个别名硬链接 `files/<2hex>/<sha>/<leaf名>`——
  模型拿到**以真实文件名结尾**的路径而对象仍按摘要去重。`fileLeafName` 手工剥两种
  分隔符（POSIX 的 `basename` 会原样保留 Windows 客户端整条路径并泄漏进日志）、
  Windows 保留设备名加前缀、255 字节 UTF-8 安全截断。
- 与 [会话持久化](../platform/session-persistence.md) 恰成镜像：append 流是
  "append 尽力而为、flush 是屏障"，附件是"发布即耐久"——可靠方向被引用关系决定：
  日志缺尾可补救，引用悬空即断真。

## 三级漏斗：admission → normalization → request variant

图片字节一生过三道**互相独立**的策略（默认值皆源码常量 [MEASURED]）：

| 阶段 | 归属 | 默认预算 | 超限行为 |
|---|---|---|---|
| admission | 部署（`ImageAttachmentLimits`） | 单图 20 MiB / 64,000,000 px / 单边 8192px；每消息 ≤20 张且合计 ≤200 MiB | **拒绝** |
| normalization | 提供方无关（`NormalizationPolicy`） | 总像素 2048×2048 + 长边 8192 + 编码 4 MiB | **降采样重编码**（不拒） |
| request variant | 每模型路由（`ImageRequestPolicy`） | DeepSeek 路由像素预算 640,000（`detail:'low'`=512²） | 按路由再缩；请求总预算超限**换下旧图** |

admission 校验链：canonical base64（解码再重编码比对，非规范形态直接拒）→ 声明
mediaType vs 魔数签名 vs **全解码事实**三层一致（`IMAGE_TYPE_MISMATCH`）→ 解码事实
含 EXIF 方向（orientation 5-8 会交换轴，宽高按"观看者感知"报告）、动画、元数据有无、
深度/色域/alpha。

normalization（`normalizeImage`）：源字节若已"干净"（非 GIF、单帧、无元数据、uchar、
srgb、全部限内）**逐字节直通**；否则 rotate→sRGB→inside 缩放（不放大）→ 质量梯子
`[85,75,60]`（有 alpha 走 WebP、否则 JPEG；WebP effort 固定 0——注释：深搜多花
3-4 倍时间省约 5%）取第一个入限者；梯子耗尽取最小者，提供方字节帽交给路由层强制。
缩放过则 `originalDimensions` 记入引用，供后续坐标换算。

request variant（`request-image.ts`）：

- `variantId = sha256(变换版本 'request-image-v5' ⊕ attachmentId ⊕ 路由预算 ⊕ 编码器
  参数表)`——同策略重放**字节一致**；磁盘缓存在 `request-images/<2hex>/<hash>`。
- 同 variant 的并发请求经 `SharedRequest` 合并（引用计数、各等待方独立取消、最后一个
  取消者才 abort）；原生编码由 `CompressionLimiter` FIFO 限流（默认并发 2、上限 8）。

## 图片如何抵达模型（消费侧）

- 日志里只有引用：`ImageBlock` 角色中立，但现行生产适配器输出声明 text-only，
  图片实际只出现在用户消息（`packages/llm/llm/src/types.ts` 注释）。
- `FileBlock` **从不原生抵达 provider**：请求组装把每处出现投影成确定性句柄文本
  （名字/字节数/只读路径），持久日志保留结构化引用供展示与授权判定。
- 适配器逐路由调 `readImageRequest(ref, policy)`（llm-deepseek 与 llm-pi-ai 同此路），
  预算取自模型目录（`imagePixelBudget`/`imageMaxBytes`，text-only 目录项声明即加载报错）。
- 请求级超限：`offloadRequestImagesWithPolicy` 按**最旧优先**把图片换成占位文本
  （DeepSeek 默认 raw 字节帽 20 MiB、张数帽 600 [MEASURED]）；`request-pricing.ts` 用
  **同一 offload 顺序**复算视觉 token 与影子计价——计量与实发内容严格一致。
- DeepSeek Files API：`variantId → fileId` 持久上传索引（formatVersion 3；scope =
  sha256(baseURL ⊕ 分隔符 ⊕ apiKey)——**哈希派生命名空间，不落密钥**），过期刷新、
  跨进程文件锁；小图可走 base64 内联（representation 'file'/'base64' 两路）。
- `resolveImageAttachmentAccess`（llm 包）把附件宿主路径映射为工具执行世界内的
  只读路径提示——附件对模型与工具开同一扇窗。

## read_image 工具链（`fs/tool-fs`）

- **组合条件注册**：`ctx.inject(['attachments'], …)`——没有挂载持久仓就没有这个工具，
  fiber 卸载即撤回（HMR 测试断言撤回顺序）；执行期仍 `ctx.get` 复核。
- 路由门比 Host 上传预检**更严**：当前生效路由必须显式声明 `image` 输入模态，解析不出
  路由也拒——"能读图的前提是这个模型看得见图"，不把退款留给适配器报错。
- 执行序不变量：全部静态门在任何 fs I/O 之前（拒绝不泄漏半读/孤儿写）；无扩展名路径
  由魔数嗅探（PNG/JPEG/GIF/WEBP 签名）；字节经 `ctx.fs.readBytes` 读取，帽
  = min(单图, 每消息合计)。**读源字节走 fs 后端（受沙箱/观测策略仲裁），持久化不经**
  ——附件仓是宿主侧服务 I/O，这条与 [沙箱执行](./sandbox-execution.md) 的边界值得说破。
- 返回前 `saveImage` 已完成：`tool/result` 事件追加时引用即指向不可变对象。尺寸/字节
  类错误码统一转成"降采样后再读"的可补救文案——超大图**绝不入持久历史**，否则它将
  搭每一轮后续请求去撞提供方墙。
- 产物是双块：envelope 文本（`<path>/<type>/<content>`，降采样时附坐标倍率建议，
  两轴取整一致才给单一倍率）+ 图片块本身；`presentationMeta` 只存 path、**不复制**
  附件引用——content 已携带完整引用，复制会造成一个事实两份记录，且 post-execute 钩子
  合法替换 content 后旧副本变脏。
- `isConcurrencySafe: true`：内容寻址写入幂等，并发读同一文件不可能冲突。

## spill：一个方法的服务与它的两个武装

三包分工：`spill`（词汇+Service Definition，`ctx.spillStore`）、`spill-local`
（宿主后端）、`spill-policy`（消费者插件）；预览机制另属 `util/output-retention`
（`TextRetainer`），seam 不管预览、不管保留策略、**也不管检索**。

- `saveText` 是唯一操作；owner 只是存储命名空间：fork **继承 seed 日志中的既有定位符**，
  不复制不重属，fork 后新 spill 归子会话（types.ts 注释）。
- 模型侧 arm（`tools/post-execute`，`prepend` 且先 `next()` 委托）：只对"最终结果是
  纯文本且 > `maxInlineBytes`"起效；显式跳过嵌套组合调用（值归日志 arm）、value 替换
  pass（展示与替换互斥）、`read` 工具（防 read→spill→再读死循环）。无 owner/无后端/
  保存失败 → warn 后保内联。
- 预算代数：先按**最坏数字位数**给 notice 定价并预留，preview 预算 = cap − reserve
  （head=`ceil(budget/2)`、tail=floor），保证替换文本**永不超 cap**；连 notice 都超帽
  （帽太小或 spill 根路径太长）就保内联，已写的 spill 文件留作孤儿待清扫。
- 第二 arm `tools/ptc-dispatch-log`：把 PTC 子调用结果的**日志副本**同样压成
  preview+locator，程序返回值原样不动（已整体越过 worker 边界）；日志 arm 不跳 `read`
  ——日志副本不是模型上下文，而 read 恰恰是制造巨型日志的头号工具。两 arm 共用同一
  替换函数，产出字节一致的投影（见 [PTC 与 code-runtime](./ptc-code-runtime.md)）。
- 本地后端：root=配置值或 tmpdir 懒建 `dsh-spill-*`(0700)；对象路径
  `session-<sha256(sessionId)[0..12]>/<随机12hex>-<encodeSegment(suggestedName)>`。
  `encodeSegment` 对全体 UTF-16 字符串**单射**：安全字符保留、其余 `~XXXX` 转义、
  `~` 自 escapes、`.`/`..` 整段转义、空串→`~`；`open('wx',0600)` 排他写挡植入符号
  链接。定位符按原样渲染进 notice，`retrievalHint` 提示用 read/grep 取回——消费方
  被明令**不解析**定位符。
- 清理=激活后一次尽力而为的启动清扫（插件 fiber 持有、disposal 等静默）：mtime 超
  `cleanupPeriodDays`（默认 30，0=关）即删、空会话目录剪、历史默认根一并发现清扫，
  活动 root 永不整体删除；POSIX 跳过他人可改写的 root/目录。留保是故意的：恢复或
  fork 的会话在老化前仍可能引用旧定位符。
- 工具也可不经 policy 直接 spill：`grep` 命中数封顶时把**完整排序列表**存仓并附定位符
  （tool-fs-search；`ctx.get('spillStore')` 机会性查找、缺仓降级为截断提示）。

## 相关

- 概览与计量：[上下文工程](../llm-layer/context-engineering.md)
- 引用块所在的事件语义：[会话事件日志](../agent-runtime/session-event-log.md)
- post-execute 瀑布：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.md)
- 文件块"句柄文本投影"与表层改写：[表层改写与压缩](./surface-compaction.md)

## 诚实边界

- 数字皆为源码常量直读 [MEASURED]；完整图片提交/拒绝文案未端到端真机复跑。
- Files API 的过期刷新调度与配额清理（`fileQuotaCleanupBatch` 细则）未逐行读，
  仅核实记录形态、scope 派生与索引格式版本。
- llm-pi-ai 各 provider profile 的 offload 参数差异未展开（与 deepseek 共享同一
  offload 助手已核实）。
- sharp/libvips 系外部原生依赖：编码器行为（如 WebP 全不透明 alpha 平面的省略豁免）
  以源码 JSDoc 与测试断言为准。
- Windows 目录耐久依赖 NTFS 日志系源码注释声明（该分支被 v8 ignore 排除在覆盖率外），
  未实测。
