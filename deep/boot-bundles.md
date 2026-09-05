---
title: "Profile 组装与启动：六个 bundle 的构成与取舍"
tags: [dsh, boot, bundle, profile, apps]
status: active
license: CC-BY-SA-4.0
evidence: "packages/boot/{app-boot,cmdline}/src 直读 + packages/bundle/* patch 与 src + apps/{cli,web} + .dsh-build 与 $DSH_HOME/profiles 真机观察 + docs/subsystems 相关页"
updated: 2026-09-05
---

# Profile 组装与启动：六个 bundle 的构成与取舍

> 边界先说破：Cordis 装载机制（Loader/Include/patch 算法/HMR 模块失效）归
> [插件组装与启动](../cordis/plugin-composition.md) 与 [热重启内幕](./cordis-hot-reload.md)，
> 浏览器端第二次 boot 归 [应用壳](../platform/web-cli-boot.md)。本篇回答的是**装载之下**
> 的问题：六个 bundle 各装什么、为什么这么拆；launcher 如何持有并移交命令行；
> profile 目录的真实形态；以及 apps/{cli,web} 两个入口包的纪律。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| 启动序第一步是 `loadLayeredEnv` | `runProfile` 实测第一步是 `installProxyFromEnvironment`（在任何插件挂载、任何请求发出之前，且从启动快照而非 `process.env` 解析——Node 的 fetch 不自理代理，`.env` 层声明的代理只能这样生效）；环境加载发生在 `bin.ts` 调 `runProfile` 之前 |
| 空根 config 只是"Loader 需要真实 include 根" | 更深一层：**每次启动都重写** `cordis.yml` 为 `[]`——Loader 的树回写（插件自卸载会 persist 当前树）会把组合后的行烘焙进根文件，下一次启动每个 bundle 的 insert 就会翻倍 |
| web-app 补丁注释称 `DSH_WEB_URL` 由 `apps/cli/src/web.ts` 注入 | 该文件在本 checkout **不存在**（apps/cli/src 下无此文件，grep 0 命中）；实况是 web-app 自己在 `webRuntime` 服务的 `resolve()` 里发布 `DSH_WEB_URL`。注释滞后于实现 |
| "一个 CLI，四个孔" | launcher 是三模式分发（profile / plugin / dump-config），profile 模板五个（web/headless/acp/sdk/sdk-minimal），`dsh web` 是硬编码别名；"孔"指应用面而非代码分支 |
| patch 三处复用同一算法 | 成立，且 dump 在其上多一层**出处标注**：对每个前缀快照各跑一次 `applyEntryPatches`、按位置 diff 出"哪层改过哪行"，输出仍是可加载 YAML |

## 清单协议：一个 dsh 节，两种角色

`package.json` 的 `dsh` 节同时承载两个互不排斥的角色（`profile.ts` 的
`DshManifestSection`）：

- **bundle 侧**：`dsh.bundle.patch` 声明本包导出的 patch 层路径（各 bundle 都是
  `./cordis.patch.yml`）。缺这个声明的包，`dsh plugin` 视其为普通依赖不入层。
- **profile 侧**：`dsh.profile { bundles, patchReload }` 声明按序叠哪些 bundle、
  用户 patch 是否活重载。层栈固定五段（application order）：
  **bundle 层（按 bundles 序）→ profile 自身 `cordis.patch.yml` → home 级
  `$DSH_HOME/cordis.patch.yml` → `--patch` overlay → 遥测开关 patch**。
  机器本地偏好（home 级）排在单 profile 层**之后**——优先级更高。
- **整 config 替换纪律**：patch 命中一行时替换其整个 `config` 而非深合并，因此
  base 的头注规定"值随模式而变的行**不进 base**，归各模式 bundle 完整重述"——
  任何一行最多"一层 bundle + 用户层"两份定义。
- **双锚解析**：bundle 名先从 dsh 安装闭包解析，再从 profile 目录的 `node_modules`；
  打包可执行文件没有符号链接，改走 ESM 代理虚拟包（`healProfilesModuleFallback`，
  跨进程 writer lock 防半程代理）。

## 六包差异矩阵 [MEASURED]

数字为当前 checkout 的 `- id:` 行数 / package.json 依赖数 / src 文件数：

| bundle | 定位 | patch 行 | 叠加 | 依赖 | 归属纪律 |
|---|---|---|---|---|---|
| base | 四个主 profile 的共享核心 | 85 | 空根上第一 insert | 84 | 无 stdout 主张 |
| web-app | 浏览器界面层 | 88 | base 之上 | 73 | stdout 打印 URL 行；浏览器插件名册在此 |
| headless | 一次性任务 | 5 | base 之上 | 6 | stdout=最终答案，stderr=推理流 |
| acp-app | ACP stdio 自动化 | 4 | base 之上 | 3 | **stdout 归 ACP 协议** |
| sdk-app | SDK JSON-RPC stdio | 4 | base 之上 | 4 | **stdout 归 JSON-RPC** |
| sdk-minimal | 独立极简 SDK 树 | 33 | **不叠 base**，整树自带 | 29 | stdout 归 JSON-RPC；sandbox-policy 默认 `danger-full-access`（base 是 workspace-write） |

sdk-minimal 是"完全替换"可行性的自证：33 行就是整棵树，无 timer、无 hmr、
无 settings 文件层，API key 走裸 `DEEPSEEK_API_KEY` 环境变量。

## base 层的取舍解剖

base 85 行里藏着整个产品的默认立场，挑关键的说破：

- **平台门互斥对**：`bash-sandbox`/`tool-bash` 在 win32 禁用、`pwsh-sandbox`/`tool-pwsh`
  在非 win32 禁用——四行成对，`!!js process.platform` 表达式在装载期求值。
- **权限三件套默认**：`sandbox-policy` mode = `DSH_PERMISSION_MODE ?? 'workspace-write'`、
  `approval` policy = danger-full-access 时 `never` 否则 `ask`、`permission` 预设表
  三档（read-only/workspace-write/danger-full-access 各配沙箱档+审批档）。
  新会话"workspace-write + ask"的产品默认就是这三行写在一起的产物。
- **遥测默认反馈门**：mode = `DSH_TELEMETRY_MODE ?? 'FEEDBACK_ONLY'`（用户 /feedback
  时才上传）；`DSH_TELEMETRY_DISABLED` **任何非空值**（含 `'0'`/`'false'`）都关闭——
  隐私开关宁误关不误开；且关闭走 launcher 打 patch 禁用整行（config 禁不了行），
  组合里没有遥测行就不生成 patch。导出排水上限：exporter 1s + 后端 3s 兜底。
- **spawn/fork 子代理不对称**（与本库前作相互印证的实况）：`tool-subagent` 行默认
  `backgroundMode: continuable`，而 `tool-subagent-fork` 行**刻意不设 model 选择**——
  行注释直言"让 provider/model 与父一致，继承的历史才能续用 KV Cache"，并引
  `.agents/notes/implemented/architecture/2026-08-10-fork-children-stay-one-shot.md`。
  本篇作者（一个 fork 子代理）实测印证：`subagent_fork` 工具签名无 model 入参。
- **装载与人格留白**：`system-prompt` persona 默认空串（人格是**部署选择**，各模式
  补丁重述同一句 "{{model}}/{{cwd}}" 模板）；`agent-loop` 的 `agents: []`——base 不
  建会话，Web 按 client 请求建、headless 由 runner 建；`hmr` 行 `disabled: true`
  （模块重载按 profile opt-in，config 观察走 launcher 的 watch-only 回退）。
- **code-runtime 不在 base**：PTC 执行器由 web-app 与 headless 各插一行
  worker-thread 后端——"PTC 是核心执行能力，不是 Web 组件"（headless 行注释原话）。

## apps/cli：唯一 bin 的三模式与启动细节

`bin.ts`（50 行）只做三件事：读版本、`parseDshArgs`、按 mode 动态 import
profile-boot / plugin / dump-config，穷尽到 `satisfies never`。真正的设计在
`args.ts` 与 `profile-boot.ts`：

- **launcher flag 先行契约**：启动器只解析 `--profile/--patch/--dump-*/plugin`，
  第一个不认识的 token 起全是应用内参——`dsh --profile web -h` 打印的是 **web 应用**
  的 help。配套纪律：`--patch` 是重复单值收集器、**永不 variadic**（variadic 会吞掉
  内参）。决策依据见 `.agents/notes/implemented/architecture/2026-08-06-app-owned-command-line.zh.md`。
- **遥测开关在组合期落地**（`resolveTelemetryPatch`），排在全部 overlay 之上——
  用户层永远盖不掉"显式退出"。
- **live 重载的组合不变式**：`composeLive` 每次重读两份用户 patch 文件（防两个 watcher
  互缝对方旧副本），bundle 层垫底、overlay 置顶（用户编辑永远挤不掉机器组合层），
  且**每代 structuredClone**——include 把 insert 行**按引用**推进活树、后续 id 补丁
  原地改这些对象；复用一份 parse 结果会把用户覆盖烘焙进 bundle 的内存行，删掉覆盖
  就再也回不了默认值。这是概览篇"整行替换"语义的资源别名侧写。
- **信号分工**：SIGTERM=监督者的常规停止→exit 0；SIGINT=用户中断→exit 130；
  两者都先 dispose 整树 + 撤代理，宽限 `PROCESS_SHUTDOWN_TIMEOUT_MS = 5000` 后强退；
  watch 设置期的异常若发生在"树已按请求退出"路径上则吞掉（`suppressShutdownError`
  以 fiber 状态与 loader 在场判活）。
- `appReady` 只在"未 abort + fiber ACTIVE + loader 在场"三条件齐时 commit——
  一次性应用（headless）与 stdio 应用（`exitOnStdinEnd`）都靠它对齐生死。
- **`dsh plugin` 是对账器不是安装器**：转发 pnpm 后按**安装实况**（而非依赖 diff）
  把声明了 `dsh.bundle` 的包排进 `bundles` 层栈——所以 `update` 能让"新版本才加上
  bundle 声明"的包入层。
- **dump 是 boot-free 的**：不跑应用的 flag provider，故拒绝带内参（打印的树与
  boot 实况可能不同就是误导）；`--dump-default-config` 连用户层都不 parse——它是
  给"坏掉的 cordis.patch.yml"留的恢复诊断。
- 安装清单还挂着一个 `dsh.configTrees`：把 `packages/preset/agent-presets/presets`
  以 scanRoster 方式挂进配置树——出厂 preset 是**装配事实**，住在 CLI 清单里。

## cmdline：三个启动器事实的交接

`provideCmdline` 在任何树条目挂载前发布 `cmdlineArgs`（冻结快照）、`appExit`、
`appReady`。应用侧两个入口：

- `parseCmdline(ctx, program)`：commander 语法错误/help/version 是**进程级终局**
  （写文本→`appExit`），action 必须"先发布服务、后 error"；flag 值经普通服务
  （如 `webStartup`）流入后续行的 `!!js` config 表达式——**没有任何行享有
  launcher 级命令行地位**，一个 provider 让全部行受益。
- `exitOnStdinEnd`：stdio 应用的生命线；EOF 到达也要等 `appReady` 才 exit(0)
  （help 路径永不 commit ready，因此"看帮助"不会把 transport 带起来）；
  `readableEnded` 早到情形用微任务补投。

## 模式装配细节

- **headless**（5 行）：startup provider 解析 `[task...]`（多词以空格拼接）发布
  `headlessStartup`，runner 行 `task: !!js ctx.headlessStartup.task` 消费；驱动
  一个全新持久化会话到静默，推理流走 stderr、最终 assistant 文本走 stdout、
  `appExit` 收尸。不挂 Host/HTTP/浏览器——同一个 launcher，最小树。
- **acp**（4 行）：零 flag；`session-title-llm` 禁用（stdout 独占给协议帧）；
  acp 行显式钉死 `provider/model` 默认。
- **sdk**（4 行）：同 acp 的 stdout 独占纪律；`maxTokensAsSuccess` 默认 true 可由
  环境变量翻转；开发形态另带 `sdk-source.cordis.patch.yml`（干净 checkout 无构建期
  Typert 贡献模块，禁 `typert-loader`；SDK 不消费 Typert 网关所以无损）。
- **web**（88 行，三段式）：①surface 专属值（persona 重述、sqlite 全文检索
  `path: ':memory:' + openAt: never`——首搜才 import node:sqlite，保 Node 22 启动安静；
  `DSH_TOOLS_MODE` 临时进程级 PTC 开关，注明"web UI 逐会话接管后移除"）；
  ②宿主/传输（webserver 默认 127.0.0.1:3080 + gzip；`--host 0.0.0.0` 直接判错——
  "会把远程代码执行暴露到网络"；`webStartup`(调用期值) 与 `webRuntime`(绑定后值)
  两段服务把行分在两把门后）；③`dsh.client` 浏览器名册（node 半边扫树合成
  `window.__DSH_BOOT__`，见 [应用壳](../platform/web-cli-boot.md)）。
- **preset 平面的迁移纪律**（web 层最有信息量的一段）：约 24 个面向模型的 tool 行
  在 web 层被 **disable 而非删除**——"某天有人重排组合，缺席的行会悄悄复活"；
  而注册表类服务（jobs、skill、goal、token-meter、subagent）留宿主面，判据注释写死：
  **realm 之外有兄弟行会 READ 它，就归两面都看得见的宿主**（entry-local realm 对
  圈外的解析不可见，曾经让 `run_in_background` 答"不可用"）。完整推理归
  `.agents/notes/implemented/architecture/2026-08-10-host-plane-ownership-after-presets.md`。

## apps/web 与前端构建链

`apps/web`（包名 `dsh-web-frontend`）是 **vite 构建壳**：`src/main.ts` 实测 5 行
——找 `#root`、`new AppWebEntry(el).run()`（shell 库归 `packages/client/web`，
见 [host/client 边界](../platform/host-client-boundary.md)）；publish 只发 `dist`，映射表
拒绝 source map 与 preview 产物。脚本面四条：`build/dev/watch`（watch 带
`--no-emptyOutDir`）与 `build:preview`（先 tsdown 两个 webworker 实验包，
再 `dsh-pack-vfs-image` 打 VFS 镜像 tar.gz）。前端 dist 的**位置解析是 bundle 的
装配事实、永不是用户配置**——web-runtime 插件自己从包根算路径再挂
frontend-static 回退席位。构建出处落在 `.dsh-build/client-build-environment.json`
（本机构建实测 [MEASURED]：formatVersion 1、commit hash+dirty 标记、version 串、
artifacts fileCount 222 + 整树 sha256）。

## 真机观察：profile 目录首用即模板

`dsh --profile web` 首跑会 `initProfile` 出 `$DSH_HOME/profiles/web/`：
`package.json`（`dsh.profile.bundles = [dsh-base, dsh-web-app]`、
`patchReload: "live"`、空 dependencies——本机现状实测）+ `cordis.patch.yml`
（带用法注释的 `[]`）+ Loader 需要的空根 `cordis.yml`。五个模板里只有 web 是
live（长驻 GUI 值得热重载），其余 startup；**自定义 profile 缺省保留历史 live**
（`DEFAULT_PROFILE_PATCH_RELOAD`）。

## 相关

- 装载机制与 patch 语义：[插件组装与启动](../cordis/plugin-composition.md)
- 重载的模块级内幕：[Cordis 热重启与热重载](./cordis-hot-reload.md)
- 浏览器侧 boot 全链：[应用壳](../platform/web-cli-boot.md)
- 双 face 传输与信任栅栏：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
- 一次性执行的 worker 后端：[PTC 与 code-runtime](./ptc-code-runtime.md)

## 诚实边界

- `boot()`/Loader 内部挂载次序只与 plugin-composition 交叉引用，未重走读；
  `healProfilesModuleFallback` 的 ESM 代理包细节以 app-boot README 声明为准，未逐行。
- `apps/cli/composition.md` 为脚本生成的全景图（`gen-doc-graphs`），本篇的行数
  以 `- id:` 行 grep 口径统计，与生成图未逐行对账。
- "`web.ts` 注释滞后"的结论以本 checkout 为准（grep 0 命中）；上游可能已修。
- 六包行数/依赖数是快照口径 [MEASURED]，上游迭代后以 `dsh --dump-default-config <profile>`
  现打现看。
- preset 平面的 `dsh-agent-presets` 出厂名册（system 只读根 + includeShippedRoot）
  只读到 web 层注释与 configTrees 挂载，未逐 preset 展开——归 N2 提示词组装篇口径。
