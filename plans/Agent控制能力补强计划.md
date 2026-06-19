# Agent 控制能力补强计划

## 记录时间

2026-06-12（2026-06-19 更新）

## 文档目的

本文档用于收束当前 Agent Loop 中与“可控、可停、可信写入”相关的未完成能力。

NovAI 的产品方向是让 AI 通过工具读写小说项目文件。这个方向成立的前提不是模型能调用工具就够了，而是用户能够清楚知道 Agent 正在做什么，并能在高风险操作前确认、中断或限制工具范围。

## 背景

当前已经完成第一版模型驱动 Agent Loop：

- 模型可调用 `ReadFile / EditFile / CreateFile / RenameFile / DeleteFile / ListDirectory / FindFiles / RagSearch`。
- 工具结果会回灌模型，支持多轮 Query Loop。
- 项目日志会记录模型调用、工具调用和工具结果。
- UI 能展示消息、工具调用和工具结果。
- Agent 已支持运行中停止（LLM 流式请求中止 + Loop 每轮与工具批次执行前检查停止信号，提交 `8feb3a4`）。

但当前仍有几个影响真实创作信任感的问题：

- 写工具会直接执行，缺少写入前确认和 diff 预览。
- 用户即时约束不稳定，例如用户说“不要阅读任何文件”时，模型仍可能调用读取类工具。
- `prompts/system.md` 在同一会话内修改后不会自动刷新到已有 Agent 消息上下文。
- Agent 执行后的 `changedFiles` 由 service 层根据工具调用和工具结果文本推导，不是工具层结构化输出。
- 旧的 `pendingFileChange / confirmPendingFileChange` 流程仍残留，但没有接入当前模型驱动 Agent Loop。

## 目标

补齐 Agent 控制能力，让用户能在正式工作台里可靠地：

1. 看见 Agent 即将修改哪些文件。
2. 在写入前查看摘要或 diff。
3. 确认、拒绝或暂停写入。
4. 在运行中停止 Agent。
5. 对本轮指令设置工具边界。
6. 在提示词变化后让 Agent 使用最新 system prompt。
7. 让 UI 从工具层拿到结构化文件变更，而不是从文本里猜。

## 非目标

本计划不直接实现：

- Git 自动提交和历史查看。
- 完整 transcript、成本统计和 hooks 系统。
- 通用 Bash 工具。
- 完整权限系统的插件化扩展。

这些能力可以在 Agent 控制能力稳定后继续设计。

## 实施计划

### Step 1：结构化文件变更结果 ✅ 已完成（`949d540`，2026-06-20）

让写入类工具直接返回结构化变更信息。

需要覆盖：

- `CreateFile` 返回 `{ type: 'created', path }`
- `EditFile` 返回 `{ type: 'updated', path }`
- `RenameFile` 返回 `{ type: 'renamed', fromPath, toPath }`
- `DeleteFile` 返回 `{ type: 'deleted', path, trashPath }`

完成后，`agent-service` 不再根据工具调用和工具结果文本推导 `changedFiles`。

> 实施说明：`ToolDefinition` 新增可选 `extractFileChange(output)`，4 个写工具各自实现（CreateFile/EditFile/RenameFile/DeleteFile）。结构化 `FileChange` 经 `AgentToolResultMessage.fileChange` 与 `tool-result` 事件透传；`agent-service.collectChangedFiles` 改读结构化字段，删除 `isSuccessfulToolResult`/`extractTrashPath`/`collectToolResultTextById`/`toChangedFile` 文本反推逻辑；`session.lastWrittenPath` 同步改读 `fileChange`。补 `extractFileChange` 与 `tool-execution` 透传 Vitest 测试。`changedFiles` 已完全来自工具层结构化结果，不再依赖工具结果文案。

### Step 2：写入前确认与 diff 预览 ✅ 已完成（`0707809`，2026-06-20）

在执行写工具前生成确认请求。

第一版可以先支持：

- 新建文件：展示目标路径和完整内容预览。
- 编辑文件：展示 `oldText -> newText` 的片段 diff。
- 重命名文件：展示 from / to。
- 删除文件：展示目标路径和回收站说明。

UI 事件使用已经预留的 `confirmation-required`。

> 实施说明：策略为「每次写操作都确认」（用户选定，最安全）。`ToolDefinition` 新增可选 `buildConfirmation(input)`，4 个写工具各自实现，从 `validatedInput` 构造 `WriteConfirmation`（edit/create 带完整文本用于 diff，rename/delete 仅路径级）。`tool-execution` 在 `tool.core.run` 前拦截写工具：调 `confirm` 回调等待用户决定，拒绝时不执行 run 并返回结构化拒绝结果回灌模型（让模型自然调整而非硬中止）；确认中断（停止）按拒绝处理。`agent-service` 新增确认注册表 `confirmationMap` + `respondConfirmation(confirmationId, accepted)`，`confirm` 回调发 `confirmation-required` 事件并 `await` UI 响应，出错/停止时 `rejectPendingConfirmations` 清理未决确认。app 层 chat store 加 `pendingConfirmation` + `confirmWriteTool`/`rejectWriteTool`，新建 `WriteConfirmationCard.vue`（抄 `ExtractionFlowPanel` 预览样式），ChatPanel 挂载。补 tool-execution 确认用例 4 个。

### Step 3：暂停、确认与拒绝 ✅ 已完成（`0707809`/`b0651e7`，2026-06-20）

Agent Loop 在遇到需要确认的写操作时进入等待状态。

需要明确：

- 确认后继续执行当前工具调用。
- 拒绝后把拒绝结果作为 tool result 回灌模型。
- 用户关闭项目或刷新时，待确认请求如何恢复或丢弃。

旧的 `pendingFileChange / confirmPendingFileChange` 可以选择清理，也可以作为参考重接，但不能长期和新 Agent Loop 并存。

> 实施说明：与 Step 2 同一批次落地（同一提交 `0707809`）。拒绝语义采用「回灌模型让其调整」而非硬中止；确认与停止正交——等待确认期间用户仍可点停止，confirm 抛错按拒绝处理，由 query Loop 的 abort 检查接管。旧 `pendingFileChange / confirmPendingFileChange` / `discardPendingFileChange` / `createPendingFileChange` / `callTool` 全套旧残留已清理（`b0651e7`，约 -529 行），连同整个 `chat/tools.ts`（旧 `ToolDefinition.call` 体系）和 `types/chat.ts` 的 `PendingFileChange`/`ToolRuntimeContext`/旧 `ToolDefinition` 类型一并删除。新确认流程完全替代旧代码。

### Step 4：停止运行 ✅ 已完成（`8feb3a4`，2026-06-17）

为 Agent Loop 增加停止能力。

需要覆盖：

- [x] LLM 请求中止（流式请求接入 `AbortSignal`，停止时抛 `AgentAbortedError` 并保留已生成内容，且不触发非流式 fallback）
- [x] 工具批次执行前检查停止信号（剩余工具跳过执行）
- [x] 工具执行完成后停止继续进入下一轮模型调用（Loop 每轮检查停止信号）
- [x] UI 状态从 `running` 回到可输入状态（停止作为正常结束，追加已停止摘要，状态回 `waiting-user`；发送按钮运行中切换为停止）

实现上优先接入 `AbortController`，再补 Query Guard 和队列策略。

> 实施说明：`AbortController` 已落地（chat-store 持有 `AbortController`，`isRunning/isStopping/abortRun` 控制状态）。**Query Guard 与队列策略仍未实现**，后续可继续在 Step 1/2/3 的基础上补齐。

### Step 5：用户即时工具约束 ✅ 已完成（`e774d2c`，2026-06-20）

在每轮用户指令进入 Agent Loop 前识别工具约束。

第一版至少支持：

- 不读文件：禁用 `ReadFile / ListDirectory / FindFiles / RagSearch`。
- 不写文件：禁用 `EditFile / CreateFile / RenameFile / DeleteFile`。
- 只读不写：只启用读取类工具。
- 只操作当前文件：写工具路径必须落在当前 active file。

约束应在工具执行层强制校验，而不是只写进 system prompt。

> 实施说明：采用「软约束 + 硬约束」双层设计，新增 `packages/core/src/core/agent/tool-policy.ts`。`ToolPolicy`（`{ allowRead, allowWrite }`）由 `parseToolPolicy(instruction)` 从用户本轮指令解析——识别「否定词（不要/别/禁止/请勿…）+ 读/写动作词」成对组合（6 字符窗口匹配，覆盖「不要读取文件」「别去查」等间隔表达），以及「不要碰文件/别动文件」这类通用动词的全禁表达。`isToolDisabledByPolicy` 用 `tool.isReadOnly` 二分判定（不硬编码工具名，新工具自动归类），`describePolicyDenial` 生成回灌模型的拒绝说明。**软约束**：`filterAvailableTools` 在 `query()` 入口把被禁工具从发给模型的 schema 列表里移除（源头过滤），`describeActivePolicy` 在 `buildAgentUserContext` 里注入本轮约束声明让模型主动避免。**硬约束**：`executeAgentTool` 在参数校验后、写工具确认流程前检查策略，命中直接返回失败 tool result 回灌模型（不依赖 prompt，模型即使无视也拦得住）。`agent-service.runTurn` 每轮从 `input.instruction` 解析策略并贯穿整条调用链。补 `tool-policy.test.ts` 覆盖读禁/写禁/双重禁/通用动词/肯定句不误触发等场景。
>
> 注：「只读不写」「只操作当前文件」这两条更细的策略未在第一版实现，当前覆盖「不读」与「不写」两类，可按需扩展。

### Step 6：system prompt 刷新 ✅ 已完成（`71c2d42`，2026-06-20）

处理 `prompts/system.md` 同一会话内变更后的刷新问题。

可选方案：

- 每轮运行前比较 system prompt 内容 hash，变化时替换已有 system message。
- 用户保存 system prompt 后重置当前 Agent 会话。
- 在 UI 显示“提示词已更新，是否应用到当前会话”。

第一版建议使用 hash 自动替换，减少用户理解成本。

> 实施说明：采用 hash 自动替换方案（用户无感，最省理解成本）。`session.ts` 的 `runChatTurn` 每轮运行前用 `buildAgentSystemPrompt({ systemPrompt, scenePrompt })` 重新拼出本轮 system 内容，经 `hashContent`（FNV-1a 32 位，与 ReadFileState / RAG indexer 复用同一算法）算出 `newSystemHash`，与 `session.systemPromptHash` 比对——不一致时调 `refreshSystemMessageContent` 原地替换 `agentMessages[0]`（system message 恒在 index 0，query 循环只 push assistant/tool，替换不破坏序列合法性），并把最新 hash 写回 `session.systemPromptHash`。这样 `system.md` 或场景提示词（`.md`）任一变化都会在下一轮自动反映。`ChatSessionState.systemPromptHash` 为新增字段。补 `session.test.ts` 覆盖正常替换、空消息、非 system 首位原样返回等用例。

## 验收标准

- 写入类工具执行前，UI 能收到确认事件。（Step 2，已完成 `0707809`）
- 用户确认后写入生效，拒绝后文件不变且 Agent 能继续解释或调整。（Step 3，已完成 `0707809`/`b0651e7`）
- ✅ Agent 运行中可以停止，停止后不再进入下一轮模型调用。（Step 4，已完成）
- 用户明确说“不读文件”时，读取类工具会被执行层拒绝。（Step 5，已完成 `e774d2c`）
- 用户明确说“不写文件”时，写入类工具会被执行层拒绝。（Step 5，已完成 `e774d2c`）
- 修改 `prompts/system.md` 后，同一会话下一轮能使用最新版提示词。（Step 6，已完成 `71c2d42`）
- `changedFiles` 来自工具层结构化结果，不依赖工具结果文案。（Step 1，已完成 `949d540`）

## 当前状态

**Step 1（结构化文件变更结果）、Step 2/3（写入前确认 + diff 预览 + 暂停/确认/拒绝 + 旧残留清理）、Step 4（停止运行）、Step 5（用户即时工具约束）、Step 6（system prompt 同会话刷新）均已全部完成**。本计划六个 Step 全部落地，Agent 控制能力主线收束。

后续可继续延伸的方向（非本计划范围）：

- Query Guard / 工具队列策略（Step 4 的 AbortController 已落地，更细粒度的队列控制仍可补）。
- 「只读不写」「只操作当前文件」这类更细的工具策略（Step 5 第一版只覆盖「不读」与「不写」两类）。

相关预留已经存在：

- `AgentUiEvent` 中有 `confirmation-required`。
- Chat 状态中有 `awaiting-confirmation`。
- 旧 `pendingFileChange / confirmPendingFileChange` 可作为历史参考。
- `query()` 与工具执行链路已经有事件回调，可扩展控制事件。

