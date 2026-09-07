---
title: "文件系统与沙箱：一个执行世界"
tags: [dsh, fs, sandbox, escalation]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{fs,sandbox}/* + docs/subsystems/{filesystem,sandbox}.zh.md；子代理D调研"
updated: 2026-09-05
---

# 文件系统与沙箱：一个执行世界

[English](filesystem-and-sandbox.md) | [中文](filesystem-and-sandbox.zh.md)

> 执行环境全组是**循环之外的可选能力 seam**。核心思想一句话：
> **fs 与 subprocess 提供方共享同一个"执行世界"**——把它们整体指向远程沙箱，
> Bash/PTY/LSP 一起搬家，无需任何提供方专用 fork（architecture.zh.md）。

## fs seam（`ctx.fs`）：能力坐标而非路径

- `FileSystem` 抽象类（`packages/fs/fs/`）：`resolve/processPath/
  processPathFromHostPath/fileUrl/contains/stat/readText/writeText/editText`…；
  `FsTarget{targetKey: Branded, displayPath}`——**targetKey 不透明，消费方
  禁止解析**，跨能力坐标只经 `processPath/fileUrl/contains` 换取；
- **写新鲜度**：`FsVersion` 不透明 token + `FsWriteIntent
  (createIfAbsent | replaceIfVersion)` → 乐观并发（stale 守卫）在文件层兑现；
  13 个稳定 `FsErrorCode`（错误码是词汇表的一部分）。
- `fs/write-intent`、`fs/edit-intent` **waterfall 单槽决策**：
  先返回者赢——本会话"读前必先 read"的 observation policy 就是占了这个槽
  （`dsh-fs-observation-policy`），**策略与实现完全解耦的活标本**。

## 提供方配对 = 世界选择

| 世界 | fs 提供方 | subprocess 提供方 | targetKey 语义 |
|---|---|---|---|
| 宿主 | `fs-local` | `subprocess-local` | ≈ realpath |
| 远程 | `fs-e2b` | `subprocess-e2b` | 远程 POSIX 路径（同取 `ctx.e2b` 句柄） |

容器/microVM/远程 = **同级完整 seam 实现**（成对替换 fs+subprocess），
**不是** `ctx.sandbox` 的提供方——沙箱 seam 只管"与宿主共享文件与内核"的
argv 限制（sandbox.zh.md 的清晰边界）。

## 沙箱 seam（`ctx.sandbox`）：包装 argv 的学问

- `SandboxProvider.confine(argv, policy) → ConfinedArgv{argv, enforcement,
  denialSignatures, runnerFailureRules}`：**逐调用携带策略**
  `SandboxExecutionPolicy{mode, workspaceRoot, sessionId}`；三模式
  （read-only/workspace-write/danger-full-access）**只管文件效果**；danger 直接
  绕过 seam。
- 后端平台链（`PLATFORM_CHAINS`）：linux=[bwrap, landlock]（多候选功能探测仲裁）、
  darwin=[seatbelt]、win32=[windows-acl]（单候选免探测、执行期拒止 fail-closed）；
  结论按 provider 生命周期缓存。
- **enforcement 分级是诚实设计**：bwrap/seatbelt 声明 full；landlock 按内核 ABI
  full/partial（launcher 探测报告）；**windows-acl 恒 partial**（Everyone 保留于
  restricting list + NTFS 硬链接别名）——能力上限写进类型而不是宣传语。
- argv 包装方言：`[runner, profile..., '--', callerArgv...]`；
  denial 分类与 **runner-failure 分类严格分离**（先 runner-failure →
  `SANDBOX_UNAVAILABLE` 抛；后 denial → 标记 `denied`）。
  `native/landlock-run`：约 300 行 C11"先限制自身再 execve"musl 静态启动器，
  内核无法强制则**拒绝运行**（fail-closed），失败退出码 125。
- Windows 侧：WRITE_RESTRICTED 令牌 + 能力 SID 的 DACL ACE
  （`sandbox-windows-acl`，koffi FFI）；**工作区 SID=路径哈希确定性生成**
  （规范化路径 SHA-256 → 两个 u32 取模 → `S-1-4-<x>-<y>`；深读实算与实机
  NTFS ACE 逐字节一致 [MEASURED]。常驻、精确 ACE 跳过让后续 provision O(1)），
  temp SID 每会话随机（域分隔防兄弟会话互借）、dispose 撤销。

## fs 内进程围栏（`fs-sandbox`）

`SandboxedFileSystem extends LocalFileSystem` 只围栏 `writeText/editText`：
danger 透传；read-only 抛 `FS_SANDBOX_DENIED`；workspace-write **重规范化后**
做 containment（防 check-here-write-there），返回 fresh target。containment=词法快路径
+ `dev/ino` 祖先走查兜底（含 Windows 8.3 别名/大小写）。
自述到位：**"信任代码对模型可控路径的策略检查，非内核边界"**——文档不夸大。

## 提权审批流（模型可见的"denied → 一次例外"）

```text
模型看到 [sandbox: file access denied under <mode> mode] + escalation hint
→ 原样重发 + sandbox_permissions + justification（必须成对非空，schema 外再查）
→ approveEscalation：执行期验证"严格更宽"（WIDER_MODES，不信 schema）
→ ctx.approval.request —— approval/request waterfall 应答者链决策（需开启的 turn）
   审计对事件 approval/asked|decided 落日志
→ 仅 allowed-once 返回授权模式，且**只覆盖这一次调用**；其余一切路径抛错未执行
```

`ApprovalOutcome = allowed-once | rejected | cancelled | unavailable`
（**unavailable=fail-closed**）。会话模式=可回放状态：`sandbox/mode` 仅日志事件 +
投影折叠；resolve 优先级=批准的显式 mode > 会话最后事件 > 部署默认（read-only）。
**权限预设**（`ctx.permissionPresets`）= `{sandbox, approval}` 两旋钮捆绑；
`set()` 先记"用户意图"（permission/preset 仅日志）再走各旋钮规范 setter；
加载校验：存在"不施加任何隔离的 shell 执行器"直接抛。

## 相关

- 消费侧（工具/审批挂点）：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)
- 进程/shell/终端：[进程、shell 与终端](./shell-process-terminal.zh.md)；
  四象限执行器与 PTY 就绪协议内幕：[Shell 与终端内幕](../deep/shell-terminal-internals.zh.md)
- code-runtime 与 LSP：[代码运行、LSP 与远程世界](./remote-and-code-runtime.zh.md)
- 档位词汇的 UI 半边：[人机问答与反馈](../augmentation/questions-and-answers.zh.md)
- 探测仲裁、令牌构造、denial/runner-failure 分类学与 landlock 启动器全细节：
  [沙箱执行内幕](../deep/sandbox-execution.zh.md)
