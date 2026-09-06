---
title: "LSP 与 E2B 远程世界：封闭导航词汇与同一世界不变式"
tags: [dsh, lsp, e2b, capability-seams, remote-execution]
status: active
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/{lsp/e2b} 六包 src 直读 + docs/{subsystems/lsp,capability-seams}.zh.md + 官方 e2e/测试对照（live E2B 受 E2B_API_KEY 门控未实跑）"
updated: 2026-09-05
---

# LSP 与 E2B 远程世界：封闭导航词汇与同一世界不变式

> 本篇回答三个问题：LSP 导航能力如何被压成 4 操作的闭合词汇表、宿主如何对抗
> 挂死与恶意语言服务器；E2B 三包如何把文件系统+shell+PTY 整体投影到一次性云
> 沙箱；以及[总览](../overview.md)设计决定 3（"seam 指向远程=整组搬家"）的接线
> 证据链。证据基准 = 六包源码直读 + 官方文档与测试对照；[MEASURED] 均指源码
> 默认值/常量（live E2B 路径受 `E2B_API_KEY` 门控，本篇未实跑）。

## 概览篇说法 → 深读结论

| 概览篇说法（[代码运行、LSP 与远程世界](../execution/remote-and-code-runtime.md)） | 深读结论 |
|---|---|
| "4 个操作封顶；结果联合也闭合" | 证实且更严：闭合联合 + `assertNever` 穷尽，加操作是 seam/提供方/工具三方编译期强制同步；`findReferences` 恒含声明，调用方无 flag |
| "扩展名→语言 provider 注册" | 补充：扩展名是**排他预留**，跨提供方冲突抛 `LSP_CONFLICT`；选择按最终扩展名、与注册顺序无关（`foo.d.ts`→`.ts`，dotfile 永不路由） |
| "`lsp-stdio` 起真语言服务器" | 实际是"一个插件实例、一张配置表、N 个隔离 provider"；每 provider 每 canonical workspace 恰好一个服务器进程，惰性 single-flight |
| "Bash/PTY/LSP/工具全部自动搬到远程，无需逐工具改造" | 接线链逐环保真（见「整组搬家」节，每一环都来自 `inject` 声明） |
| "远程执行世界的完整标本" | 降调：组 README 自述**实验性 POC、任何已发布组合不默认启用**；composition e2e 需要真实 API key |

## LSP seam：用三次拒绝定义一项能力

Service Definition `packages/lsp/lsp`（`ctx.lsp`）。词汇文件（types.ts）的模块
注释直陈拒绝清单：seam 不暴露协议类型、不暴露进程/文档控制、**没有 JSON-RPC
泛型逃生口**——只有四个语义操作。

1. **拒绝逃生口**：`LspOperation` 闭合联合四成员；`LspQueryResult` 是按 `kind`
   穷尽 switch 的闭合联合（`locations` / `hover`），新增分支在消费方编译失败。
2. **拒绝可选项**：`LspQueryRequest` 每字段必填——`workspaceRoot` 由调用方提供、
   "never defaulted"；没有 `resolve()` 步骤；超时与结果上限归消费方。
   `languageId` 从提供方注册表派生、不参与提供方选择。
3. **拒绝坐标漂移**：`locations` 变体携带 `resolvedWorkspaceUri`（提供方对
   workspace 的 canonical `file:` URI）。相对化必须用它，而不是拿请求里可能被
   符号链接改写的进程路径按宿主平台规则解析——**执行平台可能与调用方不同**，
   这一句注释就是远程世界的伏笔。

注册是原子全有或全无：先校验（非空 id、非空扩展表、表内去重、跨提供方排他），
全部通过才在同一个 `ctx.effect` 生成器里提交预留，disposer 一次性释放 id+扩展；
公开 API 是同步 fire-and-forget 的 `() => void`。错误是 `LspError`（`HarnessError`
子类），调用方按稳定 code 路由（`LSP_INVALID_PROVIDER / LSP_CONFLICT /
LSP_UNAVAILABLE / LSP_DISPOSED / LSP_UNSUPPORTED_OPERATION / LSP_MALFORMED_RESPONSE`
[MEASURED 源码枚举]），不 parse message。

## lsp-stdio：无状态查询，一个宿主两个执行世界

`inject = ['fs', 'lsp', 'subprocess']`——源码经 `ctx.fs` 读、语言服务器进程经
`ctx.subprocess` 起，**零直接 `node:fs`/`child_process`**。同一份 provider 代码
因此同时是本地宿主和远程宿主；官方 composition 夹具正是让 fixture 服务器在
E2B 沙箱内服务 `ctx.fs` 指向的远程文件。

- **配置与回滚**：`servers` 表每行注册独立 provider；先把**整张表**的 executable
  解析完再注册任何 provider（后行的坏 command 不会留下前行的孤儿），注册期的
  映射冲突逆序回滚。默认值 [MEASURED]：`maxMessageBytes` 16,000,000、
  `maxStderrBytes` 1,000,000、`maxDocumentBytes` 4,000,000、`shutdownTimeoutMs`
  5000、`killGraceMs` 2000；非正值**加载期即拒**——注释点名 "`slice(-0)`
  会保留全部 stderr" 这类静默失效。
- **瞬态打开**：`didOpen(全文) → 请求 → didClose`，无持久文档同步，从 harness
  视角查询完全无状态。每个 canonical workspace 一条序列化队列跑完整
  读源→open→query→close；已取消的等待者**不解锁队列**（tail 跟随实际前序工作）。
- **读源先于起进程**：坏源码不会留下空闲进程进池；传输失败**一次**透明换传输
  重试——因为查询只读，重试才安全。
- **握手纪律**：`processId: null`（服务器可能跑在别的 PID namespace，传宿主 PID
  只会让它监视无关进程）；position 编码只接受 utf-16；无动态注册——生命周期
  请求一律回 `null`；`workspace/configuration` 用静态配置逐项作答；
  `workspace/applyEdit` 恒拒绝（"This host never applies edits"）。
- **取消是有界的**：发出请求前 `peekNextId` 预占 id，abort 时发 `$/cancelRequest`
  并给 `killGraceMs` 的宽限等服务器确认；超宽限不静默就**整个实例拆毁**——
  挂死请求绝不能与下一条队列项的文档生命周期重叠。握手被取消/失败同样拆
  （绝不池化"永久中毒"的实例）。
- **拆除阶梯**：`shutdown → exit → await closed`，超预算升级 SIGTERM→(grace)→
  SIGKILL，然后 await 进程树退出。这里的 await **故意不设超时**（注释直陈）：
  seam 已承诺 SIGKILL，"静默是 disposal 欠调用方的后件"，再挂表只是自欺。
- **帧与翻译**：`Content-Length` 基础协议帧，header 段 64KiB 上限
  （`1 << 16` [MEASURED]）——不发终止符的服务器撑不爆缓冲；帧损坏=流位置
  不可恢复，fail-closed 拒绝全部 pending 并整组击杀（helper 不得比 leader
  多活）。协议 payload→seam 词汇全部收敛在 `translate.ts` 纯函数层，
  fake-stdio 测试逐形钉死。

## tool-lsp：模型面前只有四件事

- 参数只有 `operation/file_path/line/character`；**1-based ↔ 0-based 转换归工具
  层**（seam 与协议都是 0-based UTF-16），越界坐标拒绝非正整数。
- workspace 取自调用 agent 的会话头 `session.header.cwd`，无回退：缺失即
  `LSP_WORKSPACE_REQUIRED`——provider 必须先 canonical 一个真实工作区才能起服务。
- 上限三件套 [MEASURED 默认]：`maxLocations` 100（超出加省略行）、
  `maxResultChars` 16,000（**截断公告本身也计进上限**——`boundResult` 保证
  封顶后的总长）、`timeoutMs` 60,000 附在工具定义上，由超时策略 seam 执行。
- 输出渲染按文件分组为 `path:line:character`（回到 1-based）；`renderUri` 从
  URI 形状猜执行世界 OS（有 hostname 或前导 `/x:` 段 → Windows 盘符形态），
  注释自认 POSIX 工作区恰 rooted 于 `/c:/...` 会误渲染——仅显示面。
- 一段稳定 system prompt（`ctx.systemPrompt.section`，排序键 `TOOL_LSP`）给
  定位叙事：普通导航用 search/read，文本匹配歧义或改动前需要精确定义/引用/
  实现时才用 lsp。**授权边界不靠审批、靠能力最小化**：四个只读查询、无编辑
  通道、宿主侧拒绝 applyEdit。

## E2B：一个沙箱、两个投影、三条硬规矩

组 README 的定位原话：文件读写、shell 命令、终端全在同一个远程 Linux 世界；
**harness 进程、模型调用与会话状态永远不搬家**；沙箱短暂。三包 =
`e2b`（`ctx.e2b` 共享句柄）+ `fs-e2b`（`ctx.fs` 提供方）+
`subprocess-e2b`（`ctx.subprocess` 提供方）。

- **生命周期**：构造即开 `Sandbox.create({ secure: true, lifecycle: {
  onTimeout: 'kill' } })`，adapters 只 await `getSandbox()`；默认 `cwd` 是
  E2B 模板的用户工作区（`/home` 系 POSIX 绝对路径，字面值见包内 Config），
  `timeoutMs = 300_000`（5 分钟）[MEASURED]；
  `runtimeRoot = cwd/.dsh-e2b` 必须是真实目录（symlink 即拒）+ chmod 700；
  setup 任一步失败 → kill 回滚。disposal 容忍 `SandboxNotFoundError`。
- **控制面外联**：`e2bApiUrl` 镜像 SDK 自身的三级优先（`E2B_API_URL` >
  `E2B_DEBUG`→`http://localhost:3000` > `https://api.{E2B_DOMAIN|e2b.app}`），
  代理决策对**SDK 真正会调的 URL**做 `proxyRouteFor`——否则 bypass 名单、
  scheme 选择、调试面全都错位；egress.spec 用假代理钉死：正常流量走
  `api.e2b.app:443` CONNECT，loopback 调试面**直连**（不把 API key 递给代理）。
- **SDK 登录壳的两条硬防**：E2B SDK 的命令强制过 `/bin/bash -l -c`。对策一，
  `quoteE2BShellArg` 单引号包装一切不透明参数（零插值）；对策二，
  `e2bControlEnvs` 给每条控制命令一个**新的随机 HOME**
  （`/.dsh-e2b-control-<uuid>`），登录壳读不到用户 profile。composition e2e
  的对抗夹具：远端种一份会外带 `NPM_TOKEN` 哨兵的恶意 profile 三件套，命令与
  PTY 双双执行后断言无泄漏文件、`/proc/*/environ` 无凭证 [源：官方测试；
  本篇未 live 实跑]。

## fs-e2b：在没有锁的世界里复刻全套本地语义

`E2BFileSystem extends FileSystem`——与 fs-local 同一接缝契约，读方/edit 工具
零感知。

- **canonical 化**：`resolve` 在沙箱内跑 `realpath -mz | base64 -w0`，回传经
  base64 往返一致 + NUL 帧封 + UTF-8 fatal + 绝对路径四重校验（远端输出按
  敌意数据对待）。`targetKey`=canonical 路径、`displayPath`=沙箱 cwd 语义下
  的 posix.resolve。
- **版本令牌**：`FsVersion` = sha256（`dsh-version` 元数据 + path/type/size/
  mode/mtime/symlinkTarget）；`writeAtomic` 每次盖新 `randomUUID` 进文件元数据
  ——乐观并发（`replaceIfVersion`/`FS_STALE_VERSION`）因此可检。
- **原子写协议**：私有 staging 目录（`.dsh-<uuid>.tmp`，mkdir 撞名即拒 +
  chmod 700）→ 写临时文件（带版本元数据、chmod 沿用旧 mode、新文件 0600）→
  更新用 `files.rename` 发布；`createIfAbsent` 用 `ln -T` 区分
  created/exists——exists 即 `FS_NOT_OBSERVED`，**先读后写的闸在远程照样立**。
  提交后的 staging 清理失败不回滚成功（注释明说）。
- **读路径**：8192B 采样见 NUL → `FS_NOT_TEXT`；UTF-8 fatal；`FS_TOO_LARGE`
  双保险（stat 预检 + 流式逐块封顶）。SDK 对空文件的 stream 形态会返回
  `''` 而非流——代码注释直斥"lies"并打桥；这是**版本依赖补丁**。
- **editText** 与本地同名同语义：唯一匹配（多义 `FS_AMBIGUOUS_EDIT`）、
  CRLF 探测-还原、diff 一律 LF 归一化。远程错误无结构通道，
  `permission denied` 正则归 `FS_PERMISSION_DENIED`——对照
  [沙箱执行](./sandbox-execution.md) 的本地结构化拒绝分类，这是远程的
  降级面之一。
- 并发：进程内按 canonical path 串行 `withLock`；无跨进程协调——与持久化
  篇"进程内单写者"同一可靠性等级。

## subprocess-e2b：把进程生命周期包成远端事务

接缝要求 `spawn` **同步**返回句柄，而 E2B 只有异步控制面——全部复杂度从这里
长出来。`commandState/readyState` 双段 promise + `DeferredStdin` 先桥接再转发。

- **bootstrap 链**：`env -i <bootstrap> setsid --wait bash -c inner`。inner 先写
  自己的 pgid（`ps -o pgid=`）进私有文件，再 `env -i` 真实环境 exec 目标，
  退出码写 `exit-code` 文件。九个工具（env/setsid/bash/node/ps/tr/tee/head/rm）
  spawn 期 `command -v` 解析校验，缺一个 → exit 125。
- **环境三件套**：`readRemoteEnvironment` 用 `env -0 | base64` ASCII 通道取回
  远端环境（防 SDK 回调分块切坏 UTF-8）+ `getent passwd` 取真实登录 HOME；
  `bootstrapEnvironment` 把所有 `DSH_*` 与凭证形状名（`SENSITIVE_ENV_PATTERN`
  与 subprocess-local 同源）置空只挡登录壳引导；`serializeRemoteEnvironment`
  scrub→显式覆盖→`undefined` 墓碑删 ambient。状态目录 700、文件 600、
  environment 文件被 inner 读到即 `rm -f`。
- **输出管道**：内嵌 `node -e` 编码器把 stdout/stderr 按行 base64 帧化送回
  SDK 回调，EOF 哨兵帧收口；`E2BBase64Decoder` 逐帧往返校验，传输损坏
  fail-fast 记 `outputTransportError`。spill 模式用 `tee | head -c N >
  远端文件` 先段落盘，有效 spill 经 `readFrom` 的 `spillPath` 暴露；
  drain 超宽限 → spill 作废 + SDK 连接 `disconnect()`，远端 status 文件
  仍权威。
- **退出码双源**：wrapper 写的 status 文件优先，SDK `CommandResult` 兜底；
  collect/inherit 模式轮询 status（`pollMs` 默认 20ms [MEASURED]，每 tick
  一次控制面请求——远程的观察税）。TODO 注释自认：等 E2B 能给独立于后代
  持有 fd 的 exit 观察再拆。
- **终止是事务不是尽力**：TERM→grace 轮询组静默→KILL + `handle.kill`→
  `groupAlive` 证明（`ps -eo pgid=,stat=` 排除僵尸/死态）；证明失败就抛
  "remained live after force termination"。发布的 pgid 拒绝 ≤1（负号形式会
  `kill -- -1` 全杀）；同 uid 的远端进程可伪造 pid 文件——注释自认 userspace
  预检**关不掉数值 PGID 复用竞态**，TODO 挂在 E2B 缺 identity-bound API 上。
- **PTY 交接**：`sandbox.pty.create` 得到的是登录壳，harness 随即 `sendInput`
  `exec /bin/bash runner.bash` **把壳换掉**；runner 开场先自删四件状态文件
  再 `exec env -i` 真实 argv；`BootstrapOutputFilter` 靠随机 marker
  （`dsh-e2b-bootstrap:<uuid>`）划定输出边界——marker 之前的引导噪声永不出
  handle。清理以**整个 process session** 为单位：`ps -eo sid=,pgid=` 收全部
  组→TERM→KILL→空 session 证明，抛错必附幸存组列表。`signalForeground` 经
  `ps -o tpgid=`，但拒绝对 terminal 自身 shell 发 SIGKILL；`inputWaiting`
  恒 false 并附注原因（E2B 没有能证明 fd 0 阻塞的 /proc 访问）——接缝契约里
  诚实降级的一格。

## "整组搬家"的接线链（设计决定 3 的验证）

composition 夹具 `cordis.yml` 的首行注释就是官方**同一世界不变式**：
`e2b.cwd`、`sandbox-policy.workspaceRoot`、bash 隐式 workdir 必须指向**同一个
远程目录**。逐环 `inject` 声明核对 [MEASURED]：

| 消费方 | 途经接缝 | E2B 落点 |
|---|---|---|
| read/write/edit 工具 | `ctx.fs` ← fs-e2b | 沙箱文件 API |
| tool-bash | `ctx.shell` ← bash-local（`inject=['subprocess']`）← subprocess-e2b | `commands.run` background |
| tool-terminal | `ctx.terminals` ← terminal-bash（`inject=[...,'subprocess']`，`spawnTerminal`）| `sandbox.pty.*` |
| tool-lsp | `ctx.lsp` ← lsp-stdio（`inject=['fs','lsp','subprocess']`）| 服务器进程也在沙箱内 |

上层工具**零改造**——这正是"能力 seam 三件套"（Service Definition + Provider +
Consumer）作为部署形态开关的含义：[沙箱执行](./sandbox-execution.md) 把本地
世界做窄，E2B 把整个世界搬家，两个形态消费同一套词汇。边界同样清楚：会话
日志与提示词永远留在本地 harness 进程，沙箱生命以 `timeoutMs` 为界（默认
5 分钟，到期必 kill）。

## 相关

- 概览：[代码运行、LSP 与远程世界](../execution/remote-and-code-runtime.md) ·
  [文件系统与沙箱](../execution/filesystem-and-sandbox.md)
- 设计决定语境：[总览](../overview.md)「能力 seam 三件套」
- 本地执行对照：[沙箱执行内幕](./sandbox-execution.md)
- run_code/worker-thread 对照：[PTC 与 code-runtime 内幕](./ptc-code-runtime.md)

## 诚实边界

- live E2B 路径未实跑（`E2B_API_KEY` 门控）；远程链路结论=源码级走读+官方
  e2e/测试对照，[MEASURED] 均为源码默认值而非运行时观测。
- `lsp-stdio` 每 canonical workspace 单实例的池化行为，未在多根大工作区实测。
- fs-e2b 的空文件 stream 补丁与 `getent passwd` HOME 恢复都依赖被 pin 的
  SDK/镜像行为，上游变更是已知的版本耦合。
- 终端上层（scrollback/就绪判定）语义属 terminal 组篇目，本篇止步于
  `ctx.subprocess.spawnTerminal` 接缝。
- `subprocess-e2b/tests` 共 111KB 的规格矩阵仅抽样对照（环境泄漏、pid 安全、
  回滚静默四主题），未逐用例穷举。
