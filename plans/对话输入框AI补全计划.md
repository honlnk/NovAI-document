# 对话输入框 AI 补全计划

## 记录时间

2026-07-31

## 文档目的

本文档用于规划在 NovAI 对话输入框中加入 Copilot 式实时补全（ghost text）能力。

目标是在用户编写创作指令时，AI 能基于已输入内容预测续写，以灰色文字叠加显示在光标后，按 Tab 接受、继续打字或按 Esc 丢弃，从而减少重复提示词的打字成本。

这是一个探索性功能。产品哲学上，NovAI 的正文由 Agent 通过工具落盘生成、不鼓励用户手打正文；因此本功能的应用场景明确限定为**对话输入框内的提示词编写**，而非小说正文。补全结果作为可选项呈现，不替用户决定创作意图。

## 背景

当前对话输入框（`packages/app/src/components/layout/ChatPanel.vue` 的 `<textarea>`）是一个原生文本域，支持：

- 自适应高度、Enter 发送 / Shift+Enter 换行（含 IME 组合态保护）。
- `/提取要素`、`/生成项目记忆` 斜杠命令（token 触发 → 弹窗 → 清 token → 展开）。
- `@场景` 引用（token 触发 → 弹窗 → 激活场景提示词）。
- 选中内容引用 chip、发送三态按钮。

输入框**没有任何形式的自动补全或 ghost text**。用户每次都要从头手打创作指令。

底层的 LLM 调用已有成熟模式：

- `packages/core/src/core/llm/client.ts` 的 `streamChatCompletion` 是最轻量的 OpenAI 兼容流式客户端（system + user，SSE 解析），但**不支持中断**，且读的是 `choices[0].delta.content`。
- `packages/core/src/core/agent/llm.ts` 的 `streamAgentCompletion` 有项目里唯一成熟的「用户主动 abort vs 网络失败」流式中断实现（fetch 透传 signal + `reader.read()` 用 `Promise.race` 与 signal 竞速 + `isUserAbort` 判定）。
- `packages/core/src/core/rag/rerank.ts` 的门控模式（`if (!config.rerank.enabled) return`）是「可开关 + 独立配置」功能的现成模板。

DeepSeek 提供了 FIM（Fill In the Middle）补全专用接口，支持给定 `prompt`（前缀）和 `suffix`（后缀）补全中间内容，与「续写提示词」的形态对口。

## 目标

1. 用户在对话输入框输入时，停顿后 AI 自动给出续写建议，灰色 ghost text 叠加在光标后。
2. 按 Tab 接受建议——**逐段接受**：每次 Tab 吞掉建议的下一个分词单位（中文用 `Intl.Segmenter` 按词/短语切分，而非硬按空格），再次 Tab 继续吃下一段；继续打字或按 Esc 丢弃剩余建议。
3. 补全使用**独立的模型配置**（仿 rerank：可开关、开启后才允许配置和使用），不影响主模型。
4. 补全走 **DeepSeek FIM 专用接口**（`/completions`，`prompt` + `suffix`）。
5. 触发采用**防抖**（停顿一段时间后才请求），平衡响应性与 API 成本。
6. 支持中断：用户继续打字或主动丢弃时，立即取消未完成的补全请求。
7. 完全可关闭——不开启时输入框行为与现在完全一致。

## 非目标

本计划不实现：

- **小说正文的内联补全**：NovAI 正文由 Agent 落盘生成，没有也不鼓励手写正文的编辑器。本功能只服务于对话输入框的提示词。
- **FIM 接口以外的多引擎支持**：初版只接 DeepSeek FIM。Chat Completion 方案、多厂商自动切换等留待验证可行性后再评估。

## 关键技术约束

1. **FIM 流式 delta 字段是 `choices[0].text`**（legacy completions 协议），**不是**现有 chat 客户端读的 `choices[0].delta.content`。必须新写解析逻辑，并对 `choices[0].text` 与 `choices[0].delta.content` 做双兜底（应对代理网关改写）。
2. **FIM 走 `/completions` 接口**（非 `/chat/completions`），`base_url` 必须带 `/beta`（如 `https://api.deepseek.com/beta`）。
3. **abort 是硬需求**：补全是高频调用，用户继续打字时必须立即取消上一次未完成的请求。照搬 `streamAgentCompletion` 的 fetch-signal + reader 竞速 + isUserAbort 模式。
4. **配置门控照搬 rerank**：`if (!config.completion.enabled) return`，关闭时完全不发起网络请求。
5. **ghost text 覆盖层样式必须与 textarea 精确对齐**：字体、字号、字重、行高、字间距、white-space、word-break、padding、border、border-radius、width 全部一致，否则灰色文字会错位。建议抽共享 CSS class 规避漂移。
6. **中文 IME**：输入法组合态（keyCode 229）期间不应触发补全，沿用现有 `isImeComposing` 判定。
7. **逐段接受的中文分词**：用浏览器原生 `Intl.Segmenter('zh', { granularity: 'word' })` 切分建议，而非硬按空格。中文没有空格边界，按空格会把整句当成一段，失去逐段意义；`Intl.Segmenter` 能按词/短语切出合理单位。NovAI 只跑在 Chromium 内核上，该 API 支持无忧。
8. **延迟预期**：纯前端 + 云 API，补全延迟受网络影响，可能不如原生 Copilot 丝滑。这是探索性尝试的已知代价，通过防抖和短输出（`max_tokens` 限制）缓解。

## 实施计划

### Step 1：补全配置类型与默认值

仿照 rerank 配置，在 core 层新增 `completion` 配置分组。

需要覆盖：

- `packages/core/src/services/types.ts`：`ProjectConfigView` 加 `completion` 字段；导出 `CompletionConfigView`；`ProjectConfigPatch.completion` 设为 `Partial<CompletionConfigView>`。
- `packages/core/src/types/project.ts`：加同构 `CompletionConfig` 类型与 `ProjectConfig.completion` 字段。
- `packages/core/src/core/project/defaults.ts`：`DEFAULT_CONFIG.completion` 默认值。
- `packages/core/src/core/fs/project-fs.ts`：`normalizeProjectConfig` 加 `completion` 分支，用 `{ ...DEFAULT_CONFIG.completion, ...config.completion }` 兜底。

建议的默认值：

```ts
completion: {
  enabled: false,                              // 默认关，需用户主动开启并配置
  baseUrl: 'https://api.deepseek.com/beta',    // FIM 必须 /beta
  apiKey: '',
  model: 'deepseek-chat',
  debounceMs: 600,                             // 防抖时长
  maxTokens: 64,                               // 提示词补全不需要长，控制成本
}
```

### Step 2：FIM 流式客户端

新建 `packages/core/src/core/ai/completion-client.ts`，提供 FIM 补全的流式调用。

结构仿 `core/ai/rerank-client.ts`（职责单一的 HTTP client），SSE 读取仿 `core/llm/client.ts`，abort 模式仿 `core/agent/llm.ts`。

对外签名：

```ts
export type FimCompletionInput = {
  baseUrl: string
  apiKey: string
  model: string
  prompt: string          // 光标前文本（前缀）
  suffix?: string         // 光标后文本（后缀，提示词场景通常为空）
  maxTokens?: number
  signal?: AbortSignal
}

export type FimCompletionEvent =
  | { type: 'start' }
  | { type: 'delta'; text: string }
  | { type: 'finish'; text: string }
  | { type: 'error'; message: string }

export async function streamFimCompletion(
  input: FimCompletionInput,
  onEvent: (event: FimCompletionEvent) => void,
): Promise<string>
```

实现要点：

- **endpoint**：`resolveApiUrl(baseUrl, '/completions')`。
- **请求体**：`{ model, prompt, suffix, stream: true, max_tokens, stop: ['\n'] }`。`stop: ['\n']` 限制补全到当前行/短语，避免长篇输出。
- **delta 提取**：新写 `extractFimDeltaText`，优先读 `choices[0].text`，兜底 `choices[0].delta.content`。**不能复用** `llm/client.ts` 的 `extractDeltaText`（它只读 `delta.content`）。
- **abort**：请求前预检 `signal.aborted`、fetch 透传 `signal`、`reader.read()` 用 `Promise.race` 与 signal 竞速；捕获后用 `isUserAbort` 区分（用户 abort 静默丢弃，网络错误抛出）。
- **import 复用**：`createJsonHeaders`、`extractErrorMessage`、`normalizeBaseUrl`、`readJsonResponse`、`resolveApiUrl` 全部来自 `../ai/shared`。

新建 `packages/core/src/services/completion-service.ts`（service 层转发，仿 `generation-service.ts`）：

- 导出 `streamInlineCompletion(input, onEvent)`，内部做 `if (!enabled) return` 门控，否则调 `streamFimCompletion`。
- 类型 `FimCompletionInputView` / `FimCompletionEventView` 放 `services/types.ts`，与 `LlmStreamInputView` 同风格。
- `packages/core/src/index.ts` 导出新 service 与类型。

### Step 3：前端 composable（防抖 + abort + 状态）

新建 `packages/app/src/composables/useInlineCompletion.ts`。

状态管理仿 `useElementExtraction.ts`（模块级单例 ref），abort 模式仿 chatStore 的 `AbortController` 用法。

职责：

- 暴露 `suggestion`（当前 ghost text）、`isFetching`、`errorMessage` 等响应式状态。
- `scheduleCompletion(fullText, cursor, config)`：清防抖 + abort 上一次，配置未启用则直接返回，否则按 `debounceMs` 延迟发起请求。
- `requestCompletion`：内部持有 `AbortController`，调用 `streamInlineCompletion`，delta 事件累积到 `suggestion`。
- `clearSuggestion()` / `cancelPending()`：清空建议、清防抖定时器、abort 进行中的请求。
- `acceptNextSegment()`：逐段接受的核心。用 `Intl.Segmenter('zh', { granularity: 'word' })` 把当前 `suggestion` 切成段，取首段拼到 `inputText` 末尾、光标后移，剩余部分留作新的 `suggestion`（再次 Tab 继续吃下一段）。段定义为 `Intl.Segmenter` 的一个 `segment`（已自动过滤空白段）。建议在 composable 内懒加载一个 `Intl.Segmenter` 实例复用，避免每次 Tab 都新建。
- `isAbortError` 判定（`DOMException` + `name === 'AbortError'`），abort 时不报错、不清 suggestion（由调用方按场景处理丢弃时机）。

### Step 4：设置 UI（仿 rerank 分组）

`packages/app/src/components/settings/SettingsModal.vue` 改动：

- **新增 tab**：`{ key: 'completion', label: '输入补全' }`，`activeTab` 类型补 `'completion'`。
- **新增 `completionForm` ref**：`{ enabled, baseUrl, apiKey, model, debounceMs, maxTokens }`。
- **回填**：`onMounted` 从 `settingsStore.config.completion` 填充。
- **模板**：1:1 复制 rerank 分组（第 299-356 行）改名——toggle 开关「启用输入补全」+ `v-if="completionForm.enabled"` 条件展开 baseUrl/apiKey/model 输入框与 debounceMs/maxTokens 数字输入。
  - baseUrl 占位符显示 `https://api.deepseek.com/beta`，model 占位符显示 `deepseek-chat`。
- **保存**：`handleSaveCompletion` → `settingsStore.saveConfig`，保存成功后调 `projectStore.updateCurrentProjectConfig(savedConfig)` 同步本地（避免 ChatPanel 读到过期配置，这是项目里已知坑点，见 `changeActiveScenePromptPath` 的处理）。

### Step 5：Ghost text 覆盖层组件

新建 `packages/app/src/components/chat/GhostTextOverlay.vue`——与 textarea 同位的覆盖层 div。

- `absolute inset-0` 覆盖在 textarea 上，`pointer-events: none` 确保点击穿透。
- 渲染：已输入文本（透明，仅为撑布局）+ 灰色建议文本。
- 必须与 textarea 共享完全相同的字体/字号/字重/行高/字间距/white-space/word-break/padding/border/border-radius/width。建议把这些样式抽成共享 CSS class（如 `.chat-input-base`），textarea 和 overlay 共用。
- textarea 在补全激活时文字设为 `color: transparent`（用 class 控制，可开关），`caret-color` 保持深色让光标可见。

### Step 6：ChatPanel 集成

`packages/app/src/components/layout/ChatPanel.vue` 改动：

- **取配置**：`useProjectStore()` 读 `currentProject?.config.completion`（computed）。
- **接入 composable**：`useInlineCompletion()` 拿 `suggestion`、`scheduleCompletion`、`clearSuggestion`。
- **触发**（在 `onTextareaInput` 里加）：输入时调 `scheduleCompletion(inputText, cursor, config)`，内部自带防抖 + abort。命令菜单 / @场景 打开时不调度（避免冲突）。IME 组合态不调度。
- **键盘交互**（在 `handleKeydown` 最前面加，早于其他拦截）：
  - Tab + 有 suggestion：preventDefault，调 `acceptNextSegment()` 吞掉建议的首段（`Intl.Segmenter` 中文分词），剩余部分仍作为 ghost text 显示；建议被吃完后清空、重置高度。
  - Esc + 有 suggestion：preventDefault，清建议。
  - 继续打字：`onTextareaInput` 触发新补全，旧的自动 abort + 清空。
- **发送清理**（在 `handleSend` 里加）：发送前 `clearSuggestion()`。
- **模板**：textarea 外包一层 `relative` 容器，挂载 `GhostTextOverlay`；textarea 样式改用共享 `.chat-input-base`，补全激活且有建议时加 `text-transparent` class。

### Step 7：配置同步验证

确认设置保存后 `projectStore.currentProject.config.completion` 能响应式更新，ChatPanel 的 `completionConfig` computed 能拿到新值，补全功能「配置后立即生效、关闭后立即停止」。

## 验收标准

- 在设置中开启输入补全并配置 DeepSeek 地址/key 后，对话输入框打字停顿后出现灰色 ghost text。
- 按 Tab 能逐段接受建议（每次吞掉一个中文分词单位，多次 Tab 可吃完整条），并插入光标处；按 Esc 能丢弃剩余建议；继续打字时旧建议自动失效。
- 快速连续打字时防抖生效，不会每个字符都发起请求。
- 继续打字时上一次未完成的请求被中断，不产生错乱的建议。
- 关闭开关后，输入框行为完全恢复原状（无 ghost text、文字颜色正常、不发起任何补全请求）。
- 未配置或未开启时，功能完全不介入，不影响现有输入体验。
- IME 输入法组合态期间不触发补全。

## 当前状态

待开始。本计划已明确技术方案与实施步骤，尚未编码。

## 风险与权衡

- **样式对齐漂移**：ghost text 覆盖层与 textarea 的字体/行高必须完全一致，否则灰色文字错位。用共享 CSS class 规避，换字体/主题时需同步检查。
- **FIM 流式字段不确定性**：DeepSeek 文档未明确流式 delta 字段，按 legacy completions 协议实现为 `choices[0].text` 并兜底 `delta.content`。若实测不符需调整。
- **延迟体验**：纯前端 + 云 API，补全延迟受网络影响，可能不如原生 Copilot 丝滑。这是探索性尝试的已知代价。
- **提示词补全的价值边界**：提示词本质是表达创作意图，AI 猜意图可能干扰而非加速。这是本功能作为「探索性、可关闭」特性的根本原因——用开关让用户自行决定是否值得。

## 相关文件索引

调研阶段确认的关键可复用资产：

- `packages/core/src/core/agent/llm.ts`（第 329-358 行 `readStreamChunk`、第 411-425 行 `isUserAbort`）—— abort 模式范本。
- `packages/core/src/core/rag/rerank.ts:11`—— enabled 门控范本。
- `packages/core/src/core/llm/client.ts`（第 109-156 行 SSE 读取）—— SSE 解析范本（但 delta 提取需新写）。
- `packages/core/src/core/ai/rerank-client.ts`—— 独立 HTTP client 结构范本。
- `packages/app/src/components/settings/SettingsModal.vue`（第 299-356 行 rerank 分组）—— toggle + 条件展开 UI 范本。
- `packages/app/src/composables/useElementExtraction.ts`—— composable 状态管理范本。
- `packages/app/src/stores/project.ts`（`updateCurrentProjectConfig`、`currentProject.config`）—— 配置获取与同步入口。
