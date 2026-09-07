# DeepSeek Harness — Source Architecture Handbook

English | [中文](README.zh.md)

A handbook that explains the architecture of **DeepSeek Harness** (`dsh`, an
open-source LLM agent harness) strictly from its source code — written so a
reader with no programming background can follow it, and ending with a mental
map complete enough to design a harness of your own.

- Independent third-party work; not affiliated with the official project.
- Based on a 2026-09 source checkout; every claim carries source-repo-relative
  evidence paths and can be re-verified.
- Language pairing: every page has an English file (`foo.md`, canonical) and a
  Chinese file (`foo.zh.md`) with identical layout and content; the switch line
  sits directly under the title. Chinese-only pages (v2 translation pending)
  carry no switch until their English counterpart lands.

## Status

| Line | Content | Where |
|---|---|---|
| `v1.0.0` (tag, frozen) | complete Chinese edition: 22 overview + 29 source-level
  deep dives, covering all 51 package groups | repository Tags page; errata via `v1.x` |
| `main` (v2 line) | bilingual handbook rewrite under the style charter:
  diagram-first prose, background pages, zero-assumption wording | current view |

## Structure

| Area | English | 中文 |
|---|---|---|
| Panorama & navigation | `overview.md` | [overview.zh.md](./overview.zh.md) |
| Framework base (plugin kernel) | `cordis/` | 同名 `.zh.md` |
| Agent backbone (loop, tools, prompts) | `agent-runtime/` | 同上 |
| Model layer (adapters, context) | `llm-layer/` | 同上 |
| Platform (gateway, persistence, client) | `platform/` | 同上 |
| Execution world (fs, sandbox, shell) | `execution/` | 同上 |
| Augmentation (subagents, skills, goals) | `augmentation/` | 同上 |
| Source-level deep dives (29) | `deep/` | 同上 |

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — full text in
[LICENSE](./LICENSE) at the repository root.
