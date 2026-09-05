---
title: "沙箱执行内幕：仲裁、令牌与拒绝分类"
tags: [dsh, sandbox, windows-acl, landlock, denial]
status: active
license: CC-BY-SA-4.0
evidence: "packages/sandbox/* + native/landlock-run + packages/{fs,shell}/* 直读；windows-acl 侧为本机真机实测"
updated: 2026-09-05
---

# 沙箱执行内幕：仲裁、令牌与拒绝分类

> 概览篇 [文件系统与沙箱](../execution/filesystem-and-sandbox.md) 画出了 seam 地图；本篇回答它的下游三问：
> 多后端如何仲裁、denial 与 runner 失败如何分类、各后端把"限制"兑现到什么程度。
> 证据基线：DSH 仓库源码直读 + 本机真机实测（即在 windows-acl 沙箱 workspace-write 内）；
> 行号以当前 checkout 为准，数字标 [MEASURED] / [ESTIMATED]。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| "工作区 SID=路径哈希确定性生成"（文档级描述） | **证实+实算**：`SHA-256(规范化路径 utf8)` 取字节 0/4 处小端 u32、各 `mod (2^30-1)+1` → `S-1-4-<u32>-<u32>`（`workspace-sid.ts:35-40`）；本工作区实算与 `icacls` 读到的真实 NTFS ACE **逐字节一致** [MEASURED]，temp SID 尾缀 `-1` 域分隔实算亦一致 |
| "koffi 调 `CreateRestrictedToken`/`GetKernelObjectSecurity`/`AddMandatoryAce` 类" | **证实但部分推翻**：`CreateRestrictedToken` ✓；**无 `GetKernelObjectSecurity`、无 `AddMandatoryAce`**——实际走 `GetNamedSecurityInfoW`+`SetEntriesInAclW`+`SetNamedSecurityInfoW`（`acl.ts:122-179`），"合并 entry 进 ACL"交由 AclAPI 库函数而非手工 ACE 拼装；仅读侧 `hasExactGrant` 手工遍历 ACE |
| "denial 与 runner-failure 分类严格分离" | **证实到行**：`bash-sandbox/src/index.ts` `run()` L100-113——abort 先行 → runner-failure 抛 `SANDBOX_UNAVAILABLE` → denial 仅标记；后台 `onProcessDone` L150-167 同一优先序 |
| "containment=词法快路径 + dev/ino 祖先走查" | **证实+实测**：`containment.ts:58-76`；含 8.3 短名段（`…~1` 形态）的祖先路径对长名根**词法判 false、走查第 2 跳 dev/ino 命中放行**（真机复放）[MEASURED] |
| "会话模式=仅日志事件+投影折叠" | 证实但**归属修正**：事件声明在 `session-mode.ts:33-38`，投影单元却注册在 **sandbox-policy 包**（`index.ts:132-138`），不在 session-mode 文件里 |

## sandbox-local：仲裁链与三段式分类学

- `packages/sandbox/sandbox-local/src/index.ts`（567 行全读 [MEASURED]）平台链
  `PLATFORM_CHAINS` L159-166：linux=[bwrap, landlock]、darwin=[seatbelt]、
  win32=[windows-acl]；未知平台=空链 → `confine()` 抛 `SandboxUnavailableError`
  （fail-closed，**绝不透传原 argv**；消息内嵌各平台修复指引）。
- 探测是**功能性**的（非 `--version` 查询）且**仅链上候选 >1 才探测**（`chainVerdict`
  L499-510，单候选平台免探测；结论按 provider 生命周期缓存）：bwrap 用真实 read-only profile
  跑 `true`；seatbelt 用 `sandbox-exec -p <真profile> -- true`（`sandbox_init` 拒绝即非零）；
  landlock 用 `landlock-run --probe`（超时默认 2000ms）；windows-acl 探测函数存在但产品链
  单候选**从不调用**。
- `probeTimeoutMs` 默认 5000（L255）且强制正有限数（L194-198）：`spawnSync` 的
  timeout:0=无界这一坑被显式堵死。`runnerCommand` 覆盖=**操作员断言语义**：enforcement 恒
  'full'、跳过选择与探测（L317-324），但必须与 `runnerFailureSignatures` 成对配置否则构造即抛。
  enforcement 分级出自 `STATIC_ENFORCEMENT`（L177-187）+ probe 返回（L519-538）：bwrap/seatbelt
  =full、landlock 按内核 ABI、windows-acl 恒 partial。

### 每后端方言与三段式证据链

`DENIAL_SIGNATURES` L205-213（大小写不敏感子串）：bwrap=`read-only file system`；
landlock=`permission denied`；seatbelt=`operation not permitted`；windows-acl 三写法
`access is denied`、`access to the path`、`permission denied`（cmd/.NET/node）。
消费方经 `ConfinedArgv.denialSignatures` 拿方言，**禁配"跨后端并集"**（注释点名）。

`RUNNER_FAILURE_RULES` L231-240——三段式 = **exit 闸 → 信息行整行剔除 → 致命子串**：

| 后端 | exit 闸 | 致命签名 | 信息行剔除 |
|---|---|---|---|
| bwrap | 无（**exit 1 非契约，不设闸**） | `bwrap: ` | 无 |
| landlock | ∈{125} | `landlock-run: ` | 精确整行剔除 `partial enforcement (older Landlock ABI)` 信息行——它含 "permission" 词族，不剔除会误判成 denial |
| windows-acl | ∈{127} | `windows-acl-run: ` | 无（exit 闸防被包装命令自己打印该词误判） |

landlock 的 125 与 windows-acl 的 127 **分家**：命令自身可合法返回 125，须签名行共同判定。
辅助函数（`bash-sandbox/helpers.ts`）：`classifyRunnerFailure` 对 exit null（信号死）或 0
**永不判 runner-failure**，命中返回原始行做 detail（L81-103）；`isRunnerSpawnFailure` 只认
`ENOENT/EACCES`+syscall 字符串+`error.path===argv[0]` 的**正向归因**，且先 `statSync`+`X_OK`
验证 workdir 可用（L15-23，排"是 cwd 烂了"之冤）。窗口分类优先序：**abort > spawn 期 runner 失败 > 结算期 runner 失败 > denial**。

## windows-acl 全栈：SID、锁、令牌、runner

- 服务端 `LocalSandboxProvider`（`packages/sandbox/sandbox-windows-acl/` 10 文件全读
  [MEASURED]）standing grant **每工作区一次**（`workspaceGrants` Map L273）；每
  `(<sessionId>, workspaceRoot)` 一个随机私有 temp 目录（`mkdtempSync` 的 `dsh-` 前缀，L415）
  + 独立 temp SID（`'temp\0'` 域分隔、尾 `-1`，`workspace-sid.ts:35-54`）——防兄弟会话借工作区
  能力进彼此 temp。read-only / 无 sessionId：不建 grant 不带 SID 旗标（L360-367）。
- `assertTempRootOutsideWorkspace`（L393）：temp 母目录在工作区内直接抛——否则子项继承
  standing ACE=全放行。`dispose` 只撤 temp（ACE+目录+SID free），**工作区 ACE 永不撤**=跨会话
  复用缓存；清理失败只 warn 不抛。init 失败面撤销全部 revocable grant、`LocalFree` 所有 SID、
  聚合成 `AggregateError`，standing ACE **不回滚**（预期终态非事故残留）；`grantedPaths` **先记后 grant**。

### ACL 写入与文件锁三坑

- `withPathLock`（`acl.ts:75-109`）per-path 文件锁，锁文件位于临时目录下
  `dsh-acl-locks\<sha256(lower(path))[:16]>.lock`（L55-58）。三坑均有注释实证：**不用 OS 命名
  对象**（普通文件锁）；`LockFileEx` 传**零值 OVERLAPPED 而非 NULL**（koffi 3.1.1 收 NULL 会崩，
  `ffi.ts:142-147`）；`CreateFileW` share 位**不含 DELETE**（可删锁文件=双持锁漏洞）。
- `grantWrite`（L231-244）：读现有显式 DACL → `hasExactGrant` 精确命中则**跳过 apply**——
  O(1) 复用的实现核心（跳过避免 eager inheritance 全树再传播，源码注释估大工作区代价
  "数分钟"级 [ESTIMATED]）→ 否则 `EXPLICIT_ACCESS_W{mask=0x110156, GRANT_ACCESS, OI|CI}` →
  `SetEntriesInAclW` → `SetNamedSecurityInfoW` → 双 `LocalFree`。`GRANT_MASK=0x110156`=
  W|D|DC 去 STANDARD_RIGHTS_WRITE；**WRITE_DAC/WRITE_OWNER 不在内**——被包装进程不能改 DACL
  逃逸（`win32-abi.ts:28`；48B 的 `EXPLICIT_ACCESS_W` 布局由 `abi-probe.cpp` static_assert 钉死）。
- 内存契约（注释自述 "verified the hard way"）：返回的 ACL 指针在 descriptor 分配内部，**只能
  free descriptor**（先 free 后 merge=堆破坏）；ACE 的 SID **内联**在 mask@4+8 处，当指针读会崩
  `EqualSid`（gdb 验证）→ `hasExactGrant` 用 `koffi.decode` 逐字节比 SID（`ffi.ts:198-218`）。

### 受限令牌与 runner 子进程

- `token.ts` 调用序：`OpenProcess`(自身) → `OpenProcessToken` →
  `GetTokenInformation(TokenGroups)` 两查 → 扫属性找 `SE_GROUP_LOGON_ID` → `CopySid` →
  `CreateWellKnownSid(WinWorldSid)` → `ConvertStringSidToSidW`(能力 SID) →
  `CreateRestrictedToken` 三旗标（`DISABLE_MAX_PRIVILEGE`+`LUA_TOKEN`+`WRITE_RESTRICTED`）→
  TokenDefaultDacl 合入一条 `FILE_ALL_ACCESS(0x1F01FF)` ACE。restricting（L204-208）：
  read-only=[logon, EVERYONE]；workspace-write 再加 wsSid 与可选 tempSid。
- **保留 Everyone 的原因**：排除它 → 子进程 DLL 初始化失败 `0xC0000142`、CNG 写失败让 pwsh 崩
  `0xE0434352`（keep-alive 组）。**排除 Authenticated Users**：留着则 WMI/CIM `0x80041003`
  失败链，且系统盘根对 AU 的 `(AD)+(OI)(CI)(IO)(M)` 继承 ACE 构成建树逃逸。**排除
  INTERACTIVE/LOCAL**：Public 目录对 INTERACTIVE 授写=逃逸面（注释 L15-25）。
- **default-DACL 合并的必要性**（L96-146）：受限令牌子进程**新建对象**（匿名 stdio 管道、同步
  对象）的 DACL 来自 default DACL，不含任何 restricting SID → WRITE_RESTRICTED pass-2 拒绝创建
  （spawn EPERM 全灭）。合入 SID 选 `tempSid ?? writeSid ?? Everyone`（**优先私有 temp SID**：
  防他会话 default-DACL 对象携带共享工作区能力）。read-only 下 standing ACE **惰性失效**
  （restricting 列表不带该 SID）→ 模式升降级都 O(1)。
- `runner.ts` argv 契约（L10-14）：`--workspace/--temp/--mode` 加可选成对旗标
  `--write-sid S…`、`--temp-write-sid S…`，然后 `-- <argv…>`；两目录**两模式都验存在**。
  seam 托管对下 runner **重算 SID** 与旗标比对，不匹配即失败退出（L149-154，SID 校验=对 seam 的
  交叉验证）；standalone 则自 `mkdtemp`+自派生 SID+自撤。
- TMP/TEMP 注入：`setEnvironmentVariableW` 把**自己**环境的 TMP/TEMP 改写为私有目录
  （`$TMPDIR\dsh-XXXXXX` 形态）再 `lpEnvironment=NULL` 继承 spawn——绕开实测的"koffi 传显式
  环境块必 `ERROR_INVALID_PARAMETER`"。令牌失败即退（`windows-acl-run: `+exit 127，**永不
  无限制 spawn**）；`SetConsoleCtrlHandler(null,1)` 屏蔽自身 CTRL_C（须活到撤权+镜像退出码）。
- **退出码镜像全 32 位宽实测无截断** [MEASURED]：`0xC0000005` → `GetExitCodeProcess`
  `3221225477` → 父进程原样观察（cmd/`$LASTEXITCODE` 只是签名视图）（L207-218）。

## landlock 启动器：main.c 298 行的 syscall 序

概览篇"约 300 行"精化为 **298 行全读** [MEASURED]。syscall 序（C11+musl 静态链接，
UAPI 结构体自包含定义）：`landlock_create_ruleset(GET_VERSION)` → ABI 号 → `fs_mask_for_abi`
**逐级降格**（ABI1 全集→+REFER(2)→+TRUNCATE(3)→+IOCTL_DEV(5)；MAX_ABI=5）→ create_ruleset →
每条规则 `open(O_PATH|O_CLOEXEC)`+`fstat`（非目录 grant 只留
文件兼容位——`--rw /dev/null` 因此可行）→ `add_rule(PATH_BENEATH)` →
`prctl(PR_SET_NO_NEW_PRIVS)`（强制，**顺手废 setuid 逃逸**）→ `restrict_self` → `execvp`。

- 全失败路径同一出口：exit 125 + `landlock-run: <msg>` 行；**内核不强制=不 exec**（fail-closed）；
  限制**跨 execve 继承**，exec 失败才报错。
- `--probe`：短命进程里对 '/' 施加真 ruleset——**唯一诚实信号**（`--version` 式查询会被"有
  syscall 但拒绝强制"的内核欺骗）；partial 时正式运行在 **stderr** 打信息行（L292）——正是
  sandbox-local 要精确剔除的那行，**两文件互为契约**。
- `LandlockEnforcement='full'|'partial'|'unusable'` 三态归因**故意不可分辨**（缺失二进制 ≈
  禁用的内核 ≈ 非强制内核）。launcher 从平台可选依赖包解析，不可解析时回退**绝对但永不存在**
  的包内路径（防 cwd 劫持选二进制）；全模块**无环境变量覆盖**（`entry/src/index.ts` 明文）。

## policy 热切换："切换即事件"

- `sandbox/mode` 是**仅日志会话事件**（`session-mode.ts:24-38`）；写路径唯一=`setSandboxMode`
  append 一条事件（L53-55，注释 "switch IS its event"），无带外状态；折叠在 sandbox-policy
  注册的 `'sandboxMode'` 投影单元（L132-138），state=最后模式或 null（null=无覆盖）。
- resolve=`request.mode ?? overrideOf(session) ?? defaultMode`（L163-170），部署默认
  `read-only`（L112）；同时产 mode+workspaceRoot+sessionId，workspaceRoot **每模式都带**（即使
  read-only 不消费：调用方 resolve 一次再选执行路径）。`canonicalPath` 用 `realpathSync.native`
  ——JS 实现会先词法折叠 `..`，注释点名差异（`roots.ts:30-55`）。
- 模型可见性：systemPrompt 段 `'sandbox:policy'` 渲染 `renderPolicyContext` 三档文案（L41-55）
  ——模型运行时上下文里那三句 "Current DSH file policy: …" 的**逐字源头**；进日志的是快照，
  重放重建当时模式而不重写稳定 system prompt（模块注释 L8-11）。

## 跨包镜像与消费侧装配

- `pwsh-sandbox` 是 `bash-sandbox` 的**逐调用字面镜像**（`jscpd:ignore-start` 显式豁免查重；
  diff 实测除类名/inject 基类/措辞外逐行一致）。唯一实质差异=confine 包内层 argv：
  bash=`['bash','-c',cmd]`（`bash-local/index.ts:178`）；pwsh=`pwshPath`+`-NoLogo`
  `-NoProfile`-`-NonInteractive`-`-Command`+`ENCODING_PREAMBLE+cmd`
  （`pwsh-local/index.ts:220`）——`ENCODING_PREAMBLE`（L49）即以
  `[Console]::OutputEncoding=…` 开头，与本机工具报错现场**互证**。
- `ShellSandboxInfo{mode,denied,enforcement?,runnerFailed?}`（`shell/src/types.ts`）：danger 在
  executor 层直接 super 透传、**根本不调 confine()**（bash-sandbox L91-94），**无 enforcement**
  （未走围栏无从谈"实际施加程度"）；后台在 `onProcessDone` 落 `proc.sandbox` 条件展开 `runnerFailed`。
  `processFacts` Map（L58-65）每进程一份围栏事实：提供方可能交叠换方言，"共享最新值会把进程分错类"。
- `fs-sandbox` 只围 `writeText/editText`（读全放行；自述"信任代码对模型可控路径的策略检查，非
  内核边界"）。workspace-write 分支**重 resolve 出 fresh target 再 containment、且把 fresh 传给
  底层**（L129-143，杜绝 check-here-write-there）；denial 抛 `FS_SANDBOX_DENIED` 的 `FsError`
  （消息含 displayPath+mode），tool 层映射为两行标记（模板唯一之家 `escalation.ts:71-86`）。
- 提权面：**严格加宽只在执行期查**（`WIDER_MODES`），schema 只钉闭集目标（L28-41）；
  `approveEscalation` 失败有序：严格加宽→approver 存在→agent 存在→request→outcome（L157-189）。

## 本机实测锚点 [MEASURED]

- 内核拒绝的前台结果 `ShellSandboxInfo`：`workspace-write + denied:true + enforcement:'partial'`。
- 子进程 TMP 指向 `dsh-` 前缀私有目录；write 工具越界=两行标记模板逐字命中（fs 家族 subject=
  'operation'、bash 家族='command'）。

## Java 移植观察

1. SID 派生是**纯算法**（SHA-256+u32 取模），任何语言逐行可移植；"SID 不是秘密、能力=ACE 指名
   SID"的模型与语言无关（`workspace-sid.ts:11-14` 明文）。
2. windows-acl 全部依赖面=22 个 advapi32/kernel32 平 API（`ffi.ts` 绑定表即完整清单），JVM 侧
   Panama（JNA 亦可）能 1:1 复刻；三条实测坑要带进设计账本：`GetCurrentProcess` 伪句柄不可
   FFI 直取（要真 `OpenProcess`）；NULL OVERLAPPED 在 koffi 崩（JVM FFI 边界行为未知需自测）；
   显式环境块+受限令牌实测 `ERROR_INVALID_PARAMETER`（规避=改写自身环境再继承）。
3. `hasExactGrant` 的 O(1) 复用是**行为契约级**优化：移植丢失则每次 provision 变全树
   eager-propagation（大目录"分钟"级 [ESTIMATED]）。Java 可用 `AclFileAttributeView` 表达，
   但 SID 相等比较建议按字节（`sameSidAt` 语义）。
4. denial/runner-failure 的**字符串分类学**（stderr 子串+exit 闸+信息行剔除）本质是跨进程边界的
   脆弱契约，DSH 靠"每后端方言常量表+行号级注释+fixture 同步"管理（`RUNNER_FAILURE_RULES` 注释
   点名 partial-landlock 的 snapshot fixture）。Java 版同走子进程 argv 包装将继承同一脆弱面
   ——事实陈述，非建议。
5. `landlock-run` 是**独立于 Node 的原生资产**（musl 静态 C11 二进制+argv 契约+版本化 CLI
   contract 文档钉死）："换语言不换启动器"的活例，任何宿主语言复用同一二进制。
6. 三模式词汇在类型系统把"可 confine 子集"单独成型（`ConfinedSandboxMode = Exclude<…>`，
   danger 不入词汇表），policy 永远显式带 workspaceRoot——Java 密封接口可同样表达。
7. 环境的**双向清洗事实**：子进程 TMP/TEMP 由 runner 改写为私有目录；而 code-runtime worker
   线程内 `os.tmpdir()` 在无 TMP/TEMP/TMPDIR 时返回回退串——宿主环境与注入子进程环境是两套
   事实，移植时"谁给谁注入什么"要画清水位线。

## 诚实边界

- 待核实：`win32-process/src/ffi.ts` 绑定原语层（allocStartupInfo/decodeProcessInfo 等）只经
  `process.ts` 调用面间接确认，未逐行读。
- 待核实：bash-local/pwsh-local 执行器主体的超时、输出上限、信号升级细节（只覆盖 argv 构造与
  `runArgv`/`startArgv` 接缝）。
- 待核实：`ctx.approval` 服务侧（approval/request 应答者链、approval/asked|decided 落日志）——
  只覆盖 escalation 编排层（其调用方）。
- 待核实：bwrap/seatbelt/landlock 仅有源码+注释证据（Windows 本机无法复现探测与 enforcement 分级）；windows-acl 侧全为实测。
- 待核实：崩溃残留 temp 目录的 OS 回收路径；tool-bash/tool-fs 把围栏事实变成模型可见文本的
  装配源码（只由标记模板逐字命中反推）。
- 待核实：Authenticated Users 逃逸面与 CNG/WMI 崩溃码归因系源码注释转述的"他机实测"，本机未复现（需无沙箱对照组）。

## 相关

- 概览地图：[文件系统与沙箱](../execution/filesystem-and-sandbox.md)
- 进程与终端侧：[进程、shell 与终端](../execution/shell-process-terminal.md)
- 同级完整 seam（容器/microVM 世界）：[代码运行、LSP 与远程世界](../execution/remote-and-code-runtime.md)
- 模式的持久与重放：[持久化与崩溃恢复](./persistence-crash-recovery.md)
- 进程内执行与围栏：[PTC 代码运行时内幕](./ptc-code-runtime.md)
- 提权审批的消费侧挂点：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.md)
