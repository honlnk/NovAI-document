# Agent 控制能力补强计划

## 记录时间

2026-06-12

## 文档目的

本文档用于收束当前 Agent Loop 中与“可控、可停、可信写入”相关的未完成能力。

NovAI 的产品方向是让 AI 通过工具读写小说项目文件。这个方向成立的前提不是模型能调用工具就够了，而是用户能够清楚知道 Agent 正在做什么，并能在高风险操作前确认、中断或限制工具范围。

## 背景

当前已经完成第一版模型驱动 Agent Loop：

- 模型可调用 `ReadFile / EditFile / CreateFile / RenameFile / DeleteFile / ListDirectory / FindFiles / RagSearch`。
- 工具结果会回灌模型，支持多轮 Query Loop。
- 项目日志会记录模型调用、工具调用和工具结果。
- UI 能展示消息、工具调用和工具结果。

但当前仍有几个影响真实创作信任感的问题：

- 写工具会直接执行，缺少写入前确认和 diff 预览。
- Agent 运行中无法由用户主动停止。
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

### Step 1：结构化文件变更结果

让写入类工具直接返回结构化变更信息。

需要覆盖：

- `CreateFile` 返回 `{ type: 'created', path }`
- `EditFile` 返回 `{ type: 'updated', path }`
- `RenameFile` 返回 `{ type: 'renamed', fromPath, toPath }`
- `DeleteFile` 返回 `{ type: 'deleted', path, trashPath }`

完成后，`agent-service` 不再根据工具调用和工具结果文本推导 `changedFiles`。

### Step 2：写入前确认与 diff 预览

在执行写工具前生成确认请求。

第一版可以先支持：

- 新建文件：展示目标路径和完整内容预览。
- 编辑文件：展示 `oldText -> newText` 的片段 diff。
- 重命名文件：展示 from / to。
- 删除文件：展示目标路径和回收站说明。

UI 事件使用已经预留的 `confirmation-required`。

### Step 3：暂停、确认与拒绝

Agent Loop 在遇到需要确认的写操作时进入等待状态。

需要明确：

- 确认后继续执行当前工具调用。
- 拒绝后把拒绝结果作为 tool result 回灌模型。
- 用户关闭项目或刷新时，待确认请求如何恢复或丢弃。

旧的 `pendingFileChange / confirmPendingFileChange` 可以选择清理，也可以作为参考重接，但不能长期和新 Agent Loop 并存。

### Step 4：停止运行

为 Agent Loop 增加停止能力。

需要覆盖：

- LLM 请求中止。
- 工具批次执行前检查停止信号。
- 工具执行完成后停止继续进入下一轮模型调用。
- UI 状态从 `running` 回到可输入状态。

实现上优先接入 `AbortController`，再补 Query Guard 和队列策略。

### Step 5：用户即时工具约束

在每轮用户指令进入 Agent Loop 前识别工具约束。

第一版至少支持：

- 不读文件：禁用 `ReadFile / ListDirectory / FindFiles / RagSearch`。
- 不写文件：禁用 `EditFile / CreateFile / RenameFile / DeleteFile`。
- 只读不写：只启用读取类工具。
- 只操作当前文件：写工具路径必须落在当前 active file。

约束应在工具执行层强制校验，而不是只写进 system prompt。

### Step 6：system prompt 刷新

处理 `prompts/system.md` 同一会话内变更后的刷新问题。

可选方案：

- 每轮运行前比较 system prompt 内容 hash，变化时替换已有 system message。
- 用户保存 system prompt 后重置当前 Agent 会话。
- 在 UI 显示“提示词已更新，是否应用到当前会话”。

第一版建议使用 hash 自动替换，减少用户理解成本。

## 验收标准

- 写入类工具执行前，UI 能收到确认事件。
- 用户确认后写入生效，拒绝后文件不变且 Agent 能继续解释或调整。
- Agent 运行中可以停止，停止后不再进入下一轮模型调用。
- 用户明确说“不读文件”时，读取类工具会被执行层拒绝。
- 用户明确说“不写文件”时，写入类工具会被执行层拒绝。
- 修改 `prompts/system.md` 后，同一会话下一轮能使用最新版提示词。
- `changedFiles` 来自工具层结构化结果，不依赖工具结果文案。

## 当前状态

尚未开始实现。

相关预留已经存在：

- `AgentUiEvent` 中有 `confirmation-required`。
- Chat 状态中有 `awaiting-confirmation`。
- 旧 `pendingFileChange / confirmPendingFileChange` 可作为历史参考。
- `query()` 与工具执行链路已经有事件回调，可扩展控制事件。

