---
title: "SDK 三件套：进程外嵌入的正门"
tags: [dsh, sdk, jsonrpc, embedding, subprocess]
status: active
license: CC-BY-SA-4.0
evidence: "packages/sdk/{protocol,server,client}/src 直读 + packages/bundle/{sdk-app,sdk-minimal}
  + packages/subagent/subagent-dsh-sdk/src + packages/boot/app-boot/src/profile.ts
  + python/README.zh.md；官方口径 packages/sdk/README.md"
updated: 2026-09-05
---

# SDK 三件套：进程外嵌入的正门

[English](sdk-embedding.md) | [中文](sdk-embedding.zh.md)

> DSH 把整个 harness 塞进别的进程，靠的不是库依赖，而是**子进程 + 线路协议**：
> `packages/sdk/` 定义一套行分隔 JSON-RPC（protocol）、一个跑在插件树里的
> stdio 服务端插件（server）、一个管生老病死的 TypeScript 客户端（client）。
> Python SDK 与子代理 SDK 后端是它的两个法定消费者。概览篇
> [应用壳](../platform/web-cli-boot.zh.md)「SDK：进程外正门」一句，本篇展开。

## 组解剖与规模

| 包 | src 行数 [MEASURED] | 职责 |
|---|---|---|
| `sdk/protocol` | 425（index 27 / types 119 / transport 279） | 帧传输 + 命名线路类型（两端共享词汇表） |
| `sdk/server` | 399（index 102 / server 297） | `sdk-jsonrpc-server` 插件：把 stdio 接进已启动的插件树 |
| `sdk/client` | 1168（client 490 / api 309 / launch 157 / dispose 99 / types 83 / index 30） | 低层 `HarnessClient` + 高层 `DeepSeekHarness` + 启动解析 + 回收阶梯 |

测试面即行为文档：client 侧 4 个测试文件共 1362 行（其中 `fake-runtime.ts`
323 行是无密钥、无网络的假 dsh 子进程，整链路可驱动）[MEASURED]；server 侧
spec 1197 行 [MEASURED]。

**`packages/examples/` 的真实身份**：本 checkout 里该组与仓库根 `examples/` 均
**零个被跟踪源文件**（只剩 `node_modules` 安装壳）[MEASURED]。"消费方标本"的
法定位置其实是 `subagent/subagent-dsh-sdk` 与 `python/sdk` 两路（见后两节）。

## 线路协议：三请求、四通知

帧语法（`protocol/src/transport.ts`）：行分隔 JSON-RPC 2.0。有 `id+method` 是
请求、光 `id` 是响应、光 `method` 是通知。**JSON 语法错的行静默忽略**——新版
协议不认识的行丢掉而不是炸链路，前向兼容姿态。方法缺失回 `-32601`、处理器抛错
回 `-32603`（只带 message 字符串，栈不上线路）。请求 id 形态 `req_<uuid>`；
`request()` 的 AbortSignal 是**弃单**语义：超时后 pending 条目整个删除、不留
每调用状态——协议**没有取消**，服务端工作照跑直到进程关闭。`flush()` 用一次
零字节写回调当输出屏障；`close()` 摘监听、拒 pending，但**不销毁流**——流
所有权在调用方，这一句就是"谁拥有 stdio"的分工声明。

七个动作之外的全部词汇（`protocol/src/types.ts`）：

| 方向 | 方法 | 载荷要点 |
|---|---|---|
| C→S | `initialize` | cwd/provider/model/reasoningEffort/maxTokens——**进程级**一次 |
| C→S | `session/prompt` | sessionId + contentBlocks → 返回 `messageId`（持久入队回执，非答案） |
| C→S | `shutdown` | 空参，回 `{}` |
| S→C | `session.event` | 完整 `SessionEvent` 信封——线路词汇直接复用事件真源类型 |
| S→C | `session.status` | 整代理态 `idle · running` |
| S→C | `subagent.started` | parent/child 会话 id（血缘边，客户端建树用） |
| S→C | `subagent.finished` | provider/stopReason/status/可选末条助手消息；**只报进程内 local 子会话** |

两处值得停住的类型决定：其一，`session.event` 不发明 DTO，把 `SessionEvent`
整个信封搬上线——投影/持久化/UI 与外部消费者读的是同一份事实（见
[会话事件日志](../agent-runtime/session-event-log.zh.md)）。其二，locality 判定
在服务端 `subagent/end` 处理里用 `info.local` 快照，注释明言"匹配 id 或父
血缘单独都**不构成**局部性"——远程子代理不进这条通知。

## initialize：进程级握手与就绪门

服务端插件 `sdk-jsonrpc-server` 只 `inject = ['agents']`（LLM seam 用
`ctx.get()` 可选读）；模块头注释立了两条纪律：**stdout 是协议专用信道**，树内
禁挂 stdout logger；**插件保持具名导出、禁 default**——Loader 的 `unwrapExports`
会吞掉 default 上的 name/inject/Config/apply——`subagent-dsh-sdk` 头注直接引
`docs/postmortem/0001-acp-default-export-drops-inject` 的同一课（ACP 篇另见
[委派编排深读](./subagent-deep.zh.md)）。

`initialize` 是 SDK 的**就绪边界**：处理器先 `await ctx.get('loader')?.await()`
——等整棵当前树 settle 才应答（本插件可能排在异步兄弟条目如 MCP 初始工具发现
之前），无调度器延时凑合；手搭无 Loader 的 context 即刻可用。握手参数校验后落
进程级默认：此后每个 SDK 会话创建的 agent 继承这条 provider/model 路由。适配器
兜底克制：provider 无注册适配器时，**只有** `deepseek-official` 允许现场
`ctx.plugin(LlmDeepSeek)` 挂载，其余要求 profile 预装。响应里的
`serverInfo.name` 是 wire-stable 常量 `deepseek-harness-sdk-runtime`
（version 冻结在 `0.0.1`——身份不随包版本漂）。

## session/prompt：懒建会话与两次活性校验

未知 sessionId 首次 prompt 才 `ctx.agents.create`（客户端侧同样懒：
`harness.session()` 零线路流量）。并发懒建经 `sessionCreations` Promise 表
去重。关键防御是 `assertLiveAgent`：agent-loop 独占重载会 dispose 掉 registry
里的 agent 而服务端 `SessionRecord` 仍持有旧句柄，`followup()` 会**静默接受**
——所以投递前后各查一次 `ctx.agents.get(id)` 对码（图片 admission 跨 async
边界，故前后各一次）。会话创建**不合成 preset**（注释直说：本服务器的组合把
模型可见行留在 host 层；要 roster 的部署得先接 `agent-presets`）。内联图片经
`admitEncodedImages` 升为持久附件引用后才入消息——没挂附件仓直接报错，语义见
[附件与溢出](./attachment-spill.zh.md)。

## 收回：响应先写、根后拆

`shutdown` 的处理顺序是精心安排的：结果帧先写出，`setImmediate` 里才走
flush 屏障 → `ctx.root.fiber.dispose()`（含持久化收尾）→ `exit(0)`。单一
`exitTask` 保证竞争的多发 shutdown 只拆一次根；EOF 与信号的退出归 app bin
（`sdk-app` bundle 的 `exitOnStdinEnd`），插件只认协议内的 shutdown。
`maxTokensAsSuccess` 把 max-tokens 轮末译成 `ok` 还是 `error`——
`subagent.finished` 的 status 就出自这个部署级翻译。三处默认并不一致，值得点名：
插件 Schema 默认 **false**；`sdk-app` 的 patch 行用 `!!js` 在环境变量
`DSH_MAX_TOKENS_AS_SUCCESS` 缺省时给 **true**；`sdk-minimal` patch 写死
**false**——装载面即真相，读 Config 声明会读错出厂行为。

## 客户端两层：低层管道与高层"跑完一轮"

低层 `HarnessClient`：拥有子进程。spawn 用裸 `node:child_process` 而**不走
`ctx.subprocess` seam**——模块注释自居"seam 为 SDK 管理传输留的成文例外"
（客户端运行在任何 harness context 之外，够不着那个 seam）。stderr 维持 400 行
环形尾巴；进程死亡时 `TransportClosedError` 把 exit code + stderr tail 装进
诊断；流静止用 100ms 竞赛封顶防挂死。订阅是过滤队列 + waiter 池，filter 抛错
只毒化自己这一路；`subscribeSessionTree` 在**客户端**记 `sessionParents`
血缘表做后代遍历——服务端对全树广播，作用域归消费者。

高层 `DeepSeekHarness`：一个 runtime 进程、多个会话。`run()` 的结算条件是
"先等到自己那条消息的 `agent/inbox/spliced` 回执（防吸收先发通知），再等
`session.status = idle`"——**从入队回执到下一次 idle 的一段活动区间**，产出
`RunResult`（finalResponse 文本 + 全量 events + notifications）。线路边界全部
校验：信封畸形抛 `SdkProtocolError` 而不是 TypeError 漏出。握手失败语义：
清理成功→换新 client 可重试；清理失败→ `AggregateError` 保双因不假称静止。
`await using` 直接可用。默认路由 `deepseek-official` / `deepseek-v4-flash`。

`launch.ts` 的版本锁值得单独记：client 与 dsh 包 **package.json version 必须
逐字相等**才允许起进程（不同版本直接拒启）。构建产物缺失时源码回退：tsx loader
+ `src/bin.ts` + `sdk-source.cordis.patch.yml`（只关 `typert-loader`——
源码 checkout 没有构建期生成的 Typert 模块，SDK 应用本就不消费远程网关）。
env 语义是**整对象替换**：不给就读父环境，给了就完全归调用方（凭证策略外置）。

## 回收阶梯：EOF → SIGTERM → SIGKILL

`dispose.ts` 只解析"已证明退出"：关 stdin 协作静默（默认 6000ms，窗口够子进程
刷持久化、拆自己的嵌套进程）→ POSIX SIGTERM（3000ms 确认窗）→ SIGKILL（3000ms
有界等待）。Windows 上 Node 把 TERM/KILL 都映射为 `TerminateProcess`，故跳过
SIGTERM 直强制；计时器全部 `unref()`，不把父进程事件循环钉住。客户端
`close()` 顺序：先发有界协议 `shutdown`（1000ms），失败仅记进 stderr tail
诊断，随后阶梯权威回收。全链幂等、终身制（close 后 reuse 必拒）。

## 启动侧：PROFILE_TEMPLATES 与两个 SDK bundle

`dsh --profile sdk` 的名分在 `packages/boot/app-boot/src/profile.ts` 的
`PROFILE_TEMPLATES` [MEASURED]：`sdk = [dsh-base, dsh-sdk-app]`（patchReload
startup）；`sdk-minimal = [dsh-sdk-minimal]` **单 bundle 不叠 base**。两个
bundle 都是薄 patch：`sdk-app` insert 两行——`sdk-app-startup`（commander
零选项命令，解析成功才 publish `sdkAppStartup` 服务；server 行
`inject: [sdkAppStartup, loader]` 等它——所以 `--help` 不起 transport）与
server 本体（`maxTokensAsSuccess` 由环境变量 `!!js` 驱动）。
`sdk-minimal` 则是手摆约 30 行的**完整内核树**：sandbox/projection/invariants
全套逐行登记，persona 可被 `DSH_SYSTEM_PROMPT` 覆写，持久化
`compression: none`——连 zstd 都省了的实验最小面。

## harness 内部的客户：subagent-dsh-sdk

最妙的消费者是 DSH 自己：`subagent-dsh-sdk` 注册 provider `dsh-sdk`，把一个
子代理委派变成"起一个完整 harness 子进程"。与 fork 通道的对照是设计课
（见 [委派编排深读](./subagent-deep.zh.md)）：`inheritsParentContext = false`
——"进程边界不过对话"；能力面 `NO_START_CAPABILITIES + agentOptions` 只放行
路由四选项，输出 schema/深度/工具过滤/persona 全拒（"它们的归属跨不过进程"）。
`dshHome` 必填且必须绝对（嵌套 runtime 用隔离 home）；cwd 缺省继承委派方会话。
env 基座是 seam 共享的 `scrubbedParentEnv()` 再叠显式 env——显式密钥到得了
孩子，环境里的同名秘密不默认漏。失败面只回**固定安全事实**：
`Subagent failure (provider: DSH SDK; stage: …; category: …)`，原始 Error 走
`onError` 落宿主日志、cause 链保留，绝不带着 stderr tail 见模型。子代理输出按
`AssistantOutputFold` 规范选择规则收拢；取消无线路通道——本地先结算，再走
回收阶梯。

## Python 孪生与 examples 的正确打开方式

`python/sdk`（分发名 `deepseek-harness-sdk`）是同一协议的双胞胎实现：
api/client/errors/models 四模块对位 TS client；`python/sdk-runtime` 打包原生
adjacent 的 dsh 可执行件。官方 README 立了一条与 TS 侧不同的纪律：**Python
绝不静默读 `~/.dsh`**，每次启动必须显式选 home。它同时是"examples 该长在哪"
的答案：`python/sdk/examples` 有真实样例，`packages/examples/` 组壳则是历史
占位（见诚实边界）。

## 相关

- 四孔总览与 CLI 生命周期：[应用壳](../platform/web-cli-boot.zh.md)
- 事件词汇同源处：[会话事件日志](../agent-runtime/session-event-log.zh.md)
- 图片 admission 契约：[附件与溢出](./attachment-spill.zh.md)
- 进程内委派对照面：[子代理深读](./subagent-deep.zh.md)
- seam"整组搬家"与本文例外的语境：[LSP 与 E2B 远程世界](./lsp-e2b-remote.zh.md)

## 诚实边界

- 本篇为源码 + 测试场景名口径，未端到端起活 dsh 子进程实跑（会触发真实模型
  调用）；时间常量全部源码直读 [MEASURED]。
- server 模块头引的"single-launch Agent Note"决策注记未直读；
  `.agents/notes/` 三篇被 README 指为边界依据，本篇未展开。
- `packages/examples/` 空壳是**本 checkout 的实测事实**；SDK 组 README 未提及
  examples 包（"消费者=两路"为本篇按实测归置），向上游求证前保留此表述。
- Python 侧行为以 README 口径转述，`python/sdk` 源码未逐行走读。
- `sdk-minimal` 完整行清单在 patch 文件本身，本篇只给形态与差异点。
