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

因此，后续建议新增一层稳定的 UI 协作接口层，让 UI 主要依赖这层接口，而不是直接依赖 `core` 的内部模块。

本文档只记录方案，不立即改代码。当前与本任务并行的还有“工具系统升级”任务，二者都可能触及 Agent 与工具调用边界。为避免冲突，接口层先以文档形式冻结方向，等工具系统升级完成后再落地。

---

## 二、核心判断

当前项目已经有：

- 业务模块边界
- 类型雏形
- Agent 事件雏形
- 工具协议雏形
- 项目文件读写能力

但还没有一套真正稳定的“前端协作接口”。

更准确地说：

> `core` 已经像后台服务，但还缺一层面向 UI 的本地 SDK / Application Service。

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

建议后续形成如下结构：

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
├── services/             # 面向 UI 的稳定协作接口层
│   ├── project-service.ts
│   ├── file-service.ts
│   ├── settings-service.ts
│   ├── agent-service.ts
│   ├── rag-service.ts
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

---

## 四、ProjectService

`ProjectService` 负责项目生命周期。

建议接口：

```ts
type ProjectService = {
  createProject(name: string): Promise<ProjectView>
  openProject(): Promise<ProjectView>
  restoreLastProject(): Promise<ProjectView | null>
  forgetLastProject(): Promise<void>
  closeProject(projectId: string): Promise<void>
  refreshProject(projectId: string): Promise<ProjectView>
  inspectProject(projectId: string): Promise<ProjectStatusView>
}
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

---

## 五、FileService

`FileService` 负责项目文件树、文件预览与写入。

建议接口：

```ts
type FileService = {
  listFiles(projectId: string): Promise<ProjectFileNodeView[]>
  readFile(projectId: string, path: string): Promise<FileContentView>
  writeFile(projectId: string, path: string, content: string): Promise<FileContentView>
  refreshFiles(projectId: string): Promise<ProjectFileNodeView[]>
}
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

- `writeFile` 是 UI 主动保存用接口，不等同于 Agent 工具写入。
- Agent 写入应通过 `AgentService` 事件通知 UI，例如 `file-changed`。
- 文件路径统一使用项目内相对路径。

---

## 六、SettingsService

`SettingsService` 负责项目配置读写和模型连接测试。

建议接口：

```ts
type SettingsService = {
  getConfig(projectId: string): Promise<ProjectConfigView>
  updateConfig(projectId: string, patch: ProjectConfigPatch): Promise<ProjectConfigView>
  testLlm(config: LlmConfigView): Promise<ConnectionTestResult>
  testEmbedding(config: EmbeddingConfigView): Promise<ConnectionTestResult>
  testRerank(config: RerankConfigView): Promise<ConnectionTestResult>
}
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

---

## 七、AgentService

`AgentService` 是最关键的协作接口。UI 不应直接知道 `query()`、`runChatTurn()`、底层 `AgentMessage` 或工具执行细节。

建议接口：

```ts
type AgentService = {
  createSession(projectId: string): Promise<ChatSessionView>
  getSession(projectId: string): Promise<ChatSessionView | null>
  runTurn(input: RunAgentTurnInput): Promise<RunAgentTurnResult>
  stopRun(projectId: string, sessionId: string): Promise<void>
}
```

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
  writtenPath?: string
  changedFiles: Array<{
    path: string
    changeType: 'created' | 'updated'
  }>
  session: ChatSessionView
}
```

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
  | { type: 'file-changed'; path: string; changeType: 'created' | 'updated' }
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

建议接口：

```ts
type RagService = {
  getIndexStatus(projectId: string): Promise<ProjectIndexStatusView | null>
  rebuildIndex(projectId: string): Promise<IndexBuildResultView>
  search(projectId: string, query: string): Promise<RetrievalResultView>
}
```

注意当前代码现状：

- 已有基础 Embedding 请求、IndexedDB 向量索引、余弦检索与 Rerank 调试链路。
- 当前尚未真正引入 Orama 依赖。
- 当前 `RagSearch` 尚未接入新的 Agent Loop。

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

## 十二、建议落地顺序

考虑当前还有“工具系统升级”任务并行，建议等工具系统边界稳定后再开始落地接口层。

推荐顺序：

1. 新增 `src/services/types.ts`
   - 定义 `ProjectView`
   - 定义 `FileContentView`
   - 定义 `ChatSessionView`
   - 定义 `AgentUiEvent`
   - 定义 `NovAiError`

2. 新增运行时项目注册表
   - 由 service 内部维护 `projectId -> ProjectSnapshot`
   - UI 不再持有 `FileSystemDirectoryHandle`

3. 实现 `ProjectService`
   - 包装创建、打开、修复、恢复、关闭项目
   - 统一返回 `ProjectView`

4. 实现 `FileService`
   - 包装文件树、文件读取、刷新

5. 实现 `SettingsService`
   - 包装配置读写和模型连接测试

6. 实现 `AgentService`
   - 包装 `runChatTurn`
   - 映射 Agent 内部事件到 `AgentUiEvent`
   - 输出 `file-changed`、`run-finish` 等 UI 事件

7. 让 Pinia store 只调用 service
   - 逐步减少页面直接 import `core/*`

8. 最后再让正式 UI 只依赖 store 或 service
   - 前端展示层不再直接碰内部实现模块

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

建议先完成工具系统升级，再根据最终工具事件模型落地 `AgentService`。

本文档在此之前作为协作记忆，不急于转成代码。

---

## 十四、一句话结论

NovAI 后续需要把 `core` 当作本地后台能力层，把 `services` 当作面向 UI 的稳定接口层。

你可以继续演进 `core`，UI 合作者只依赖 `services/types.ts`、service 方法和 `AgentUiEvent`。这样双方可以并行开发，减少互相踩代码的概率。
