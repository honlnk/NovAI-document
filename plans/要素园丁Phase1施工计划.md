# 要素园丁 Phase 1 施工计划

## 记录时间

2026-09-25 立稿（依据《[要素园丁子代理设计](../architecture/要素园丁子代理设计.md)》v3，`8785f6c`）

## 状态

已完成（2026-09-25 结项，代码 `3ac5dd2` / `05c1ea0` / `2311afa`）。未验证清单已于 2026-09-29 经用户真机复核全部通过（见施工日志文末补记）。

## 1. 背景与输入

设计文档 v3 已定稿落档。本计划把设计 §8 的 Phase 1（最小闭环）拆成可施工、可验收的阶段：

> 委派基建（工具 + 子代理运行壳 + 工具白名单）+ 一次性园丁 + `/整理要素` 斜杠命令 + 聊天区子代理任务组呈现 + `prompts/ELEMENT.md`（默认内容 + 初始化/补齐 + PromptList「要素规范」分组）。
> 验收：用户发起整理任务，能看到园丁逐步工作、逐卡确认、最后收到结构化汇报。

施工前代码侦察已核实的关键事实（本计划的落点全部以此为准）：

| 现状 | 文件 | 对施工的含义 |
| :-- | :-- | :-- |
| 主 Agent 每轮 `createAgentTools()` 组装全量工具表，`Record<AgentToolName, …>` 强制全键 | `core/agent/tools.ts` | 委派工具入表需把 `AgentRunnableToolMap` 放宽为 `Partial<Record<…>>`；未知工具本就有兜底分支 |
| `query()` 是纯函数式循环（view + tools + confirm + signal + onEvent 全部入参注入） | `core/agent/query.ts` | 园丁子代理 = 拿一套独立入参再跑一次 `query()`，零 Loop 改造 |
| 写确认链：`executeAgentTool` → `decideWriteToolPermission`（按档位判定）→ `ConfirmHandler` | `core/agent/tool-execution.ts`、`permission.ts` | 园丁内层写工具复用同一条链 → 确认卡、档位继承（决策 3）天然成立；委派工具自身无 path 字段，必须标记外层 `isReadOnly`（它不直接写文件） |
| 子代理事件回聊天：session 层 `onEvent` 把 tool-call/tool-result push 成 `ChatMessage` | `core/chat/session.ts` | 消息加可选 `agent?: 'gardener'` 标记即可，旧会话无字段照常渲染 |
| 斜杠命令 `/生成项目记忆` = 常量驱动 prompt 走普通发送链路 | `app/constants/slash-commands.ts`、`ChatPanel.vue` | `/整理要素` 同款交互：无二级界面，驱动 prompt 指示主 Agent 调委派工具 |
| 提示词家族初始化：`createProject` 写默认、`repairProject` 缺失补齐（不覆盖用户版本） | `core/fs/project-fs.ts` | `prompts/ELEMENT.md` 完全套用 `NovAI.md` 的初始化/补齐策略 |
| 测试基建：session-driver 全链路测试 stub 全局 fetch 编排 SSE；fs 测试有内存目录句柄 | `core/chat/session-driver.test.ts`、`core/fs/project-fs.test.ts` | 园丁端到端测试照抄 fetch stub 模式，无需真实 LLM |

## 2. 定稿决策（无开放选择题）

以下影响用户体验的点全部拍死，施工方不得另选：

| # | 决策点 | 拍板 |
| :-- | :-- | :-- |
| D1 | 委派工具名 | `DelegateToGardener`（英文工具名，随现有工具命名） |
| D2 | 工具入面范围 | 主 Agent 工具面常驻该工具（不只在 `/整理要素` 后临时注入）；园丁自己的工具面绝无此工具（深度写死 1） |
| D3 | 园丁工具白名单 | 可读：`ListDirectory` / `FindFiles` / `ReadFile` / `RagSearch`（全项目可读）；可写：`CreateFile` / `EditFile`（运行层强制 `elements/` 前缀，路径闸拒绝即报错回灌）；无 `DeleteFile`、`RenameFile`、`GetFileChangeHistory`、`WebSearch`、`WebFetch` |
| D4 | `/整理要素` 交互形态 | 无二级界面，与 `/生成项目记忆` 同款：选中即发送驱动 prompt（常量 `GARDENER_TASK_PROMPT`），由主 Agent 调用 `DelegateToGardener`；图标 🌿，描述「委派园丁子代理整理 elements/ 要素库」 |
| D5 | PromptList 分组 | 第四分组「要素规范」，置于「项目总览」之后、「场景提示词」之前；文件图标 🌿；右侧徽标文案「园丁专用」（对照 system.md「恒激活」、NovAI.md「恒注入」） |
| D6 | 聊天区呈现 | 园丁的 tool 行折叠为嵌套任务组，组头「🌿 园丁整理 · N 次工具调用」；复用 TurnProcessGroup 折叠形态与 `hidden="until-found"`；默认展开/收起跟随外层任务组同规则（运行中最后一轮展开、结束收起、用户可手动切换并记住） |
| D7 | 确认卡归属标识 | 园丁触发的写确认卡带「园丁」徽标（header 前缀），其余与主 Agent 卡完全一致 |
| D8 | 园丁汇报形态 | Phase 1 不做 outputSchema JSON 强约束；persona 规定固定 Markdown 报告结构（改动文件清单 / 合并与迁移 / 存疑事项待拍板 / 建议删除），全文作为委派工具结果回灌主 Agent 消化总结 |
| D9 | 委派工具外层语义 | `isReadOnly: true`（本工具不直接写文件；写确认全部发生在园丁内层工具上）、`isConcurrencySafe: false`（长任务，串行） |
| D10 | ELEMENT.md 缺失兜底 | 园丁启动时文件缺失（理论不该发生，repair 已补齐）→ persona 后附一行「规范文件缺失，按内置守则工作」，不阻塞运行 |

## 3. 范围与落点

### 阶段 1：`prompts/ELEMENT.md` Schema 层

- `core/project/defaults.ts`：新增 `DEFAULT_ELEMENT_SCHEMA` 常量，内容按设计 §7.3 六节：六类目录职责与边界（含 entities vs worldbuilding 判断原则）、命名规范（一人一文件、文件名即要素名）、frontmatter 维护规范（tags 去重、`relatedChapters` 取并集、`lastUpdatedChapter` 取最新）、链接约定（素材→章节写 `chapters/第NNN章-标题.txt` 路径、素材↔素材 `[[要素名]]`、正文不写链接）、园丁工作守则（合并保留旧文件全部事实、按模板小节归位、存疑不合并列入汇报、不删除文件）、模板结构说明（五类模板分节名，`templates.ts` 代码兜底声明）。
- `core/fs/project-fs.ts`：`createProject` 写入 `prompts/ELEMENT.md`；`repairProject` 缺失补齐（已存在不覆盖）；新增 `readElementSchema(rootHandle)`（缺失返回空串，照 `readNovAiOverview`）。
- `app/components/category/PromptList.vue`：第四分组（D5）。
- 测试：默认内容六节齐全；初始化写入 / 补齐 / 不覆盖已有版本；readElementSchema 缺失空串。

### 阶段 2：园丁运行壳 + 委派工具（core）

- 新文件 `core/agent/gardener.ts`：
  - `GARDENER_PERSONA` 硬编码（设计 §4：要素库园丁、从不动 `chapters/` 正文、ELEMENT.md 是本书要素规范必须遵守、报告固定结构）。
  - `buildGardenerSystemPrompt(elementSchema)`：persona + ELEMENT.md 全文（空 schema 走 D10 兜底）——园丁提示词栈**只有**这两层（设计 §7.4）。
  - `createGardenerTools()`：从 `createAgentTools()` 过滤出白名单六工具（D3），`CreateFile` / `EditFile` 包一层 validateInput 路径闸（`elements/` 前缀，拒绝时报错信息说明「园丁只能写 elements/ 下的文件」）。
  - `runGardener(input)`：全新 `ModelView` + system(persona+schema) + user(task[+scope]) → 复用 `query()`（config 继承主 Agent：同模型、同压缩阈值、同 `agentMaxTurns` 安全阀）→ 返回 `{ report, aborted, toolCallCount }`。
- 委派工具：`AgentToolName` / `CoreToolName` / `ChatToolName` / `ToolNameView` 四处联合类型加 `DelegateToGardener`；`isAgentToolName` 同步；`AgentRunnableToolMap` 放宽为 `Partial<Record<…>>`（主 Agent 面由 session 层组装为全量 + 委派工具）。
- 工具定义（入 `gardener.ts` 或独立 `delegate-tool.ts`）：schema `{ task: string 必填, scope?: string }`；描述写明「子代理看不到当前对话，task 必须自包含」（照抄 dsh tool-subagent providerWording 原则）；`run` 时读 `prompts/ELEMENT.md` → `runGardener` → 内层工具事件透传给外层 `onEvent`（事件带 `agent: 'gardener'` 标记）；确认回调包装加 `agentLabel: '园丁'`（D7）。
- `core/chat/session.ts`：`runDriverTurn` 组装 `tools = { ...createAgentTools(), DelegateToGardener: … }`（闭包注入 confirm/signal/onEvent 转发）；`onEvent` 处理 `agent === 'gardener'` 的 tool 事件 → push 带标记的 ChatMessage；园丁写成功的 `fileChange` 照常进 `changeLedger`（同 runId，归本轮 change-summary 面板）。
- `WriteConfirmationRequest` / `FileChangeConfirmationView` 加可选 `agentLabel`；`agent-service.ts` 透传。
- 测试：persona 组装（含/缺 schema）；白名单精确性；路径闸（拒 `chapters/`、拒 `prompts/ELEMENT.md`、收 `elements/characters/x.md`）；fetch stub 端到端（主 Agent 调委派 → 第二次 fetch 断言园丁 system 含 persona+规范、tools 无 DeleteFile/DelegateToGardener → 园丁返回报告 → 主 Agent 收到工具结果）；`isAgentToolName` 覆盖。

### 阶段 3：`/整理要素` + 聊天区呈现（app）

- `app/constants/slash-commands.ts`：注册 `{ id: 'gardener', label: '/整理要素', … }`（D4）。
- `agent-service.ts` 导出 `GARDENER_TASK_PROMPT` 常量（指示主 Agent 立即调用 `DelegateToGardener`，task 自包含：整理整个 `elements/`、核对同名合并/近名去重/frontmatter 规范/链接约定，存疑列汇报）。
- `ChatPanel.vue`：`handleSlashCommandSelect` 加 `gardener` 分支（同 `init`：直接发送）。
- `ChatMessageView`（services/types.ts + mappers + agent-service）：tool-call/tool-result 视图加可选 `agent?: 'gardener'`。
- `app/stores/chat-render.ts`：新 `SubagentGroupItem`——轮内连续的 gardener tool 行折叠为嵌套组（D6）；`ChatPanel.vue` 模板渲染嵌套 TurnProcessGroup（组头文案换「🌿 园丁整理 · …」）。
- `app/components/chat/ToolCallRow.vue`：`DelegateToGardener` 行的名称/图标映射。
- `app/components/chat/WriteConfirmationCard.vue`：`agentLabel` 徽标（D7）。
- 测试：chat-render 园丁行折叠嵌套组（连续标记成组、无标记不混入、旧消息无字段不受影响）。

### 阶段 4：收尾

- 全量门禁：`pnpm test` + `pnpm typecheck`（core 与 app 双 typecheck）。
- 文档同步：设计文档状态流转（Phase 1 完成）、本计划施工日志、`docs/plans/README.md` 索引、`docs/project/当前进度.md`。

## 4. 迁移条款

| 旧物 | 处置 |
| :-- | :-- |
| 旧项目无 `prompts/ELEMENT.md` | 打开走 `repairProject` 补齐默认内容；已存在（含用户改过）不覆盖。不补的历史会话不受影响 |
| 旧会话消息无 `agent` 字段 | 可选字段，渲染层无标记照常平铺为普通 tool 行，无需迁移 |
| `AgentRunnableToolMap` 类型放宽（`Record` → `Partial<Record>`） | 纯编译期变化；主 Agent 工具面由 session 层组装后仍是全量 12+1 工具，模型可见面不变（仅新增委派工具） |
| 旧配置 | 零新配置项（Phase 1 无设置页改动；决策 3/4 的独立覆盖是 Phase 2） |

## 5. 验收点

| # | 验收点 | 验证方式 |
| :-- | :-- | :-- |
| A1 | ELEMENT.md 默认内容六节齐全、初始化/补齐/不覆盖行为正确 | 单测（defaults + project-fs，内存目录句柄） |
| A2 | 园丁提示词 = persona + ELEMENT.md 全文，无其他提示词家族文件混入 | 单测（buildGardenerSystemPrompt） |
| A3 | 园丁工具白名单精确（6 个）、路径闸拒绝非 `elements/` 写入 | 单测 |
| A4 | 委派端到端：主 Agent 调 `DelegateToGardener` → 园丁独立请求（system 含规范、工具面正确）→ 报告回灌主 Agent | 单测（fetch stub 编排两次 SSE，照 session-driver 模式） |
| A5 | 园丁 tool 行折叠为嵌套「🌿 园丁整理」组；无标记消息不受影响 | 单测（chat-render） |
| A6 | 园丁写文件走确认卡（带「园丁」徽标）且进 changeLedger | 单测覆盖事件标记与账本累积；卡片视觉归 A7 真机 |
| A7 | 真机全流程：`/整理要素` → 聊天区见园丁逐步工作 → 逐卡确认 → 结构化汇报 → change-summary 面板含园丁改动 | **降级验证**：无浏览器自动化环境，A4 集成测试兜底；交用户真人复核（清单见施工日志） |
| A8 | 全量 `pnpm test` 绿 + core/app typecheck 干净 | 门禁命令 |

## 6. 明确不做（本期诱惑项）

- 主 Agent 自主判断何时委派（Phase 2；本期委派只由用户意图触发）
- 快捷键 / 一键按钮入口（Phase 2）
- 设置页园丁独立确认闸口 / 模型配置（Phase 2，本期全继承）
- `[[要素名]]` 链接解析与 UI 渲染、反向链接、索引页 / 最近变更视图（Phase 3；本期 ELEMENT.md 只按约定**积累**链接文本）
- 「每章写完建议整理」触发评估（Phase 3）
- outputSchema / JSON 强约束汇报（D8：Phase 1 用 persona 规定 Markdown 结构）
- 子代理再嵌套子代理、进程外后端、常驻并行双 Loop（设计 §10 明确不做）
- 园丁改动后自动重建 RAG 索引（维持现状：设置页手动重建；园丁改动经 file-changed 事件刷新文件树）
- `RenameFile` / `DeleteFile` 进园丁白名单（设计拍板：改名合并后旧文件去向列入汇报交用户，园丁不删不改名）

## 7. 新旧机制替代表

| 机制 | 去向 |
| :-- | :-- |
| `/提取要素` 流程（useElementExtraction → element-service） | **并存不替换**：提取 = 从章节初次产出候选；园丁 = 存量库治理。职责边界写进 ELEMENT.md 守则与委派工具描述 |
| 主 Agent 直接编辑 elements/ 的能力 | **并存**：用户控制哲学下主 Agent 仍可写任何文件；园丁只是新增的专职整理路径 |
| PromptList 三分组 | 变四分组（新增「要素规范」），旧三组不动 |
| 斜杠命令二项 | 变三项（新增 `/整理要素`），旧二项不动 |

## 8. 文档同步计划

- 本计划：施工日志随阶段追加（日期 + 阶段 + 提交 hash + 验收结论 + 偏差记录）
- 《要素园丁子代理设计》：状态从「设计定稿，待立施工计划」流转为「Phase 1 施工中」→ 完成后更新
- `docs/plans/README.md`：本计划进「进行中」，结项后转「已完成」
- `docs/project/当前进度.md`：进行中清单同步
- 提交纪律：长任务分段提交，每阶段门禁（测试全绿 + typecheck + 文档同步）过后提交一次；docs 子模块先提交、父仓指针后推进

## 施工日志

### 2026-09-25 阶段 1：ELEMENT.md Schema 层（`3ac5dd2`）

- `DEFAULT_ELEMENT_SCHEMA` 六节默认内容落 `core/project/defaults.ts`；模板分节名与 `templates.ts` 由测试锚定一致（每类首小节名 + worldbuilding 无模板声明）。
- `createProject` 写入 / `repairProject` 缺失补齐（不覆盖已有）/ `readElementSchema` 缺失空串，全套用 NovAI.md 同款策略。
- PromptList 第四分组「要素规范」（🌿 / 徽标「园丁专用」/ 缺失提示「修复项目后自动补齐」）。
- 门禁：全量 446 测试绿 + core/app 双 typecheck 干净。验收点 A1 ✓。
- 偏差：无。

### 2026-09-25 阶段 2：园丁运行壳 + 委派工具（`05c1ea0`）

- 新文件 `core/agent/gardener.ts`：`GARDENER_PERSONA` + `buildGardenerSystemPrompt`（两层，schema 缺失附兜底提示）、`createGardenerTools`（白名单六工具，CreateFile/EditFile 包 validateInput 路径闸，前缀判定大小写不敏感）、`runGardener`（全新 ModelView 复用 query 循环，内层工具事件带 `agent: 'gardener'` 透传）、`createDelegateToGardenerTool`（confirm 包装加 `agentLabel: '园丁'`）、`GARDENER_TASK_PROMPT` 常量。
- 工具名四处联合扩展：`AgentToolName` / `ChatToolName` / `ToolNameView` + `isAgentToolName`；**`CoreToolName` 不扩**（见偏差 1）。
- `AgentRunnableToolMap` 放宽 `Record` → `Partial<Record>`；session 层 `runDriverTurn` 组装全量工具 + 委派工具，提取 `appendToolEventMessage` 统一主/子代理工具事件的转写与账本累积（记录带 agent 归属）。
- 视图层：`ChatMessage`/`ChatMessageView` 的 tool-call/tool-result、`FileChangeRecord`/`FileChangeRecordView`、`FileChangeConfirmationView` 均加可选 agent/agentLabel 字段；agent-service 映射透传。
- 测试 +10：提示词组装、白名单精确、路径闸三向（拒 chapters/、拒 prompts/ELEMENT.md、收 elements/、大小写不敏感）、委派端到端（fetch stub：园丁请求 system 两层断言、工具面断言、汇报回灌、子代理事件标记、外层无标记）、写确认带「园丁」标签、task 空串校验拒绝。验收点 A2/A3/A4/A6（自动化部分）✓。
- 门禁：全量 456 测试绿 + core/app 双 typecheck 干净。
- 偏差：
  1. **`CoreToolName` 未加 `DelegateToGardener`（偏离计划 §3 阶段 2 原文「四处联合类型」→ 实际三处 + CoreToolName 保持不动）。** 原因：`tools/index.ts` 的遗留 core 注册表以 `Record<CoreToolName, …>` 全键约束，委派工具需要 confirm/signal/事件转发闭包、不是 ToolRuntime 可驱动的纯工具，进注册表既不成立也会破坏全键约束；`AgentRunnableTool.core` 以 `AgentToolName` 参数化，类型链不需要动 CoreToolName。为容纳会话层工具，`ToolDefinition` 的 `TName` 约束从 `CoreToolName` 放宽为 `string`（TName 在类型体中仅出现在 name 字段，无行为影响）。对用户无影响，纯类型层取舍。
  2. 委派端到端测试曾因假项目缺 `config` 字段失败（园丁从 `project.config` 取配置）——测试修复，非产品代码问题。

### 2026-09-25 阶段 3：/整理要素 + 聊天区呈现（`2311afa`）

- 斜杠命令注册 `{ id: 'gardener', label: '/整理要素', icon: '🌿' }`；ChatPanel 分支与 `/生成项目记忆` 同款——发送 `GARDENER_TASK_PROMPT`（core 导出，经 agent-service 转出）。
- `chat-render.ts`：新增 `SubagentGroupItem` + `foldSubagentRuns`（轮内连续 `agent === 'gardener'` 工具行折叠为嵌套组）；嵌套组整体算一个过程项；默认展开/收起与外层同规则、覆盖互不干扰；外层组工具计数只计直接行（不与嵌套叠加）。旧消息（无 agent 字段）行为不变，测试锚定。
- `TurnProcessGroup.vue`：支持 `label`（组头「🌿 园丁整理 · N 次工具调用」）与 `nested` 缩进；ChatPanel 模板递归渲染嵌套组。
- `WriteConfirmationCard.vue`：`agentLabel` 徽标（园丁触发的确认卡可辨识请求方）；`TurnChangesPanel.vue`：文件行「园丁」徽标（该文件改动含园丁记录时）。
- 测试 +4（chat-render：嵌套折叠结构/折叠默认与覆盖/旧数据不变/两段不相邻园丁工作段各自成组）。验收点 A5 ✓。
- 门禁：全量 460 测试绿 + core/app 双 typecheck 干净。
- 偏差：无（ToolCallRow 按计划检查后确认无需改动——工具名直显，委派行摘要由 summarizeInput 提供中文文案）。

### 2026-09-25 阶段 4：收尾结项

- 终门禁：全量 460 测试绿 + core/app 双 typecheck 干净 + 生产构建通过（`pnpm build`，1.11s 无错）。
- 文档同步：设计文档状态 → Phase 1 已完成；本计划状态结项 + 本日志；`plans/README.md` 本计划迁「已完成」；`docs/project/当前进度.md` 园丁条目迁已完成段并回改两处「见进行中」引用。
- 验收点对账：A1 ✓（defaults 11 测 + project-fs 初始化/补齐/不覆盖/读取）；A2 ✓（buildGardenerSystemPrompt 两层组装断言）；A3 ✓（白名单精确 + 路径闸三向 + 大小写）；A4 ✓（fetch stub 双请求端到端：园丁 system 两层、工具面、汇报回灌、事件标记）；A5 ✓（chat-render 嵌套折叠 +4）；A6 自动化部分 ✓（事件标记 / 账本 agent 归属 / 确认请求 agentLabel），卡片视觉归 A7；A7 计划内降级（无浏览器自动化环境，A4 兜底），真人复核清单见下；A8 ✓。
- 未验证清单（移交用户真机复核）：
  1. 真实 LLM 下 `/整理要素` 全流程：园丁逐步工作、逐卡确认（确认卡「园丁」徽标）、结构化汇报回聊天、change-summary 面板含园丁改动与「园丁」徽标；
  2. 老项目打开触发 `repairProject` 自动补齐 `prompts/ELEMENT.md`（PromptList 出现「要素规范」组且不覆盖用户版本）；
  3. 嵌套「🌿 园丁整理」组的展开/收起交互、缩进与配色视觉。
- 尾巴（本期不做，去向明确）：园丁改动后 RAG 索引不自动重建（设置页手动重建，§6 明确不做）；汇报 Markdown 结构靠 persona 约束、无 JSON 硬校验（D8 拍板）；PromptList 缺文件时以提示文案兜底（「修复项目后会自动补齐」）；委派深度固定 1 层（设计 §3.2）。

### 2026-09-29 真机复核通过（用户执行）

阶段 4 未验证清单三项经用户真机复核全部通过，未发现问题：

1. 真实 LLM 下 `/整理要素` 全流程：园丁逐步工作、逐卡确认（确认卡「园丁」徽标）、结构化汇报回聊天、change-summary 面板含园丁改动与「园丁」徽标；
2. 老项目打开触发 `repairProject` 自动补齐 `prompts/ELEMENT.md`（PromptList 出现「要素规范」组且不覆盖用户版本）；
3. 嵌套「🌿 园丁整理」组的展开/收起交互、缩进与配色视觉。

本计划验证尾巴全部闭环。

