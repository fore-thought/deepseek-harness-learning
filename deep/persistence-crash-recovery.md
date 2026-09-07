---
title: "Persistence Formats & Crash Recovery"
tags: [dsh, persistence, zstd, crash-recovery, atomic-write, i18n-stub]
status: draft
updated: 2026-09-06
license: CC-BY-SA-4.0
evidence: "DSH 仓库 packages/{session,storage,checkpoint-policy,jsonl} 源码直读 + 真机实测（真实落盘只读解析、截断副本恢复实验）"
---

# Persistence Formats & Crash Recovery

[English](persistence-crash-recovery.md) | [中文](persistence-crash-recovery.zh.md)

> Placeholder: the English translation of this page is pending. The source
> of truth is the [Chinese version](persistence-crash-recovery.zh.md); once translated the
> two versions share identical layout and content (repo conventions).

## Honest boundary

See the 诚实边界 section of the [Chinese source](persistence-crash-recovery.zh.md).
