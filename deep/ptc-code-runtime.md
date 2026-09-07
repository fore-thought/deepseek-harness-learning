---
title: "PTC & code-runtime Internals"
tags: [dsh, ptc, run-code, code-runtime, worker-thread, i18n-stub]
status: draft
updated: 2026-09-06
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/core/tools/src/{ptc.ts,index.ts}、packages/agent-tool-presentation/src/index.ts、packages/code-runtime/{code-runtime,code-runtime-worker-thread}/src/*、packages/core/agent-loop/src/tool-calls.ts 源码直读 + 本机 Node v24 对已构建产物真机实测（精选 16 项）"
---

# PTC & code-runtime Internals

[English](ptc-code-runtime.md) | [中文](ptc-code-runtime.zh.md)

> Placeholder: the English translation of this page is pending. The source
> of truth is the [Chinese version](ptc-code-runtime.zh.md); once translated the
> two versions share identical layout and content (repo conventions).

## Honest boundary

See the 诚实边界 section of the [Chinese source](ptc-code-runtime.zh.md).
