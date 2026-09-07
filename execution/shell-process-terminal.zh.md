---
title: "进程、shell 与终端：从一次性 spawn 到持久 PTY"
tags: [dsh, subprocess, shell, terminal]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{subprocess,shell,terminal}/*；docs 同名页；子代理D"
updated: 2026-09-05
---

# 进程、shell 与终端：从一次性 spawn 到持久 PTY

[English](shell-process-terminal.md) | [中文](shell-process-terminal.zh.md)

## subprocess（`ctx.subprocess`）：最底层的显式

- `resolveExecutable / spawn / spawnTerminal`；`SubprocessSpawnSpec` **完全显式零默认**（没有"稍微继承一下环境"的暗通道）；
- **环境清洗**：`scrubbedParentEnv()` 按
  `/KEY|PASSWORD|SECRET|TOKEN/i` 剔除 + `DSH_ENV_PREFIX` 命名空间——
  子进程拿不到宿主机密是**默认**不是选项；`DSH_*` 托管变量的分层供给与
  启动冻结快照语义见 [工程基建](../deep/engineering-base.zh.md) 机制四；
- 终止语义：`handle.terminate()` 是唯一动词（SIGTERM→宽限→SIGKILL，
  **进程树范围**；POSIX 负 pid 组信号 / Windows taskkill /T /F 提供方内消化）；
- 输出：`collected.readFrom(offset)` **非消费式读取器** + 尾保留 +
  `CollectedOutput{text(尾), truncated, spillPath}`——大输出落 spill 仓给路径
  （[上下文工程](../llm-layer/context-engineering.zh.md) 的 SpillStore 在这里兑现；你见过的"完整输出已存文件"即此）。
- 读取语义两副面孔：seam 的 offset 读取**非消费**（可重放），
  `ShellProcess.readOutput` **消费式**（读过即前进）——内幕见 [Shell 与终端内幕](../deep/shell-terminal-internals.zh.md)。

## shell（`ctx.shell`）：一次性执行的词汇表

- `resolve(request) → ShellExecSpec → run/start`；结果四事实**正交独立**：
  `exitCode · signal · timedOut · aborted`（+ `sandbox?` 围栏事实
  `ShellSandboxInfo{mode, denied, enforcement?, runnerFailed?}`）——
  "超时被杀"与"用户取消"永不混为一谈；
- 前台 bash 链：提权参数校验 → `ctx.sandboxPolicy.resolve` →
  `SandboxBashExecutor.run`：danger 直发原始 argv；confined 经
  `confine(['bash','-c',cmd], policy)` 包装（见 [文件系统与沙箱](./filesystem-and-sandbox.zh.md)）；
- **shell 不管后台身份**：返回无任务概念的进程句柄，`job` 语义
  （id=`<kind>-N`、owner 鉴权、first-wins）归 `ctx.jobs` 适配层
  （[后台任务与外部触发](../augmentation/background-and-triggers.zh.md)）——关注点切分干净。
- `runHook` 通道为外部 hooks 生态定制（清洗环境/进程组取消/超时，
  见 [能力供给](../augmentation/skills-mcp-hooks.zh.md)）。

## terminal（`ctx.terminals`）：持久 PTY 会话

- `TerminalBackend/TerminalBackendSession` 提供方注册；node-pty 实现
  （终端分配/前台组/信号/整会话停稳）；**confined 时 PTY argv 同样过
  `confine()`**——沙箱不因交互而旁路；
- `terminal-bash` 的受控 bash 提示符协议：OSC `CONTROLLED_PROMPT`
  标记 + `PROMPT_COMMAND` 重置 PS1；命令就绪检测=提示符/stdin 等待/静默超时
  三路；`@xterm/headless` 仿真渲染拿 scrollback——
  **"让代理像人一样用终端"被做成协议，不是黑盒截图**；
  `TerminalWaitReason = stdin_read | inferred_idle | timeout | session_exit`；
- 有趣的护栏：**会话开启期间禁止切换 sandbox/mode**（WeakMap fence）——
  模式不能中途换车，防止权限与既成事实脱节。

## guard（杂务防线，packages/guard/）

- `timeout-policy`：包裹 `tools/execute` 给声明 `timeoutMs`
  的工具装 deadline（`TOOL_TIMEOUT`）；`MAX_TIMER_DELAY_MS`
  （setTimeout 2^31-1 坑的工程化）；替换结果**仅在自持原因码命中时**发生——
  嵌套外层计时器读作上游取消，不冒充本次超时；
- `repeat-tool-reminder`：重复调用**顾问式提醒不否决**——护栏分"硬闸"和"耳提示"，
  后者不改决策权。

## 相关

- argv 包装与提权：[文件系统与沙箱](./filesystem-and-sandbox.zh.md)
- 沙箱后端与拒绝分类内幕（含 pwsh-sandbox 镜像与 ENCODING_PREAMBLE 互证）：
  [沙箱执行内幕](../deep/sandbox-execution.zh.md)
- 工具六段流水线挂点：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)
- 后台任务生命周期：[后台任务与外部触发](../augmentation/background-and-triggers.zh.md)
- 四象限逐调用镜像、spill 硬化与 PTY 就绪协议内幕：
  [Shell 与终端内幕](../deep/shell-terminal-internals.zh.md)
