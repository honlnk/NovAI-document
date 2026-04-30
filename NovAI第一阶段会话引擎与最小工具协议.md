# NovAI 第一阶段会话引擎与最小工具协议

## 一、文档目的

本文档用于回答一个更接近实现的问题：

> 在明确 NovAI 要走“可视化小说写作智能体”路线后，第一阶段的会话引擎和工具协议到底应该长什么样？

这份文档不追求覆盖最终完整形态，只定义：

- 第一阶段会话引擎的最小目标
- 第一阶段消息模型
- 第一阶段 Agent Loop
- 第一阶段最小工具协议
- 当前预览对象作为默认操作目标的规则
- `RagSearch` 作为 NovAI 特有工具时的接入方式

相关文档：

- [Claude Code 借鉴与映射设计](ClaudeCode借鉴与映射设计.md)
- [技术架构设计](技术架构设计.md)
- [开发前最小契约文档](开发前最小契约文档.md)
- [UI设计文档](UI设计文档.md)

当前实现状态：

- 已新增 `src/core/agent/` 作为第一版 Agent Loop 核心层
- 已新增 `src/core/tools/` 作为项目文件工具底层实现
- 已接入 OpenAI-compatible `tools / tool_calls`
- 已实现 `ReadFile / EditFile / CreateFile / RenameFile / DeleteFile / ListDirectory / FindFiles` 项目文件工具
- 已实现项目级运行日志 `.novel/logs/agent.log.jsonl`
- 已实现最近项目 IndexedDB 记忆与恢复

本文档中未完成能力仍按“目标设计”理解，已完成能力以当前实现状态为准。

---

## 二、第一阶段的真实目标

第一阶段不要试图一口气做成完整版 Claude Code。

第一阶段真正要完成的是这条链路：

1. 用户在对话区输入自然语言意图
2. 系统结合“当前预览目标”判断默认操作对象
3. AI 以工具方式读取目标文件与必要上下文
4. AI 在必要时调用 `RagSearch`
5. AI 返回结构化结果
6. 系统将结果写回本地 Markdown 文件
7. 内容区更新预览，对话区显示操作摘要

也就是说，第一阶段不是“做一整套万能智能体平台”，而是：

> 让 NovAI 先具备一个能围绕当前小说文件真正工作的最小会话智能体。

---

## 三、第一阶段不做什么

为了压住复杂度，第一阶段明确不做：

- 多 agent
- 计划模式 / Plan mode
- 通用 slash commands
- MCP
- LSP
- IDE 桥接
- 复杂权限系统
- 后台任务系统
- 自由组合任意工具链
- 完整通用对话压缩体系

第一阶段只做一个“单会话、单主线程、围绕当前小说项目文件工作”的 agent。

---

## 四、会话引擎的定位

### 4.1 会话引擎是什么

NovAI 的会话引擎不是简单的“发送聊天请求”函数。

它应当是一个长期存在于当前项目工作区中的对象，负责：

- 保存当前会话消息
- 保存当前会话状态
- 接收用户输入
- 根据 UI 当前状态组装默认上下文
- 驱动 AI 调用工具
- 接收工具结果并继续推进
- 输出对话摘要与文件变更结果

### 4.2 一句话定义

第一阶段会话引擎可以定义为：

> 一个围绕“当前项目 + 当前预览对象 + 最小工具集”运作的单线程小说写作 Agent Loop。

---

## 五、第一阶段会话范围

第一阶段建议把会话范围控制为“当前项目中的一个主工作会话”。

### 5.1 会话粒度

建议：

- 一个打开中的小说项目，对应一个主会话
- 这个主会话持续存在，直到用户关闭项目或刷新页面

### 5.2 会话保存策略

第一阶段先不强求复杂持久化，可以先采用：

- 页面内存中保存当前消息列表
- 刷新丢失可接受
- 后续再补会话恢复

### 5.3 为什么先这样做

因为第一阶段最重要的是验证：

- 用户是否愿意以“当前预览文件 + 对话驱动”的方式写作
- AI 是否能真正围绕项目文件工作
- 工具调用链路是否稳定

不是先把“历史恢复、跨会话连续性、复杂记忆系统”做满。

---

## 六、第一阶段消息模型

NovAI 第一阶段就不应该只保留：

- `instruction`
- `result`

而应引入最小结构化消息模型。

### 6.1 最小消息类型

第一阶段建议至少支持以下消息类型：

```ts
type ChatMessage =
  | UserTextMessage
  | AssistantTextMessage
  | AssistantActionSummaryMessage
  | ToolCallMessage
  | ToolResultMessage
  | ErrorMessage
  | ContextSummaryMessage
```

### 6.2 用户消息

```ts
type UserTextMessage = {
  id: string
  role: 'user'
  kind: 'text'
  text: string
  createdAt: string
}
```

用途：

- 保存用户原始意图
- 作为会话推进的触发点

### 6.3 AI 普通文本消息

```ts
type AssistantTextMessage = {
  id: string
  role: 'assistant'
  kind: 'text'
  text: string
  createdAt: string
}
```

用途：

- AI 给用户的说明
- 澄清问题
- 非工具类分析结论

### 6.4 AI 操作摘要消息

```ts
type AssistantActionSummaryMessage = {
  id: string
  role: 'assistant'
  kind: 'action-summary'
  summary: string
  targetPath?: string
  relatedPaths?: string[]
  createdAt: string
}
```

用途：

- 告诉用户“做了什么”
- 不承担正文展示职责

示例：

- 已修改 `chapters/第003章-密境遇险.md`
- 已保存 `prompts/scenes/scene-001.md`
- 已为本次修改补充 4 条相关要素上下文

### 6.5 工具调用消息

```ts
type ToolCallMessage = {
  id: string
  role: 'system'
  kind: 'tool-call'
  toolName: 'ReadFile' | 'EditFile' | 'CreateFile' | 'RenameFile' | 'DeleteFile' | 'ListDirectory' | 'FindFiles' | 'RagSearch'
  inputSummary: string
  createdAt: string
}
```

用途：

- 给 UI 一个显示“AI 正在做什么”的钩子
- 形成执行轨迹

### 6.6 工具结果消息

```ts
type ToolResultMessage = {
  id: string
  role: 'system'
  kind: 'tool-result'
  toolName: 'ReadFile' | 'EditFile' | 'CreateFile' | 'RenameFile' | 'DeleteFile' | 'ListDirectory' | 'FindFiles' | 'RagSearch'
  ok: boolean
  resultSummary: string
  createdAt: string
}
```

用途：

- 记录工具执行结果
- 支撑调试和用户可见摘要

### 6.7 错误消息

```ts
type ErrorMessage = {
  id: string
  role: 'system'
  kind: 'error'
  message: string
  recoverable: boolean
  createdAt: string
}
```

### 6.8 上下文摘要消息

```ts
type ContextSummaryMessage = {
  id: string
  role: 'system'
  kind: 'context-summary'
  summary: string
  createdAt: string
}
```

用途：

- 第一阶段先可用于显示本轮组装了哪些上下文
- 未来再演进为压缩后的会话摘要

---

## 七、第一阶段会话状态

### 7.1 最小会话状态

```ts
type ChatSessionState = {
  sessionId: string
  projectId: string
  messages: ChatMessage[]
  status: 'idle' | 'running' | 'waiting-user' | 'error'
  currentDraftText: string
  currentTarget: ChatTargetContext | null
  lastRagResult: RagSearchResult | null
}
```

### 7.2 当前目标上下文

这是 NovAI 和 Claude Code 最大的差异之一。

```ts
type ChatTargetContext = {
  type: 'chapter' | 'prompt-system' | 'prompt-scene' | 'element' | 'project'
  primaryPath?: string
  groupName?: string
  displayName: string
  derivedFrom: 'preview' | 'selection' | 'explicit-user-intent'
}
```

### 7.3 这层状态的意义

AI 不应总是从零猜用户想操作哪个文件。

系统应该先把当前 UI 中最明确的上下文转成结构化状态，再交给会话引擎使用。

---

## 八、当前预览对象作为默认操作目标

这是第一阶段最关键的体验设计之一。

### 8.1 规则

若用户没有明确指定文件路径，则：

- 当前预览的章节文件，视为默认目标
- 当前预览的提示词文件，视为默认目标
- 当前预览的要素文件，视为默认目标

### 8.2 目标判定优先级

建议采用以下优先级：

1. 用户在本轮消息中明确指定的文件或对象
2. 当前内容区正在预览的文件
3. 当前打开的文件分组
4. 项目级默认目标

### 8.3 示例

例 1：

- 当前预览：`chapters/第003章-密境遇险.md`
- 用户输入：“把结尾改得更紧张一些”

系统默认目标：

- `type = chapter`
- `primaryPath = chapters/第003章-密境遇险.md`

例 2：

- 当前预览：`prompts/scenes/scene-001.md`
- 用户输入：“这个场景提示词要更压抑一点”

系统默认目标：

- `type = prompt-scene`
- `primaryPath = prompts/scenes/scene-001.md`

### 8.4 对会话引擎的影响

会话引擎在每次提交用户输入前，应先接收这份 `ChatTargetContext`，并把它作为本轮上下文的一部分。

---

## 九、第一阶段 Agent Loop

### 9.1 最小循环

第一阶段不要做开放式无限循环，而采用一个受控的最小循环：

1. 用户输入意图
2. 系统生成本轮默认上下文
3. AI 决定需要调用哪些工具
4. 工具执行
5. AI 基于工具结果产出结果
6. 若需要落地则写文件
7. 输出操作摘要
8. 结束本轮

### 9.2 一轮中的建议阶段

```txt
用户输入
-> 目标判定
-> 上下文组装
-> 模型决策
-> 工具调用
-> 结果整合
-> 文件落地
-> UI 更新
-> 输出摘要
```

### 9.3 上下文组装内容

第一阶段建议组装这些输入：

- 当前用户输入
- 当前目标上下文
- `prompts/system.md`
- 若命中场景提示词，则补充对应 `scene prompt`
- 目标文件内容摘要或全文
- 必要时的 `RagSearch` 结果
- 必要时的近期章节上下文

### 9.4 何时调用 `RagSearch`

建议不是每轮强制调用，而是按规则触发：

- 当前任务与剧情、人物、设定相关
- 当前任务是章节生成或章节修改
- 当前任务可能依赖故事背景一致性

对以下情况可跳过：

- 纯提示词措辞修改
- 纯格式性修改
- 明显局部的文案润色

---

## 十、第一阶段最小工具协议

### 10.1 设计原则

工具协议要尽量简单，但要保留后续扩展空间。

建议每个工具至少具有：

- 名称
- 输入 schema
- 输出 schema
- 执行函数
- 用户可见摘要函数

### 10.2 最小定义

```ts
type ToolDefinition<TInput, TOutput> = {
  name: string
  description: string
  validateInput: (input: unknown) => TInput
  call: (input: TInput, context: ToolRuntimeContext) => Promise<TOutput>
  summarizeInput: (input: TInput) => string
  summarizeOutput: (output: TOutput) => string
}
```

### 10.3 运行时上下文

```ts
type ToolRuntimeContext = {
  project: ProjectSnapshot
  target: ChatTargetContext | null
  session: ChatSessionState
}
```

第一阶段不需要把权限系统做得很重，但需要保证工具知道：

- 当前项目
- 当前目标
- 当前会话

---

## 十一、第一阶段工具定义

当前第一版已实现七个项目文件工具：

- `ReadFile`
- `EditFile`
- `CreateFile`
- `RenameFile`
- `DeleteFile`
- `ListDirectory`
- `FindFiles`

`Bash` 已明确不再作为 NovAI 工具开发目标。`RagSearch` 保留为后续正式接入 Agent Loop 的扩展目标。

## 11.1 ReadFile

### 输入

```ts
type ReadFileInput = {
  path: string
  offset?: number
  limit?: number
}
```

### 输出

```ts
type ReadFileOutput = {
  path: string
  content: string
  numberedContent: string
  startLine: number
  endLine: number
  totalLines: number
  truncated: boolean
  empty: boolean
  offsetBeyondEnd: boolean
  fileSizeBytes: number
  notice?: string
}
```

### 用途

- 读取默认目标文件
- 读取补充上下文文件
- 为 `EditFile` 提供精确原文

### 规则

- 优先用于 Markdown / JSON / TXT
- 输出带行号内容，方便模型定位
- 可通过 `offset / limit` 分段读取长文件
- 第一阶段无需支持复杂二进制文件

## 11.2 CreateFile

### 输入

```ts
type CreateFileInput = {
  path: string
  content: string
}
```

### 输出

```ts
type CreateFileOutput = {
  path: string
  contentLength: number
  linesAdded: number
  created: true
}
```

### 用途

- 新建章节
- 新建提示词
- 新建要素文件

### 规则

- 目标文件已存在时失败
- 父目录不存在时自动创建
- 用于明确“新建”语义，不承担覆盖已有文件职责

## 11.3 EditFile

### 输入

```ts
type EditFileInput = {
  path: string
  oldText: string
  newText: string
  replaceAll?: boolean
}
```

### 输出

```ts
type EditFileOutput = {
  path: string
  occurrences: number
  contentLength: number
  linesAdded: number
  linesRemoved: number
}
```

### 当前实现

当前已从“整体覆盖”升级为精确文本替换：

- `oldText` 必须来自 `ReadFile` 返回内容，不应包含行号前缀
- 如果匹配多处且未启用 `replaceAll`，工具会失败，并提示使用相邻行组成唯一片段
- `oldText === newText` 会失败
- 空 `oldText` 会失败，新建文件请使用 `CreateFile`
- 对直引号和中文弯引号做了兼容处理

这比整体覆盖更适合后续做 diff、权限确认和局部修改预览。

## 11.4 RenameFile

用于重命名或移动工作区内单个文本文件。

### 输入

```ts
type RenameFileInput = {
  fromPath: string
  toPath: string
}
```

### 输出

```ts
type RenameFileOutput = {
  fromPath: string
  toPath: string
  contentLength: number
}
```

### 使用边界

- `fromPath` 必须存在
- `toPath` 不能已存在
- `toPath` 父目录不存在时自动创建
- 只支持 `.md / .json / .txt`
- 不允许操作 `novel.config.json` 或 `.novel/**`

## 11.5 DeleteFile

用于把工作区内单个文本文件移入回收站。

### 输入

```ts
type DeleteFileInput = {
  path: string
}
```

### 输出

```ts
type DeleteFileOutput = {
  path: string
  trashPath: string
  contentLength: number
  linesRemoved: number
}
```

### 使用边界

- `path` 必须存在
- 只支持 `.md / .json / .txt`
- 不直接永久删除，而是移动到 `.novel/trash`
- 不允许操作 `novel.config.json` 或 `.novel/**`
- 用户意图不明确时，不应主动删除文件

## 11.6 ListDirectory

用于查看工作区内某个已存在目录的直接子项，不读取文件正文。

```ts
type ListDirectoryInput = {
  path?: string
  showHidden?: boolean
}
```

```ts
type ListDirectoryOutput = {
  path: string
  entries: Array<{
    name: string
    path: string
    kind: 'file' | 'directory'
    hidden: boolean
  }>
}
```

## 11.7 FindFiles

用于按文件名或路径模式查找工作区内文件，不读取文件正文。

```ts
type FindFilesInput = {
  pattern: string
  path?: string
  includeHidden?: boolean
  limit?: number
}
```

```ts
type FindFilesOutput = {
  pattern: string
  path: string
  filenames: string[]
  numFiles: number
  truncated: boolean
}
```

## 11.8 RagSearch

### 输入

```ts
type RagSearchInput = {
  query: string
  topK: number
  filters?: {
    type?: Array<'character' | 'location' | 'timeline' | 'plot' | 'worldbuilding'>
    lastUpdatedChapter?: string
  }
}
```

### 输出

```ts
type RagSearchHit = {
  id: string
  sourcePath: string
  type: 'character' | 'location' | 'timeline' | 'plot' | 'worldbuilding'
  name: string
  summary: string
  score?: number
}

type RagSearchOutput = {
  hits: RagSearchHit[]
}
```

### 用途

- 为生成章节提供背景
- 为修改章节补充上下文
- 为设定相关修改提供故事事实支持

### 说明

这是 NovAI 区别于 Claude Code 的核心能力之一，因此必须从第一阶段就进入工具协议，而不是只停留在测试按钮层。

当前 `RagSearch` 暂未接入新的 Agent Loop，后续应作为模型可主动调用的正式工具加入 `src/core/agent/tools.ts`。

---

## 十二、第一阶段模型输入格式建议

第一阶段无需完全复刻 Claude Code 的复杂消息协议，但建议模型输入至少包含以下几类信息：

### 12.1 系统层指令

包括：

- 内置系统提示词模板
- `prompts/system.md`
- 当前工作模式说明
- 工具使用规则

### 12.2 当前回合上下文

包括：

- 用户输入
- 当前目标文件信息
- 当前内容区预览对象
- 可用工具列表

### 12.3 文件与检索上下文

包括：

- 当前目标文件内容
- 补充提示词内容
- `RagSearch` 召回结果
- 近期章节（按需）

### 12.4 输出预期

第一阶段建议要求模型在每轮中明确返回以下两类之一：

1. 需要调用工具
2. 已完成本轮任务并给出结果摘要

这有利于后续把执行链路稳定下来。

---

## 十三、第一阶段推荐的工作流模板

## 13.1 修改章节

```txt
用户说：把这一章结尾改得更悲壮一些
-> 系统发现当前预览目标是 chapter
-> ReadFile(当前章节)
-> RagSearch(与当前章节结尾相关的人物/情节要素)
-> 模型生成局部修改方案
-> EditFile(精确替换章节片段)
-> 内容区刷新预览
-> 对话区显示“已修改章节”摘要
```

## 13.2 修改提示词

```txt
用户说：把这个场景提示词改得更有压迫感
-> 系统发现当前预览目标是 prompt-scene
-> ReadFile(当前 scene prompt)
-> 模型生成局部修改方案
-> EditFile(精确替换提示词片段)
-> 内容区刷新预览
-> 对话区显示“已修改提示词”摘要
```

## 13.3 生成新章节

```txt
用户说：写下一章，主角在废墟中发现旧王朝密卷
-> 系统识别为生成任务
-> 读取 system prompt / 相关 scene prompt
-> 读取近期章节
-> RagSearch(人物 / 地点 / 情节 / 世界观)
-> 模型生成章节内容
-> CreateFile(新章节)
-> 后续触发要素提取与索引更新
-> 内容区切换为新章节预览
-> 对话区显示生成摘要
```

---

## 十四、当前落地状态与下一步

### 14.1 已落地

当前已经完成：

1. 建立 `stores/chat.ts`
2. 建立最小消息类型
3. 建立 `src/core/tools/` 文件工具层
4. 建立 `src/core/agent/` Agent Loop 层
5. 将当前预览文件接入 `ChatTargetContext`
6. 将 Session Lab 接入新的 `query()` 循环
7. 建立项目级日志系统
8. 建立最近项目恢复能力

### 14.2 下一步建议

下一批建议按这个顺序推进：

1. 写工具权限确认
   在 `EditFile / CreateFile` 真正执行前暂停 Query，让用户确认或拒绝。

2. Query Guard / AbortController
   避免重复提交，支持用户停止正在运行的 Agent。

3. 工具调用分组与 Query Step UI
   让用户能看到每轮模型调用和工具批次。

4. 把 `RagSearch` 接入会话引擎
   不再只是测试按钮

5. 要素提取与索引更新
   让生成或修改章节后的结构化知识进入 `elements/` 与后续检索链路。

---

## 十五、第一阶段成功标准

如果以下体验已经成立，就说明第一阶段方向是对的：

1. 用户无需描述精确文件路径，也能修改当前预览文件
2. AI 修改章节时，能够自动读取该章节而不是盲改
3. AI 在需要时能够自动补充相关要素上下文
4. 对话区展示的是操作摘要，而不是大段正文堆积
5. 文件区始终是正文和提示词的主要呈现位置
6. 用户能明显感受到“AI 在操作项目文件”，而不是“AI 在聊天窗口里写作文”

---

## 十六、一句话结论

NovAI 第一阶段的会话引擎，不应该是“聊天接口的 UI 包装”，而应该是一个围绕当前小说项目、当前预览目标和最小工具集运作的写作 Agent Loop。
