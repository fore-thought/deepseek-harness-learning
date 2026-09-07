---
title: "HTTP 载体与控制器分层：host/api 两组的装配侧内幕"
tags: [dsh, host, api, webserver, controllers, directory-picker]
status: active
license: CC-BY-SA-4.0
evidence: "packages/host/*/src + packages/api/{gateway,remotes,session-controller,settings-controller,workspace-controller}/src 直读 + docs/api-gateway.zh.md + docs/subsystems/web-server.zh.md"
updated: 2026-09-05
---

# HTTP 载体与控制器分层：host/api 两组的装配侧内幕

[English](host-gateway-webserver.md) | [中文](host-gateway-webserver.zh.md)

> Host 侧从"一个监听端口"到"类型化 Remote 命名空间"之间，站着 `packages/host/`
> 5 组与 `packages/api/` 5 组共十来个包。协议本体（`@Remote` 标记、生成管线、
> 分发顺序、三流帧协议）已在 [typert 远程协议与代码生成](./typert-remote-protocol.zh.md)，
> 一元链/流链/信任栅栏的接线总览在 [host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)——
> 本篇只写**装配侧**：HTTP 载体本身、回退席位语义、index 注入双渲染器、控制器分层纪律，
> 以及两个宿主本地服务（插件清单、目录选择 seam）。

## 两组成员与体量（[MEASURED] 直读行数）

| 包 | 角色 | ctx 键 / Remote namespace |
|---|---|---|
| host/webserver | 浏览器 HTTP 载体（334+111 行） | `ctx.webServer` |
| host/frontend-static | 回退席位占有人：SPA dist 服务器（133 行） | 消费 `ctx.webServer` + `ctx.connection` |
| host/directory-picker | 目录选择 seam（103+37 行） | `ctx.directoryPicker`（抽象） |
| host/directory-picker-{native,browse,auto} | 两个后端+启动自适应挂载（653/307/177 行） | 注册/挂载一个实现 |
| host/plugin-inventory | Loader 只读投影（83+63 行） | Remote `pluginInventory/list` |
| api/gateway | Remote 分发双端点（index.ts 1,137 行 + client 子面） | `ctx.typertGateway` / `ctx.remote` |
| api/remotes | Host 装配壳 + 事件白名单（152+33 行） | 注册 Remote 事件源 |
| api/{session,settings,workspace}-controller | 三大业务命名空间（host 面 3,293/495/723 行） | `session`/`settings`+`credentials`/`workspace` |

## webserver：一个不懂 harness 概念的 HTTP 载体

官方口径先立住（`docs/subsystems/web-server.zh.md`，源码逐条证实）：它不是 agent
loop 的一部分、也不是能力 seam，**不了解任何 harness 概念、不服务任何文件**；
Electron 走 `file://` + IPC，根本不经过它。路由表三件套决定全部请求命运：

- **匹配顺序固定**：exact 表 → 前缀表最长匹配 → 回退席位（`match()` 源码序）。
  注册顺序不携带请求语义——具名路由在组合上必须互不相交，重复
  `(kind, path)` **抛异常**（"路由模式是组合层约定，冲突即配置错误"）。
- **upgrade 表独立于路由表**：exact-path、一路径一协议所有者；已升级 socket 由
  服务自己持集合追踪——因为 **Node 的 `closeAllConnections()` 不含已升级 socket**，
  SSE/WS 这类永不自行结束的连接若不由载体显式销毁，拆卸就挂起。这是 dispose
  把 `close()` 与 `closeAllConnections()` 配对的真实原因。
- **回退席位只有一个所有者**：第二次认领抛异常——"两个回退无法组合"。
- **gzip 包在载体内部**：handler 继续直接持有 `ServerResponse`，服务不新增写出
  API。过滤链保持 identity 的场景：已有 content-encoding、`no-transform`、
  范围响应、`text/event-stream`、ZIP，以及**无 socket 的 Web Worker 隧道**
  （打包 `.gz` VFS 镜像传输，压缩会毁掉它）。协商用 Negotiator 只认 gzip/identity
  两值，再以 `Object.create(req)` 影子副本改写 accept-encoding 喂给 compression
  中间件——不改原请求。
- **异常兜底纪律**：任何 per-request 抛错（畸形 % 转义撞 `decodeURIComponent`、
  客户端半路断体）记 warn 后应答 400（响应头已发出则销毁 socket），**绝不杀进程**。
- 该包从不打印 URL——URL 行归 shell 所有。
- Config 是封闭二选一：`host: '127.0.0.1' | '0.0.0.0'`。载体**不拥有** TLS、认证、
  Origin 策略，绑非回环即裸奔，除非组合层提供；随附 `dsh web` 只选回环并拒绝
  `--host 0.0.0.0`。压缩默认 `none`，随附 Web 组合选 gzip level 1 / 1024B 阈值
  [MEASURED]。

## index 注入：一张表喂两个渲染器

`injections.ts`（111 行）把"插件往启动 HTML 里塞东西"做成**结构化行**：
`IndexInjection` 六行 kind（`global`/`script`/`script-src`/`script-preload`/
`style`/`html`），全部 JSON-serializable，因为一张表有两个消费者——served 形态
直接渲染进 index.html 文本；静态 worker 部署把同样的行经 boot payload 交给页侧
解释器。渲染细节三处硬约束 [MEASURED]：

1. `global` 行的 JSON 文本一律把 `<` 转成 `\\u003c`——行数据永远不能提前闭合
   script 元素越狱；
2. 表尾追加 `__DSH_BOOT_READY__` deferred 的 settle 脚本（`??=`
   `Promise.withResolvers()`）：异步 bootstrap 形态先建后结，served 形态一步建+结，
   client 入口读任何注入状态前先 await 它；
3. head/body 两组各按表序插入开标签之后；无 `<head>` 的夹具页前插、无 `<body>`
   的碎片的后追加——都有注释写明理由。

结构化行装不下的东西走 `tapIndex` 原始 html→html 逃生口，永远在行渲染**之后**
按注册序应用。每次渲染前 `collectIndexInjections()` 现场 `emit`
`webserver/index-inject` 收表——订阅者在 emit 时刻读活状态（模块图、主题偏好），
没有缓存视图。

## frontend-static：回退席位的完整语义

认领席位 = 实现一套固定语义（`serveStatic()` 133 行全读）：index 响应先过
`ctx.connection.authorizeIndex` 认证再读字节，**非 index 资产保持公开**；
非 GET/HEAD → 405（具名路由自管方法，405 只属回退）；越出 dist 根的遍历 → 403；
缺失或不是文件 → 空 404；未知扩展名 → octet-stream；`.gz` 按自身字节发送、
**永不加 Content-Encoding**（worker 自己解压，传输级再编码只会让它解一个已解码档）。

- 遍历护栏比较用平台 `sep` 不是 `/`：`resolve()` 在 Windows 吐反斜杠路径，
  `/` 后缀会把**每一条合法子路径**误判成遍历。
- 只有 `ENOENT/EISDIR/ENOTDIR` 三种 fs 失败算 404，其余**上抛**给 webserver 的
  400 兜底——静默吞掉真实故障比承认失败更糟。
- index 渲染顺带在 `<head>` 开标签后钉 `<base href="/">`：dist 以相对 base 构建
  （同一份文件可挂任意静态目录），但 SPA 深链回退页面的相对资产 URL 会挂在请求
  目录下，served 形态必须锚回站点根。
- `distIndex` 是组合应用的工作区知识，"部署永不硬编码"——典型经 `!!js` 表达式给。

## plugin-inventory：没有第二真源的直接投影

`list()` 每次现读 `ctx.loader.entries()` 把非组条目映射成公共行——**刻意不缓存**：
Cordis 的 plugin/status 事件已经维护 `Entry.fiber` 与 `Fiber.state`，再加一层缓存
= 多一个要同步的生命周期真源。FiberState 六态映射到公共阶段词汇时 `disposed`
折叠为 `null`——阶段**从不区分**"从未启动"与"已被释放"，因为两者对读者等价。
const enum 用运行时镜像对象过桥（跨包 const enum 不可靠）。

可选伙伴 `agentPresets` 每次 `ctx.get` 解析：组合了 roster 时快照携带每个预设的
压平组合行。三条判定值得记：已有会话挂载过的预设由其**最新 standing 世代**作答
（文件事后坏了也照实回答——挂载才是那些会话实际运行的组合）；从未挂载的读组合
文件、`!!js` disabled 门用 Loader 上下文求值且**读取从不挂载预设**；求值不了的门
标 `conditional`，坏预设带原因保留在列表里。服务**刻意不声明同进程 Cordis
merge**——Remote-only，客户端经 `api-remotes` 显式装配消费。

## directory-picker：seam 暴露判别联合，不是一套方法

后端交互**形状**不同（不是机制不同），所以 `capability()` 返回判别联合
（`native{pick}` vs `browse{list,createDirectory}`），消费者 switch on
`kind`；遇到未知的 kind，文档默认行为是**隐藏选择 affordance 而不是报错**。
能力注册表 `DirectoryPickerCapabilities` 是 merge 可扩展 map（新后端声明合并进表、
条目 `kind` 字面量必须等于键），seam 包永不修改。一个 context 一个实现（重复注册
抛——cordis 标准重复服务行为），capability 对象在服务生命周期内必须稳定。

- **auto 后端**：boot 采样一次宿主事实（webserver 实际 bindHost、platform、
  `SSH_CONNECTION/SSH_TTY/DISPLAY/WAYLAND_DISPLAY`、PATH 里有无 zenity/kdialog），
  纯函数解析出 kind，然后把 **Host 后端 + Client 界面成对**作为普通 Loader 条目
  挂进内存根树——根树 `write()` 是 no-op，挂载行**永不回写进配置文件**。卸载
  disposer 逆序 `loader.remove()`（事务性拆除，chooser 卸载完成=两面连同依赖者
  全部静默）；setup 失败路径自己回滚已建条目，否则重试会撞自己的
  `directoryPicker` 注册。两个 `Record<kind, string>` 包名表导出是
  `verify-cordis-config` 静态门的需要（运行时字符串引用静态门看不见）。
- **browse 后端**：为远程客户端服务（宿主屏幕零渲染）。三个算法/护栏细节：
  ① **有界窗口**流式扫描——`maxEntries` 默认 1,000（对齐 GitHub 网页目录截断
  [MEASURED]），窗口开 `keep+1` 槽证明切位，`boundedInsert` 满窗且名字≥尾部
  O(1) 拒绝、否则二分插入，10 万子目录对 1,001 窗口不趋近 10⁸ 次比较；窗口内
  候选后来被证实不可入（坏符号链接）不回补——已被逐出就诚实报 `truncated`。
  ② `raceAbort` 包装每个 fs await——Node 文件操作不可撤回，调用方离场后
  迟到 settle 就地吞掉、句柄由中止方关闭；aborted 退出路径**不许等 close**
  （close 排在 in-flight read 后面，等它就等于把刚逃离的网络盘挂回请求上）。
  ③ `fullyQualified` 栅栏：Windows 的 `\foo`、`/foo`、`\\server` 都过
  `isAbsolute` 却按进程当前盘 rebase——wire 值必须命名固定文件系统位置，否则
  `directory-unreadable`。隐藏行只按 POSIX 点前缀判（Windows hidden 属性 dirent
  给不出，明记限制）；符号链接目录 stat 探针后可见。
- **native 后端**：macOS 走 `osascript`（"User canceled"/-128 → null）；Linux
  zenity→kdialog 两级降级（两者皆无→给出安装指引的显式错误）；Windows 是
  **koffi 绑定的子进程**（不是 worker 线程——模态对话框必须是"进程的第一个窗口"
  Windows 才免手动 foreground 激活），协议三帧 `showing{threadId}`/`done`/
  `error`；中止靠向对话框线程重发 `WM_CLOSE`（150ms × 20 次预算后 `kill()`
  兜底），settle 后 `unref()` 防止卡在模态里的子进程吊住宿主退出。**无降级层**：
  PowerShell 回退层已被官方决策注记移除（简化，2026-08-04），失败原样上抛。
- **Remote 接线在 api 侧**：`DirectoryPickerController`（namespace
  `directoryPicker`）住在 workspace-controller 包里——抽象 seam 永不是 Loader
  条目，wire 动词由这个宿主入口携带；后端没组合时子插件停在 pending、
  **不注册命名空间**，"不能被服务的动词被拒绝，而不是被近似"。`pick`/`list`/
  `createDirectory` 三动词各经 `requireCapability(kind, verb)` 门控，能力形状
  不匹配时给稳定错误。

## api 侧：控制器分层纪律（只写装配面）

- **gateway 认领判定**（`claimsEndpoint` [MEASURED]）：恰两段 endpoint +
  （typert 注册表命中 ∪ `hasSeen` ∪ SRC claims）。两个容易漏的点：
  `hasSeen` 让**注册过后被撤回**的严格 endpoint 继续被认领——热卸载后绝不降级到
  SRC 弱推断（校验强度不悄悄松）；`collectSrcClaims` 是运行时全扫服务
  `typertRemote` binding + 原型 `remoteMethods()`。另外 `invoke`/`stream`
  互斥双门：一元方法走流载体、流方法走一元载体都吃 `gateway/signature-invalid`，
  流方法返回值还要求真 Iterable，否则 `result-invalid`。网关级故障词汇是 17 个
  `gateway/*` 码声明合并进共享 `RemoteErrorDetailsMap` [MEASURED]——与领域
  （`session/*`）分层，载体错误永不与业务错误混淆。
- **remotes 装配壳**：白名单 18 项、单一源文件双 face 共享（16 emit + 恰 2
  waterfall：`approval/request`、`user-questions/request`——浏览器应答要**回到
  Host 参与决策**）。桥接机制 `RemoteEventQueue`：同步 Cordis 监听 → Deque +
  单 waiter → AsyncIterable；源结束时把未消费的 pending waterfall dispatch
  逐个 reject，且 `queue.push` 返回 false（源已尽）时监听器**当场改走 next()**——
  瀑布链永不因广播通道关闭而悬挂。`assertJsonArgs` 把每个转发参数过无损 JSON 门。
- **session-controller**（host 面 13 文件 3,293 行 [MEASURED]）：
  `TypertRemoteService` + namespace `session`，聚合 agent.ts（激活/组合/模型选择
  策略）、commands(598 行)、history(407)、list(314)、control、catalog、
  skill-catalog 等子控制器。身份策略四型失败值得单独记：冷会话不存在
  （`session/not-found`）、**subagent 所有权拒绝**（header.origin 即 subagent，或
  parentSession 被某 live agent 持有 → `session/agent-busy` + "改用 subagent
  delivery"——泛型会话路由永远不抢委派通道拥有的 identity）、显式 id 改 cwd/换
  preset 的**收养冲突**（记录的归属与请求不符就抛，绝不静默改判）。冷检查
  `inspectApiSession` 经 `sessionQuery.observeSession(projectionMode:'none')`
  走查询面，不挂 agent-loop。
- **settings-controller**：一个包挂两命名空间（`settings` + 旁挂
  `credentials` 插件）。输出侧纪律：因 typert 运行时**不 parse 返回值**（见
  typert 篇），描述符投影函数 `namespaceView` 逐字段重建 wire 视图——provider
  描述符若多挂了可枚举属性，不会随响应溜给调用方。所有远程读固定
  `redactSecrets: true`，写侧把 provider 拒绝分类为 `settings/conflict` 或
  `settings/rejected`。两个命名空间在无 provider 时**保持注册**——调用得到
  "缺提供方"的可操作诊断，而不是"方法不存在"。
- **workspace-controller**：6 个一元动词（create/rename/delete/两级
  insertBefore/archiveSession）+ `follow` 流（baseline+增量，feed.ts 170 行），
  兼挂 DirectoryPickerController（上节）。
- 官方分层一句（api-gateway.zh.md 边界节）：API 各层按
  `remotes → gateway → connection → webserver` 组织；需要流式/浏览器原生响应的
  功能注册精确 Fetch 路由而**不伪装成 Remote 方法**。

## 说法核对（官方口径 vs 源码）

| 说法 | 结论 |
|---|---|
| web-server 页"具名路由组合上互不相交、回退单主" | 证实（重复即抛，源码两处） |
| api-gateway 页"Gateway 只认领存在严格描述符或活跃 SRC marker 的 endpoint" | 证实+补充：还有 `hasSeen`——撤回过的严格 endpoint **不降级** SRC |
| host README"选择器后端可在共享 seam 后互相替换" | 证实：auto 只是挂载器，pin 某交互=直接组合那一对 |
| workspace 子系统页与工作区注册表 | 工作区记录本体在 `packages/workspace/`，归配置与凭据深读 |

## 相关

- 协议与分发内幕：[typert 远程协议与代码生成](./typert-remote-protocol.zh.md)
- 双树总览与信任栅栏：[host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)
- 应用壳与启动序：[应用壳](../platform/web-cli-boot.zh.md)
- 配置面 seam 本体：[配置面](../augmentation/settings-and-credentials.zh.md)
- 预设 roster 供给侧：[能力供给](../augmentation/skills-mcp-hooks.zh.md)

## 诚实边界

- session-controller 的 commands/history/list/catalog 子文件只读头部与官方口径，
  未逐行；其 client 面（manager.ts 946 行等）属 Web Client 架构主题，本篇按
  包名提及未展开。
- win32 对话框的 koffi FFI 绑定层（bindings/logic 两文件）核了结构未执行；
  macOS/Linux 分支的取消语义以源码文案为准。
- webserver README 的开发模式 bundle 监视流水线未展开（装配面之外的构建话题）。
- 本篇 [MEASURED] 数字为行数/条目数直读与默认常量实读；未做浏览器端到端复放
  （认证换 cookie 的 index 门、SSE 保持 identity 等以源码路径与注释为据）。
