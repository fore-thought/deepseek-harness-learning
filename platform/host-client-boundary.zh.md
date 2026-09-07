---
title: "host/client 分层与 API 网关"
tags: [dsh, host, client, remote, typert]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{api,host,client,typert}/*；docs: api-gateway / web-client；子代理E"
updated: 2026-09-05
---

# host/client 分层与 API 网关

[English](host-client-boundary.md) | [中文](host-client-boundary.zh.md)

> **Host** = 一个 Node 进程（`dsh` 起的 Cordis 树）：拥有权威状态、持久化、
> mutation 顺序。**Client** = 浏览器内**另一个独立 Cordis 应用**：只维护 Host 状态的
> 镜像与 UI。两侧不是"一个进程分两半"，而是**两棵树通过窄线对话**——这是 DSH Web 版
> 最重要的结构事实。

## Typert：从 TS 类型生成远程边界

`packages/typert/`（protocol/registry/loader/generator 四包）：

- 服务端方法标 `@Remote`（运行时**零包装**：只在原型写字符串键 marker，
  跨协议包副本可读；非 symbol）→ **构建期**用 ts.Program 静态分析生成
  descriptor + Zod schema + 双端声明合并产物；`TypertLookupMap` 传身份不传快照，
  **wire id 即领域 id、解析是 store 的无状态 resolver（无引用计数/GC）**——
  生产实体全集只有 `agent`/`session` 两个（深读修正）；
- `ctx.typertGateway`（Host 侧分发）：args 精确匹配 → 输入**双向强校验**
  （client parseInput + host zod parse + JSON-safe 断言）→ lookup 解析（失败
  `RemoteError session/not-found|agent-busy`）→ 调实时服务方法 → **输出无运行时
  parse**（深读修正：`descriptor.result` 的 codec 在任何运行路径都不执行——
  result schema 的真实角色 = 生成期约束 + 编译期类型，"返回值同样校验"不成立）；
- 流类型三件套：`RemoteStream / RemoteSnapshotStream / RemoteJournalStream`；
- `RemoteResult{ok}` **永不因载体 reject**——错误码按领域所有者分层（封闭/开放混合），
  线级错误与业务错误永不混淆。

HTTP 载体与控制器装配面（路由三件套/回退席位/17 个 gateway 码分层）：
[HTTP 载体与控制器分层](../deep/host-gateway-webserver.zh.md)。

## 一元链与流链

```text
一元：ctx.remote.ns.method → connection.rpc.call('/api', '<ns>/<method>')
      → POST /api/…（Host/Origin + cookie 信任检查前置于分发）→ typertGateway.invoke
流：  follow = opening snapshot{header, cursor, tail records, projections baseline}
        + 逐个 event 帧  → WebSocket /api/remote.mux（2s ping）
      → client 校验 seq 闭区间、follow-before-page、page() 补缺口
        （深读补：seq 校验全在 client；重连纪律=每连接代至多重试 1 次；
        背压=无应用层协议，靠单条共享写链 await 串行 + 2s 心跳 missed2 判死）
      control 流 = 每代 baseline + queue/jobs/projection 替换帧（投影帧链：
      session/event → registry.apply → onChanged → 'projection' 帧）
```

- 关键纪律：**`session/event` 不走"事件转发 allowlist"**——follow 流按会话定制，
  allowlist（`packages/api/remotes/src/remote-events.ts`，18 项）只管跨切面事件；
  名单**单一源文件双 face 共享**（host 与 client 编译同一清单）；
- 名单里只有 `approval/request` 与 `user-questions/request` 两项是
  **waterfall**：浏览器应答要**回到 Host 参与决策**（你收到提问卡片的那条线就是这个）；
- 快照流恢复语义**按数据类型定制**（每代 opening baseline 替换、普通通知不 replay），
  没有统一 resync API——"帧链"显式编码进各控制器。

## 连接与信任

- `ctx.connection`：generation 计数 + **信任栅栏**（token 换签名 host-only cookie
  再 302 干净根路径；LAN trust 采样）；首帧 `{type:'ready', clientId, host:{home}}`。
- 双 face 包结构：api/remotes、gateway、session-controller、workspace-controller、
  client/connection 各拆 `tsconfig.host.json/tsconfig.client.json`——
  **一个 npm 包两种编译形态**，构建序 Host 生成物 → Client 消费。
- Electron 走 file:// + IPC，不经过 webserver（同 client 树换传输）。

浏览器侧连接内幕（三道门/generation 唯一源/两形态 journal）：
[Web Client 架构](../deep/web-client-architecture.zh.md)。

## 为什么这么分

Host 权威 + Client 镜像 = 多标签页/多设备看同一会话、断线重连按 seq 补帧、
UI 崩溃不伤事实。而"浏览器里也是 Cordis"意味着**前端同样一切皆插件**：
chat 节点（`ConversationNodeDefinition` + keyed renderer）、渲染槽位
（`ui-slots`）都是注册项——客户端的"一切皆插件"和宿主同构。

## 相关

- SDK/ACP 两个进程外面孔：[应用壳](./web-cli-boot.zh.md)
- 投影基线从哪来：[会话事件日志](../agent-runtime/session-event-log.zh.md)
- 提问/审批的领域语义：[人机问答与反馈](../augmentation/questions-and-answers.zh.md)
- 协议与代码生成的源码级内幕（含概览篇三处修正的完整证据链）：
  [typert 远程协议与代码生成](../deep/typert-remote-protocol.zh.md)
