---
title: "文件系统深读：观测策略、本地后端与工具层内幕"
tags: [dsh, fs, observation-policy, ripgrep, atomic-write]
status: active
license: CC-BY-SA-4.0
evidence: "packages/fs/ 全组 7 包 29 个 src 文件直读 [MEASURED] + docs/subsystems/filesystem.zh.md；打包 rg 真机复放（win32-x64）"
updated: 2026-09-05
---

# 文件系统深读：观测策略、本地后端与工具层内幕

> [概览篇](../execution/filesystem-and-sandbox.zh.md)画出了 fs seam 与沙箱的地图，
> [沙箱执行内幕](./sandbox-execution.zh.md)走读了内核边界那半边。本篇补齐剩下的两大块：
> **fs-local 后端的身份/并发/原子发布机制**，以及**观测策略插件与 5 个模型工具的实现细节**。
> 沙箱仲裁、令牌构造、denial 分类不在本篇重复，链接指回。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| "targetKey 不透明，消费方禁止解析" | 证实+机制：本地后端 targetKey=`realpath`；文件缺失时 realpath 最近存在祖先再拼回缺失后缀，**创建前后身份稳定**（`fsio.ts` 的 `resolveLocalTarget`） |
| "observation policy 占了 waterfall 单槽" | 证实+定性：先到先得"是部署约定，非强制不变式"（`fs/index.ts` 注释原文）；状态为 `WeakMap<session, Map<targetKey, FsObservation>>` |
| "先读后写/编辑" | 证实到文案：两个守卫码各有专属补救话术（`remediateFsError`），code 原样保留供机器路由 |
| "workspace-write 重规范化后 containment" | 证实+自述：模块注释明言"**信任代码里的策略检查，不是内核边界**"；残余 TOCTOU 靠重规范化收窄并被威胁模型接受 |
| "13 个稳定 FsErrorCode" | 证实 [MEASURED]；另发现**搜索组有独立的 4 码 `SEARCH_*` 词汇**，刻意不并入 FsErrorCode（spawn 型发现≠provider 操作） |

## FsTarget 身份力学：realpath、祖先回退与 symlink 语义

- `resolveLocalTarget(cwd, path)`：displayPath=拼接出的绝对路径（**不**解链接，给人看），
  targetKey=realpath（给机器守身份）。目标不存在时对最近祖先 realpath + 重拼缺失段；
  POSIX 的 ENOTDIR（路径段穿文件）直接判 `FS_NOT_FOUND`，而 **Windows 把同一情形报成
  ENOENT**——回退路径上补一次 `stat().isDirectory()` 修复语义分歧 [MEASURED: 源码注释]。
- 所有读写编辑 I/O 都走 targetKey，所以**经 symlink 写入更新的是目标文件，链接本身不被替换**；
  同一文件的多个别名共享新鲜度守卫。
- `lstat` 刻意保持**路径形**（收原始 path 而非 target）：resolve 会跟随末段链接，
  有信任边界规则的消费方先 lstat 见 `symlink` 即在解析前拒绝。
- `processPath/processPathFromHostPath/fileUrl/contains` 是跨能力坐标出口——
  本地后端里 `processPath = String(targetKey)`，宿主绝对路径直通（isAbsolute 即映射）；
  远程后端才需要真映射（fs-e2b 见概览篇的提供方配对表）。

## 版本 token 与字面编辑：FsVersion = dev:ino:size:mtimeNs:ctimeNs

- `probe()` 用 **bigint stat**，`versionOf` 拼 `dev:ino:size:mtimeNs:ctimeNs`
  五元组——纳秒级时间戳防同秒改，ino/dev 防"删了重建同名文件"骗过守卫。
  消费方永远不解读，只回交。
- `applyLiteralEdit`（fsio）：0 命中→`FS_EDIT_NOT_FOUND`；>1 且非 replaceAll→
  `FS_AMBIGUOUS_EDIT`；`countOccurrences` 不重叠计数（index += needle.length）。
- 行尾三部曲：匹配一律在 **LF 归一化**文本上做（CRLF→LF，孤立 CR 不动）；
  `detectLineEndings` 取头 4096 字符样本按 CRLF/LF **计多数**定风格；写回
  `restoreLineEndings` 先再归一再换回，防已是 CRLF 的内容被双重转成 CR-CRLF。
  write 的 `before/after` 都是 LF 口径——CRLF 文件整体覆写不能被呈现成"每行都改了"。
- editText 的守卫顺序是**先验版本后匹配**：陈旧内容报 `FS_STALE_VERSION` 而非
  对新内容报"找不到"；目标缺失在带守卫/裸编辑两条路径下都报 STALE。

## per-targetKey FIFO 锁：并发变更的确定性

- `LocalFileSystem.withLock(targetKey, op)`：每 key 一条 promise 链（前序失败也接链，
  `prior.then(op, op)`），执行完仅当自己仍是链尾才从 Map 删除（自清洁）。
- 后果：同一文件的并发 write/edit **确定性排队**——一个赢，其余见新版本被 STALE 拒绝；
  "读→守卫→写"临界区在锁内不可交错。这也是 read 工具
  `isConcurrencySafe: true` 的底气：观测与变更赛跑是安全的，
  因为守卫复查在提供方锁内重做（fail-closed）。
- `writeText` 流程：probe（非普通文件拒 `FS_NOT_REGULAR_FILE`）→ 守卫判定 →
  尽力预读 diff 基准 → 原子发布 → 再 probe 取新 version。

## 原子发布：私有 staging 目录与三条发布路线

- 暂存：目标目录内建 `.<名>.<pid>.<uuid>.tmpdir`（`mkdir 0o700` + 显式
  `chmod 0o700` 双保险），临时文件 `<名>.tmp` 以 `'wx'` 0o600 独占创建，
  写完 `handle.sync()`，再 chmod 回原 mode（保留 POSIX 位）。
- 发布三分叉：
  1. **createIfAbsent** → `link()` 硬链无覆盖原语；EEXIST 或竞态后目标已存在时
     检查条目再分类（目录/特殊文件→`FS_NOT_REGULAR_FILE`；文件→`FS_NOT_OBSERVED`），
     **检查后验**避免把"缺硬链支持的平台"误报成碰撞；
  2. **Windows 覆写既有文件** → 预拷贝目标 DACL 到 temp
     （`GetFileSecurityW/SetFileSecurityW`），再 `ReplaceFileW`
     （koffi 懒加载 advapi32/kernel32）保描述符语义；ENOENT 回退 rename；
  3. **普通覆写** → `rename()`。
- 提交点之后 staging 清理失败**不回滚成功**（"已提交的写入不能被呈现层残留变成失败"）。
- diff 基准 `readTextForDiff` 是尽力而为：任何 IO 故障/超限/被删只回 `null`
  （呈现降级为全文件 diff），**绝不杀死已决定的写入**；唯独用户取消（abort）照常上抛。
  上限 `diffBasisMaxBytes` 默认 10 MiB [MEASURED: 常量]；多读 1 字节检测 stat 后增长。

## 观测策略插件：三决策、同步契约与 invariant 伴生

- `fs-observation-policy` 不注册服务、不 inject——只挂 3 个 `fs/*` 事件监听，
  全部状态在自己那两层 Map 里。**没它时 seam 完整可用**：write 无条件创建/覆盖、
  edit 无条件替换——策略是叠加态，不是必需品。
- owner 推导链：工具把 `exec` 原样当不透明 `object` actor 传入事件 → 插件窄化成
  `FsObservationActor{agent?.session?}` 取 `agent.session` 做 WeakMap key——
  **不 import tools/agent/session 任何包**（seam 三件套的解耦标本）。
  无 agent 的直调=可自由读，但**永远无法获得写授权**。
- 三决策（`ObservedStateGate`）：
  - `writeIntent`：未见/确认缺失 → `createIfAbsent`；已见 →
    `replaceIfVersion(observed)`；
  - `editIntent`：未见 → 抛 `FS_NOT_OBSERVED`；缺失 → `FS_NOT_FOUND`；
    已见 → 版本守卫；
  - `observe`：记 present(version) 或 absent。
- 两个易被忽略的实现细节：守卫决策包在 `Promise.resolve().then` 里——**同步抛变成
  rejection**，符合 waterfall 的 Promise 契约不逃逸；waterfall 单槽不 `next()`，
  完全决策而非组合。
- `fs/observed` 契约是**同步、只记、不抛**：emit 不 await，抛异常的监听方可能顶掉
  read 本该返回的错误、或把已成功的变更翻成 isError——WeakMap.set 天然满足。
- 处置即清（换新 WeakMap）：HMR 重载后策略状态从零开始且可观测。
- 伴生 `fs/src/invariant.ts`：`fs-invariant` 插件 inject invariants 服务，全局监听
  `internal/dispatch`，对三个 `fs/*` 事件断言 targetKey/displayPath/version 非空——
  **事件词汇表的运行时不变量**。

## tool-fs 四工具：窗口化、授权与回放契约

- **read 一次 stat 三用**：缺失→先 emit absent 再抛 `FS_NOT_FOUND`（缺失观测只授权
  带守卫重建，不授权编辑）；存在→类型闸门（非普通文件 `FS_NOT_REGULAR_FILE`）+
  size 路由 + 新鲜度授权。**没有 full/partial 视图之分**——任何窗口化读取都携带
  stat 的 version emit present，文件没变就能授权后续 write/edit。
- 窗口三 cap 默认：2000 行 / 单行 2000 字符 / 选区 50 KiB [MEASURED: 常量]，
  全部走 plugin Config 可部署改写（构造期正整数断言）。
  `buildWindow` 是流式扫描器：行缓冲 cap=maxLineLength+1（一个无换行的巨型单行
  也撑不爆内存）；**字节 cap 命中后继续扫描只为数总行数**，totalLines 恒精确；
  offset 过 EOF 抛 `FS_NOT_FOUND`（空文件 offset=1 豁免）。页脚三分支
  （capped/继续/EOF）各给下一条 offset 建议。
- 授权与渲染解耦的少见的另一面：策略插件**不做**任何 fs IO，窗口化只在工具侧；
  提供方永远看不到 offset/limit。
- **write 不 stat**：waterfall 拿意图直接 `writeText`（守卫复查在提供方锁内），
  模型可见输出只有一行 Created/Updated 确认——**不回显内容**；before/after 走
  结构化 value → `presentationMeta` 的 hunk diffs（每 hunk 3 行上下文；
  unified-diff 的无换行符标记行剔除、不混入内容）。
- `remediateFsError`：`FS_NOT_OBSERVED` 统一成"没读过——先读再重试"（路径入话术）；
  `FS_STALE_VERSION` 保留提供方原因追加"重读后重试"。code 不变，UI/重试按码分支。
- 每个工具自带提示词存在感：`ctx.systemPrompt.section` 按
  `getSectionOrder('TOOL_READ')` 等槽位注入用法纪律——工具自身管自己的提示词段，
  组装机制归提示词组装专题篇。
- 回放防御密度值得单独记：`readMetaFromMeta` 除形状外还校验语义（offset≥1 整数、
  totalLines≥0、行号严格递增、不越 totalLines），任何违例**降级通用卡而非抛**——
  旧日志里过时输出不得炸历史查看器。`langFromPath` 用 `Object.hasOwn` 查表，
  防特殊文件名把原型成员塞进 meta 的 JSON 校验。
- **read_image 是组合条件工具**：`ctx.inject(['attachments'])` 内注册——没挂
  attachment store 的部署里它根本不存在；执行体仍为直调者复查。门禁顺序刻意严格：
  所有预读门（扩展名/部署媒体类型白名单/**路由显式声明 image 输入**）在文件 IO 前
  跑完，拒绝永不留下半次读取或附件写入。无扩展名路径放行到签名嗅探
  （PNG/JPEG/GIF/WEBP magic bytes），最终解码裁判在 attachment 服务；
  byteCap=min(单图上限, 单消息图片总额上限)；**先落盘后返回**——tool/result 事件
  追加时必须引用已持久对象。6 类 `AttachmentError` 各配可恢复话术
  （尺寸/像素/字节超限→"缩小后再读"；16-bit PNG 转换失败、签名与字节打架→
  "改名或转格式"）。

## fs-sandbox：只管接口衔接（仲裁内幕指回沙箱篇）

- 换世界=换提供方：装载 `fs-sandbox` 顶替 `fs-local` + 提供
  `ctx.sandboxPolicy`，模型工具零改动。基类 `sandboxMode` getter 是
  **能力事实**：裸 local 报 undefined，围栏后端报部署默认档。
- 工具层据此**组装期门控 schema**：有围栏才把 `sandbox_permissions/justification`
  字段对铺进 write/edit 的 parameters——无围栏后端时校验器直接拒收这两个参数。
  bash 与 fs 共用同一套提权词汇（`ESCALATION_TARGETS/approveEscalation`/
  `sandboxDenialMarker`），审批链全节见概览篇。
- `checkedTarget` 的关键不变量：**被检查的身份=被变更的身份**——workspace-write
  现场重 `resolve`（realpath 反映刚被换掉的祖先链接）、containment 通过后才把
  **fresh target** 交下去，杜绝 check-here-write-there。`read-only` 全拒 mutation
  （读永放行），`danger-full-access` 透传。
- containment 判定 `isPathUnder`：词法快路径（大小写敏感性按平台）→ 不等则
  **祖先逐跳 dev/ino 身份比对**，Windows 8.3 短名与大小写别名骗不过去；
  真机复放见 [沙箱执行内幕](./sandbox-execution.zh.md)。

## 搜索组：打包 rg 直连、零 shell 层

- `glob/grep` 由单一插件注册（`tool-fs-search`），inject
  `tools/systemPrompt/subprocess`——**刻意不 inject fs**；spill 走
  `ctx.get('spillStore')` 机会读取。执行固定为 `ctx.subprocess.spawn`
  前台直调：不进 shell、不起后台任务、不经任何引号层。
- 二进制随 npm 依赖交付（`@vscode/ripgrep`；真机复放
  `@vscode/ripgrep-win32-x64@1.18.0` [MEASURED]）；单文件打包构建改用
  `<exe>-rg(.exe)` sidecar（原生二进制无法从 pkg 虚拟 FS 里 spawn）。
  路径解析**延迟到首次调用**：缺二进制=当次 `SEARCH_FAILED`，不炸 Loader 组合。
- argv 安全三件套：`--no-config` 置前——宿主 `RIPGREP_CONFIG_PATH` 或二进制旁
  `rg.conf` 可注入 `--pre` 让 rg 对每个匹配文件执行**任意预处理命令**，
  这是真实的提权面；模型值一律 `--flag=value` 形态；搜索根放 `--` 之后，
  dash 开头路径永不解析成 flag。grep 固定 `--json` NDJSON，非 match 记录
  （begin/end/context/summary）视为框线跳过，缺字段=整次 `SEARCH_FAILED`
  （raw 输出是内部运输件，不容"部分成功"）。
- 退出码契约：0 有结果、**1 是"成功但零匹配"**、其余分类成 4 码
  `SEARCH_INVALID_PATTERN / SEARCH_FAILED / SEARCH_RAW_OUTPUT_OVERFLOW /
  SEARCH_ABORTED`。
- 七层预算 [MEASURED: 常量]：raw stdout 20 MiB、grep 250 条内联、单行预览 2000B
  （保 UTF-8 边界）、glob 100 路径、presentationMeta 64 KiB、stderr 64 KiB、
  协作超时 30s + 终止宽限 3s。meta cap 有独立理由：spill-policy 只缩 `content`
  不缩 `meta`，而 meta 随日志持久并**每请求重发**——`capMetaBytes` 尾部整组丢弃
  并标 truncated，单组超限也保留（宁大勿空）。
- glob=`rg --files --sort=modified --no-ignore --hidden` + 六个 VCS 元数据目录
  （.git/.svn/.hg/.bzr/.jj/.sl）各**两条**反向 glob：裸形在遍历中剪枝，
  `/**` 形管"搜索根本身就在目录内"时剪枝不触发的场景。超 cap 的两种呈现：
  默认取 mtime 头部，或按顶层条目**轮询抽样**（`sampleAcrossTopLevel`，
  每条目先占一槽再发第二槽；开关 `sampleOverCapGlobResults` 无默认值、部署必填）；
  两种模式下完整结果都尽力溢写为格式化文本（`trySaveFormattedResult`：
  无后端/无 session owner/写失败只 warn——搜索成功不因存储缺席而翻车）。
- 显示相对化 `toWorkdirRelative` 纯展示；"返回的路径能直接续 read"是 v1
  **部署前提**（workdir 与 fs 根同工作区），源码注释明言不做运行时校验。

## str_replace_editor：兼容层的另一条编辑流水线

- 四命令 view/create/str_replace/insert，面向外部协议的兼容层：路径**强制绝对**
  （相对路径报错话术直接给 `/` 前缀建议）；view 支持 `view_range`
  （末位 -1=到 EOF）；目录 view=2 层深、排除隐藏项/node_modules/Python 缓存目录、
  码点排序、`maxOutputChars` 截断加 `<response clipped>` 哨兵。
- 与 tool-fs 的关键实现差异——**变更绕开提供方 editText**：工具侧
  `readText → indexOf 数命中（多命中报行号清单）→ 拼接 → writeText(replaceIfVersion)`。
  匹配在**未归一化的 readText 原文**上做，跨了提供方的 LF 归一化与单一临界区
  （靠 stat version 补偿新鲜度）——一套 seam 上并存两条编辑流水线，兼容层的固有代价。
- waterfall 默认 thunk 也不同：create 在无策略时兜底 `createIfAbsent`
  （tool-fs write 兜底是无条件）；str_replace/insert 有 intent 用 intent 的 version、
  无则用本次 stat 的 version——**该工具永远带守卫**，观测缺失照样被
  `FS_NOT_OBSERVED` 拒。
- 观测照样 emit（view 命中→present；缺失→absent+NOT_FOUND；变更成功→present 新版本）。
- 值得记的不对称：MutationPolicy 只做 resolve/mapError，**参数里没有提权字段对**，
  `FS_SANDBOX_DENIED` 只贴裸 marker 不附同轮升级提示——**这个工具被沙箱拒了无法
  原样重试提权**，与 write/edit 行为不同（是否有意为之未考证，见诚实边界）。

## 诚实边界

- owner 取 `agent.session`：同一会话里多 agent 是否**应共享**观测集合，源码未言；
  本篇按实现如实记录。
- str_replace_editor 无升级字段是设计还是疏漏，无注释佐证，未闭环。
- `ReplaceFileW` 的 flags=0/backup=null 语义组合未与平台文档逐条对照（按源码直录）。
- 远程后端 fs-e2b 不在本篇——见概览篇提供方配对表与 [远程世界](../execution/remote-and-code-runtime.zh.md)。
- 真机复放限于 rg 版本/输出与 containment 指认（转引沙箱篇）；Windows DACL 三分叉
  未在本机做故障注入实测。
- 各默认常量（2000/50KiB/250/100/20MiB/30s 等）为写作时 checkout 值，上游可调。

## 相关

- seam 地图与提权审批流：[文件系统与沙箱](../execution/filesystem-and-sandbox.zh.md)
- 内核边界与 denial 分类：[沙箱执行内幕](./sandbox-execution.zh.md)
- 工具注册表（本组工具的宿主）：[工具流水线](../agent-runtime/tools-pipeline.zh.md)
- 图片附件与溢出词汇：[上下文工程](../llm-layer/context-engineering.zh.md)
- 远程后端与 LSP：[代码运行、LSP 与远程世界](../execution/remote-and-code-runtime.zh.md)
