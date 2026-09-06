---
title: "工作流引擎与 Agent Teams：脚本编排、Ralph 循环与持久协作"
tags: [dsh, workflow, ralph, agent-team, orchestration, worker-thread]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{workflow,experimental}/* 源码直读 + docs/subsystems/{workflow,agent-team}.zh.md"
updated: 2026-09-05
---

# 工作流引擎与 Agent Teams：脚本编排、Ralph 循环与持久协作

> 本篇是[委派与编排](../augmentation/subagent-orchestration.md)概览篇 workflow/Ralph/agent-team
> 三节的深读展开：`packages/workflow/` 4 包全量源码走读 + `packages/experimental/` 8 包承接
> （agent-team 组源码级；其余以官方 README/治理规则承接，深度声明见「诚实边界」）。

## 概览篇口径 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| 每 run 一个 worker + vm | 证实；另有 built/unbuilt 双启动形态与 ready→go 握手门（下详） |
| globals 五钩子 | 证实；另加 `args` 全局；无 fs/network/timers——脚本只协调、agents 干活 |
| fatal 穿透不消融 null | 证实；11 码封闭；fatal 判定用宿主 realm `instanceof`，脚本伪造不了 |
| 取消后有界宽限强判 | 定量：`disposeGraceMs` 默认 5000ms；强判前合成 stranded `agent-end` |
| Ralph = workflow+subagent 组合 | 更强：部署方固定脚本 + fresh-provider 能力闸 + 报告双重解码 |
| 团队协调无新执行原语 | 证实：复用 steer/冷恢复/投影；新增仅 4 枚领域事件 + CAS/DAG 校验 |
| （概览未提） | 并发/总量上限全家、`tool-workflow/*` 持久记录层、realm 物质化纪律 |

## 家族谱系与 seam 契约

三个事实先立住（`packages/workflow/workflow/src/index.ts`，203 行）：

1. **可选能力**：workflow 与 subagent 一样不属于 agent loop，类型与操作记录在 workflow 包而非 core。
2. **单提供方**：每 ctx 只允许一个引擎实现（与 bash 同款），无命名提供方注册表——换引擎靠插件
   配置替换而非并存。Provider = `dsh-workflow-worker-thread`；模型消费方 = `dsh-tool-workflow`。
3. **浏览器安全词汇分裂**：`types.ts`（131 行）只放可进 Client 的数据词汇；持 `Agent` 引用的
   `WorkflowStartRequest`/`WorkflowRun` 单放 `runtime-types.ts`（49 行），Host-only。

契约红线（JSDoc 即规范）：

- `meta`/`args` 是**纯 JSON 数据**——引擎校验 meta 形状、绝不求值脚本文本来获取它们；`parent`
  必填（脚本起的每个孩子都归属它，谱系/深度经 subagent seam 传递）；`subagentProvider` 与
  `maxTotalAgents` 是**引擎级**覆盖，脚本不可观察、不可替换。
- `WorkflowRun.result` **永不 reject**：脚本失败兑现为 `stopReason:'error'`。`WorkflowStopReason`
  封闭三态 completed|cancelled|error；消费方把非 completed 映射成 isError 工具结果，绝不把部分
  输出报成功。`dispose()` 有界：取消→等结果与子代停稳→无论如何 terminate worker。
- 观察事件 `workflow/{start,phase,log,agent-start,agent-end,end}` 6 枚：payload 一律带
  `WorkflowRunInfo`（id+meta 快照）而**非活句柄**——订阅者拿不到 `cancel/dispose`；`workflow/end`
  刻意**省略 result value**（监听者不得持有调用方值的可变别名）；每次 emit 逐监听器隔离：
  抛错只记日志不传播，payload 各自克隆。
- `WorkflowError` 的 11 个 fatal 码 [MEASURED 并集计数]：`SCRIPT_PARSE / META_INVALID /
  INVALID_ARGUMENT / UNSUPPORTED_OPTION / UNSUPPORTED_SCHEMA / AGENT_CAP / ITEM_CAP / AGENT_START /
  AGENT_RESULT / RESULT_UNSERIALIZABLE / CANCELLED`。组合器纪律：fatal 直接重抛杀脚本——拼错的
  选项必须大声报错，绝不能消融成"看起来像孩子失败"的 `null`；逐项 `null` 只留给孩子失败与
  普通阶段错误。

## worker-thread 引擎：每 run 一线程 + 可逃逸 vm

9 文件 ≈1960 行 [MEASURED: host.ts 625 / runtime.ts 488 / index.ts 205 / session.ts 201 /
realm.ts 151 / protocol.ts 101 / types.ts 94 / meta.ts 82 / worker.ts 14]。

- **启动门**：worker 侧 `runWorkerSession`（session.ts:124-201）post `ready` 后等 `go` 才
  `drive()`；若 `Cancel` 先到，它兼作门释放——脚本**一行都不执行**即报 cancelled（挡住启动竞态
  里的同步前缀）。
- **双编译形态**：built 消费者加载 `./worker.cjs`；仓库内（`.ts`）消费者用
  `data:text/javascript` bootstrap 在 worker 内先注册 tsx 的 cjs+esm 变换再 import `worker.ts`。
  两形态都清空 `execArgv`。
- **环境洗刷**（host.ts `workerSpawnEnv`）：worker 只见 win32 的 `TMP/TEMP` 与 unbuilt 形态的
  `TSX_TSCONFIG_PATH`，其余全剥。注释明说为什么**故意不给代理策略**：worker 线程本就不继承宿主
  全局 dispatcher，与其塞一个可能携带 `user:password` 的代理 URL 给执行模型脚本的线程，不如让
  workflow 的请求直连——与 code runtime 同款围堵（`docs/defensive-patterns.md` 要求）。
- **脚本编译**：包装成 `(async () => { … })()`，`lineOffset:-1` 让栈帧回到脚本自身行号。宿主在
  `start()` 里用**同一 wrapper** 预解析一次（换取 seam 的同步 `SCRIPT_PARSE` 承诺，一次冗余解析
  是刻意买价）；以 `export const meta` 开头的 body 得到点名纠错——模型最可能犯的书写错。
- **线程协议**（protocol.ts，101 行）：worker→host 8 种（ready/phase/log/agent-start/agent-end/
  child-start/child-dispose/result），host→worker 7 种（go/cancel/child-started/child-start-error/
  child-settled/child-failed/child-disposed）。tag 字符串枚举 + payload 映射表单一真源，判别联合
  + `assertNever` 收口；所有 payload 按构造即 JSON，structured clone 无损。`child-start` 的
  callId 由 worker 单调分配，一次 start 恰一个回复（started|start-error）。
- **孩子 = subagent seam 的 RPC 镜像**：`agent()` 经 `ChildPort.startAgent` 到宿主
  `subagents.start(provider, {prompt, parent, signal, outputSchema?, agentOptions})`——编排原语
  复用，不另造执行通道。孩子结果用 `snapshotJsonValue` 无损 JSON 快照过界，失败即 `child-failed`
  （基础设施故障，fatal——provider 坏了不得读成"孩子失败"）。
- **取消语义**（host.ts + runtime.ts 合读）：宿主 `cancel()` 三件套 = post Cancel（worker 侧
  hooks 自**下一个任意 hook 边界**起抛 `CANCELLED`，含 `phase/log`——catch 了失败也续不了命）+
  abort 一个全体孩子共享的规范 signal + 武装宽限计时器（unref，不吊进程）。排队等槽的
  `agent()` 直接 reject；acquire 后/过界前的取消竞态在续段重查。start 往返后发现已取消：
  **先 dispose 新孩子再抛**——不给死脚本留活孩子。
- **终态 first-wins**：`terminalClaimed` 在 Result / grace 强判 / worker 死亡三入口间原子认领；
  `workerDeathObserved` 作为**逻辑收报屏障**先关闭消息准入（Node 可能在 `error` 事件后仍投递排队
  消息——迟到的不许造孩子、不许在 `workflow/end` 后旁白）。取消跨线程竞态里脚本恰好完成：宿主把
  非 cancelled 的 result 改判 cancelled（"completed 会撒谎"）。
- **配对账本**：`liveAgents` 保证每个已报 start 恰有一个 end——worker 还能说话时自己报
  （`agent-end` 在取消后**不压制**），线程已死/宽限到点时宿主合成 `outcome:'cancelled'` 且全部
  排在 `workflow/end` 之前。`agentsStarted` 相应分优雅（脚本侧计数）/退化（宿主侧观察）两口径。
- **realm 物质化**（realm.ts，151 行）：离开脚本 realm 的值逐路径走查成宿主纯 JSON——非有限数/
  bigint/函数/symbol/嵌套 undefined/循环/稀疏数组/带非索引属性的数组/异型原型（Date/Map/class）
  一律拒并带路径报错；`__proto__` 键用 `defineProperty` 落成 OWN 属性防原型污染。**信任模型照实
  交代**：vm 不是安全边界，worker 提供宿主循环隔离与强杀能力、不是敌意值围堵；getter 在走查中
  照常执行脚本代码。
- **上限与默认** [MEASURED worker-thread index.ts static Config]：

| 配置 | 默认 | 说明 |
|---|---|---|
| `provider` | `spawn` | 孩子跑的 subagent 提供方 |
| `maxConcurrentAgents` | `0`→`min(16, cores-2)` | 并发 `agent()` 槽，FIFO |
| `maxTotalAgents` | `1000` | 每 run 总孩子数，失速循环兜底 |
| `maxItemsPerCall` | `4096` | 单个 parallel/pipeline 的项数 |
| `syncTimeoutMs` | `5000` | vm 初始同步片超时（worker 内） |
| `disposeGraceMs` | `5000` | 取消到强判+强杀的宽限 |

## 工具面与持久记录：tool-workflow

`tool-workflow/src/index.ts`（334 行）注册模型工具 `workflow`（可改名），参数
`{script, meta, args?}`；**工具的 description 就是模型侧规范**：五个 hook 的准确语义、结构化输出
schema 子集（仅 `type/properties/required/additionalProperties/items/enum/const/oneOf`，无
pattern/format/数值边界）、"misused hooks 必杀脚本"。运行**前台**：本调用直到脚本整体结束才返回。

- 用法策略以系统提示段落随工具自带（`TOOL_WORKFLOW` 槽）：**仅当用户明确要求**才用 workflow；
  一两次委派用普通 subagent。
- **持久记录只归顶层 run**：`exec.parent === undefined` 时向父 Session 写
  `tool-workflow/{run-start,agent-start,agent-end,run-end}` 4 枚 log-only 事件；嵌套工具调用不写。
  首次 append 失败即**禁用本 run 后续记录**——日志保持空或合法前缀，工具结果不受影响（记录是
  投影，不是所有权）。`run-end` 只在结果已取得且 dispose 完全停稳后写。
- **两枚不变式伴生**：`workflow/invariant.ts`（136 行）在事件总线上校验 start/end 配对与 meta
  一致（`internal/dispatch` 预演 + 事件消费提交两段式）；`tool-workflow/invariant.ts`（169 行）
  对每条候选 Session 事件增量折叠校验：成员序号正且唯一、end 必配对、run 结束后不得续写——
  **尾部缺 end 是合法的中断证据，不是损坏**。
- 结果渲染：JSON 值截顶 `maxResultChars = 50_000` [MEASURED]，首行
  `workflow "<name>" completed (N agents).`；presentation 为 args-only 通用卡。聊天侧把 4 事件折成
  一个 `workflow-run` 节点的是 `dsh-client-ui-workflow-run`（Client 组，另篇承接）。

## Ralph：固定脚本 + 严格解码的前台循环

`tool-ralph/src/index.ts`（477 行）**没有新机制**——`RALPH_SCRIPT` 是部署方所有的固定字符串
（`String.raw`），经 `ctx.workflowEngine.start` 起跑，并把 `maxTotalAgents` 压到本轮数上限：

- **模型只供数据**：`{objective, maxRounds?}`；循环体、报告 schema、provider 路由不可被模型改写
  ——引擎级覆盖（`subagentProvider`/`maxTotalAgents`）恰好是脚本观察不到的那两个旋钮。
- **fresh 闸门**（`requireFreshProvider`）：配置的 provider 必须已注册、具备
  `capabilities.outputSchema`、且 `inheritsParentContext === false`，否则加载即拒。这就是概览篇
  "每轮零上下文新子会话"的强制点。
- **每轮提示模板**（脚本内）明写：无父会话/无先前子会话；"你现在就是 Ralph 的一轮，别再调
  ralph"；共享工作区 = 长期记忆；上一份报告只是**有界交接**，须对照工作区核实。
- **报告契约**：`{status, summary, evidence[], nextSteps[], blocker}` 五字段封闭；逐状态规则
  （continue 要有 nextSteps 且 blocker 空；complete 要有 evidence、nextSteps 空、blocker 空；
  blocked 要有具体 blocker）；序列化 ≤ `maxHandoffChars`（默认 16_384 [MEASURED]）。**校验两遍**：
  脚本内 `validateReport` + 宿主侧 `readRunResult/readReport` 按排序键名表**严格重解码**（
  budget-limited 必须恰好烧满轮数；首轮失败必须 `lastReport === null`）——跨 provider 边界一个
  字都不信。
- **四终态**：complete | blocked | budget-limited | round-failed（仅孩子失败回 `null` 时；报告
  畸形属脚本异常 → 整个 run 以 error 收口）。渲染文案刻意是 "Ralph worker **reported**
  completion"——自我报告不是独立认证。
- 其余默认 [MEASURED Config]：provider `spawn`、maxRounds 256、父方面文本截顶 16_384。

## Agent Teams：Lead 日志上的持久协作

`experimental/agent-team` 17 源文件 ≈2500 行 [MEASURED 抽查: index.ts 324 / roster.ts 487 /
mailbox.ts 331 / projection.ts 316 / task-board.ts 297 / types.ts 234 / lifecycle.ts 87 /
activity.ts 87 / journal.ts 73 / task-graph.ts 69]。一句话：**普通 root 会话即隐式 Lead**，
`TeamId` 品牌化自 `SessionId`，无创建事件——持久状态从第一条成员/消息/任务记录开始。

- **四枚领域事件**（types.ts 声明合并进 `SessionEventMap`）：`team/member`、`team/task`、
  `team/message/queued`、`team/message/delivered`，payload 各带 `version: 2`。它们**只存在于
  日志、从不进入会话表面**——协作记录不影响任何成员的模型可见历史；顺序与时间由事件
  `seq/time` 承担，快照不重复存。（注意这个 `version` 是 Team 记录自身的 schema 计数，与会话
  文件格式代次 `session.v2.jsonl.zstd` 是两套编号；代次机制见
  [会话投影与遥测](./session-projection-telemetry.md)。）
- **事务与读侧**：`TeamJournal`（73 行）按 Lead Session 串行化 mutation 尾链，`appendAndFlush`
  先落盘 checkpoint 再唤醒等待者；读侧状态 = `ctx.sessionProjections` 注册的 `agentTeam` 投影
  （projection.ts：zod 严格重放每条 Team 事件，ContentBlock 逐变体精确校验、插件扩展变体保留
  unknown 标签）。这是[会话事件日志](../agent-runtime/session-event-log.md)"投影派生"主线在
  协作域的满血消费者。
- **provisioning 对账**：`spawnTeammate` 先 append+flush 一条 `provisioning` 成员记录，再要
  provider 造孩子；provider 失败也持久化为 `failed`。恢复期把未终结的 provisioning 对照孩子独立
  Session 对账（直接 parent + continuable descriptor 匹配 + 初始用户消息已记录 → `active`，否则
  `failed`）；同进程竞态下 creator 接受终态或报 `TEAM_PROVISIONING_CONFLICT` 并 drain 该 child。
  **名字永久**：`^[a-z0-9]+(-[a-z0-9]+)*$`，连 failed 成员也占名、永不复用；`maxMembers`（默认 8）
  含 failed。成员状态读侧 = 持久 phase（provisioning|active|failed）+ 运行时派生（running|idle|
  inactive），后者绝不回写记录。
- **mailbox 的 exactly-once 是进程内的**：`sendMessage` 先 `queued`+flush 再投递——queued 减
  delivered 即恢复队列；`delivered` 确认**只在目标 Session 持久收下**之后才写（判定折叠
  `user/message` 历史与 pending inbox 的 `agent/inbox/spliced`，session-message.ts）。去重键 =
  target 侧 `TeamMessageSource{kind:'team-message', messageId, senderId}`（并入 `MessageSourceMap`）。
  投递三态复用 subagent 路由：running 在最近 step 边界 steer、idle 起轮、inactive 冷恢复；Lead
  目标直接 `Agent.steer()`，teammate 走 host-only 的 continuation-owner 路径——**sibling 消息绝不
  伪装成 Lead 的公开消息操作**。模型可见格式：`Team message <id> from <name>:` 前缀 + 原样内容块。
- **任务板 = CAS + DAG**：整快照随每条事件持久化，`expectedRevision` 不匹配报
  `TEAM_TASK_STALE_REVISION` 而非覆盖；8 种 action（claim/release/edit/set_dependencies/complete/
  reopen/reassign/delete），`reassign` 仅 Lead。id 为安全整数后缀的 `task-<n>`，耗尽报
  `TEAM_TASK_LIMIT` 绝不复用；`deleted` 留 tombstone（不占 `maxTasks`、不进 list）。
  `assertTaskGraphCandidate` 对**全活跃图**校验：自块/重复边/blocker 缺失或已删 → 拒；DFS 访问
  集环检测 → 拒。`writeScopes` 是规范化 workspace 相对前缀，重叠只出**警告**——提示性，非锁。
- **等待与中断**：`waitForChange` 限 10_000..3_600_000ms（越界抛 `TEAM_INVALID_TIMEOUT`），一次性
  waiter（activity.ts：notify 即释放并摘账），`agent/status` 边也喂通知；dispose 释放全部等待者。
  `interrupt` 仅 Lead、委托 continuable-subagent 路径 `keepInbox`——不删排队邮件、不释放 owner。
  **owner 永不自动释放**（idle/中断/退出都不还）。
- **限额全家** [MEASURED index.ts:46-51]：maxMembers 8 / maxTasks 256 / 每成员 pending 64 /
  单消息 65_536 字节 / disposalTimeoutMs 5_000。dispose = 关准入→等已准入事务→停 roster 内确切
  live 孩子及后代（Lead 的非 Team 孩子不受影响）→失败显式 `AggregateError`。

### 工具层与浏览器面

`tool-agent-team`（411 行）按 **Agent 作用域**注入 9 个工具：`spawn_teammate / send_message /
list_agents / wait_agent / interrupt_agent / team_task_create / team_task_list / team_task_get /
team_task_update`——前五个与 subagent 家族的通用工具**同名不同装**：Team 作用域路由到
`ctx.agentTeams`，`tryMembership` 非抛探针在 `agent/created/disposed` 边上装卸。系统提示
`TEAM_POLICY` 段落逐成员动态注入角色行；fresh/fork provider 默认 `spawn`/`fork`。`wait_agent`
有模型专属捷径：同一同步跨度内读 roster，无活跃 peer 立即回 `noProgress{no-active-peer}` 并指导
先 `send_message` 唤醒——不白等。

`TeamService` 继承 `TypertRemoteService`，`@Remote` 三方法（view/createTask/updateTask）供浏览器
读权威状态：Team 业务拒绝走**传输成功内的 `ok:false`**（`team-task-conflict` 与其他
`team-rejected` 区分），transport 失败仍留在外层 `RemoteResult`。`client-ui-agent-team`（98 行
README）只用这条 remote、不扩展稳定 Proxy、不存 Team 状态，teammate 导航复用 addressed-subagent
路径。

## experimental 治理口径与余下地图

`packages/experimental/AGENTS.md`（9 行 [MEASURED]）四条治理：①只有**整个公开契约**都实验/内部
才入组（发布包里的实验选项留在主人家）；②统一 `dsh-experimental-*` npm 前缀 + `private: true` +
无 `publishConfig`，由 workspace 约束门禁执法、发布家族排除本目录；③发布包**不得**依赖本组
（测试走 devDependencies、示例显式加载可以）；④**实验身份不放宽任何工程约束**——安全/文档/
生命周期/测试/不变式/快照照旧。两个 profile 包是"运行时空模块 + `cordis.patch.yml`"的 bundle
patch 层（package.json `dsh.bundle.patch` 声明）——显式源码 checkout 启用的标准形态，呼应
[插件组装](../cordis/plugin-composition.md) 的 profile 叠加。

余下 4 包按官方 README 承接职责地图（内部机制各值得一篇独立深读，见诚实边界）：

| 包 | 一句话机制 |
|---|---|
| `code-runtime-python` | code seam 的 CPython 子进程后端；fd-3 JSON-lines，入站帧按敌意重建 |
| `inspector` | 跨 realm CDP hub；worker 独占 CDP 状态，活对象传输前投影成 snapshot |
| `webworker-runtime` | 整棵插件树跑进 dedicated Web Worker：VFS 镜像 + CJS 加载器 + HTTP 隧道 |
| `webworker-packer` | 镜像供给侧：roster→发布视图→可达性 sweep；不编译源码 |

- `code-runtime-python`（138 行 README + src/protocol.ts）：每 `run()` 全新 CPython 3.10+ 子进程，
  `language:'python'、isolation:'process'` 注册进 `ctx.codeRuntime`；stdout/stderr 留给程序、协议
  走 fd 3；隔离 = 仅含 TMPDIR 的环境 + `RLIMIT_CPU/AS` + 墙钟 + SIGTERM→宽限→SIGKILL 进程组拆卸，
  默认预算 cpu 60s / 墙钟 600000ms / 地址空间 512MB（Darwin 不生效）/ 日志 64KB / 值 32KB /
  宽限 3000ms [MEASURED README 声明值]。信任等级与模型 bash 持平——**非安全边界**。seam 家族见
  [PTC 与 code-runtime](./ptc-code-runtime.md)。
- `inspector`：Host 插件起 worker 并连专用 MessagePort；Client 侧读注入的
  `globalThis.__DSH_INSPECTOR__` bootstrap 直连 worker 的鉴权 WebSocket；DevTools 每条连接在
  worker 里独占一个 `node:inspector.Session`，Host 暂停时 Console/Sources/断点仍可用。
- `webworker-runtime`：一条 tsdown 管线出装配库/worker 束/页面半三产物；VFS 镜像边下载边解压挂载，
  overlay 只能替换 `home/` 与 `workspace/`；`module-proxies.ts` 是唯一平台叉口（`node:*` 走
  VFS/隧道/浏览器原语，做不到的结构化 stub，native 包替换后端）；`node:child_process` 是**实现**
  不是 stub——spawn = 本束自己的另一个 worker，首帧告知"你是 shell 进程"；无编译器，未降低的
  镜像挂载即拒。
- `webworker-packer`：三层标准栈（合成 profile 的 roster → 每包 npm 发布视图 → 从真实加载器解析
  出发的可达性 sweep，pack 期把可达模块降低到包装契约）；CLI `dsh-pack-vfs-image`；预览调试看到
  的就是 served 交付的字节。

## 诚实边界

- 未跑真机 workflow/team：本会话所在组合不挂 team profile、启动一次真 run 需持久存储+显式启用层；
  机制叙述 = 源码 + 官方 type-equiv 双口径，无行为实验定量。
- 未读全：`roster.ts` 120 行之后、`mailbox` 的 `tryDispatch` 本体、`task-board.ts` 命令映射细节、
  `projection.ts` 60 行之后——关键契约已用 README/类型表/事件声明互证；`inspector` 与
  `webworker-runtime` 的内部（Node 兼容层规模远超一篇余量）、`client/ui-workflow-run` 折叠细节
  未展开，已在覆盖施工表分别挂后续批次。
- `RALPH_SCRIPT`、`POLICY`、工具 description 等固定字符串按当前 checkout 引用即快照；上游改词须勘。
- 概览篇"Ralph 是策略不是模式"的判断在源码层完全成立，本篇升级为其实现证据：固定脚本 + 两个
  引擎级旋钮 + 能力闸。

## 相关

- 概览承接：[委派与编排](../augmentation/subagent-orchestration.md)（本篇深读其 workflow/Ralph/team 三节）
- 孩子从哪来：[会话事件日志](../agent-runtime/session-event-log.md)；持久底座：
  [会话持久化与存储](../platform/session-persistence.md)
- 代次与投影机制：[会话投影与遥测](./session-projection-telemetry.md)（同阶段 R1 篇）
- 工具执行与结构化输出子集：[工具注册表与执行流水线](../agent-runtime/tools-pipeline.md)
- code seam 家族：[PTC 与 code-runtime](./ptc-code-runtime.md)；profile/patch 形态：
  [插件组装与启动](../cordis/plugin-composition.md)
