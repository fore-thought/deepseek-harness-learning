---
title: "turn/step 主循环：agent loop 的心脏"
tags: [dsh, agent-loop, runtime]
status: active
license: CC-BY-SA-4.0
evidence: "packages/core/{agent,agent-loop}/src；docs: agent-lifecycle / core；子代理B"
updated: 2026-09-05
---

# turn/step 主循环：agent loop 的心脏

> `ctx.agentLoop`（包 `packages/core/agent-loop`）是默认驱动器；
> `ctx.agents`（包 `packages/core/agent`）只是**接口 + 注册表 + 事件词汇**。
> 消费方（UI、远程、subagent 提供方）都面向 `Agent` 句柄编程，不依赖循环包——
> Provider 反转：**循环注入 agents，agents 不知道循环**。换循环=换一行 patch。

## 层级词汇（术语表 docs/glossary.zh.md）

- **轮次 turn**：一次对已接纳输入的排空，含 0..n 个步骤；
- **步骤 step**：一次模型请求 + 其引发的工具执行；
- **Round**：外层策略迭代（Goal Round/Ralph Round），归策略所有，不是循环内概念。

## 一次 turn 的骨架（`agent.ts` 的 `turn()`）

```text
turn/start (seq 落盘)
 loop:
   preStep(target): inbox.claim —— 先 durable 记账再领取
     (agent/inbox/spliced + 逐条 agent/inbox/claimed)
   → systemPrompt.assemble(作用域链合并的提示词段 + 工具 schema)
   → RuntimeContext 投影(动态上下文变化才生成注入消息)
   → agent/pre-step  waterfall   ← 拦截点①：改写/拒绝模型可见输入
      reject → blocked 收轮；step0 空批次 → completed 收轮(无步骤轮也留痕)
   step/start → 逐条 append user/message
   → agent/request   waterfall   ← 拦截点②：改请求 config
   → llm/stream（适配器调用；逐 chunk append assistant/chunk）
      失败 → agent/request-error waterfall ← 拦截点③：retry 决策(重试/压缩见 llm-layer/context-engineering.md)
   → assistant/message 落盘(usage+chunk seqs)
   → 工具调度(见 tools-pipeline.md)
   step/end
   工具欠请求 或 新 next-step 输入到达 → target='next-step' 继续
 end loop
 → agent/turn-stopping serial 终检 ← 拦截点④：续跑/收尾策略(goal driver 在此；
    消费方最完整示例见 deep/goal-plan-todo-schedule.md)
turn/end{reason: completed|aborted|blocked|error|max-tokens|interrupted}
```

要点：

- **inbox 是唯一输入闸**：`send(msg, target, wakeup)`，
  `target ∈ {next-turn, next-step}`；follow-up 立刻唤醒、steer 注入当前轮、
  inject 的上下文可以静躺。`Inbox` 内存态只是 durable `agent/inbox/spliced`
  事件的重放投影（先持久化再改内存），`claim` 是纯删除+记账——
  **被拒绝的批次保持已移除**（claim-then-enter），不重投。
- **相位状态机**（私有）：`idle | maintenance | running` + `AbortController`；
  `AgentStatus` 对外只有 idle/running，且 running 覆盖整个 driver 排空区间。
  `maintenance`（`runMaintenance`）窗口里跑压缩类后台工作——**对外仍显 idle**，
  且与驱动轮次不并发。
  `kick = while (await turn()) {}`；轮末 inbox 仍有 pending → 换新 AbortController 续跑。
- **max-tokens 粘性**：因 max-tokens 结束的步骤不降级，坚持到轮结束。
- **取消**：`cancel(cause, {keepInbox})` → 默认清 inbox（durable canceled；`keepInbox` 则
  保留收件箱且**不记 canceled splice**）+
  abort(类型化 cause：user/parent/hook/disposed 复制进 `AbortSignal.reason`)；
  持久化为 `turn/end{aborted}`；取消途中到达的新输入由 `wakingAfterAbort`
  **先分类再插入**。崩溃时未闭合轮由 resume 合成 interrupted（见 [会话事件日志](./session-event-log.md)）。
- **创建事务**：`prepare()` 三源融合取消（调用方 signal/所属 fiber 卸载/factory 拆迁）；
  发布顺序 `sessions.enter → agents.enter → announce → agent/session-start`，
  任一步失败**逆序回滚**。构造即 `createScope(...)$——**Agent 对象本身就是 scope key**
  （[工具注册表与执行流水线](./tools-pipeline.md) 的作用域层由它派生）。
- **因果归属**：`AgentRegistry` 用 AsyncLocalStorage 双槽记 initiator（谁触发了这次
  agent 工作），支撑权限审批与遥测的"人-机-钩子"归因；`withoutInitiator` 可显式清除。

## request/header 与请求纪元

步骤前按"渲染后 system + tools + LlmCallConfig"做 `canonicalHeader + headerEquals`
比较，**变了才 append `request/header`**（reason: initial/resume/change/series；
series 触发**含 surface 代际变化**——header 内容不变也会写一条）。生产侧三裁决与
`request/context` 三元组差量见 [主干循环内幕](../deep/agent-loop-internals.md) 与
[LLM 适配器与计量内幕](../deep/llm-adapters-metering.md)。
这是"模型可见即已记录"的落地：回放任何一步都能精确重建当时的请求纪元，
也告诉下游"从这里开始历史被改写"（配合 surface replace）。每个请求都记录请求纪元——
提示词/工具集变更是显式事件，而不是隐式状态。

## 相关

- 事件词汇与投影：[会话事件日志](./session-event-log.md)
- 步骤内工具怎么跑：[工具注册表与执行流水线](./tools-pipeline.md)
- 循环挂上来的增强策略：[自组织](../augmentation/goal-plan-todo.md) · [上下文工程](../llm-layer/context-engineering.md)
- 主干实现层内幕：[agent-loop-internals](../deep/agent-loop-internals.md)；pre-step 消费者谱系：
  [提示词组装与上下文](../deep/prompt-assembly-context.md)
