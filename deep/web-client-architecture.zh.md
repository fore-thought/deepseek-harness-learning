---
title: "Web Client：浏览器里的第二个 Cordis 应用"
tags: [dsh, web-client, browser, slots, conversation, boot]
status: active
license: CC-BY-SA-4.0
evidence: "packages/client/*（45 包）package.json 全量清点 + docs/subsystems/{web-client,client-modules,slots,conversation}.zh.md + {web,modules,connection,store,ui-conversation,ui-tool,ui-settings}/README.zh.md 直读"
updated: 2026-09-05
---

# Web Client：浏览器里的第二个 Cordis 应用

[English](web-client-architecture.md) | [中文](web-client-architecture.zh.md)

> 总览全景图里那句"Web Client（浏览器内独立 Cordis 应用）"不是修辞：浏览器端有**自己的**
> 插件树、自己的模块系统、自己的服务注入与事件装配。本篇按架构级走读
> `packages/client/` 全组 **45 包** [MEASURED]（38 个 ui-* + 7 个底座，
> 其中 41 个在 package.json 声明 `dsh.client` [MEASURED]）；
> Host 侧网关与代码生成边界见 [host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)，
> 本篇只写浏览器这一侧，不重复。

## 总图：一条依赖方向线

官方分层（`docs/subsystems/web-client.zh.md`）把所有包钉在一条单向依赖链上：

```text
Host 权威状态 → Remote 传输 → Client model → UI adapter → Conversation/呈现 → Slots → React
```

反向只走 callback：用户操作进入**注入的** Client service 或生成的 Remote namespace。
三条铁律贯穿全组——组件绝不接收 Cordis `ctx`；功能包不得运行时 import 另一功能包的值
（共享运行时值必须有职责收窄的静态 owner：`store`/`ui-primitives`/浏览器安全 util）；
presentation 绝不复制 transport state。

## 启动：一张图的旅程

浏览器应用由 Host 组合出的 `WebBootGraph` 驱动，链路上四个关键设计：

1. **声明即入图**：包在 package.json 写 `dsh.client`（`platform: 'web'` + 可选
   `inject` 边 + `immediately`）并导出 `./client` bundle，node 半侧
   `ctx.clientModules`（`ClientModuleRegistry`）就把它变成 `/plugins` 下的
   带版本 bundle 与图上一行。扫描是**单包增量**的——不存在全量重扫代码路径；
   fiber 构造/释放把 entry 名标脏，一次微任务 flush 对账。
2. **loud/quiet 双姿态**：激活趟里坏声明聚合成一个 `AggregateError` 点名每个坏包
   （fiber 进 FAILED）；稳态里坏包只留一条警告、不得殃及他包。同一实现、相反姿态。
3. **index 注入即协议**：Host 向 `<head>` 依次注入 facade script、application combo 的
   preload、parser-blocking 的 bootstrap combo，最后才是 `window.__DSH_BOOT__` 图本身
   （`<` 已转义，插件可控字符串逃不出 script 元素）。解析器拒绝畸形 row——
   没有有效 manifest 的页面**无法启动**，而不是带病运行。
4. **两阶段 boot**：`dsh-client-web` 的 `AppWebEntry.run()` 先模块阶段（接纳 bootstrap
   批次、按图预取 `immediately` 层），再插件阶段（创建全部 entry、Cordis 注入决定激活
   顺序、等待 settled、审计激活）。全部就绪后才把带标记的启动 DOM 交给 `ui-renderer`
   hydrate，并触发唯一一次 `renderSlot('root')`。

白屏防护值得单记：启动页是**无框架**原生 DOM + 本地 CSS，spinner 弧随 entry 激活增长、
逐 entry 报告失败原因——bundle 挂了、插件缺服务挂了，都可见而非空白。
"用更少的代码扛住启动期"本身就是插件树外的 bootstrap 语义。

## 惰性 CJS 模块表

浏览器半（`ctx.modules`，`ClientModuleSystem`）是一张 lazy CommonJS 表，关键性质：

- **加载 ≠ 执行**：materialize bundle 只注册 factory；一切模块副作用（含 CSS 注入）
  在首次物化时才跑。factory 依赖未物化模块则递归物化；require 环**直接抛错**——
  factory 形态的 CJS 给不出部分导出，宁错不糊。
- **解析顺序固定**：platform seed 表 → 记忆化缓存 → 启动图 row → 已注册 factory，
  其余一律抛错。`PLATFORM_MODULES`（React、Cordis、静态 UI 库）是冻结基座，
  `dsh.client.external` 只允许"基座之外的精确请求"——纯类型 import 被擦除不产生请求。
- **combo 与 rev**：资源分组进 `/plugins/??a.js,b.js&rev=...`，每条启动 URL 按 UTF-8
  字节 ≤ 3 KiB [MEASURED] 贪心切分；初始 rev 是进程 nonce + 序号（启动期**不哈希**
  每个插件产物），只有 HMR 观察到变化才换成 bundle+map 的哈希。map 一律
  Indexed Source Map v3；未知/被改的资源列表、缺 rev、陈旧 rev 全部 404，
  绝不让 SPA fallback 把 HTML 当 JS 吐回去。
- **vendored Loader 的唯一消费点**是 `EntryTree.import`——模块系统是"插件代码如何到达
  浏览器"的唯一可替换实现（[Cordis 热重启](./cordis-hot-reload.zh.md)的浏览器镜像）。

## connection：三道门与一个 generation

物理层 `client/connection` 是浏览器侧最"安全工程"密度高的一包，官方口径拆出三道门：

| 门 | 机制 | 失败姿态 |
|---|---|---|
| 会话认证 | 每进程随机启动令牌；`GET /?token=...` 换签名 cookie 后重定向干净 `/` | cookie 缺失/过期/authority 不符 → RPC 分发前 401 |
| 请求信任 | `Host` 必须 loopback 或匹配 `trustedHosts`（WHATWG 归一化）；带 `Origin` 必须等于 Host；`sec-fetch-site: cross-site` 一律拒 | 信任失败 403；信任但未认证 401 |
| 密钥归属 | cookie 签名密钥 = `ctx.credentials` 里 `client-connection/browser-session` 的 grant（`$DSH_HOME/.credentials.yaml`）；删除并重启 = 撤销全部会话 | host-only、`Path=/`、`HttpOnly`、`SameSite=Strict`；loopback HTTP 刻意不设 `Secure` |

（表注：信任检查"防 DNS rebinding 与跨站请求，**绝不建立身份**"——身份是 token 门的事。）

generation 机制解决一个竞态：`$events` logical stream 是**唯一** generation source，
其 opening `ready` frame 携带 Host home，且 Host 在**所有增量 listener 同步挂好后**才发它
——baseline 永远不会跑在增量前面。失效（stream 结束/首项非 ready/畸形事件）→
Controller 按 500ms、1s、2s、4s、8s、10s 上限的 50%–100% 抖动梯重试；`offline` 暂停、
`online` 重置回第一档。没有统一 `Runtime` 或 `resync()` API——恢复语义按数据语义
分散在各自的 model 里，这是明示的架构决定。

## Client model：镜像而非第二真源

每个 API controller 包有配对的 Host face / Client face。以 sessions 为例，
`ClientSessions → SessionManager → Session` 三级：list baseline、惰性会话实例、
projection store、subagent catalog、queue 都归 Manager；每个 `Session` 持一段连续的
`SessionEventLikeEntry` 窗口。持久读路径的重连语义最讲究：

- `follow()` 首帧 = 当前 header + tail page + cursor + **完整 projection baseline**；
  journal 先校验每条 record 的逻辑 seq 闭区间，Client 端**免转换**直接保留为窗口条目；
  每代 opening snapshot **原子替换**保留窗口，其后按 seq append。
- `page()` 只用于更早历史与 gap repair——它不是一条平行时间线。
- control/Workspace 这类瞬态流：断开期间**保留最后发布的值**，重连用新 baseline 替换；
  普通转发通知不 replay——要可靠恢复的域必须自带 baseline/cursor/query。

"配对不产生第二份业务真相"：Host 定持久状态与 mutation 结果，Client model 只做
identity 稳定的最新可用投影，并把"迟到响应 vs 替换 baseline"的合并规则写死在代码里。
这与 [会话事件日志](../agent-runtime/session-event-log.zh.md)的"唯一真源+投影"哲学同构——
浏览器只是又一个投影消费者。

## Slots：类型化的 UI 组合代数

`ui-slots`（纯核心、零 React、不声明 `dsh.client`）+ `ui-renderer`（唯一绑 React、
唯一 `useSyncExternalStore` 处）撑起组合面。声明侧是编译期 `SlotMap`：key、
cardinality、scope、owner props、keyed props、slot 级 inject face 一次写全。
两个正交维度：**cardinality** `single`/`list`/`keyed`/`chain`（chain=每人提供纯
`select(owner)`，第一个非 null 者当选、以 `matched` 传组件）；**scope**
`root`/`session`/`session-maybe`。

生命周期语义是 Cordis 式的：一个声明同时产生三效果（key 生效、授权 parent
`renderSlot`、记录 dispatch 规格）；**每个声明只能有一个存活 owner**；owner 销毁时
递归折叠它声明的 child slots——所以向别人的 slot 贡献一律走
`ctx.slots.inject(key, cb)`，cb 在每段声明生命周期内重跑。可选功能 entry 作为
一个生命周期单元让整棵子树出现或消失。

组件收到的 props 是**推导**出来的六件套（`PropsRuntime`/`PropsRenderSlots`/`PropsStore`/
`InjectFace`/`PropsLocale`/`matched`），配一条四级分诊口诀：owner 渲染时已知的值走
owner props；单 entry 私有走注册项 `inject`；全体 occupant 共享的能力走 slot 级
inject face（`conversation.chat.node` 的 `useTurnData(key)` 即此）；跨 entry 可变
视图状态走声明的 store。React node 通过 child slot 组合，不通过注入值传。

## Conversation：事件流到视图的装配层

`ui-conversation` 拥有双 registry（`events`/`views`，拒重复 key、保注册顺序、返回幂等
disposer）与逐 Session identity-stable binding。数据模型四个概念：Definition 按稳定
`(kind, id)` 关联事件、折叠确定性 State；Context 是一个 `(kind,id)` 的有序 Match 集；
Location 由持久 boundary 事件推导 Turn/Step 坐标；View Definition 产 target 自有
snapshot。**可回放 event family 契约**是写给所有业务包的：start 唯一、delta 携带稳定
id 且按 seq 升序确定性回放、只有 update 时留 pending Context 等分页补齐——
"Client 绝不把 update 猜给最近一个未完成 Context"。

Assistant 流式的处理最巧：实时 delta 以 Client-only `assistant/live-chunk` 瞬态 entry
进入，持久 `assistant/message` 内嵌完整紧凑 stream 供历史回放——同一组
`match`/`update` 消费两种形态，于是**重连与分页历史无需持久 token 行就能复现相同
Assistant 状态**（与 [表层改写与压缩](./surface-compaction.zh.md)的"模型侧表面"互为一体两面：
那边改写给模型的输入，这边装配给人的输出）。
target 激活集合单调增长；View 选择固定为"持久化偏好 > 已注册 `chat` > 不渲染"，
绝不选第一个注册者。

## 一个 UI 插件的标准形状（三例横切）

- **`ui-tool`（keyed 分发 exemplar）**：`ui-conversation` 把排好序的 `tool-call` node
  经匹配 key 交给它，它再经 keyed slot `tool.call.toolview` 按 **wire 工具名**分发原子
  视图；root/subcall 拓扑、call/result 配对归运行时，业务包只注册一张卡——没注册的
  工具走 generic fallback。owner 载荷是 `ToolCallOwnerProps`（冻结 block、会话授权
  `loadImage`、`openFile`/`inspect` 回调）；Host 侧 `presentCall`/`presentResult`
  值**不进 Client**——呈现逻辑也是"发布即耐久"式的分层。
- **`ui-settings`（底座 exemplar）**：`ctx.settingsScope.bind(spec)` 给功能一个
  按命名空间的 scope，每次写以命名空间 revision 作 `expectedRevision` 围栏——
  并发写**被拒绝而非静默覆盖**；它声明 `settings.section` 等 slot 类型但自己
  不渲染任何东西（外壳在 `ui-settings-general`），因此任何功能都够得到它。
  浏览器只持**一面** describe 镜像，在 `settings/document-updated` 事件与
  `connection/reset` 时刷新——首连也算，堵死"提交落在急切读取与订阅之间"的窗口。
- **`ui-conversation`（复杂度上限 exemplar）**：乐观提交流水——Enter 同事务清草稿/
  撤销历史、发送作为 detached attempt 运行（发送期可继续输入）；回显经
  `session.beginSubmission` 注册、以 observed 退休只发生一次；图片走浏览器
  `FileReader`、文件走"已暂存凭证 + FIFO 后台上传队列"（`maxConcurrentFileUploads`
  默认 2 [MEASURED，README 口径]），切 Session 时传输与字节进度由 service 续持。
  一个 composer chain 还是审批/提问包的**临时接管位**（`chain` cardinality 实战，
  见 [人机问答与反馈](../augmentation/questions-and-answers.zh.md)）。

## 45 包目录表（按域分组，每包一行）

底座机件（7）：

| 包 | 一句话 |
|---|---|
| `web` | 启动内核：模块系统 + vendored Loader + 无框架启动页（不声明 dsh.client，它就是外壳） |
| `modules` | 双面孔模块表：node 半组合 `__DSH_BOOT__`，浏览器半惰性物化 |
| `connection` | 线层：token/cookie 认证、信任检查、generation 与重试 |
| `file-upload` | agent-scoped 浏览器文件上传、流式 intake、staged receipt |
| `hmr` | 开发热重载驱动：SSE rebuilt 帧 → invalidate/prefetch → fiber 重启 |
| `locale` | 语言包：Host 偏好 + 可扩展语料目录 + 浏览器回退 + 类型化字典 |
| `store` | 零 React 的 observable/snapshot-store 契约（Zustand/Immer 引擎共享） |

装配层（3）：`ui-slots`（registry 纯核心）、`ui-renderer`（唯一 React 绑定与 root 渲染）、
`ui-layout`（三栏 AppFrame + `ctx.layout` 视图态）。

model 适配（4）：`ui-session`（session scope adapter 与标准 hooks）、`ui-workspace`
（workspace picker 注册进 sidebar/empty-state）、`ui-conversation`（装配+shell+composer）、
`ui-commands`（全局目录缓存、`/` 源、popupSelect registry）。

对话呈现 target（5）：`ui-chat`（Chat target 与 node renderer）、`ui-trajectory`
（事件账本 + 时间线总览）、`ui-tool`（调用树与 keyed 工具视图）、`ui-deliverables`
（产出文件 turn tail 与可点文件引用）、`ui-reference`（统一 @file/@session 引用源）。

功能面（26）：`ui-approval`（composer 接管审批瀑布）、`ui-user-questions`（提问接管与
plan 审阅）、`ui-goal`（GoalBar 读 goal 投影）、`ui-plan`（plan 席位）、`ui-subagent`
（子代理目录与续谈路由）、`ui-workflow-run`（持久 workflow node）、`ui-jobs`（header
后台任务列表）、`ui-schedule`（只读活动日程）、`ui-skill`（技能引用与工具行）、
`ui-message-feedback`（逐消息反馈条）、`ui-input-trigger`（`/`、`@` 检测与候选菜单）、
`ui-model-selection`（模型选择）、`ui-permission-presets`（权限档位）、`ui-agent-preset`
（preset 席位与组合编辑器）、`ui-attachment`（附件呈现）、`ui-sidebar`（会话树/搜索/分组）、
`ui-theme`（Host bootstrap 调色 + DOM-free ThemeRuntime）、`ui-brand-official`
（官方品牌位占位）、`ui-directory-picker-browse` / `ui-directory-picker-native`
（两种目录选择面）、`ui-settings-general`（设置外壳与 onboarding）、`ui-settings-models`
（模型设置）、`ui-settings-plugins` / `ui-settings-plugin-inventory`（插件分区与只读
Loader 清单）、`ui-primitives`（纯 React 原子件：controls/icons/markdown/JSON inspector，
零 cordis、不声明 dsh.client）、`ui-settings`（设置领域底座，见上节）。

## 相关

- Host 侧边界与 typert 生成：[host/client 分层与 API 网关](../platform/host-client-boundary.zh.md)
  · [typert 协议深读](./typert-remote-protocol.zh.md)
- 事件日志与投影：[会话事件日志](../agent-runtime/session-event-log.zh.md)
- 主干装配（对照"第二个应用"的第一应用）：[插件组装与启动](../cordis/plugin-composition.zh.md)
- 工具呈现的运行时侧：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)

## 诚实边界

- ui-* 包按"架构级"口径以 README + package.json 全量清点成文；38 个 ui-* 的内部源码
  未逐行读，目录表一句话职责来自各自 package.json description（作者口径）。
- 未做浏览器端真机实测（无 GUI 环境）：启动页行为、combo 404 姿态、SSE 广播均系
  README/子系统页口径；三处关键 README 与 docs/subsystems 交叉核对无矛盾。
- `api-gateway`/`session-controller` 的 Host 侧实现细节属 host-client-boundary 篇管辖；
  `host/webserver` 载体与 index 渲染挂接未展开（归 R5 的 host 篇）。
- slot 声明树只摘录了 root 起始段；完整 roster 以 `docs/subsystems/slots.zh.md`
  "当前层级"为准。
- 数字口径：45/41/38 为本 checkout package.json 实测 [MEASURED]；3 KiB URL 帽、
  重试梯度、`maxConcurrentFileUploads=2` 为 README/子系统页声明，未逐项复算常量源码。
