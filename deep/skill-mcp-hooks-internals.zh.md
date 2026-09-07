---
title: "技能·MCP·hooks 内幕：目录经济学、桥接克制与双方言单引擎"
tags: [dsh, skills, mcp, hooks, catalog, bridge]
status: active
license: CC-BY-SA-4.0
evidence: "packages/{skill,mcp,hooks}/* src 直读 + docs/subsystems/skills.zh.md + Agent Notes（.agents/notes/implemented/feature/2026-06-30-hook-bridges.md、2026-07-07-mcp-client-plugin.md）；mcp/hooks 无独立子系统页，口径以源码为准"
updated: 2026-09-05
---

# 技能·MCP·hooks 内幕：目录经济学、桥接克制与双方言单引擎

[English](skill-mcp-hooks-internals.md) | [中文](skill-mcp-hooks-internals.zh.md)

> [概览篇](../augmentation/skills-mcp-hooks.zh.md) 的 skills/MCP/hooks 三节深读展开。
> commands 归交互组承接篇、preset 归[提示词组装](./prompt-assembly-context.zh.md)、
> extensions/identity 归热重启与交互两篇——本篇不越界。三块共享同一主题：
> **能力从 harness 外面来，进来时先过词汇收窄**。

## 概览篇说法 → 深读结论

| 概览篇说法 | 深读结论 |
|---|---|
| rank 六档 100→600 | 证实+补一漏：runtime 技能固定 **rank 250**（`RUNTIME_RANK`），单层内插在 project-agents(200) 与 custom(300) 之间 [MEASURED skill/src/index.ts] |
| "command 钩子经 `ctx.shell.runHook`" | 措辞修正：`runHook` 是 hook-protocol 库函数，把注入的 `ctx.shell` 当执行器调 `resolve()/run()`；shell seam 上并不存在 runHook 方法 [MEASURED runner.ts] |
| "不代管 resources/prompts——能力词汇表保持小" | 取舍原因闭环（Agent Note 原文）：Resources/Prompts **deferred**——"harness 侧消费机制尚不存在，且设计空间大"；Client 初始化 `capabilities: {}` 空声明 [MEASURED connection.ts + note] |
| "exit 2 = 阻塞、其它失败不阻塞" | 完整解码契约见「codec」节：只有 exit 0 且以 `{` 开头的 stdout 才尝试结构化 JSON；畸形 JSON 宽容降级为纯文本；信号死亡（exitCode null）→ undefined=非阻塞；基础设施失败被造成为"无退出码"结果——**runHook 永不抛** |
| （概览未提）updatedInput | 解析但**不生效**——输入改写被明确推迟；CC 桥遇之仅 warn，Codex 桥根本没有该通路 |
| "`hook/invoked|`hook/result` 配对事件在轮次内落日志（invariant 伴生强制）" | 强制细节闭环：伴生插件在 `internal/dispatch` 预提交阶段校验"发布前必经验证"、turn 括封、配对守恒（result 无主=失败）；SubagentStart/Stop 与 SessionStart 属 detached 点**不落** `hook/*`——无开轮可记 [MEASURED invariant.ts/events.ts] |

## ctx.skills：分层目录与一次重试的缓存

注册表与[工具注册表](../agent-runtime/tools-pipeline.zh.md)同形（host+per-scope 层）：
注册落进调用方上下文 scope 的层——preset 常驻组合注册的提供方只对那个 scope 可见；
读取合并全局层与观察 scope 链，**最近层的重名条目直接赢**，rank 只在层内裁决，
层内次序 = rank → 提供方注册序 → 本地序 [MEASURED]。

- **缓存键含 scope 链**：`{cwd, scopeIds, revision}`，scope 经 WeakMap 折成稳定数字 id。
  注释点名一个易错场景：空会话重组（recompose）会换 scope 父级而**不触碰注册表**，
  只有键里带链，下一次读取才看得见新 preset。
- **revision 竞态协议**：发现进行中注册表变了 → 重试一次；再变 → 返回最新候选但
  标记不完整、**不予缓存**。`snapshot()` 返回 `{skills, complete}`，不完整观测永不进缓存——
  消费方靠它决定"敢不敢发布目录"。
- 提供方 `list()` 抛错=记录 warn+剔除结果+缓存降级为不完整；但 `signal` abort 例外，
  向上抛（调用方自己取消了就如实死）。
- `validateCandidate` 防冒充：candidate.provider 必须等于提供方自己的 name；
  `get()` 加载后定义与候选 name 不符 → 精确失效该提供方条目。
- `skills/change` 无过滤无 diff；监听器抛错/reject 全被吞（"refresh 不 load-bearing"）。

## skill-filesystem：发现与失效的双通道

- 两种布局：`<name>/SKILL.md` 目录包 或 `<name>.md` 平铺；**不支持** 递归
  `**/SKILL.md`（官方页明说）。用户 dsh 根跳过 `.system` 子目录；本地提供方
  从不合成内置技能——bundled 技能必须显式配根。
- **读文件双通道**：`ctx.fs` 在 → resolve/stat/readText 全走文件系统 seam
  （远程/沙箱工作区同样可发现技能，git 根上行探测也一样）；`ctx.fs` 不在或
  trustedHost（bundled 根）→ node fs 直读。后者是宿主信任面的显式豁免。
- frontmatter 严格化：首行必须恰为 `---`；name+description 必填；
  invocation 只认 kebab 键（`disable-model-invocation`/`user-invocable`），
  camelCase 旧键抛错并指路正规键名；布尔宽容接受 `1/0/yes/no/on/off` 系。
- watcher 参数 [MEASURED]：chokidar depth 1、atomic、awaitWriteFinish
  稳定窗 200ms、轮询 100ms、项目 watch 上限 128（最老项目整体腾退）。
  失效相关性只认两层：根直属 `.md` 与 `<entry>/SKILL.md`——**bundle 内资源文件
  变更不触发目录失效**（正文没变，变的只是素材）。
- **不存在的根也能 watch**：从最近存在祖先起步，watchFile 轮询第一个缺失段，
  目录长出来即升格挂 chokidar；开窗前后双查 watch mode 防竞态。watcher 故障
  → 不健康标记 + 返回"不完整观测"（可读候选照常给出）。
- 失效是**双通道**的：chokidar 事件之外，模型 `write/edit` 经 `fs/observed`
  事件同步直达（actor 名必须是 edit|write 才认），两通道都汇到
  queueMicrotask 合并的一次 invalidate。
- 源码 TODO 点名：chokidar/watchFile 这套值得抽成独立 Cordis 文件观测服务。

## tool-skill：目录是持久消息，不是提示词碎片

会话技能目录走[事件日志](../agent-runtime/session-event-log.zh.md)路线而非
system prompt 路线——这是一条 `user/message`，`source.kind='skill-catalog'`
且**自带 entries 镜像**（name+description 逐条）。注释原文：呈现方不得回解
`<available_skills>` 块——伪 XML 是给模型的框，持久事实独立于框。

- **身份=摘要不认渲染**：目录指纹是 sha256(JSON 逐条目规范化)，注释解释了
  为什么不能用分隔符——任何分隔符都可能是 description 合法字符，只有 quoting
  使边界精确。description 空白归一+截 500 字符（下限 3）[MEASURED]。
- 发布决策由 `catalogHistory()` 反向扫事件得出 {visibleDigest, published}：
  当前可见目录指纹未变 → 不发布且清掉批内冗余；变了 → 追加**替换式**
  update 目录（框文直说"本目录取代此前一切列表"）；从未发布且零技能 → 只清理。
  不完整 snapshot 一律按兵不动。
- **可见性匹配定义身份**：`ctx.tools.get(name, agent) === skillTool` 恒等比较——
  一个只叫 `skill` 的 scoped shadow 不能继承这套目录；工具面没了，目录 guidance
  同时消失。
- `/name` 手势（用户直呼技能）：正则 `(^|\s)/kebab(?=\s|$)`，第二个 `/` 或非边界
  即破——注释点名 `/usr/bin` 与 `5/8` 因此进不来。只扫 `source.kind='user'`
  的消息（"外部文本不能伪造手势"）；不认识/非 user-invocable 的 token 保持原文
  当普通散文。**这是 disable-model-invocation 技能的唯一入口**，目录与
  `skill` 工具永远看不见它们。
- 注入落位：手势正文排在**所有**背景注入之后（注释：workspace 规则、运行时策略、
  目录都是背景，"要行动的材料离答案最近"）；渲染与工具结果共用
  `renderSkillContent` → 两条路径模型所见逐字节一致。
- 敌意种子姿态：目录事件 entries 字段读不懂 → 当"不是本插件的目录"处理。注释
  明说为什么：抛出去会**毁掉该会话后续每一轮**。
- `skill-badge` 是提供方最小标本（单候选+assets resourceBase+bundled rank）；
  官方页记录 CLI 交付默认禁用它——启用即显式 opt-in。

## MCP 桥：命名是纯函数，世代是全有全无

- **serverName 是用户配置，绝不取远端 `serverInfo.name`**（Note 三条理由：
  不受信、跨部署不唯一、升级会变——全都不得静默改名模型可见工具）。重名跨实例
  = 配置错误：后来者在装载期响亮失败，无静默 shadow；占用表 WeakMap 按注册 scope
  分桶——两个 Agent 的 MCP 服务器可复用同名。
- `publicToolName(server, raw)` 纯函数：`mcp__<server>__<raw>`；越 64 字符
  DeepSeek 函数名契约或含非法字符时，规范化+截断+**12-hex sha256(`server\0raw`)
  后缀**，防不同身份折叠同名。raw 名只在 wire 上走；public 名永不反解析 [MEASURED]。
- `syncTools` 两段式：fetch 全部翻页并建定义（服务端重列同一 raw 名=整次失败）
  → swap（先 dispose 旧世代再注册新的；注册冲突=外来户占了这个 namespace，
  **整代回滚、零工具留场**）。只有启动期且 `failOnStartupError` 才向上 throw。
- 监督器数字 [MEASURED RECONNECT_DEFAULTS]：500ms 起、指数×2、帽 30s、每中断
  10 次；连上后存活超过 maxDelayMs（稳定窗=最长退避间隔）→ 视为上次中断结束、
  预算清零；**crash-loop（秒连秒断）仍会烧光预算**——防无限重启。烧光=工具全部
  注销并停手，只有 HMR/重载可救。
- 世代关闭有 5s fail-closed 帽（SDK 自握两个 2s 终止宽限，+1s 等 close 证明），
  超时宁可停监督也不让子进程叠罗汉；重连 timer 一律 `unref()`。
- "uncached"姿态三处：`tools/list` 手发请求绕 SDK 分页校验缓存；`tools/call`
  用 `z.record(unknown)` 接原始结果——桥自己拥有 transport 之后的校验；模型给出
  非对象参数时兜底 `{}`，让服务端报具体缺参错误给模型学。
- **图片结果=信任边界全景**：mediaType 限四格式、canonical base64 解码重编码
  相等性双验、当前路由必须**正面声明** image 模态（解析不出路由也拒）、
  `saveImages` 批量入附件仓；任一被拒 → 整批图片退化为诊断文本
  （"...raw image data remains available to programmatic callers"）——
  模型上下文与 PTC 值各保各的形状（详见[附件与溢出](./attachment-spill.zh.md)）。
- `finalizeContent` + `WeakMap<ToolExecution, PreparedProjection>`：execute 返回
  canonical `McpResult`，富投影（图片块替换）只在 value 与 fallback content 双双
  深相等时兑现——事后改写进不了持久内容 [MEASURED]。
- 边界两条：`taskSupport:'required'` 的工具直接运行期拒绝（桥不支持任务型执行）；
  stdio 子进程 spawn 由 MCP SDK 持有、环境经 `scrubbedParentEnv()` 清洗
  （凭证形状与陈旧 `DSH_*` 剔除）但**不经沙箱仲裁**——与模型
  [子进程](../execution/shell-process-terminal.zh.md)世界的待遇不同。

## hooks：一个引擎、两套方言、"兼容适配器"定位

`hook-protocol` 是库不是插件：codec/matcher/merge/runner/detached/invariant 六件套
共享；两个桥各自拥有 payload 形状、环境规则、matcher 模式与 Decision 映射。
Agent Note 的核心律令：**桥是兼容适配器，不是权力工具**——桥能做的一切
（阻塞/注入/强续轮），原生 Cordis 插件都能更有力地做：类型化返回、完整 ctx、
无序列化边界。

- **codec 决策双通道折叠**：顶层 legacy `decision` 只有 `approve/block` 合法
  ——越界的 `{"decision":"deny"}` 无效被忽略；`allow/deny/ask` 只能来自
  `hookSpecificOutput.permissionDecision` 且**覆盖**顶层值；块内
  `hookEventName` 与触发事件不符（或缺席）→ 事件级字段整体丢弃，但判别值仍记录
  （畸形块要能看出它自称是什么）。
- merge：rank 制 `deny(3)>ask(2)>allow(1)`，`block`折进 deny、`approve`折进
  allow；**理由只从获胜 rank 收集**（赢的人说话）；第一个 `continue:false`
  粘住；additionalContext/systemMessage 按钩子顺序累加不拼接——拼法归桥。
- matcher：CC 纯 `[A-Za-z0-9_|]+` 判为字面管道多选，否则正则；Codex 恒为不锚定
  正则。非法正则双时态：配置期 diagnostic 抛 SyntaxError **整份配置拒收**
  （监听器一个都不注册），运行期被禁为永不匹配。
- runHook 走 shell seam：默认超时 10 分钟（wire 单位是秒）、`trailingNewline`
  方言各异（CC 有/Codex 无）、每次运行记 wall-clock duration 进
  `hook/result`（stderr 摘要截 500 字符 [MEASURED]）。
- **detached 静默**：SessionStart/Subagent* 是 emit 形——没人 await。track 的是
  全链（含续作），drain 先 abort（**杀掉**还在跑的十分钟钩子而非等超时），
  再循环清空注册表；注释点名："未被 track 的 .catch 就是把失败变成沉默"。
- ask 不是终态：CC 的 ask → `PreToolDecision.ask` → 经 approval seam 找
  answerer；无 ApprovalService 或无人应答 → **fail-closed 成 deny** [MEASURED note]。
- 失标守卫（mislabel guard）：桥注入的每条上下文显式
  `source={kind:'plugin', plugin:桥名}`，单测钉死它不被记成用户发言。
- 偏差清单（以代码为准，README 管全量库存）：`transcript_path` 恒空
  （''/null 各方言形状不同）——持久化 seam 不暴露工件路径；`stop_hook_active`
  恒 false——被 Stop 钩子强续的轮次里钩子无法自知，TODO stop-loop-guard；
  CC 的 Subagent* matcher 主语恒 `general-purpose`（harness 无"种类"概念，
  具体 kind 的配置永不触发）；`merged.stop` 只记录不执行（无 run 级 halt 机制）；
  两份 config 都是进程级读一次（TODO per-session 项目本地发现）；
  Note 里 "inject only bash" 相对现码滞后（实为 `['shell','sessionProjections']`）
  ——口径以源码为准。

## 双桥差异矩阵

| 轴 | Claude Code 桥 | Codex 桥 |
|---|---|---|
| 点位 | 7（含 SubagentStart/Stop） | 5 |
| matcher | 字面快路径或正则 | 恒正则 |
| stdin | JSON + 尾换行 | JSON 无尾换行 |
| 纯 stdout | 不升上下文 | SessionStart/UserPromptSubmit 在 exit 0 且无结构化字段时升为 additionalContext |
| pre-tool | deny + ask（→approval） | 仅认 deny（faithful-but-degraded） |
| payload | CC 形状（camel 混合） | snake_case + `model`/`permission_mode`/`turn_id` |
| 占位符替换 | `${CLAUDE_PLUGIN_ROOT}`/`${CLAUDE_PROJECT_DIR}` | 无 |
| 非 command 型钩子 | prompt/agent/http → skip+warn | async:true 亦 skip |
| handlerId | `claude-code:<point>:<n>` | `codex:<point>:<n>` |
| 注入源标记 | `plugin: 'hooks-claude-code'` | `plugin: 'hooks-codex'` |

## 克制为什么成立（三条设计逻辑）

1. **两级 token 经济学**：目录只送 name+description（500 字符帽），正文按名拉取；
   MCP 只桥 Tools，resources/prompts 等消费机制出生再进——进来的能力先过词汇收窄。
2. **faithful-but-degraded**：方言里支持的子集如实映射到类型化 Decision，不支持的
   字段解析、忽略、留痕——宁可降级也不臆造语义。
3. **身份是纯函数**：`publicToolName`、`renderSkillContent`、目录摘要指纹全为确定性
   纯函数——HMR/重连换世代后，会话历史与权限规则仍然指得准。

## 相关

- 概览：[能力供给](../augmentation/skills-mcp-hooks.zh.md)；命令/审批：
  [工具流水线](../agent-runtime/tools-pipeline.zh.md)
- 图片入上下文：[附件与溢出](./attachment-spill.zh.md)；PTC 值形状：
  [PTC 与 code-runtime](./ptc-code-runtime.zh.md)
- 注入落位与表层：[会话事件日志](../agent-runtime/session-event-log.zh.md) ·
  [表层改写与压缩](./surface-compaction.zh.md)
- preset 深读：[提示词组装与上下文注入](./prompt-assembly-context.zh.md)
- extensions 动态化：[Cordis 热重启](./cordis-hot-reload.zh.md)

## 诚实边界

- mcp/hooks 无官方子系统页；本篇口径 = 源码直读 + 两篇 Agent Note + 包 README
  引用。README 的"不支持事件全量清单"未逐条核对。
- CC/Codex 的"参考协议一致性"未对上游真实产品实测（codec 的字段白名单是
  按源码注释所引 schema 子集转述）。
- 未端到端真机运行一份 hooks.json（本部署未配置钩子）；runner 失败语义以
  codec/runner 源码与其单测断言为准。
- MCP 未实测外接真实 server：分页、uncached、图片 admission 全为源码走读
  （deep-qa/pub-scan 机检通过）。
- 目录"追加+框文声明替换"与表层 replace 的交互（旧目录是否会被压缩改写）
  未对照 [表层改写与压缩](./surface-compaction.zh.md) 实测时间线。
- `/name` 手势与斜杠命令（command 注册表客户端侧先行解析）的边界只按源码注释
  转述，未追到 client 侧代码验证全链。
