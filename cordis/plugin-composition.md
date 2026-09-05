---
title: "插件组装与启动：loader、patch、profile"
tags: [dsh, cordis, boot, profile]
status: active
license: CC-BY-SA-4.0
evidence: "vendor/{loader,include,hmr}、packages/{boot,bundle}/*；docs/architecture；子代理A"
updated: 2026-09-05
---

# 插件组装与启动：从 YAML 到运行中的产品

> [Cordis 内核](./cordis-kernel.md) 回答"插件是什么"；本页回答"一次 `dsh web` 启动时，
> 那棵插件树是怎么被**配置**出来、又是怎么被**装载**起来的"。

## 配置树：Entry / EntryTree / patch

- `vendor/loader`：配置驱动的装配器。配置是一棵 **EntryTree**，条目
  `EntryOptions { id, name, config, group, disabled, inject, intercept, isolate }`；
  装载 = 逐条目动态 import → `registry.plugin`。
- `vendor/include`：把 YAML/JSON 文件当 EntryTree 载体（`cordis.yml`/`cordis.patch.yml`），
  并实现 **patch 语义**：按 `id` 定位条目、**整 config 替换**（不深合并）、
  `insert` 插新行。表达式语法 `!!js` 在插件上下文插值
  （如 `disabled: "!!js !ctx.webServer"`）；Include 保留嵌套行表达式直到目标行激活。
- **事务化更新**：`Group` 对一批 create 先并行全量执行；失败的精确回滚形态 =
  逆序移除新增行 + **重建全部旧行**；条目自我卸载（自检服务不可用）会把
  `disabled: true` 写回 YAML（真机证实：`!!js` 表达式写回时原样保留）。

## 产品即分层叠加：profile 与 bundle

```text
运行中的 dsh = 空列表
  ⊕ bundle 层（按 profile 声明顺序：dsh-base → dsh-web-app …）
  ⊕ profile 自带 cordis.patch.yml
  ⊕ home 级 patch（$DSH_HOME 用户自己的）
  ⊕ --patch 命令行 overlay
```

- **profile** = `$DSH_HOME/profiles/<name>` 里的具名组装：`package.json` 的
  `dsh.profile { bundles, patchReload }` 声明叠哪些 bundle。随附模板：
  `web / headless / sdk / sdk-minimal / acp`。
- **bundle（组合包）** = Cordis 配置项 + 挂载代码的分发格式。
  `packages/bundle/base`（dsh-base）是四个主 profile 的共享第一层：模型适配器、
  工具、持久化、沙箱与审批、设置、凭据、遥测；web-app/headless/sdk-app/acp-app 各加一层。
  `sdk-minimal` 刻意不叠 base——自带完整显式树（演示"完全替换"的可行性）。
- 同一棵 `applyEntryPatches` 算法三处复用：启动装载、`dsh --dump-config`、
  profile 组合——**配置树可完整外视**是审计与二次定制的基础。

## dsh 启动序（`apps/cli/src/profile-boot.ts`）

1. `loadLayeredEnv`：继承环境 > 调用目录 `.env` > home `.env`（bootstrap 变量黑名单拒载）；
2. 组合 patch 层（`structuredClone` 防 insert 引用别名污染）；
3. `installFailLoud`：未处理拒绝 → 恢复终端 → exit 1（宁崩不静）；
4. `boot()`：new Context → 挂 Loader → prepare 里 provide 启动环境快照与
   命令行三服务（`CmdlineArgs/AppExit/AppReady`，`packages/boot/cmdline`）；
5. `loader.await()` + **启动审计** `assertEntriesActivated`：点名 FAILED 条目与
   PENDING 所缺服务——"配置了但没起来"绝不静默；
6. `patchReload: live` 的 profile（web）挂文件监视：用户 patch 一编辑 →
   条目 config 整替 → Include 重算树 → 相关 fiber epoch 变 → **热重启**该子系统；
   headless/sdk/acp 只在启动时应用一次（一次性生命周期换依赖会自毁）；
7. `appReady.commit()` → web profile 起 HTTP 监听并开浏览器（见 [应用壳](../platform/web-cli-boot.md)）。

## 应用入口纪律

所有受支持的 Node 应用都从 `dsh` CLI + 具名 profile 启动；
`scripts/verify-application-entrypoints.ts` 把每个包 bin/可执行源码归入显式类别，
**拒绝任何绕过 dsh launcher 的 Node 应用路径**。Python SDK 同理：把普通 dsh CLI
打包成平台 wheel，客户端默认起 `dsh --profile sdk`。

## 热重载（HMR）速写

`vendor/hmr`：文件监视 → 借 Node ESM 内部 loadCache/ModuleJob 失效模块 → 重挂该条目。
任何插件的注册都是 effect（[Cordis 内核](./cordis-kernel.md)），所以"卸载→重载" = 逆序回卷 + 重放，
产品状态（会话日志）不受影响。三级决策表与双缓存失效细节见 [Cordis 热重启与热重载内幕](../deep/cordis-hot-reload.md)。
**这是 JS 运行时私有技巧**，Java 对应物（classloader
热替换）代价高得多——移植时的真实架构分叉点。

## 相关

- 内核语义：[Cordis 内核](./cordis-kernel.md)
- 启动后的应用壳与浏览器端 boot：[应用壳](../platform/web-cli-boot.md)
- host/client 进程边界：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
