---
title: "Topic Navigation"
tags: [dsh, moc]
status: active
license: CC-BY-SA-4.0
updated: 2026-09-06
---

# DSH Source Architecture · Topic Navigation

[English](dsh-harness-index.md) | [中文](dsh-harness-index.zh.md)

> Goal: understand the architecture of DSH (an open-source TypeScript LLM agent
> harness) — pure subject knowledge; how to build something with it belongs to
> other topics. Read [the overview](./overview.md) for the panorama; enter
> sub-pages by layer.

## Suggested reading order

1. [DSH Source Architecture Overview](./overview.md) — panorama & design decisions
2. Framework base: [The Cordis Kernel](./cordis/cordis-kernel.md) →
   [Plugin Composition & Boot](./cordis/plugin-composition.md)
3. Agent backbone: [Session Event Log](./agent-runtime/session-event-log.md) →
   [Turn/Step Main Loop](./agent-runtime/turn-step-loop.md) →
   [Tool Registry & Execution Pipeline](./agent-runtime/tools-pipeline.md)
4. Model layer: [The LLM Layer](./llm-layer/llm-vocabulary.md) →
   [Context Engineering](./llm-layer/context-engineering.md)
5. Platform: [Session Persistence & Storage](./platform/session-persistence.md) →
   [Host/Client Split & API Gateway](./platform/host-client-boundary.md) →
   [Application Shells](./platform/web-cli-boot.md)
6. Execution world: [Filesystem & Sandbox](./execution/filesystem-and-sandbox.md) →
   [Processes, Shell & Terminal](./execution/shell-process-terminal.md) →
   [Code Runtime, LSP & Remote Worlds](./execution/remote-and-code-runtime.md)
7. Augmentation: [Delegation & Orchestration](./augmentation/subagent-orchestration.md) →
   [Capability Supply](./augmentation/skills-mcp-hooks.md) →
   [Self-Organization](./augmentation/goal-plan-todo.md) →
   [Q&A & Feedback](./augmentation/questions-and-answers.md) →
   [Background Tasks & Triggers](./augmentation/background-and-triggers.md) →
   [Settings Plane](./augmentation/settings-and-credentials.md)

## Source-level deep dives (29 · full coverage)

Source-level walkthroughs plus real-machine measurements. Every one of the **51
package groups** under `packages/` has at least one primary carrier page (the
table below doubles as the "who owns what" index). Suggested usage: first pass
reads the overview layers (1→7) to build the skeleton; deep dives are the second
pass. Each page is self-contained with re-verifiable evidence paths:

| Deep dive | Primary package groups | Overview companion |
|---|---|---|
| [Agent Loop Internals](./deep/agent-loop-internals.md) | core (all 8 pkgs) | [Turn/Step Main Loop](./agent-runtime/turn-step-loop.md) · [Tool Pipeline](./agent-runtime/tools-pipeline.md) |
| [Prompt Assembly & Runtime Context](./deep/prompt-assembly-context.md) | system-prompt · context (6 pkgs) · preset | [Turn/Step](./agent-runtime/turn-step-loop.md) · [Context Engineering](./llm-layer/context-engineering.md) |
| [Projections, Titles, Telemetry & Format Generations](./deep/session-projection-telemetry.md) | session (18 derivation pkgs) | [Session Persistence](./platform/session-persistence.md) · [Session Event Log](./agent-runtime/session-event-log.md) |
| [Session Query Internals](./deep/session-query.md) | session-query (4 pkgs) | [Session Persistence](./platform/session-persistence.md) |
| [Surface Rewriting & Compaction](./deep/surface-compaction.md) | compaction (4 pkgs) | [Session Event Log](./agent-runtime/session-event-log.md) · [Context Engineering](./llm-layer/context-engineering.md) |
| [LLM Adapters & Metering](./deep/llm-adapters-metering.md) | llm (7 pkgs incl token-meter) | [The LLM Layer](./llm-layer/llm-vocabulary.md) |
| [Attachments & Spill](./deep/attachment-spill.md) | attachment · spill (5 pkgs) | [Context Engineering](./llm-layer/context-engineering.md) |
| [Persistence Formats & Crash Recovery](./deep/persistence-crash-recovery.md) | session (persistence side) · storage (seam/backends) | [Session Persistence](./platform/session-persistence.md) |
| [HTTP Carriers & Controller Layering](./deep/host-gateway-webserver.md) | host (5 pkgs) · api (5 pkgs) | [Host/Client Split](./platform/host-client-boundary.md) · [App Shells](./platform/web-cli-boot.md) |
| [Web Client Browser Architecture](./deep/web-client-architecture.md) | client (all 45 pkgs) | [Host/Client Split](./platform/host-client-boundary.md) |
| [typert Remote Protocol & Codegen](./deep/typert-remote-protocol.md) | typert (4 pkgs) | [Host/Client Split](./platform/host-client-boundary.md) |
| [The SDK Trilogy](./deep/sdk-embedding.md) | sdk (3 pkgs) · examples | [App Shells](./platform/web-cli-boot.md) |
| [Profile Assembly & Six Bundles](./deep/boot-bundles.md) | boot (2) · bundle (6) · apps | [Plugin Composition](./cordis/plugin-composition.md) · [App Shells](./platform/web-cli-boot.md) |
| [Cordis Hot Restart & Hot Reload](./deep/cordis-hot-reload.md) | extensions (4) · vendor | [Plugin Composition](./cordis/plugin-composition.md) |
| [Filesystem Observation](./deep/filesystem-observation.md) | fs (all 7 pkgs) | [Filesystem & Sandbox](./execution/filesystem-and-sandbox.md) |
| [Sandbox Execution](./deep/sandbox-execution.md) | sandbox (4 pkgs) | [Filesystem & Sandbox](./execution/filesystem-and-sandbox.md) |
| [Shell & Terminal Internals](./deep/shell-terminal-internals.md) | shell (10) · subprocess · terminal · guard | [Processes, Shell & Terminal](./execution/shell-process-terminal.md) |
| [LSP & the E2B Remote World](./deep/lsp-e2b-remote.md) | lsp (3) · e2b (3) | [Code Runtime, LSP & Remote Worlds](./execution/remote-and-code-runtime.md) |
| [PTC & code-runtime Internals](./deep/ptc-code-runtime.md) | code-runtime (2 pkgs) | [Code Runtime…](./execution/remote-and-code-runtime.md) · [Tool Pipeline](./agent-runtime/tools-pipeline.md) |
| [Web Tools](./deep/web-tools.md) | web (all 6 pkgs) | [Tool Registry & Execution Pipeline](./agent-runtime/tools-pipeline.md) |
| [Delegation Internals](./deep/subagent-deep.md) | subagent (9 pkgs) · acp | [Delegation & Orchestration](./augmentation/subagent-orchestration.md) |
| [Workflow Engine & Agent Teams](./deep/workflow-agent-team.md) | workflow (4) · experimental (8) | [Delegation & Orchestration](./augmentation/subagent-orchestration.md) |
| [Self-Organization: Four Domains](./deep/goal-plan-todo-schedule.md) | goal · plan · todo · schedule | [Self-Organization](./augmentation/goal-plan-todo.md) |
| [Interaction, Approvals & Permission Tiers](./deep/interaction-approval-feedback.md) | interaction (5) · feedback · identity | [Q&A & Feedback](./augmentation/questions-and-answers.md) |
| [Settings Plane Deep Read](./deep/settings-credentials-workspace.md) | settings · credentials · workspace · storage-domain | [Settings Plane](./augmentation/settings-and-credentials.md) |
| [Skills, MCP & Hooks Internals](./deep/skill-mcp-hooks-internals.md) | skill (4) · mcp · hooks (3) | [Capability Supply](./augmentation/skills-mcp-hooks.md) |
| [Background Jobs & Webhooks](./deep/jobs-and-webhooks.md) | jobs (3 pkgs) · webhook (2) | [Background Tasks & Triggers](./augmentation/background-and-triggers.md) |
| [Invariants Registry](./deep/invariants-registry.md) | runtime-diagnostics · cross-cutting | [Session Event Log](./agent-runtime/session-event-log.md) |
| [Engineering Base](./deep/engineering-base.md) | util (13) · test-support · native · python | (new topic, no overview counterpart) |

## Evidence baseline

All conclusions come from direct reading of DSH source (a local source checkout);
evidence uses source-repo-relative paths and can be re-verified. Unresolved
points carry an inline `pending verification` marker. Baseline snapshot: 2026-09;
dated "current/corrected" markers on deep-dive pages are check-back points.
