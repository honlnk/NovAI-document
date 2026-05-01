# NovAI UI 协作接口契约设计

## 一、文档目的

本文档记录 NovAI 在多人协作开发时，`core` 层与 UI 层之间应如何划分接口边界。

当前 NovAI 的实现重点集中在 `src/core/`：

- 本地项目文件系统
- Agent Loop
- 文件工具
- RAG 调试链路
- 项目日志
- 最近项目恢复

这些模块已经具备“本地后台服务”的雏形，但它们目前仍偏内部实现 API。后续如果 UI 直接依赖大量 `core/*` 文件，前端展示层会被迫理解底层文件句柄、Agent 内部消息、工具执行细节和日志细节，协作时容易互相影响。

因此，项目已经新增第一版 UI 协作接口层，让 UI 主要依赖这层接口，而不是直接依赖 `core` 的内部模块。

本文档最初用于记录方案；截至 2026-05-01，第一版接口层已经落地到 `src/services/` 与 `src/stores/`，并完成 `SessionTestView`、`TestLabView` 的迁移验证。

当前状态：

- `src/views` 和 `src/stores` 不再直接 import `src/core/*`。
- UI 页面主要通过 Pinia store 使用能力。
- Pinia store 通过 `src/services/*` 调用 core。
- `src/services/types.ts` 已经提供第一版显式 view type / event type。
- `pnpm build` 已通过。

因此，本文档后续同时承担两件事：

1. 记录已落地的 UI 协作接口。
2. 标记仍然属于临时实现或后续演进的部分。

---

## 二、核心判断

当前项目已经有：

- 业务模块边界
- 类型雏形
- Agent 事件雏形
- 工具协议雏形
- 项目文件读写能力

并且已经有了第一版可用的“前端协作接口”。

更准确地说：

> `core` 已经像后台服务，`services` 已经承担第一版面向 UI 的本地 SDK / Application Service。

不过，这一层目前仍是 v0.1 形态：

- 足够支撑 UI 合作者开始接入。
- 足够让测试页不直接依赖 core。
- 还没有完成 Agent 停止、写入确认、会话持久化、结构化工具变更结果等高级能力。

后续 UI 开发者理想上只需要关心：

- 打开项目
- 读取文件树
- 打开文件
- 保存配置
- 发送一轮 Agent 指令
- 订阅 Agent 运行事件
- 展示结果和错误

而不需要知道：

- `FileSystemDirectoryHandle`
- `ProjectSnapshot` 的完整内部结构
- `query()` 如何循环
- `runChatTurn()` 如何拼 prompt
- 工具结果如何回灌模型
- 日志何时写入
- 最近项目如何存在 IndexedDB

---

## 三、建议分层

当前已经形成如下结构：

```txt
src/
├── core/                 # 底层业务能力和内部实现
│   ├── fs/
│   ├── project/
│   ├── agent/
│   ├── chat/
│   ├── tools/
│   ├── rag/
│   └── logging/
│
├── services/             # 面向 UI 的稳定协作接口层（已落地第一版）
│   ├── project-service.ts
│   ├── file-service.ts
│   ├── settings-service.ts
│   ├── agent-service.ts
│   ├── generation-service.ts
│   ├── rag-service.ts
│   ├── element-service.ts
│   ├── mappers.ts
│   ├── project-runtime.ts
│   └── types.ts
│
├── stores/               # Pinia，负责 UI 状态适配
│
└── views/components/     # UI 展示层
```

职责建议：

| 层级 | 职责 |
|:-----|:-----|
| `core` | 提供项目、Agent、工具、RAG、日志等基础能力；允许内部重构 |
| `services` | 组合 `core` 能力，向 UI 暴露稳定动作和事件协议 |
| `stores` | 保存页面状态、loading、当前选择、消息展示数据 |
| `components/views` | 渲染 UI，触发 store 或 service 动作 |

协作约定：

- UI 优先依赖 `services` 和公开 view types。
- Pinia store 可以调用 service，但不应承载复杂业务逻辑。
- UI 不直接依赖 `FileSystemDirectoryHandle`。
- UI 不直接调用 `core/agent/query.ts`、`core/chat/session.ts` 等内部入口。
- `core` 可以继续迭代，但要尽量保持 `services` 的外部合同稳定。

当前迁移验证：

- `SessionTestView` 已迁移为 `projectStore / settingsStore / chatStore`。
- `TestLabView` 已迁移为 store + service 调用。
- 页面层不再直接读取 `FileSystemDirectoryHandle`、`ProjectSnapshot`、`runChatTurn`。

---

## 三点五、当前实现文件索引

当前服务层文件：

| 文件 | 职责 |
|:-----|:-----|
| `src/services/types.ts` | UI 协作接口类型，包含 project、file、settings、agent、RAG、element view types |
| `src/services/project-runtime.ts` | service 内部运行时项目注册表，维护 `projectId -> ProjectSnapshot` |
| `src/services/mappers.ts` | 将 core 内部结构映射为 UI view type |
| `src/services/project-service.ts` | 项目创建、打开、恢复、关闭、刷新、检查 |
| `src/services/file-service.ts` | 文件树、文件读取、章节写入、刷新 |
| `src/services/settings-service.ts` | 配置读写、system prompt 读写、模型连接测试 |
| `src/services/agent-service.ts` | Agent 会话、运行一轮指令、事件映射 |
| `src/services/generation-service.ts` | 流式生成调试接口 |
| `src/services/rag-service.ts` | 索引状态、索引重建、RAG 调试 |
| `src/services/element-service.ts` | 要素提取预览 |
| `src/services/index.ts` | service barrel export |

当前 store 文件：

| 文件 | 职责 |
|:-----|:-----|
| `src/stores/project.ts` | 当前项目、最近项目、当前文件、项目生命周期状态 |
| `src/stores/settings.ts` | 当前项目配置、system prompt、连接测试状态 |
| `src/stores/chat.ts` | Agent 会话视图、运行事件、变更文件、默认目标 |

对 UI 合作者来说，优先使用 store：

```txt
页面 / 组件
-> useProjectStore()
-> useSettingsStore()
-> useChatStore()
```

需要更细的调试能力时，再直接使用 service。

---

## 四、ProjectService

`ProjectService` 负责项目生命周期。

当前接口：

```ts
isProjectAccessSupported(): boolean
createProject(name: string): Promise<ProjectView>
openProject(): Promise<ProjectView>
restoreLastProject(): Promise<ProjectView | null>
getLastProjectSummary(): Promise<LastProjectSummaryView | null>
forgetLastProject(): Promise<void>
closeProject(projectId: string): Promise<void>
refreshProject(projectId: string): Promise<ProjectView>
inspectProject(projectId: string): Promise<ProjectStatusView>
```

UI 面向的项目结构建议使用 `ProjectView`，不要直接暴露完整 `ProjectSnapshot`：

```ts
type ProjectView = {
  id: string
  name: string
  rootName: string
  files: ProjectFileNodeView[]
  config: ProjectConfigView
  activeFilePath?: string
}
```

`ProjectService` 内部可以维护 `projectId -> ProjectSnapshot` 的运行时注册表：

```ts
type ProjectRuntimeRegistry = {
  get(projectId: string): ProjectSnapshot | null
  set(project: ProjectSnapshot): void
  delete(projectId: string): void
}
```

这样 UI 只拿稳定的 `projectId`，底层仍可以保留浏览器目录句柄。

当前注意事项：

- `openProject()` 内部会调用 `repairProject()`，因此打开项目时会温和补齐缺失默认结构。
- UI 不需要再单独展示“修复项目结构”作为常规流程。
- `ProjectSnapshot` 和 `FileSystemDirectoryHandle` 只保留在 service/runtime 内部。

---

## 五、FileService

`FileService` 负责项目文件树、文件预览与写入。

当前接口：

```ts
listFiles(projectId: string): Promise<ProjectFileNodeView[]>
readFile(projectId: string, path: string): Promise<FileContentView>
refreshFiles(projectId: string): Promise<ProjectFileNodeView[]>
writeChapter(projectId: string, fileName: string, markdown: string): Promise<FileContentView>
```

建议 view type：

```ts
type ProjectFileNodeView = {
  name: string
  path: string
  kind: 'file' | 'directory'
  children?: ProjectFileNodeView[]
}

type FileContentView = {
  path: string
  name: string
  format: 'markdown' | 'json' | 'text'
  content: string
  updatedAt: string
}
```

注意：

- 当前暂未暴露通用 `writeFile` 给 UI。
- `writeChapter` 是 Test Lab 调试生成结果落盘用接口。
- Agent 写入应通过 `AgentService` 事件通知 UI，例如 `file-changed`。
- 文件路径统一使用项目内相对路径。

---

## 六、SettingsService

`SettingsService` 负责项目配置读写和模型连接测试。

当前接口：

```ts
getConfig(projectId: string): Promise<ProjectConfigView>
updateConfig(projectId: string, patch: ProjectConfigPatch): Promise<ProjectConfigView>
readSystemPrompt(projectId: string): Promise<string>
writeSystemPrompt(projectId: string, content: string): Promise<void>
testLlm(config: LlmConfigView): Promise<ConnectionTestResultView>
testEmbedding(config: EmbeddingConfigView): Promise<ConnectionTestResultView>
testRerank(config: RerankConfigView): Promise<ConnectionTestResultView>
```

建议 UI 使用 patch 更新配置，而不是每次提交完整 `novel.config.json`：

```ts
type ProjectConfigPatch = Partial<{
  project: Partial<ProjectInfoView>
  llm: Partial<LlmConfigView>
  embedding: Partial<EmbeddingConfigView>
  rerank: Partial<RerankConfigView>
  settings: Partial<ProjectSettingsView>
}>
```

内部实现仍可以整文件覆盖写回配置文件。

当前已经有 `useSettingsStore()` 封装常用 UI 状态：

```ts
const settingsStore = useSettingsStore()

await settingsStore.loadSettings(projectId)
await settingsStore.saveConfig(projectId, patch)
await settingsStore.saveSystemPrompt(projectId, content)
await settingsStore.testLlmConfig(config)
```

---

## 七、AgentService

`AgentService` 是最关键的协作接口。UI 不应直接知道 `query()`、`runChatTurn()`、底层 `AgentMessage` 或工具执行细节。

当前接口：

```ts
deriveTargetFromPath(path?: string | null): ChatTargetView | null
createSession(projectId: string): Promise<ChatSessionView>
getSession(projectId: string): Promise<ChatSessionView | null>
runTurn(input: RunAgentTurnInput): Promise<RunAgentTurnResult>
```

`stopRun()` 尚未落地，后续需要和 `AbortController`、Query Guard、队列策略一起设计。

建议输入：

```ts
type RunAgentTurnInput = {
  projectId: string
  sessionId?: string
  instruction: string
  activeFilePath?: string
  onEvent?: (event: AgentUiEvent) => void
}
```

建议输出：

```ts
type RunAgentTurnResult = {
  projectId: string
  sessionId: string
  targetPath?: string
  changedFiles: ChangedFileView[]
  session: ChatSessionView
}
```

当前文件变更类型：

```ts
type ChangedFileView =
  | { type: 'created'; path: string }
  | { type: 'updated'; path: string }
  | { type: 'renamed'; fromPath: string; toPath: string }
  | { type: 'deleted'; path: string; trashPath?: string }
```

当前限制：

- `changedFiles` 目前由 `agent-service` 根据工具调用和工具结果文本保守推导。
- 后续更稳的做法是让 core 工具层直接返回结构化变更结果。
- 写入确认、暂停恢复、停止运行尚未落地。

---

## 八、Agent UI 事件协议

UI 最应该依赖的是稳定事件，而不是内部 Agent 实现。

建议先定义如下事件合同：

```ts
type AgentUiEvent =
  | { type: 'run-start'; runId: string; sessionId: string }
  | { type: 'message'; message: ChatMessageView }
  | { type: 'assistant-delta'; text: string; fullText: string }
  | { type: 'model-start'; step: number }
  | { type: 'model-finish'; step: number; toolCallCount: number; finishReason?: string }
  | { type: 'tool-call'; toolCall: ToolCallView }
  | { type: 'tool-result'; toolResult: ToolResultView }
  | { type: 'file-changed'; file: ChangedFileView }
  | { type: 'confirmation-required'; request: FileChangeConfirmationView }
  | { type: 'run-error'; error: NovAiError }
  | { type: 'run-finish'; result: RunAgentTurnResult }
```

其中部分事件可以先预留：

- `confirmation-required` 等写工具确认机制落地后启用。
- `file-changed` 在 Agent 写入文件后触发，用于 UI 刷新预览。
- `model-start/model-finish` 用于展示 Query Step。
- `tool-call/tool-result` 用于展示工具轨迹。

建议消息 view type：

```ts
type ChatMessageView =
  | {
      id: string
      role: 'user'
      kind: 'text'
      text: string
      createdAt: string
    }
  | {
      id: string
      role: 'assistant'
      kind: 'text' | 'action-summary'
      text: string
      targetPath?: string
      relatedPaths?: string[]
      createdAt: string
    }
  | {
      id: string
      role: 'system'
      kind: 'tool-call' | 'tool-result' | 'context-summary' | 'error'
      text: string
      ok?: boolean
      toolName?: string
      createdAt: string
    }
```

这层可以和当前 `ChatMessage` 一一映射，但不要求完全暴露内部结构。

---

## 九、RagService

RAG 当前适合先作为调试和状态接口暴露。等 `RagSearch` 正式进入 Agent Loop 后，UI 主要通过 Agent 事件观察检索过程。

当前接口：

```ts
inspectIndex(projectId: string): Promise<ProjectIndexMetaView | null>
rebuildIndex(projectId: string): Promise<IndexBuildResultView>
runRagDebug(projectId: string, query: string): Promise<{
  draft: GenerationContextDraftView
  explanations: RetrievalExplanationView[]
  recalledCount: number
}>
```

注意当前代码现状：

- 已有基础 Embedding 请求、IndexedDB 向量索引、余弦检索与 Rerank 调试链路。
- 当前尚未真正引入 Orama 依赖。
- 当前 `RagSearch` 尚未接入新的 Agent Loop。

---

## 九点五、GenerationService 与 ElementService

这两组接口主要服务 Test Lab 调试能力。

`GenerationService`：

```ts
streamGeneration(
  input: LlmStreamInputView,
  onEvent: (event: LlmStreamEventView) => void,
): Promise<string>
```

`ElementService`：

```ts
previewElementExtraction(input: {
  chapterMarkdown: string
  chapterPath?: string
  systemPrompt?: string
}): Promise<ElementExtractionResultView>
```

当前定位：

- `streamGeneration` 是单次流式生成调试接口，不是 Agent 主工作流。
- 正式创作流程应优先走 `AgentService.runTurn()`。
- `previewElementExtraction` 当前仍是预览/占位能力，后续要和要素写入协议一起完善。

因此接口文档中应避免把“Orama 已落地”作为实现事实。

---

## 十、统一错误模型

当前内部多处使用 `throw new Error(message)`。这适合快速开发，但不利于 UI 做稳定交互。

建议 service 层统一转换为结构化错误：

```ts
type NovAiError = {
  code: NovAiErrorCode
  message: string
  recoverable: boolean
  detail?: unknown
}

type NovAiErrorCode =
  | 'PROJECT_NOT_OPEN'
  | 'PROJECT_PERMISSION_DENIED'
  | 'FILE_NOT_FOUND'
  | 'FILE_WRITE_FAILED'
  | 'MODEL_CONFIG_MISSING'
  | 'MODEL_REQUEST_FAILED'
  | 'TOOL_INPUT_INVALID'
  | 'TOOL_EXECUTION_FAILED'
  | 'RAG_INDEX_EMPTY'
  | 'RAG_REQUEST_FAILED'
  | 'RUN_ABORTED'
  | 'UNKNOWN_ERROR'
```

原则：

- `core` 可以继续抛普通错误。
- `services` 负责把内部错误转换成 `NovAiError`。
- UI 根据 `code` 决定展示 toast、弹窗、重试按钮、授权提示或配置入口。

---

## 十一、状态归属约定

建议约定：

```txt
core:
  负责能力，不负责 UI 展示状态。

services:
  负责把 core 能力组合成 UI 可用动作，并输出稳定 view type / event。

stores:
  负责页面状态，例如当前项目、当前文件、消息列表、运行状态、loading、错误展示。

components:
  负责渲染和用户交互，不直接承载业务流程。
```

典型调用路径：

```txt
UI Component
-> Pinia Store action
-> Service
-> Core
-> Service 映射结果 / 事件
-> Store 更新状态
-> UI 重新渲染
```

---

## 十二、落地状态与下一步

当前已完成：

1. 已新增 `src/services/types.ts`
   - 已定义 `ProjectView`
   - 已定义 `FileContentView`
   - 已定义 `ChatSessionView`
   - 已定义 `AgentUiEvent`
   - 已定义 `NovAiError`

2. 已新增运行时项目注册表
   - `src/services/project-runtime.ts`
   - 由 service 内部维护 `projectId -> ProjectSnapshot`
   - UI 不再持有 `FileSystemDirectoryHandle`

3. 已实现 `ProjectService`
   - 包装创建、打开、恢复、关闭、刷新、检查项目
   - 统一返回 `ProjectView`

4. 已实现 `FileService`
   - 包装文件树、文件读取、刷新、章节写入

5. 已实现 `SettingsService`
   - 包装配置读写、system prompt 读写和模型连接测试

6. 已实现 `AgentService`
   - 包装 `runChatTurn`
   - 映射 Agent 内部消息到 `ChatMessageView`
   - 输出 `file-changed`、`run-finish` 等 UI 事件

7. 已让 Pinia store 调用 service
   - `projectStore`
   - `settingsStore`
   - `chatStore`

8. 已迁移测试页面
   - `SessionTestView`
   - `TestLabView`
   - `src/views`、`src/stores` 当前不再直接 import `src/core/*`

建议下一步：

1. 将 `services/types.ts` 按领域拆分
   - `project-types.ts`
   - `agent-types.ts`
   - `rag-types.ts`
   - `settings-types.ts`

2. 补齐结构化错误转换
   - 当前 `NovAiError` 类型已定义
   - 但多数 service 仍沿用内部普通 Error 或 store 中的 message 兜底

3. 补齐 Agent 停止与写入确认
   - `stopRun`
   - `confirmation-required`
   - 写入 diff 预览
   - 暂停 / 恢复

4. 将工具层结构化变更结果上抛到 `AgentService`
   - 取代当前基于工具文本摘要推导 `changedFiles` 的临时方式

5. 让正式 UI 只依赖 store 或 service
   - 前端展示层继续避免直接碰内部实现模块

---

## 十三、与工具系统升级任务的关系

本接口契约与工具系统升级高度相关，但优先级不同：

- 工具系统升级解决“Agent 能调用哪些工具、工具怎么执行、如何确认写入”。
- UI 协作接口解决“前端如何稳定使用这些能力”。

两者可能冲突的地方：

- `AgentUiEvent.tool-call`
- `AgentUiEvent.tool-result`
- `confirmation-required`
- `file-changed`
- `RunAgentTurnResult.changedFiles`
- `ToolCallView / ToolResultView`

当前工具系统升级已经完成第一轮，`AgentService` 也已经落地第一版。但以下接口仍会继续受工具系统演进影响：

- `ToolCallView`
- `ToolResultView`
- `ChangedFileView`
- `confirmation-required`
- `file-changed`
- `RunAgentTurnResult.changedFiles`

因此，UI 可以先使用这些接口做展示，但不要把它们视为最终协议。

---

## 十四、一句话结论

NovAI 当前已经开始把 `core` 当作本地后台能力层，把 `services` 当作面向 UI 的稳定接口层。

你可以继续演进 `core`，UI 合作者优先依赖 `stores`，必要时依赖 `services/types.ts`、service 方法和 `AgentUiEvent`。这样双方可以并行开发，减少互相踩代码的概率。
