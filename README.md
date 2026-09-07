# DeepSeek Harness — Source Architecture Handbook

English | [中文](README.zh.md)

A handbook that explains the architecture of **DeepSeek Harness** (`dsh`, an
open-source LLM agent harness) strictly from its source code — written so a
reader with no programming background can follow it, and ending with a mental
map complete enough to design a harness of your own.

- Independent third-party work; not affiliated with the official project.
- Based on a 2026-09 source checkout; every claim carries source-repo-relative
  evidence paths and can be re-verified.
- Every page exists twice: `foo.md` (English) and `foo.zh.md` (Chinese), same
  directory, identical layout and content. Pages still marked `status: draft`
  with an i18n-stub tag are placeholders whose translation is pending.

## Status

| Line | Content | Where |
|---|---|---|
| `v1.0.0` (tag, frozen) | complete Chinese edition: 22 overview + 29 source-level |
  deep dives, covering all 51 package groups | repository Tags page; errata via `v1.x` |
| `main` (v2 line) | bilingual handbook rewrite under the style charter:
  diagram-first prose, background pages, zero-assumption wording | current view |

## Structure

- Panorama: [overview](./overview.md) · [topic navigation](./dsh-harness-index.md)
- Framework base: [The Cordis Kernel](./cordis/cordis-kernel.md) ·
  [Plugin Composition & Boot](./cordis/plugin-composition.md)
- Agent backbone: [Session Event Log](./agent-runtime/session-event-log.md) ·
  [Turn/Step Main Loop](./agent-runtime/turn-step-loop.md) ·
  [Tool Registry & Execution Pipeline](./agent-runtime/tools-pipeline.md)
- Model layer: [The LLM Layer](./llm-layer/llm-vocabulary.md) ·
  [Context Engineering](./llm-layer/context-engineering.md)
- Platform: [Session Persistence & Storage](./platform/session-persistence.md) ·
  [Host/Client Split & API Gateway](./platform/host-client-boundary.md) ·
  [Application Shells](./platform/web-cli-boot.md)
- Execution world: [Filesystem & Sandbox](./execution/filesystem-and-sandbox.md) ·
  [Processes, Shell & Terminal](./execution/shell-process-terminal.md) ·
  [Code Runtime, LSP & Remote Worlds](./execution/remote-and-code-runtime.md)
- Augmentation: [Delegation & Orchestration](./augmentation/subagent-orchestration.md) ·
  [Capability Supply](./augmentation/skills-mcp-hooks.md) ·
  [Self-Organization](./augmentation/goal-plan-todo.md) ·
  [Q&A & Feedback](./augmentation/questions-and-answers.md) ·
  [Background Tasks & Triggers](./augmentation/background-and-triggers.md) ·
  [Settings Plane](./augmentation/settings-and-credentials.md)
- Source-level deep dives (29, one per package group cluster):
  [topic navigation table](./dsh-harness-index.md#source-level-deep-dives-29--full-coverage)

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — full text in
[LICENSE](./LICENSE) at the repository root.
