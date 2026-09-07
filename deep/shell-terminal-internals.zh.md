---
title: "Shell 与终端内幕：四象限执行器、spill 硬化与 PTY 就绪协议"
tags: [dsh, shell, subprocess, terminal, pty, guard]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{shell,subprocess,terminal,guard}/* 源码直读 + 本 harness 会话内真机实测；docs/subsystems/{shell,subprocess,terminal}.zh.md"
updated: 2026-09-05
---

# Shell 与终端内幕：四象限执行器、spill 硬化与 PTY 就绪协议

[English](shell-terminal-internals.md) | [中文](shell-terminal-internals.zh.md)

> [概览篇](../execution/shell-process-terminal.zh.md) 讲接缝词汇与分层；本篇下钻一层：
> bash/pwsh 两个执行器家族如何"逐调用镜像"、subprocess 的零配置与环境清洗如何兑现、
> 持久 PTY 的就绪协议是什么、guard 双保险挂在流水线哪里。证据 = 源码直读 +
> 本篇写作时身处真实 DSH 会话的真机实测（常量与路径形均已脱敏）。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| "bash/pwsh local+sandbox 四象限" | 成立；shell 组实测 **10 包**（4 执行器 + seam + shell-env + 4 工具包）[MEASURED] |
| "大输出落 spill 仓给路径" | 实测约 100KB stdout：`truncated=true` + spill 路径回投；内存窗**保尾**，头段只能从 spill 文件找回 |
| "后台 job id=<kind>-N" | 实测：后台 pwsh 调用返回 id `pwsh-1`（kind 来自 `JobKindMap` 声明合并） |
| "DSH_* 托管环境" | 实测快照 4 枚见末节；内置 3 枚 + 插件贡献 1 枚，registry 机制直接命中 |
| "会话开启期间禁切换 sandbox/mode" | 属实（terminal-bash 侧的 WeakMap fence），内幕已录 [沙箱执行内幕](./sandbox-execution.zh.md)，本篇不重复 |

## 四象限：mirror-by-design

执行器家族是刻意的**逐调用镜像**：pwsh 两包源码里成对出现 `jscpd:ignore-start`
注释，明言 "deliberate call-for-call mirror of dsh-bash-local"——平台差异太大，
复制优于抽象，仓库用工具豁免承认这笔债。镜像之上的**差异清单**：

| 维度 | bash 家族 | pwsh 家族 |
|---|---|---|
| argv | `bash -c <command>`（命令字符串域） | `<pwshPath> -NoLogo -NoProfile -NonInteractive -Command <preamble+command>`（单 argv 元素、无中间 shell、无引号层） |
| 环境覆写 | NO_COLOR / TERM=dumb / PAGER / GIT_PAGER 等 5 项 | 3 项——`TERM=dumb` 是 POSIX 概念，**刻意缺席** |
| 编码 | 无需处理 | `ENCODING_PREAMBLE`：命令前置两段 UTF-8 输出设置语句，骑第 1 行以分号分隔——**保住 PowerShell 错误行号不失真**；为最后兜底的老版 Windows PowerShell 而设（新版默认 UTF-8 不受影响） |
| 可执行文件 | 裸 `bash` 走 PATH | 候选链：Program Files PS7 → PATH 各项（去引号，兼容商店安装）→ System32 老版 → 裸 `pwsh`；存在性用 **lstat**（stat 会撞商店应用执行别名的 ACL） |
| 子类挂点 | `runArgv/startArgv`（收 argv） | `argv(spec)` 保护钩子产 argv，再走同名 run/start |

两处点名：pwsh 的可执行解析独立成零依赖模块 `resolve.ts`，因为仓库的覆盖率
门禁探针必须与执行器**共享同一定义**（探针解析不同 = 给错误文件开免检）；
pwsh 执行器内部超时原因码仍写作 `'BASH_TIMEOUT'`——镜像保真的痕迹。

**沙箱执行器**（继承 local 家族）的裁决纪律：

- `danger-full-access` 直通底层，但结果仍带事实 `{mode, denied: false}`——事实通栏不缺帧。
- confined 时整条 argv 交给 `ctx.sandbox.confine()`（包装内幕见[沙箱执行内幕](./sandbox-execution.zh.md)）。
- **runner 失败优先于拒绝分类**：命令根本没跑成，stderr 混着拒绝词也不能误判。
- `isRunnerSpawnFailure` 是正向归因的证据链：code ∈ {ENOENT, EACCES} +
  syscall/path 精确命中 `spawn <argv0>` + 独立排除 cwd 不可入；注释自承 cwd 检查与
  spawn 非原子——"并发路径替换可能改变归因，但**不可能放行一次未受限执行**"。
- 信号死亡不算拒绝（exitCode 为 null 直接 false）。
- `processFacts` 按**进程**存封装事实而非共享 latest-wrap——重叠调用可能来自不同
  provider，事实不能串档。

## 执行默认值与设置热更

`resolve(request) → ShellExecSpec` 补全并封顶，默认值实测于源码 schema [MEASURED]：

| 字段 | 默认 | 语义 |
|---|---|---|
| `timeoutMs` | 120,000 | 前台超时（后台**不适用**超时） |
| `maxTimeoutMs` | 600,000 | 模型可请求的上限 |
| `maxOutputBytes` | 64,000 | 每流内存窗（保尾） |
| `maxSpillBytes` | 64 MiB | spill 全流上限，超了只剩内存尾 |
| `graceMs` | 3,000 | SIGTERM→SIGKILL 宽限（≤`MAX_TIMER_DELAY_MS`） |

设置节命名空间 `'shell'` **归 seam 所有**而非任一提供方：宿主组合恰好一个
`ctx.shell` 提供方（win32 层把 POSIX 行换成 pwsh 行，双挂撞重复服务注册），
于是提供方共享命名空间而不重复登记，**同一份设置文档跨平台都能解析**。
每个字段经 getter 按命令读取，热提交零重建；唯一派生事实（pwshPath 文件系统探针）
挂在 `onChange` 上按声明值变化重探。非法配置**在写入处拒绝**
（`assertServiceable*Config`），不是下一条命令才炸。

超时/取消分类学：一个融合 deadline 同时承载 timeout 与上游取消，
`timedOut` 与 `aborted` 按**第一因**互斥归位——外层 deadline 先到记为 abort，
只有本执行器自持原因码才算 timedOut。

## subprocess：零配置提供方与两副读取面孔

`LocalSubprocessRuntime` **没有任何配置**：一切预算随 spec 显式进来，
"部署可变的选择不属于隐藏的子进程服务默认值"。要点四件：

- **环境基座** `scrubbedParentEnv()`：按 `/KEY|PASSWORD|SECRET|TOKEN/i` 与
  `DSH_*`（Windows 不分大小写——防 `dsh_*` 残留被子进程读成托管名）双清洗；
  PATH/HOME/locale/代理变量幸存。代理变量还要做**还原覆盖**：子 Node 无标志位会
  无视继承的代理名，stdio 型 MCP 服务器会直连而父进程在走代理——同一条 overlay
  恢复用户原值、撤销规范化，`undefined` 直接删名。
- **可执行查找**：含分隔符的相对路径**拒绝解析**（解析基准未定义 → fail loud，
  绝不猜）；Windows 按 `PATHEXT` 展开候选。
- **tail-keep + spill 硬化**：内存窗按 chunk 丢头、超界字节精确修头（诊断尾必须
  持住最后 N 字节）；首次溢出开 spill 文件并把**已收全部 chunk 回放**写入；全流超
  spill 上限弃文件只保内存尾。安全姿态：私有 0700 随机目录 +
  `pid-序号-随机6字节-标签` 文件名 + `'wx'` + 0600——防路径预测、防共享 tmp 预埋
  符号链接；完成态 spill **保留**待外部清理，SIGKILL 的宿主连清理机会都没有。
- **整棵树的生命周期**：句柄从 live 集合释放的条件是 `waitForExit()` **全树**
  消失——trap 了 TERM 的 helper 不能比 fiber 活得久；宿主退出用
  `prependListener('exit')` 同步强杀。终止动词唯一：`terminate()`
  （SIGTERM→grace→SIGKILL；POSIX 负 pid 组信号 / Windows `taskkill /T /F`），
  `win32-process` 包向 Windows ACL 沙箱供 Job Object/FFI 原语（机制见
  [沙箱执行内幕](./sandbox-execution.zh.md)）。

**两副读取面孔**是有意的分工：`ctx.subprocess` 的 collected 读取器按全流字节
offset、**非消费**（独立读取器互不偷增量）；而 `ShellProcess.readOutput()` 是
**消费式**（连续读不重复投递）——后台作业每次轮询要的正是后者，`lossy` 时
指路 stdout/stderr 两份 spill。

## PTY 读侧：注册表、就绪协议与仿真用途

`TerminalSessionService`（`ctx.terminals`）拥有 id、发布、授权与**等待式清理**，
后端只拥有终端机制。授权对象是**确切 Agent 身份**，不是名字也不是可猜的 id；
8 枚 `TerminalErrorCode` 全部机器可路由（`SEND_ACTIVE`/`OWNER_NOT_LIVE`/…）。
`hasOwnerActivity` 覆盖 spawn-to-close 全程——发布与创建之间**没有授权空窗**。
回滚纪律：清理失败不被调用方取消原因替换，双败抛 `AggregateError`。

`terminal-bash` 后端默认值实测 [MEASURED]（并断言边界组合合法，如"handoff
宽限 ≥ 一个轮询间隔——宽限窗里必须还有一轮 poll"）：

```text
rows×cols = 40×160        scrollback = 10,000 行 / 4 MiB     单次读 ≤ 256 KiB
poll = 50ms               exact-probe 延迟 = 150ms
idleSilence = 3,000ms     handoffGrace = 500ms
send 超时 = 30,000ms      disposeGrace = 3,000ms
```

**就绪三路**对应 `TerminalWaitReason`：`stdin_read`（前台进程组在等输入）、
`inferred_idle`（静默 3s + 交棒宽限）、`timeout`，外加 `session_exit`。
提示符协议：受控 shell 每次提示符前发 OSC 标记（`133;D;` 前缀）+ 可打印
`'dsh> '`。源码钉着一个已知竞态：**bash 会在内核把前台组所有权交回之前
就打出 PROMPT_COMMAND**——"看见标记"只是证据，轮询确认 shell 重新掌组才是权威
（TODO `pty-delayed-signal-prompt` 记录在案）。

`@xterm/headless` 仿真的用途常被误读为"渲染给模型看"：实为**终端协议
应答机**——shell 的行编辑器会反查光标位置，emulator 的 onData 把应答序列化
排队写回 PTY；输出侧走流式 sanitizer（剥 CSI/OSC 短序列、跨 chunk 残段保载）。
取尾按字符退行不切码点；`read()` 对 scrollback 做相对最新行的分页有界读。
进程表观察：**每轮就绪轮询至多读一次表**（Linux /proc、macOS fork ps、
Windows Toolhelp32；Windows 存活判定按句柄等待不需要表），但发信号前走
`isAlive` 现值——"批量过滤读快照、信号栅栏读现值"。

## persistent shell：模型看到的"有状态"是标记协议

`tool-bash-persistent`/`tool-pwsh-persistent` 给模型的承诺是"cwd 与环境变量
跨调用保留"，实现 = owner 级 PTY 会话 + **每次调用现场包标记**：

- 会话注册表：`WeakMap<Agent, Promise>` 去重——同 owner 并发首调共享同一个
  创建 promise；初始化只发 `stty -echo`（压回显、**不改提示符**，后端标记就绪
  检测继续有效）。配置默认 timeoutMs=300s、maxOutputChars=16k [MEASURED]。
- 命令包裹：printf 起始标记 → `eval -- <quoted>` → 取 `$?` → printf 结束标记+状态，
  start/end 各带 `randomUUID` nonce。为什么必须单物理行：交互式 bash 对嵌入换行
  先打 PS2，提示符与标记源码会漏进模型可见输出。bash 侧用美元单引号转义域；
  pwsh 侧用其反引号转义符（反引号加美元抑制展开、反引号加 n 表换行、
  反引号加 e 表 ESC、CR 直接剥离）——同一目的两套方言。
- 状态回收：等待超时、会话死亡 → 渲染 shell 退出标记 + **静默 reset** +
  一句 `SHELL_RESET_MESSAGE`（"下次调用从 workspace 全新开始"）；截断话术同样
  面向模型自解释（`LOST_PREFIX_MESSAGE` 承认 scrollback 吃过头）。
- 工具层轮询 25ms、后端就绪轮询 50ms——两层两钟，各管各的等待。

`tool-terminal` 则把裸 PTY 直接开成六件模型工具：
`terminal_open/send/read/signal/close/list`；后台 send 注册 job kind
`'pty-send'`；单结果封顶 256 KiB、schema 下限 64B。

## 工具面：组合决定 schema

`tool-bash`/`tool-pwsh` 的参数表**按组合动态生成**：执行器不封装沙箱时
`sandbox_permissions`/`justification` 根本不发布；封装执行器存在而 `ctx.sandboxPolicy`
缺失 → **插件装载即抛**（拆分组合活不过加载）。描述文本按 `enableRunInBackground`
与升级目标拼装；跨调用纪律进系统提示词段（`ctx.systemPrompt.section('tool:bash')`），
单次调用的 schema 不背行为准则。

执行时序的三处讲究：**先审批后执行**（共享 fail-closed 序列，只替换 mode 字段
且必须严格更宽）；**workdir 与 confinement 同一身份**（policy.workspaceRoot 优先于
会话 header cwd，相对路径按 session workspace 解析）；**DSH_* 最后合**——
不受信任的当前事实不可能从宿主进程继承过期值，caller env 也永远压不动托管键。

后台移交 `ctx.jobs`：`jobs.start(kind='bash'|'pwsh', owner, run)`；tool-call 信号的
所有权在 jobs 提交 detached 所有权**之前**留在调用方（动手前先查一次
`exec.signal.aborted`）。`processOutcome` 把非零退出报成 completed（码进 detail）
——"报告而非判错"；TODO `background-infrastructure-outcome` 自承认基础设施失败
目前与 killed 混影。

**标记协议**（模型侧契约）：stdout → `[stderr]` 段 → 拒绝标记（+ 升级提示）→
`[timed out after Nms]` → `[killed by signal: X]` 或 `[exit code: N]`——exit 标记**永远
垫底**，因为 `parseExitStatus` 锚定末尾；命令 trap 了 SIGTERM 再以 0 退出，超时
照样报告。解析归 seam 所有（`dsh-shell` 导出），pwsh 工具复用同一半契约。

## guard 双保险（packages/guard/）

- `timeout-policy`：**合作式**超时闸。工具自己声明 `timeoutMs` 且承诺尊重
  `exec.signal`；wrapper 换上派生 deadline 再在 finally 恢复上游信号
  （post-execute 监听者不会看到本插件已死的 signal）；`timeoutOf` 按自持原因码
  限定——嵌套外层计时器先到 = 普通上游取消，不算本闸超时；超时才替换结果为
  `error.code = TOOL_TIMEOUT` 的结构化 isError（重试/沙箱插件可按码路由）。
- `repeat-tool-reminder`：**顾问式**耳提示，不否决不改写。链键 = 工具名 +
  深度键排序的规范化 args JSON（属性顺序不影响识别）；阈值默认 `[3, 5, 8]`，
  首档温和、后续档点名工具/连击数/参数预览（截 500 字符，只束显示不束识别）；
  **post-execute 计数含被拒调用**——反复锤被拒的调用正是最值得断的环；用户插话 =
  上下文变了，链清零；提醒以 `source:{kind:'plugin', form:'notice'}` 注入——
  标签承重，没标签会被派生历史渲染成用户发言。

## 实测证据（本篇写作时身处真实会话）

- `DSH_*` 快照 [MEASURED]：`DSH_HOME`、`DSH_SHELL=1`、`DSH_SESSION_ID` 为
  shell-env 内置；`DSH_WEB_URL` 是 web 组合里贡献者注册的活样本。
- 无态性 [MEASURED]：前一次调用设置的环境变量后一次读到空，且两次 PID 不同。
- tail-keep+spill [MEASURED]：约 100KB stdout 触发截断，路径形
  `<tmp>/dsh-subprocess-<rand>/dsh-subprocess-<pid>-<seq>-<hex>-stdout.log`；
  探针自己的 LanguageMode 行**被自家保尾策略吃进 spill**——机制自证。
- 语言模式配合 [MEASURED]：workspace-write 组合下 `FullLanguage`
  （read-only → ConstrainedLanguage 出自工具描述与沙箱篇，本会话未亲验）。
- 后台链 [MEASURED]：`run_in_background` → job id `pwsh-1` → `job_output` 增量读 →
  completed / exit 0。

## 诚实边界

- `process-inspector`/`windows-inspector`（486+310 行）只核了头注设计与 seam 承诺，
  未逐行走查树扫描实现。
- PTY 通路（node-pty 分配、`terminal_*` 六工具、交棒宽限）本组合无真机面，
  叙述停在源码级。
- hooks 桥使用 `stdin/env` 受信任通道的细节转引自 seam JSDoc，未查 hooks 源码。
- pwsh 家族与 bash 家族"逐调用镜像"的**全等性**只抽验了头部注释与转义函数，
  未做 diff。
- 工具描述文里的升级纪律长段是**给模型的规范**，本篇只述其形，不为其背书。

## 相关

- 概览层词汇：[进程、shell 与终端](../execution/shell-process-terminal.zh.md)
- 沙箱仲裁/令牌/拒绝分类内幕：[沙箱执行内幕](./sandbox-execution.zh.md)
- spill 的上下文侧兑现：[上下文工程](../llm-layer/context-engineering.zh.md)
- 后台作业生命周期：[后台任务与外部触发](../augmentation/background-and-triggers.zh.md)
- 设置文档与热提交：[配置面](../augmentation/settings-and-credentials.zh.md)
- 工具流水线挂点（tools/execute、post-execute）：
  [工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)
