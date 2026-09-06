---
title: "配置面深读：层叠解析、热轮换、凭据账本与工作区登记"
tags: [dsh, settings, credentials, authorization, workspace, storage-domain]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{settings/settings,settings/settings-file,credentials/credentials,credentials/credentials-local,credentials/authorization,workspace/workspace,storage/storage-domain} src 直读 + docs/subsystems/{settings,credentials,workspace}.zh.md + 消费方 grep 实测"
updated: 2026-09-05
---

# 配置面深读：层叠解析、热轮换、凭据账本与工作区登记

> 本篇是[配置面概览](../augmentation/settings-and-credentials.md)的源码级展开，回答四个问题：
> 配置树如何逐层解析、拒陈旧写、保注释地落盘；机密如何"住在配置之外"——引用、热轮换、
> 分层信任与脱敏；以及"配置面谁也供不出的凭据"从哪来（authorization）。三者脚下的
> 领域 KV（storage-domain）与工作区登记簿一并展开。存储后端本体（json/sqlite）的
> 读写与崩溃恢复已由[持久化深读](./persistence-crash-recovery.md)承接，此处不重复。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| "解析层叠 defaults → base → user" | 证实：`mergeLayers` 只对 plain object 递归合并，其余值（含数组）整体替换；合并结果交给 schema 调用完成运行时准入 |
| "写串行 + `expectedRevision` 拒陈旧写（CAS）" | 精化：CAS 检查发生在**队列槽位**而非调用时刻；revision 是对**原始 user 分节**的单调计数，外部编辑经 publish 同样 bump；`expectedRevision` 是可选参数 |
| "`redactSecrets` 给一切外显路径兜底" | **有声明的缝**：脱敏器只走 object/dict/array 三种容器，藏在 union/transform 分支里的机密原样放行，源码以 `TODO(settings-wire-redaction)` 自报 fail-open |
| "本地层解析顺序 env > file > project-env > user-env" | 证实（`credentials-local` 模块头注释画出该栈）；且只有顶层 process env 不可写，两级 `.env` 在托管仓写入后**失去效力但不报错** |
| "workspace 两写标记崩溃恢复" | 修正为**三段写**：create 走"标记→记录→顺序"三阶、各有回滚；启动恢复只有一种动作——把被标记的删除做完，从不猜测反向恢复 |
| "storage-domain 全库仅 3 域" | 证实 [MEASURED]：`defineDomain` 全库恰 3 处——workspace、message-feedback、session-projection-cache |
| "`applies` 标注生效时机" | 精化：`applies` 是 **UI 提示而非机制**——`restart` 的 owner 从不 watch，值在构造期一次读定（`settings.zh.md` 原文，与实现一致） |

## settings seam：注册、层叠与三条写路径

Service Definition 抽象类 `SettingsProvider`（packages/settings/settings/src/index.ts，
893 行 [MEASURED]）把接缝切成两半：**提供方只实现 `load`/`persist` 两个 IO**，注册、
解析、校验、变更检测、提交事件全在基类——配置文档的语义归接缝，存储只是载体。

- **namespace 双重把关**：运行时 `/^[a-z][a-z0-9-]*$/` + 品牌 `SettingsNamespace`；
  另有一套编译期模板类型（`SettingsNamespaceInput` 的逐字符递归）把大写字母、连字符
  开头这类拼写错误拦在类型层——配置键名获得与普通 API 字符串不同的字面量纪律。
- **注册 = fiber effect**：`ctx.effect` 登记，宿主 fiber 卸载即移除 namespace 与全部
  观察者。注册时刻若存储里已有分节且解析失败，**拒绝注册本身**——这是 schema 能审判
  它的最早时刻；注册之后同样的失败改为"保留 last good + warn"（见 publish）。
- **解析**：`schema(mergeLayers(base, userSection))` 之后才跑 `options.validate`——
  跨字段约束钩子看到的就是 owner 将来拿到的完整解析值。llm-pi-ai 用它把"自己服务不了
  的提供方 profile"拒在写入处，而非存下后让整条 namespace 逐路由失效。解析值 `deepFreeze`。
- **写队列的四个细节**：①入队前 `cloneJsonShaped` 一次遍历完成"快照+JSON 形状校验"——
  Date/Map/BigInt/环以 `$` 起根路径拒绝，对象里的 `undefined` 键跳过（稀疏 patch 语义）、
  数组里的 `undefined` 拒绝；②每命名空间一条串行链，前驱失败**不毒化**队列
  （`previous.catch(() => undefined)`）；③出队时重查 stopped 与 registration 存活性；
  ④合并基线取**槽位时刻**的 section——"绝不由调用者见过的状态推导"。
- **persist 成功但 registration 已被换掉**：文档缓存仍更新（"写已进存储，缓存必须
  说实话"），只是不 commit 不通知——半路换主的中途写不得冒充新 owner 的提交。
- **三条写路径**：`update`（稀疏 merge 进 user 层）、`replace`（缺席键回继承，
  `replace({})` 即整区重置）、`mutate`（路径寻址的 set/unset 序列，空路径指分节根）。
  `mutate` 的存在理由与脱敏咬合：持**脱敏视图**的调用方做整体 replace 会静默删光它
  从未见过的 secret 字段，路径写让它"只点名自己改的字段，不重述整个分节"。
- **`installSection`（消费方通用姿势）**：组合 entry 作 base 注册；provider 在场时
  `setSource(scope.get)`，provider 离场回退 entry——回退前先判 `isUnloading(owner)`：
  自己下线无需回退工作，只有"peer 消失"才做。FiberState 是 const enum，无运行时对象，
  这里以数值镜像（4/5）比对，注释自注与 CLI boot 的镜像同源。
- **两个提交事件**：`settings/updated`（解析值变了，deep-equal 门控）与
  `settings/document-updated`（**原始分节**变了——同一解析值下"字段从继承变为
  恰好等于它的覆盖"也要通知配置面，否则另一个标签页不知道手里的 revision 已过期）。
  fan-out 是手写的逐 listener 派发：普通 emit 停在第一个抛错的监听器、饿死后半程，
  这里反着来——非 INVARIANT 失败逐个 containment+log，INVARIANT 全员跑完后重抛；
  且重抛只能来自同步监听器，所以接缝注释明言"不变量检查不得写成 async 函数"。
  包内伴生插件（invariant.ts）对该事件立四条断言：只为已注册 namespace 发、只在
  解析值真变时发、携带的必须是权威解析值、等值复用即违规。

## secret 脱敏与 YAML 诊断纪律

`redactSecrets(schema, value)` 沿 schema 形状走对象/dict/数组三种容器，把
`role('secret')` 字段从值里**删除**、在旁边车记 `{path, set}` 槽位——配置的
三层（value/base/user）同走；对象属性上的 secret 槽位**未设值也枚举**（表单要渲染
这个空位），dict/array 里的只在值存在处枚举。输入永不原地改动。已知缝照抄源码：
union/intersection/transform 分支里可达的机密原样放行且无记录，`TODO` 标注
"应 fail-closed"，属上游自己声明的未闭环。

- 与凭据分工：机密的本分是**根本不住设置文档**——配置只存 `CredentialRef`；
  本 build 内 `role('secret')` 的真实字段仅 1 处 [MEASURED]（web-search-deepseek 的
  `apiKey`）——secret 槽机制是兜底而非主路。
- 线侧强制：api/settings-controller 的两处 `describe` 全部传
  `{ redactSecrets: true }` [MEASURED]；`SettingsNamespaceView` 的注释把规矩写成类型：
  跨 Remote 的字段是 `JsonValue` 而非 `unknown`——无约束数据不上线。
- **YAML 错误不回显原文**：解析失败只报 `code + 行列`、永不使用
  `error.message`——解析器消息会引用出错源行，而在这两类文档里那行就是机密本身。
  settings-file 与 credentials-local 共享同一纪律，注释同文互证。

## settings-file：注释、竞态窗口与 last-good

默认文档 `<harness home>/settings.yaml`（显式 `path` 优先；扩展名限
yaml/yml/json，格式由扩展名派生）。

- **启动语义**：init 先 `load()+publish()` 一次；文档存在而不可解析 = **启动失败**
  ——"一份存在但不能信任的文档"绝不当作"没有配置"。watcher（chokidar）在 super.init
  之后才挂，`awaitWriteFinish.stabilityThreshold = debounceMs`（默认 100ms [MEASURED]）。
- **两个竞态补丁**：①初始读与 watcher 就绪之间写入的变更永远不会触发事件——
  `ready` 时补一次 reconcile 关闭窗口；②自写抑制靠 **text 缓存**：内容等于缓存的
  watcher 事件是 no-op，自家写盘因此不惊叫。
- **写路径**：所有命名空间的写与 reload 串在**同一条** operation 链（一份文档托底
  全部 namespace，异链会让 render 基于陈旧文本、兄弟分节从盘上蒸发）→
  `withFileLock` → `reconcileFromDisk`（先折进未观测的盘上状态再渲染）→
  YAML 走 `patchNode` 叶级 diff：仅变化的叶子 `setIn`、仅消失的键 `deleteIn`，
  未动节点连同**被改分节内部兄弟的注释**都按原样存活；JSON 格式则整键替换。
  落盘经 `writeFileAtomic`（0600/目录 0700）。
- **两种失败姿态共用一个 `reconcileFromDisk`**：reload 侧 warn 并保留 last good
  （热更新永不掀翻活进程）；write 侧 fail loud（不能理解文档就拒绝覆盖用户的
  手工编辑）。INVARIANT 失败从两侧都穿透到队列错误面——毒提交要可见，但不杀掉
  重载循环。

## credentials seam：两个键空间与四层信任栈

一条接缝回答两个问题，两把键语法**刻意不相交**：

- `CredentialRef`：POSIX 环境变量名（`/^[A-Za-z_][A-Za-z0-9_]*$/`），答
  "这个 env 名背后是什么"——可分层；
- `CredentialKey`：`<scope>/<id>`（两段各小写 kebab），答"这个插件为某 id
  存了什么"——不可分层（授权凭据没有 env 可读），**记录存在本身就是全部事实**。
  `/` 使两语法天然不相交，两个键空间永不串台。scope 用**属主插件名**而非业务域名：
  记录载荷按属主格式书写，两插件同名会互读对方载荷；卸载插件遗留的记录可经
  `credentialKeyScope` 辨认为孤儿（配置界面须报孤儿而非"可用的凭据"）。

接缝级规则：**空的存储值在一切层面都视为不存在**——`resolve` 跳过、`describe`
报未配置，空串绝不冒充已配置的机密。外部来源的名字先过
`isCredentialRefName`/`isCredentialKeySegment` 预检——不合语法的名字读作
"没有这条"，而不是抛错。

`credentials-local`（935 行 [MEASURED]）的解析栈按"该层多可信"排序，模块头
注释即文档：

| 层 | 载体 | 可写性 | 理由 |
|---|---|---|---|
| 1 process env | 继承的环境 | **只读且赢** | `KEY=… dsh`/CI secret 是"本次运行的显式意图"，进程内改不动它，必须**可见地**只读而非静默遮蔽写 |
| 2 托管文档 | `<harness home>/.credentials.yaml` | 可写 | 产品拥有的存储 |
| 3 project `.env` | 调用 cwd | 只读兜底 | 信任"从中启动的那个项目"；但永远压不过托管仓——Models 页写的 key 不被某个 checkout 携带的旧 key 顶掉 |
| 4 user `.env` | harness home | 只读兜底 | 最泛的位置排最后 |

环境事实来自 `launch-environment` 的**启动冻结快照**：之后的 chdir、工作区切换、
恢复的会话看到的都是开机时解析的层——"消费方绕过摊平的 `process.env`"是显式设计。

- **shadowed 写拒绝**：写前判 `inherited(ref)`——env 遮蔽中的引用 set 会"表面成功、
  解析持续返回遮蔽值"，直接拒绝并指路"去启动 dsh 的那个 shell 里 unset"。入队时
  判一次、出队时**再判一次**（排队期间环境可能变了）。
- **解析即信任**：文档只允许 `version/refs/records` 三个顶层键、每条都校验、
  未知字段/错型/空值/未知 kind **全部拒绝而非跳过**——"我存的凭据没生效"比报错
  难堪更糟。grant payload 进出双向 `assertJsonValue`（JSON 往返不了的 Date/实例/
  `bigint` 拒之门外，诊断永不引用值本身）。
- **预发布布局就地迁移**：识别器精确到"无 version 键、无文档指令、标量键全部
  合语法的非空字符串映射"才认定旧布局，把原文逐行缩进挂进 `refs:` 之下——
  注释与拼写按字节存活；迁移在锁内重读再写，认错的文档保持响亮拒绝。
  注释自注"首个 tagged release 时连同预发布姿态一起删除"。
- **权限面**：`assertOwnerOnly` 在**读内容之前**检查 mode——组/其他位非零即拒启，
  错误文案直接给出 chmod 指引；每次 reload 与写前复查（外部编辑器或还原备份可能
  事后放宽）。POSIX-only：Windows 无 mode 可查，**跳过而不装样子**。
- **锁窗口按操作定价**：`withFileLock` 默认 2s（够"渲染+改名"的文件工作）；凭据文档
  统一 `waitMs = 30_000` [MEASURED]——`modifyRecord` 的 mutate **持锁决策**，token
  刷新意味着一次网络往返，按默认等待会让同一文档的其余全部写方在窗口内超时。
  refs 与 records 共享一个文件一把锁，等待定价按最长持有者算，注释明说这一点。
  锁本体 = `<file>.lock` 兄弟文件 `wx` 独占创建 + rename 提交 → 读侧无锁；
  Windows 的 EPERM 仅当 lstat 证明锁存在才算争用；孤儿锁**永不由后来者按文件年龄
  猜测回收**——恢复是运维动作。
- **事件只在提交后**：`notifyUpdated`/`notifyRecordUpdated` 于 durable 写完成后触发
  ——"坏观察者不能让已持久的写看起来失败"。reload 路径先 diff 快照再逐条发事件；
  快照**整体替换**，被删条目不会在内存里还魂。

## authorization：配置供不出的凭据怎么进来

接缝管**对话与生命周期**，永不管协议：懂自家协议的插件注册一个 flow
（key/label/methods/run），协议方言被关在 flow 里——"第二个授权协议以另一个 flow
到来，而不是以另一个 seam"。

- **每 key 一 flow 一尝试**：`DUPLICATE_FLOW`/`ALREADY_IN_FLIGHT` 都响亮拒绝；
  第二调用方**不被合流**——两个尝试会通过同一 flow 向不同人问不同的题。
  interaction 随 `begin()` 传入而非注册制：提示只送达发起它的那个页面，headless
  调用方给一个"逢问必拒"的 interaction 即可。
- **提交见证是硬契约**：`run()` 正常返回 ≠ authorized——seam 在尝试期间 watch
  `credentials/record-updated` 亲见 committed、事后 `describeRecord` 仍在，两关齐过
  才报 `authorized`，否则 `NOT_COMMITTED`。重授权时旧记录本来就在，光凭"存在"
  会让一个什么都没写的 flow 把旧凭据报成刚授权的。
- **人的"不"与机器的"坏"分家**：`AuthorizationDeclinedError` 只许人类抛出——
  prompt 自带的 signal 被撤回（例如 flow 竞速两路、撤掉败者）必须用别的拒绝，
  否则随后的真实失败会被误读成拒绝。withdrawn 与 run 用 `Promise.race`：
  对 abort 无反应的 flow 不能把 key 锁到进程终——孤 run 被放弃自行落地，它若仍
  提交了记录，那依然是人授权过的凭据。
- `authorization/settled` 在 running 槽位**释放之后**发射：监听到事件就发起下一次
  尝试的界面，不会撞上刚结束的那次；`failed` 只存在于事件流（自己的调用方看到的是
  抛错）——旁观的第二个标签页需要另一种方式区分"拒了"与"坏了"。
- 本 build 唯一注册点 [MEASURED]：`llm/llm-pi-ai/src/login.ts`。claude-code/codex
  等外部宿主的登录路径未走此 seam（grep 无命中），其真实形态见各自承接篇。

## storage-domain：域路由侧

storage 枢纽上挂载的"领域数据形式"（枢纽与后端注册归
[持久化深读](./persistence-crash-recovery.md)）：消费方永不直触后端。

- **路由在域侧插件 config**：`backend` 必填（"没有普适正确的介质"）+
  `routes` 按域名覆盖；路由指向未注册后端在 `open` 时以 `backend-not-found`
  响亮失败。
- **声明即校验**：`defineDomain` 在属主包**模块加载时**跑配置自检——域名/表名合
  语法、版本非负整、`compatibleVersions` 均低于当前版、global schema 拒收
  `null`（`null` 是介质的"从未写过"哨兵，可空的 global 在重开时分不清"没写过"与
  "存了个 null"，静默回退 initial）。
- **运行时模型**：内存权威快照 + 每域一条写链。**先等后端 durable、再动内存、
  再发 `domain/changed`**——读与介质永不背离；事件携带的值 = 发射时刻的内存值，
  且按写链顺序到达。表句柄同步读、`entries()` 是快照迭代（排队写落地不影响进行
  中的遍历）；`update` 是链上原子 read-modify-write，`delete` 的存在判定也推迟到
  自己槽位——先前排队的 put 会被看见。
- **坏记录两姿态**：权威域 `open` 整体拒绝（`invalid-record` 带表/键定位）；声明
  `invalidRecords: 'backup-and-skip'` 的可弃派生域把失败记录的文档**移到一边**、
  记日志、缺席继续 open；`layout: 'per-record'`（投影缓存用它）按记录文档独立
  版本校验，不接受的记录丢弃而非迁移，写入永远盖当前版戳。
- `DomainChanged` 是封闭 union（put 带新快照 / deleted 无值）——**不携带旧值**，
  diff 型消费者自存上一份快照；events.ts 注释自报"跨进程变更推送是后续阶段"，
  当前事件仅进程内消费。

## workspace：稳定 id、候选账本与三段写

工作区 = 规范路径上的稳定 uuid + 标题 + 会话有序账本；**宿主侧可选能力、对模型
完全不可见**（无工具、无提示词文本、无会话事件——官方页自述，包内核实）。

- **id 与路径分居**：id 永不用路径——"路径会被规范化改写，引用锚点必须稳定"；
  唯一性规范只有 `realpathNormalize` 一套（尾斜杠/`..`/symlink 全解析，指向
  已拥有目录的符号链接按串相等判冲突）。相对路径与 Windows 盘根不完备拼写在
  `realpath` 之前先拒——绝不借宿主 cwd 或当前盘符兜底。
- **账本双校验**：成员资格 = id 在账上 ∧ 会话 header 的 canonical cwd === 工作区
  路径。读侧 `sessionIds` getter **同步过滤**（缺 header/cwd 不合/目录解析失败都
  不进视野），每个被接受的写顺带 durable prune 掉被过滤候选；attach 校验
  header.cwd 必须解析为**存在且等于**工作区目录；已在账上免重验——两个输入
  （存储 header、工作区路径）都不可变。move 语义 DOM `insertBefore` 式，
  不在账者 `WorkspaceMoveInvalidError`，原地移动 no-op。
- **no-op 不写介质**：实体唯一写径 `mutate` 在域写链上跑，返回原记录 + 剪枝
  无变化时以**内部 sentinel** 中止本次槽位——介质不重写、事件不发。
- **三段写 + 恢复只朝一个方向**：create = 置 `pendingMutation{create}` → 写记录
  → 改全局顺序；每段失败逆序回滚，双败抛 `AggregateError` 并说明"标记仍可恢复"。
  启动时 `recoverPendingMutation`：见标记就**把删除做完**再清标记（pending create
  语义同样被完成为"确保不存在"）——恢复从不反向猜测"这行是不是该留"；标记与
  顺序表自相矛盾（待处理 id 仍在 order 里）则 fail loud。delete 尾段"标记没清掉"
  降级为 warn：删除已是事实，不把已达成说成失败，留下的标记交给下次启动收尾。
- **启动一致性四查**（`validateStoredState`）：顺序不重复、顺序引用恒存在、
  initialized 后无游离记录、路径与 session 各归一主（一主账本不变式）。
- **历史 bootstrap**：`initialized: false` 区分"合法的空中枢"与"待引导"——从
  `sessionPersistence.list()` 全量 header 按 canonical 路径分组、组内按创建时间倒序、
  组间按最新时间倒序建工作区并合并记账，两步写 `false→true` 即崩溃安全开关。
  持久化是**硬依赖**：`inject` 含 `sessionPersistence`，对等件不可用时插件保持
  pending——绝不把"看不见持久化"误读成"历史为空"而提交 initialized 戳。
- **归档是叠加在账本上的 registry 全局集**：归档会话保留账本槽位（取消归档即
  恢复原位），故归档集永不参与一主账本不变式；`archiveSession` 只认"确切的
  miss"——存储故障原样上抛，不冒充 unknown session。

## 三缝一样式（源码里能读出来的"家规"）

- **containment fan-out 是被评审过的生命周期契约**：settings/credentials/
  authorization 三处手写逐 listener 派发 + INVARIANT 最后重抛，形状逐字相同；
  两处包内 `jscpd:ignore` 注释明说"这是对称性偏好、提取公共 helper 反而会耦合
  两条接缝的事件语义"——复制是决定，不是懒。
- **settings-file 与 credentials-local 是同一家族的镜像**：Config 四字段、
  text 缓存、单操作链、watcher ready 补扫、last-good 策略逐段互引"故意对称"。
- **"boot fail loud / live keep last good"** 贯穿五处：settings-file、
  credentials-local、storage-domain（invalid-record vs backup-and-skip）、
  workspace（四查 vs 过滤 warn）、以及 credentials 文档的预发布识别器（认得就迁、
  认不得就响）。

## 相关

- 概览与浅层叙事：[配置面](../augmentation/settings-and-credentials.md)
- 存储枢纽与后端、会话日志崩溃恢复：[持久化格式与崩溃恢复](./persistence-crash-recovery.md)
- 注册即 fiber effect 的热重载语义：[Cordis 内核](../cordis/cordis-kernel.md) ·
  [插件组装与启动](../cordis/plugin-composition.md)
- 设置界面对 describe/凭据视图的消费：[host/client 分层与 API 网关](../platform/host-client-boundary.md)
- workspace 作为 webhook/会话归属容器：[后台任务与外部触发](../augmentation/background-and-triggers.md)

## 诚实边界

- 后端介质实现（storage-json/storage-sqlite 的 `KvUnit` 细节、锁文件在
  各后端的形态差异）未展开——归持久化深读管辖；本篇只走到 `descriptorOf`
  递交给后端的形状。
- api/settings-controller 的远程写路径（`SettingsPathOpView` 在 wire 上的调用
  细节）只核实"两处 describe 均强制 redact"，未逐调用走读。
- authorization 本 build 仅一例 flow 注册；subagent-claude-code/codex 的登录路径
  是否绕开此 seam 未逐一验证（grep 无 `registerFlow` 命中）。
- 多进程竞态（`withFileLock` 争用、凭据迁移双 boot）以源码注释与单测声明为据，
  本机未开双进程复放。
- Windows 上凭据文档的保护语义（"create/replace APIs 所能表达的"）系源码注释
  声明，未做 ACL 实测。
- 观测数字：`defineDomain` 3 处、`role('secret')` 真实字段 1 处、
  `credentials.resolve` 消费 4 处——均为本 checkout 的 grep 快照 [MEASURED]。
