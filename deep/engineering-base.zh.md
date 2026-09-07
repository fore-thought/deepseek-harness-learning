---
title: "工程基建：util、测试支持与原生侧的机制收口"
tags: [dsh, util, testing, native, packaging, landlock]
status: active
license: CC-BY-SA-4.0
evidence: "packages/util/* 13 包（2,525 行 src）+ packages/test-support/* 6 包 + native/landlock-run + python/{sdk,sdk-runtime} 源码与文档直读；消费者接线按各包 index.ts import 行扫描；DSH_* 托管变量为本会话真机观测；DSH 仓库相对路径可复核"
updated: 2026-09-05
---

# 工程基建：util、测试支持与原生侧的机制收口

[English](engineering-base.md) | [中文](engineering-base.zh.md)

> 本篇收口 harness 的"非领域"地基：`packages/util/` 13 包（任务口径常说 12，实测目录 13 个
> [MEASURED]）、`packages/test-support/` 6 包、`native/landlock-run` 启动器包族、
> `python/` 双 wheel。不逐包记流水账，按**机制**组织——这些包全部是"库不是插件"：
> 不接 `ctx`、不注册服务、不发事件（launch-environment 只提供一个 ctx 插槽键），
> 所以它们不出现在任何一篇主题概览里，却支撑着每一篇的接线质量。

## 规模与消费面速览

| 块 | 实测 | 口径 |
|---|---|---|
| packages/util/ | 13 包 / 2,525 行 src（最大 http-proxy 680、output-retention 441）[MEASURED] | 排除 test 与 lib 产物 |
| packages/test-support/ | 6 包 / 17,537 行（session-snapshot 独占 9,475）[MEASURED] | 含测试目录源码 |
| util 消费面 | brand/values 被 ≥9 组引用，atomic-write 仅 settings+credentials，output-retention 仅 spill+jobs [MEASURED] | 各包 index.ts 的 import 行扫描，抽样口径见诚实边界 |

## 机制一：原子写与跨进程锁（atomic-write）

会话日志**不用**这个包（持久化篇已证：日志后端走自己的 link/MoveFileW 路径）；
它的正身消费方只有两个——settings 与 credentials 的"读-改-写"用户文件。两个原语：

- `writeFileAtomic`：随机后缀兄弟文件 + `wx` 独占创建 + rename 覆盖。三点设计决定：
  ①`mode` 参数**强制在类型里**（权限决定必须出现在每个调用点）；②新鲜 inode 携带
  mode 穿过 rename，替换宽权限文件即收窄、无 chmod 竞态；③rename 替换符号链接目标
  **本身**而非写穿到指向物。Windows 瞬态干扰（EACCES/EBUSY/EPERM）有界重试：
  8 次、20ms 起指数、上限 200ms/次 [MEASURED]。崩溃耐久（fsync）**故意不归它管**——
  源码 TODO 明言 settings 耐久不在范围内，与会话日志"每帧同步 fsync"形成有意的
  可靠性分层：**被崩溃恢复依赖的写才配得起 fsync**。
- `withFileLock`：`<file>.lock` 兄弟（wx，内容=pid）。读者完全无锁（rename 提交原子）。
  争用判别有个 Windows 细节：`EEXIST` 直接算争用，`EPERM` 要再 `lstat` 证实锁存在才算
  ——防止把无关权限错误吞成"锁被占"。默认等待 2s（按"渲染+rename"时代校准），
  持锁跑网络往返的调用方必须自己声明 `waitMs`。**孤儿锁永不自动回收**：
  文件年龄不能证明持有者已死，回收是运维动作。

## 机制二：值纪律底座（brand / values / crypto / time / timeout / deque）

主干篇的"品牌化 id + `…Map→派生联合`"类型模式，其运行时零成本底座就是 brand 包：

- `Branded<B>` 仅编译期（`declare const BRAND: unique symbol`），**不持有任何运行时身份**
  ——这是"重复安装安全"的关键：同一包被装成两份，产出的值仍可互换。values 同声明
  此纪律（重复安装安全），并给全仓提供 `assertNever`（闭合联合逃逸即抛，带 switch 点
  标签）、跨 realm 纯对象判定（查 `Array`/`Object` 构造器**逐字对**源码串"native code"，
  防伪造原型的 JSON 注入）、`JsonValue` 契约。
- crypto 包 41 行解决一个真实环境裂缝：`crypto.randomUUID` 是**安全上下文专属** Web API，
  局域网 HTTP 页面上没有；统一走 `getRandomValues` 手拼 RFC 9562 v4，并用
  `no-restricted-properties` lint 规则把 `randomUUID` 调用点全部指向这里。
  `bytesToBase64` 以 32KiB 分块防参数溢出。
- time：只做 IANA 时区的**线端校验与规范化**（`Intl.DateTimeFormat.resolvedOptions`
  取 canonical 名）——别名不入库，因为时区标识要跨进程再解析；格式化与各端错误
  词汇**故意不管**（各边界自抛）。timeout：`clampTimeout(requested, def, max)` 一段式
  收口 + `TimeoutReason`（能力自有 code + 用时）走 abort signal 传播；Node 定时器
  钳位值 2^31−1 ms 显式建模为 `MAX_TIMER_DELAY_MS`。deque 给跨异步工作的循环双端队列
  （api 组消费）。

## 机制三：预览代数（output-retention）

[附件与溢出篇](./attachment-spill.zh.md) 写过 spill"预览机制另属 util/output-retention"，
这里兑现那句展开。它回答一个纯机械问题：**预算内留了什么、精确漏了多少**。

- 两个 retainer 因资源模型不同而不同名：`ItemRetainer`（有序逻辑单元，v1 只有 head
  策略）与 `TextRetainer`（字节流，head/tail/headTail 三策略共用一组前/后累加器，
  驻留 ≤ prefixCap+tailBytes+一个 chunk，流再大不涨）。
- **语义红线**：`truncated` 只表示"因预算省略"，绝不含"上游本来不全"——权限失败、
  跳过的二进制、提供方部分失败留在工具域字段，永不混入。恢复文案归工具
  （`RetentionNotice` 只带机械事实：只有工具知道"收窄模式/取更具体 URL/去读 spill"）。
- 字节级 UTF-8 边界安全：前切 `trimTrailingPartialUtf8`（回退至多 3 个连续字节找
  lead，声明长度超出实有即裁），后切丢前导连续字节——**裁剪自身绝不产生 U+FFFD**。
- 消费者恰是"上下文有预算"的包：spill-policy、jobs 输出镜像 [MEASURED]。

## 机制四：环境与网络面（launch-environment / http-proxy / home-paths / native-command）

- **launch-environment**：不可变启动快照，每值记来源层，信任序
  `process > project-env > user-env`；Windows 名折叠大写防大小写分裂优先级，POSIX 保持
  精确。快照在任何 config 表达式求值前由启动器灌进 ctx 插槽，之后的 chdir/切工作区/
  恢复会话都看不到漂移；无启动器时退化为"仅 process 一层"。**真机旁证** [MEASURED]：
  本会话子进程可见托管变量 `DSH_HOME`、`DSH_SESSION_ID`（UUID 形）、`DSH_SHELL`、
  `DSH_WEB_URL`——subprocess 篇的"DSH_* 环境词汇"即由这条链供给。
- **http-proxy**：Node 内置 fetch 无视 `HTTP_PROXY`，所以启动器解析**一个** policy
  装成 undici 全局 dispatcher——LLM 适配器、web 搜索、MCP、遥测零改动全覆盖。
  拆两半：policy.ts 纯、零 undici 依赖（浏览器 worker 也能算路由）；install.ts 管传输。
  三条硬规矩：①环回四值（localhost/127.0.0.1/::1/[::1]）**强制并入**每条 policy 的
  noProxy——否则代理把 Web UI 自己变成路由环（`[::1]` 也列是因为 undici 匹配器把裸
  `::1` 读成 host `:` port `1`）；②`proxyRouteFor` 把决策**连同**当时装好的 dispatcher
  一起交回——install/dispose 落在分支判断之间也不发错路；③子进程环境由 policy 发布
  回写（含 `ALL_PROXY` 兜底与合并后的 bypass），但全局 dispatcher 从不读这些变量。
  第四个函数 `clearedProxyEnv` 专供测试重放隔离（loader-smoke 就 import 它）。
- **home-paths**：`DSH_HOME` 三源优先级（显式配置 > 环境变量 > `~/.dsh`，空白值算未设
  ——防把 home 解成 cwd）；`dshHomeDisplay` 永不返回本机绝对路径（`~/.dsh` 或
  `$DSH_HOME` 中性形——本库文档遵守的写法就出自这里）。
  `canonicalizeWatchPath` 给原生文件监视器唯一拼法：最深存在祖先 realpath + 可枚举
  目录证明，防 Windows 把"文件当父目录"报成普通不存在、防短名别名混进长路径。
- **native-command**：宿主原生调用的**无 shell 合同**（`execFile`+argv 数组永不拼
  shell 串、utf8、windowsHide、signal 终止子进程）；path-opener 按意图分双路
  （可渲染文档 [.html/.svg 等] 先问默认浏览器、命名不了再回默认应用；文本编辑意图
  从不问浏览器；WSL 检测靠内核 release + 环境标记，路径逐条翻成 Windows 桌面拼法）。

## 机制五：landlock-run（原生启动器包族，一句话机制+全量治理）

Linux 沙箱消费内幕见 [沙箱执行](./sandbox-execution.zh.md)（"~300 行 C11+musl 静态链接"
即此包）。本篇只收**发布与治理面**，它自成一格：

- "先限制自身、再执行"：规则集跨 `execve` 继承，包装命令及其后代全受限、调用进程
  不受限；内核不能执行就**不跑并退出**（fail-closed 同沙箱侧口径）。
- esbuild 式包族：入口包 + 按平台 optionalDependencies；**有意无安装期构建回退**
  ——编译回退把"干净的 fail-closed 降级"变成"看环境的也许"；`probe()` 是唯一可用性
  信号，缺二进制与内核不执行**故意不可区分**（都报 `unusable`），消费方只有一条降级路。
- 125 归因陷阱：成功 exec 的子进程也可能退 125，故消费方须**同时**看到致命诊断行与
  该状态才归因启动器——沙箱篇拒绝分类学的上游合同。
- 打包门控有真实事故形状：平台 tarball 用 `npm pack`、入口用 `pnpm pack`，**故意分家**
  ——观察到 pnpm 会规范化文件模式、剥掉可执行位，发出"无法 spawn 的启动器"；
  prepack 逐字节验 ELF `e_machine` 与声明 cpu 一致、装后再做真实拘禁世界证明。
- 主仓 workspace 同锁文件：启动器约定变更与消费方更新在**同一改动**里落地共测。

## 机制六：测试支持三层无钥匙阶梯（test-support）

"无钥匙"= CI 不需要任何模型 API key；阶梯从"真 boot"到"真回放"到"像素级快照"：

1. **loader-smoke**：起真实 `cordis.yml` 过 app bin 的共享子进程壳。双模式解析器：
   dev 下 `tsx` 直跑 TS 源（走 tsconfig paths 映射），CI 设 `DSH_EXAMPLE_MODE=lib`
   用纯 Node 走真 `exports`——**同一 example 覆盖开发路径与安装消费者路径**；
   启动即 `clearedProxyEnv` 隔离网络策略。
2. **llm-replay**：把**录制的会话日志本身**当测试脚本——从 v2 嵌入的 assistant 流与
   显式标记的本地压缩调用导出每次模型调用的脚本，新会话按首叫顺序绑定父/子脚本；
   抛错与挂起场景日志无法重建、必须显式 override。文件顶部一道守卫：
   `sessionFormatCatalog.currentVersion !== SESSION_FORMAT_VERSION` 即抛——
   **格式迁移与回放脚本的锁步**是这套体系的地基（持久化勘误背景见
   [投影/遥测/代次迁移](./session-projection-telemetry.zh.md)，在途篇按名引用）。
   mock-server 补协议层负样本：24 种行为（connection_reset/stream_eof/stall/
   malformed_event/quota_exceeded…）一次请求消耗一个，服务器**从不**重试或解读
   harness 策略——恢复语义归被测方。
3. **session-snapshot**：ACP 黑盒快照四层（共享 launcher / 脚本化 scenario harness /
   纯归一器 / describe-it 套件工厂）；归一器剥请求头、系统提示、会话 id——
   快照对比"可回放事实"而非环境噪声。
- 另两员：**client-runtime** 在 jsdom 里装配**生产实现**的 SlotRegistry/渲染器/store
  （一字不拷，防测试与产品漂移），不进产品插件图；**agent-loop-testkit** 只挂前置
  服务（SessionStore/SystemPrompt/ToolRuntime/LlmRuntime），**故意不挂 AgentLoop**、
  不注册适配器——把装载拓扑留给被测测试自己。

## 机制七：Python SDK 打包形态（python/）

双包：**sdk**（高层轮次 API + 低层 JSON-RPC 客户端，按行分隔 JSON-RPC 走 stdio 驱动
子进程 runtime）与 **sdk-runtime**（把整棵封闭的 Node 依赖树包进原生可执行程序的
平台 wheel——**用 SDK 不需要系统 Node.js**）。四个值得说破的决定：

1. **wheel-only**：sdist 构建直接抛错；平台矩阵五目标显式清单化
   （linux-x64/arm64、macos-x64/arm64、win-x64；**不发** win-arm64），wheel tag 与
   载荷严格互验（hatch 构建钩子拒绝混平台载荷）。
2. **绝不静默读 `~/.dsh`**：每次启动强制显式 `DSH_HOME`，缺失或空白即拒绝——
   与 home-paths 的"空白=未设"同源，但 Python 侧更严（连默认都不给）。
3. 可执行程序内置 VFS 装插件，而**操作系统符号链接进不去 VFS**：打包运行在
   `$DSH_HOME/profiles/node_modules` 下维护小型真实 ESM 代理包，镜像声明的 exports、
   记录原包身份、再导出虚拟模块 URL——内置配置与外部插件 peer 因此共享同一
   Cordis 实例（热重载家族的又一环）。
4. ripgrep 与 macOS PTY spawn-helper 作为可执行伴随文件随 wheel 分发；仓库开发专用
   node 载体（系统 Node ≥22.19 + `runtime/node/`）**既不进 wheel 也不进 sdist**。

## 诚实边界

- 消费面扫描只看各包 index.ts 的 import 行（72 命中）；非入口文件的引用未计，
  "仅 settings+credentials 消费 atomic-write"这类结论在此口径内成立。
- `DSH_SESSION_ID` 在 worker-thread 与 pwsh 子进程里观测到两种形态（带 `session-`
  前缀的会话 id 与裸 UUID），供给链未逐环追（归 subprocess/launch-environment 消费侧
  深读）；本篇只声明"存在托管注入"，不猜两形态的归一规则。
- output-retention 读了策略代数与 UTF-8 边界函数，`formatRetentionNotice` 文案矩阵
  未逐条贴；http-proxy 的 `applyPolicyEnv` 恢复闭包细节只核到注释声明。
- landlock-run 的 C 源码未读（归沙箱篇 main.c 口径），本篇全部治理面证据来自其
  README/docs 与包元数据；Python 侧未真机构建 wheel，py 文件直读 + platforms.json。
- llm-replay 3,744 行为含测试的粗计数；"v2 嵌入流"口径随格式代次演进，回放锁步
  守卫本身即上游对此的自觉。

## 相关

- 原子写镜像与 fsync 分层：[持久化格式与崩溃恢复](./persistence-crash-recovery.zh.md)
- 预览代数的下游大用户：[附件与溢出](./attachment-spill.zh.md)
- landlock 消费侧（拒绝分类/令牌）：[沙箱执行](./sandbox-execution.zh.md)
- 品牌化模式的领域应用：[主干内幕](./agent-loop-internals.zh.md)
- 配置与凭据面（atomic-write 消费方）：「配置·凭据·工作区深读」篇（N14 在途，
  成文后由主会话回填为相对链接 `deep/settings-credentials-workspace.md`）
- 应用壳的 DSH_* 供给上游：[应用壳](../platform/web-cli-boot.zh.md)
