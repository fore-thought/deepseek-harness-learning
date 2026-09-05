---
title: "委派与编排：subagent、workflow、Ralph、agent team"
tags: [dsh, subagent, workflow, orchestration]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{subagent,workflow,experimental}/*；docs: subagent / workflow；子代理F"
updated: 2026-09-04
---

# 委派与编排：subagent、workflow、Ralph、agent team

> `ctx.subagents` 是"把一轮工作交给别的 agent"的统一 seam——**同一个接口后面
> 从进程内新建子 agent 到委派给另一个产品（Claude Code/Codex/任意 ACP）**。
> 文档明示：subagent 与 bash 一样是**可选能力，不属于 agent loop**。

## 一次性 run 与可继续 Activation

- **one-shot**：`tool-subagent → ctx.subagents.start`：能力旗标校验
  （缺能力**拒 `UNSUPPORTED_CAPABILITY`，绝不接受了再静默忽略**）→ 铸造
  durable `SubagentDescriptorData`（按模式判别 one-shot|continuable）→
  provider 发布 `SubagentRun` → `subagent/start` emit → await 结果 →
  dispose 到完全停稳 → `subagent/end`。`signal` 是唯一取消通道
  （发布前清资源不发事件对；发布后取消轮次工作）。
- **continuable（fork/spawn）**：`startContinuable()` 预留 durable childId →
  provider 给 `ContinuableCreateSpec`——**fork = 带 seed**：父日志至最后一个
  `turn/end` 的**平衡已完成轮次前缀**，经 `CreateAgentOptions.seed` 灌入；
  spawn 无 seed；外部提供方（ACP/Codex/Claude Code）**拒绝 agentOptions**——唯一例外
  是 dsh-sdk：进程外通道里**只有一路**能传 agentOptions（含 agentRouteDefaults 白名单）。
  此后**冷恢复不经 provider**：管理器折叠 descriptor 事件 → `ctx.agents.resume()`。
- **消息路由三态**（`send_message`）：running=同 Activation `steer` 最近 step；
  waiting=唤醒后 steer；无 Activation=冷恢复再 steer。鉴权=确切在线 sender 的
  直接父子边。`interrupt` = `Agent.cancel(keepInbox:true)` 同步返回。
- **状态不设第二状态机**：Activation 的 running|waiting|settled 由
  "完全停稳 + ownedChildren 集合"**推导**；settled 时向持久 parent 投递
  `subagent-settled` 通知（与模型消息 kind 分离——你现在收到的完成通知就是这个）。
- 拆卸 **child-first**；`drainContinuableDescendants` 按 host 拥有的 Agent 界定。
- 冷列表身份 = `subagent/descriptor` 事件 **last-wins 投影折叠**
  （子自己的覆盖 fork 种子里的祖先记录）。

## workflow：脚本化的扇出（`packages/workflow/`）

`WorkflowEngine.start({script, meta, args, parent, signal})`：
每 run 一个 `node:worker_threads` + `node:vm` context，脚本编译为
`(async () => { body })()`，globals `agent/parallel/pipeline/phase/log`；
`agent()` 经 worker↔host RPC **走 subagent seam** 起子（编排原语复用，
不另造执行通道）。并发槽 FIFO+上限；**fatal `WorkflowError` 穿透
parallel/pipeline 不消融为逐项 null**；`result` 永不 reject，取消后有界宽限
强判 cancelled 并 terminate worker。观察事件对
`workflow/start|phase|log|agent-start|agent-end|end`（payload 克隆隔离）。

**Ralph**（`packages/workflow/tool-ralph`）不是新机制：= workflow + subagent
原语组合出的**前台全新 agent 循环**——每轮零上下文新子会话、共享工作区当持久记忆、
有界结构化交接（"Ralph 是策略不是模式"的实现）。

## agent team（experimental，`ctx.agentTeams`）

可继续 subagent 之上的**私有显式启用**协作 seam：持久 roster、任务板
（DAG：revision CAS + blockedBy 无环）、mailbox（queued-minus-delivered）。
观察：团队协调**没有加新执行原语**，全部复用 Activation + 事件投影。

## 相关

- 委派对象的创建/取消语义：[turn/step 主循环](../agent-runtime/turn-step-loop.md)
- fork 种子的日志边界：[会话事件日志](../agent-runtime/session-event-log.md)
- ACP 双面孔（被委派目标/对外服务器）：[应用壳](../platform/web-cli-boot.md)
- 实现层内幕（能力双闸门/Activation 驻留/四路传输/结算竞态窗口）：
  [委派内幕](../deep/subagent-deep.md)
- 工作流引擎与 Agent Teams 深读（上限表/配对账本/mailbox 语义）：
  [工作流与 Agent Teams](../deep/workflow-agent-team.md)
- 进程外嵌入通道（"不过对话边界"的后端）：[SDK 三件套深读](../deep/sdk-embedding.md)
