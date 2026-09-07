---
title: "能力供给：skills、commands、MCP、hooks、preset"
tags: [dsh, skills, mcp, hooks, preset]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{skill,mcp,hooks,preset,extensions}/*；docs: skills / commands；子代理F"
updated: 2026-09-04
---

# 能力供给：skills、commands、MCP、hooks、preset

[English](skills-mcp-hooks.md) | [中文](skills-mcp-hooks.zh.md)

> 这一组回答"模型/人类的能力从哪来、怎么挂上"。**skill=按需加载的提示词包，
> command=人类斜杠孔，MCP=外部工具桥，hooks=外部事件桥，preset=整棵子树套餐**。

## skills：发现 → 目录 → 按名加载

- 提供方 `skill-filesystem` 按 **rank 分层**扫根（100 `<project>/.dsh/skills` →
  200 `.agents/skills` → 300 custom → 400 `<dshHome>/skills` → 500 agentsHome →
  600 bundled；项目根=最近含 `.git` 祖先，经 `ctx.fs` 探测）；
  层合并 nearest-wins，rank 只在层内裁决。
- 失效双观测：chokidar watch + **模型 write/edit 观测**（沙箱写文件也算变更信号）→
  `provider.invalidate()` → `skills/change`（**不带 diff**，消费方按自身
  选项重拉 snapshot）；目录缓存按解析后 scope 链为键；**`ctx.skills.get()` 永不缓存正文**。
- 模型侧只注入 name/description 目录（从不注入正文/绝对路径）；`skill` 工具
  按 `name` 加载 content（你收到过的 skill 卡片即此）。
  `SkillInvocationPolicy{modelInvocable, userInvocable}` 双向门控；
  `SkillResourceBase{directory|url|opaque}` 决定附属资源可达方式。

## commands：人类命令平面（不过模型）

`ctx.commands`：`CommandRuntime{register, list, parse, dispatch}`；
斜杠指令由面向人类的适配器执行、**不成为模型消息**；输出属 UI 状态，
除非处理器显式改持久领域。skills 的 `userInvocable` 入口也注册在这里——
同一注册表两种来源。

## MCP：只桥接 Tools 的最小面

`packages/mcp/mcp-client`：连接监督器 + **世代原子交换**（重连成功才换轨）、
指数退避预算、stdio spawn 走清洗环境；工具稳定命名
`mcp__<serverName>__<tool>`。桥接克制：不代管 resources/prompts——
**能力词汇表保持小**。

## hooks：外部生态事件桥（Claude Code / Codex 两套配置方言共用一个引擎）

`packages/hooks/hook-protocol` 是唯一引擎（唯一差异轴=matcher pattern 语义）。
桥注册的 Cordis 监听器即挂点：
`agent/session-start`(SessionStart)、`agent/pre-step`(UserPromptSubmit，可阻塞+注入)、
`tools/pre-execute`(PreToolUse→`PreToolDecision`)、`tools/post-execute`(PostToolUse)、
`agent/turn-stopping`(Stop)、`subagent/start|end`。
command 钩子由 hook-protocol 库函数**经 `ctx.shell` 执行器**执行（环境清洗/进程组
取消/超时）——不是"存在叫 runHook 的 shell 方法"这层意思（深读纠偏）；
**exit 2 = 阻塞**（stderr 成模型可见原因）、其它失败不阻塞；决策合并
`deny > ask > allow`。`hook/invoked|hook/result` 配对事件在轮次内落日志
（invariant 伴生强制）。

## preset：agent 能力整棵树的外卖

`ctx.agentPresets`：`<preset>/.dsh/agent.cordis.yml`（信任根判别
`PresetRoot{id, trust}`）**常驻挂载**：同 id single-flight 挂到 standing scope，
多个 agent"认父 scope key 加入"；组装文件 mtime+size=代际 stamp 驱动重挂；
被挂子树 `write()` 抑制（preset 的 YAML 不被运行时写回）。
会话级能力集切换=`agent-preset/selected` 事件+投影。
→ "让某个会话拥有不同能力集合"的官方答案（architecture.zh.md 归属映射表）。

## extensions 与 identity（一句话定位）

- `packages/extensions`：运行中的 agent 可动态定义/运行/移除 Cordis 包的
  双半沙箱 runner（`ctx.dynamicCordisRunner`）+ 模型侧工具 `tool-cordis`——
  "harness 能被 harness 现场改装"（详见其文档）。
- `packages/identity`：每 harness home 一个匿名 id（遥测/反馈/提供方请求共用）。

## 相关

- 三桥内幕（revision 缓存/MCP 命名折叠/双方言单引擎/stdio 不沙箱边界）：
  [技能·MCP·hooks 内幕](../deep/skill-mcp-hooks-internals.zh.md)

- hooks/approval 共用的决策点：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.zh.md)
- 挂载语法（!!js / isolate 行）：[插件组装与启动](../cordis/plugin-composition.zh.md)
- 后台任务与外部触发：[后台任务与外部触发](./background-and-triggers.zh.md)
