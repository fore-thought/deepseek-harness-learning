---
title: "配置面：settings、credentials、workspace"
tags: [dsh, settings, credentials, workspace, config]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{settings,credentials,workspace}/*；docs 同名页；子代理F/E"
updated: 2026-09-04
---

# 配置面：settings、credentials、workspace

## `ctx.settings`：命名空间化、可 watch 的配置树

- 插件注册 `SettingsNamespace`（schemastery schema，含 `role('secret')`
  标注 → UI 表单与脱敏由 schema **驱动生成**，配置界面不是手写的）；
- 解析层叠：defaults → 组合 base → user 分节；`update` 稀疏 patch 只进 user 层、
  `replace` 整体替换（缺席键回继承）；**写串行 + 深冻结快照 +
  `expectedRevision` 拒陈旧写**（CAS 又一处一致姿势）；
- `watch`/`settings/document-updated|updated` 事件推外部编辑
  （settings-file provider 监视 `$DSH_HOME/settings.yaml`）；
- 密钥在设置里**永远只是引用**（见下），`redactSecrets` 给一切外显路径兜底——
  但 union/transform 分支是**源码 TODO 自报的 fail-open 缝**，非绝对兜底（深读口径）。

## `ctx.credentials`：值与引用分离

- `CredentialRef` = branded 的 POSIX env-var 名；配置只存**引用**，
  `resolve(ref)` **每操作一次** → **无重启热轮换**（换 key 不打断在途流，
  LLM 适配器的连接事实同理，见 [LLM 层](../llm-layer/llm-vocabulary.zh.md)）；
- 本地层解析顺序：env > file > project-env > user-env（`credentials-local`）；
- `describe` **永不带值**——无值槽位设计使"读半边"可以跨 Remote 线给设置 UI；
  env 供值的引用标 `writable:false`。
- 权限分级哲学（standards.md 同向）：运行令牌代理可见、支付级仅用户。

## `ctx.workspaceRegistry`：工作区一等实体

- `WorkspaceId`（uuid）≠ path：注册即身份，`realpath` 规范化保证路径唯一，
  **header-cwd 双校验**成员资格；storage-domain 落盘 + 两写标记崩溃恢复；
- "删除注册**不删数据**"；workspace 是 webhook/session/附件的归属容器
  （`WebhookSessionRequest` 必落规范 workspace，见 [后台任务与外部触发](./background-and-triggers.zh.md)）。

## 配置读取优先级（项目组规范的实现印证）

"项目 → 组 → 卡片问用户"在代码里=服务注入 + waterfall 应答：缺配置不是错误路径，
`NO_PROVIDER` 类显式失败把决定权送回人（[人机问答与反馈](./questions-and-answers.zh.md)）。

## 相关

- 配置面深读（层叠解析/写链四细节/信任栈/三段写恢复）：
  [配置面深读](../deep/settings-credentials-workspace.zh.md)

- `$DSH_HOME` 目录布局与 profile：[插件组装与启动](../cordis/plugin-composition.zh.md)
- 密钥文件与 gitignore 纪律：组级 standards.md §2（工作区外只读引用）
