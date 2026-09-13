# Agent Loop 重构开发计划

## 记录时间

2026-09-14

## 文档目的与使用方式

本文档是 **NovAI Agent Loop 重构的可执行施工计划**，用于驱动后续开发（可能由能力较弱的模型或分期推进）。

**给执行者的三条元规则：**

1. **本计划是「按阶段推进、每阶段独立验收」的**。每个阶段（Stage）末尾都有「验收」小节，**必须全部通过后才能进入下一阶段**。不要跨阶段并行开发，不要在没有通过当前阶段验收的情况下开始下一阶段。
2. **遇到与计划冲突的现实，停下来报告，不要擅自改方向**。计划里标注了「⚠️ 决策点」的地方是允许你提出异议的；其余地方如果你发现代码现状与计划描述不符（比如文件已改名、函数签名已变），**以代码现状为准继续，但在交付说明里显式记录这处偏差**。
3. **每个阶段是一个独立的、可提交的变更集**。完成后运行 `pnpm test`（即 `vitest run`，见根 `package.json`）+ 手动验收，再提交。

**前置阅读（必读，按此顺序）：**

1. [core功能梳理与冗余问题诊断](./core功能梳理与冗余问题诊断.md) —— 知道哪些要删、为什么
2. [参考系切换：Claude Code 转向 DeepSeek Harness](./参考系切换-ClaudeCode转向DeepSeekHarness.md) —— 知道往哪个形状写
3. 本计划

**总目标：** 把 NovAI 的 Agent Loop 从「无流式、无压缩、轮次硬顶、消息双结构、每次写都确认」重构为「流式渲染、自动压缩、范围权限、单一消息源」的现代形态。设计蓝本是 dsh，但**只抄形状，不抄框架与存储**。

**总非目标（这些明确不做，避免范围蔓延）：**

- ❌ 不引入 Cordis 或任何插件框架
- ❌ 不引入事件溯源存储（append-only 事件日志 + JSONL 投影）
- ❌ 不做多协议模型适配（anthropic/gemini 生成链路）
- ❌ 不做 subagent / 多智能体 / plan mode
- ❌ 不重写工具层、RAG 层、要素提取层（它们是好的，保留）
- ❌ 不改变 FIM 输入补全（它是独立链路，保留原样）

---

## 全局背景：现状与目标对照

| 维度 | 现状（病灶） | 目标（本计划交付后） |
| :--- | :--- | :--- |
| 流式渲染 | `query.ts:84` 丢弃 delta，UI 整段蹦出 | 打字机式实时渲染 |
| 上下文管理 | `agentMessages` 无限累积、全量重发 | 阈值触发自动压缩，摘要替换旧区间 |
| 写入安全 | 每次写操作都强制确认 | 项目工作区内静默放行，越界才确认 |
| 长文本 | 4000 字章节整段塞上下文 | 超长结果 spill 落盘，上下文放预览+引用 |
| 消息模型 | ChatMessage（7 类展示）与 AgentMessage（4 类模型）双结构手工同步 | 显示层 ChatMessage 保留；模型层独立维护「消息 + 压缩替换表」 |
| 轮次上限 | `DEFAULT_MAX_TURNS=8` 写死 | 可配置，超限优雅收尾 |
| 约束解析 | regex tool-policy（已决定删除） | 删除，用「范围权限」替代 |

---

## Stage 0：清理（前置，独立提交）

> **为什么先做：** 在干净的代码基线上重构，避免把要删的东西一起重构进去。这一阶段**不改任何运行时行为**，纯删除。

### 范围与动作

**0.1 删除 tool-policy 全链路**（最高优先，诊断文档 2.4 已定案）

按以下清单逐项删除，每删一处跑相关测试：

- 删除文件 `packages/core/src/core/agent/tool-policy.ts` 及其测试 `tool-policy.test.ts`
- `packages/core/src/core/agent/query.ts`：删除 `filterAvailableTools` 调用与 import；工具 schema 不再按 policy 过滤，全量下发
- `packages/core/src/core/agent/tool-execution.ts`：删除 `toolPolicy` 相关的布尔禁用拦截与路径拦截分支
- `packages/core/src/core/agent/prompt.ts`：删除 `describeActivePolicy` 及 system prompt 里的「本轮工具约束」段落
- `packages/core/src/services/agent-service.ts` `runTurn`：删除 `parseToolPolicy(instruction, activeFilePath)` 调用与 `toolPolicy` 透传
- `packages/core/src/types/chat.ts`：`ChatTurnInput` 删除 `toolPolicy` 字段；删除 `import type { ToolPolicy }`

**0.2 删除死代码与遗物**（诊断文档「第二桶」）

- 删除 `packages/core/src/services/generation-service.ts`（app 零调用）
- 删除 `packages/app/src/components/file-tree/TreeNode.vue`（未被引用）
- 删除 `stores/settings.ts` 的系统提示词读写残留能力（`readSystemPrompt`/`writeSystemPrompt`），并同步修正 `FirstTimeGuide.vue` 第三步引导文案（改为指向现有功能，或直接移除该步骤）
- 删除 `rerank.mode: 'multimodal'` 死枚举（`types/project.ts`），Rerank 固定为 text
- 修复默认值漂移：统一 `conversationTokenLimit` 默认值为 **12000**（`SettingsModal.vue` 里的 8000 改为与 `defaults.ts` 一致）

**0.3 处理僵尸配置**（诊断文档 2.3）

六件套的处置分两类，**不要一刀切**：

- **保留配置项，标注「规划中」**：`conversationTokenLimit`、`compressionKeepRecentTurns`（本计划 Stage 3 会实现它们，现在保留并在 UI 加「即将上线」tooltip 或灰置说明，避免用户误以为已生效）
- **移除配置项**：`generationRecentChapters`（近章注入改由 Loop 重写时的上下文组装统一处理，不再单独配置）、`proofreadDefaultChapters`、`organizeDefaultChapters`、`enableBackgroundIndexing`（这三个对应功能无明确近期实现计划，从 `types/project.ts`、`defaults.ts`、设置面板 UI 一并移除，避免误导）

### 验收

- [ ] `pnpm test` 全绿（删除 tool-policy 后相关测试同步删除）
- [ ] 全仓 grep `toolPolicy`、`ToolPolicy`、`generationRecentChapters`、`proofreadDefaultChapters`、`organizeDefaultChapters`、`enableBackgroundIndexing` 仅剩压缩两项（保留的）无其他残留
- [ ] 手动验证：发一条消息让 Agent 读改一个文件，全流程正常（确认卡、写回、文件树刷新）
- [ ] 提交信息建议：`refactor(core): 清理 tool-policy 与死代码，为 Loop 重构铺路`

---

## Stage 1：打通流式渲染（独立价值，最先见效）

> **为什么最先：** 这是体感提升最大、风险最低、且不依赖后续重构的一步。做完即使用户只用现有 Loop，也能看到逐字输出。

### 背景

`agent/llm.ts` 的 `streamAgentCompletion` 已经在解析 SSE delta，但 `query.ts:84` 调用时把 `onEvent` 回调传成了 `() => {}`，delta 被丢弃。UI 要等 `finish` 事件才拿到整段文本。

### 范围与动作

**1.1 core：把 delta 事件透传出来**

- `query.ts`：`streamAgentCompletion` 的回调不再传空函数，而是把 `delta` 事件向上转发为一个新的 `AgentQueryEvent`：
  - 在 `AgentQueryEvent` 联合类型（`query.ts:16-26`）中新增 `{ type: 'assistant-delta'; text: string }`
  - `streamAgentCompletion(input, (e) => { if (e.type === 'delta') input.onEvent?.({type:'assistant-delta', text: e.text}) })`
- `chat/session.ts` `runChatTurn`：在 `query` 的 `onEvent` 里处理 `assistant-delta`，转发给 UI（见 1.2 的事件流）

**1.2 service：事件流加 delta**

- `services/types.ts` 的 `AgentUiEvent`（`:427-437`）新增 `{ type: 'message-delta'; sessionId: string; messageId: string; text: string }`
- `services/agent-service.ts` `runTurn`：把 core 的 `assistant-delta` 翻译成 `message-delta` UI 事件。需要为本轮的 assistant 文本消息预先分配一个稳定 `messageId`，让 UI 能把连续 delta 累积到同一条消息上。

**1.3 app：UI 流式渲染**

- `stores/chat.ts` `handleAgentEvent`：处理 `message-delta`——若该 `messageId` 的消息不存在则创建一条占位的 assistant 文本消息，存在则追加 text
- `components/chat/MessageItem.vue`：assistant 文本消息本就 Markdown 渲染，确认流式追加时能增量刷新即可（Vue 响应式天然支持）；可选加一个「生成中」的光标指示
- 注意：工具调用事件（tool-call/tool-result）仍在完成后才出现，这是正确的——delta 只针对纯文本输出

### 关键约束

- **delta 不落盘**：流式 delta 只用于实时渲染；最终完整 assistant 文本仍走现有的 `assistant-message` 完成事件落盘，避免持久化一堆碎片。即：delta 是「瞬态渲染态」，完成事件是「持久事实」。
- **abort 时已生成内容不丢**：`llm.ts` 的 `AgentAbortedError` 携带 `partialContent`，`query.ts:88-100` 已会保留为 assistant 消息——确保流式渲染下，abort 后 UI 上已显示的 delta 与最终落盘的 partialContent 一致。

### 验收

- [ ] `pnpm test` 全绿（为 `assistant-delta` 事件转发补单测）
- [ ] 手动验证：发一条会让 Agent 纯文本回复的消息（如「介绍一下这个项目」），文字**逐字/逐段出现**，而不是整段蹦出
- [ ] 手动验证：生成中途点「停止」，已显示的文字保留，且与最终落盘内容一致
- [ ] 提交信息建议：`feat(agent): 模型输出流式渲染到 UI`

---

## Stage 2：统一模型消息源（为压缩打地基）

> **为什么先于压缩：** 压缩要「替换一段区间」，前提是模型看到的消息历史有一个干净、可切片、带 token 估值的单一来源。现状是 ChatMessage（展示）和 AgentMessage（模型）双结构手工同步，必须先理顺。

### 背景

- 展示层：`ChatMessage` 7 类（`types/chat.ts:81-88`），给用户看，进 `.novel/sessions/*.json`
- 模型层：`AgentMessage` 4 类（`core/agent/messages.ts`），发给模型，存于 `ChatSessionState.agentMessages`
- 问题：两套结构靠 `runChatTurn` 的事件流手工同步（`session.ts`），压缩时很难干净地「切一段替换」

### 目标结构（参照 dsh 的 surface 概念，但不引入事件溯源）

引入一个独立的「模型视图」概念，与显示层解耦：

```
显示层 ChatMessage[]  —— 不变，继续负责 UI 展示与持久化，完整保留所有历史
模型层 ModelView      —— 新增。维护：
   - messages: AgentMessage[]（当前模型可见的消息序列）
   - 支持「区间替换」：把 [startIdx, endIdx] 替换为一条摘要消息
```

**关键决策（已定）：不引入 dsh 的事件日志。** `ModelView` 就是一个内存里的结构 + 持久化一个轻量的「替换记录」。显示层 `ChatMessage[]` 一字不动。

### 范围与动作

**2.1 新增 `ModelView` 抽象**

- 新建 `packages/core/src/core/agent/model-view.ts`
- 定义：
  ```ts
  type ModelView = {
    messages: AgentMessage[]           // 当前模型可见的消息
    // 压缩替换记录：把原 [startSeq, endSeq] 的消息替换为 summaryMessage
    // 显示层不受影响，仅模型层在 buildRequest 时应用
  }
  ```
- 提供方法：
  - `appendUser(content)` / `appendAssistant(msg)` / `appendToolResults(results)` —— 追加
  - `replaceRangeWithSummary(startIdx, endIdx, summaryMessage)` —— 区间替换（Stage 3 用）
  - `estimateTokens()` —— 用 `ceil(text.length / 4) + 每消息结构开销` 的启发式估算（参照 dsh `token-meter`，中文偏低可接受，阈值留余量）
  - `toRequestMessages()` —— 产出最终发给模型的 `messages` 数组
- 持久化：`ChatSessionState` 增加 `modelView` 字段替代现有 `agentMessages`（或并行迁移）。**持久化的是「当前模型视图」**（即压缩后的样子），不是全量历史——全量历史在显示层 `messages` 里。

**2.2 迁移 runChatTurn 使用 ModelView**

- `session.ts`：`buildAgentMessages` 不再直接拼 `previousMessages`，改为操作 `ModelView`
- system prompt 的 hash 刷新逻辑（`refreshSystemMessageContent`，`session.ts:465-473`）改为更新 `ModelView.messages[0]`
- `query.ts` 的 `input.messages` 来自 `modelView.toRequestMessages()`

**2.3 保持显示层不变**

- `ChatMessage[]` 的推送逻辑（`pushMessage`）完全不动——它继续完整记录用户消息、assistant 文本、tool-call/tool-result、context-summary、action-summary、error
- **从此显示层与模型层彻底分离**：显示层是全量 transcript（持久化、UI 展示），模型层是发给 LLM 的当前视图（可压缩）。不再有两套结构「手工同步」的问题——它们本来就各司其職。

### 验收

- [ ] `pnpm test` 全绿（为 ModelView 的 append/replace/estimateTokens 补单测）
- [ ] 手动验证：多轮对话（含工具调用）行为与重构前完全一致，模型能正确看到历史
- [ ] 数据验证：会话 JSON 里 `modelView`（或等价字段）存在，且压缩前与显示层消息内容一致
- [ ] 提交信息建议：`refactor(agent): 引入 ModelView 统一模型消息源`

---

## Stage 3：上下文压缩（核心能力，兑现承诺）

> **这是本计划的核心交付物。** 算法照抄 dsh 的 compaction，存储用 Stage 2 的 ModelView 区间替换表。

### 算法规格（照抄 dsh，见参考系切换文档 3.1）

**触发（两条路都要）：**

1. **压力触发**：每次模型请求发出**前**，`modelView.estimateTokens() >= thresholdTokens` 则先压缩。`thresholdTokens = floor(contextWindow × 0.8)`。contextWindow 从哪来见「配置」小节。
2. **溢出触发**：模型返回 context-window-exceeded 类错误时，强制压缩一次并重试该请求（重试上限 1 次，避免死循环）。

**选区（保留什么）：**

1. 系统提示（`messages[0]`）**永不**进压缩区
2. 从**尾部向前**累加 token，直到达到 `retainTokens = floor(contextWindow × 0.16)`——即保留最近约 16% 的原文
3. **工具配对回退**：若保留区起点切开了一对 assistant(toolCalls) ↔ tool 结果，则保留起点继续**向前**移，直到切线平衡——绝不让模型看到悬空的 tool result（没有对应 call 的 result，或有 call 没 result）
4. 若保留区已盖到系统头（无可压区间），放弃本次压缩

**摘要调用（KV-cache 友好）：**

- 发给模型的内容 = 会话自己的系统提示（messages[0]）+ 被压区间的原始消息 + **末尾追加一条 user 消息**作为压缩指令
- 压缩指令要求模型输出**固定八节 Markdown 检查点**（直接照搬 dsh 的模板）：
  `## Primary Request and Intent` / `## Key Technical Concepts` / `## Files and Code` / `## Errors and Fixes` / `## Pending Jobs` / `## Current Work` / `## Next Step` / `## Critical Context`，空节写 `(none)`
- 指令里强调：保留精确的文件路径/章节号/人物名/伏笔/用户纠正；**不得提及这是压缩请求**；若输入中已有 `<compacted-summary>` 块（旧检查点），不照抄，保留仍为真的事实并合并成单一总结（**这条规则使多轮压缩收敛**）
- **安全阀**：摘要输出若被 max-tokens 截断或为空，视为失败，**不落地**（宁缺毋滥）
- **收缩校验**：摘要的 token 估值必须**严格小于**被压区间的估值，否则放弃（摘要不比原文小就没意义）

**落地（回注）：**

- 用 Stage 2 的 `modelView.replaceRangeWithSummary(startIdx, endIdx, summaryMessage)`
- `summaryMessage` 是一条 `role: 'user'` 的消息，内容 = 固定 preamble + `<compacted-summary>` + 摘要 + `</compacted-summary>`。preamble 指示模型「把捕获的上下文当作既定背景，直接继续任务，不要回应此检查点」
- 显示层 `ChatMessage[]` **不动**（用户仍能看到完整历史）；可选在显示层追加一条 `kind: 'context-summary'` 消息告知用户「已压缩 N 轮对话」（复用现有 `ContextSummaryMessage` 类型，这正是它设计的用途）

### 配置（兑现 Stage 0 保留的两个配置项）

- `conversationTokenLimit`：**重定义为「触发压缩的 token 阈值」**（替代 dsh 的 contextWindow × 0.8，因为我们不一定拿得到准确的 contextWindow）。默认 12000。UI 文案从「接近上限触发上下文压缩」改为准确描述。
- `compressionKeepRecentTurns`：保留最近 N 轮**不压缩**的轮数，作为 `retainTokens` 的「轮数口径」补充——两者取并集（既保留最近 16% token，也至少保留最近 N 轮原文）。
- 在 `ProjectSettingsPanel.vue` 把这两项从「灰置/即将上线」恢复为可用。

### 范围与动作

- 新建 `packages/core/src/core/agent/compaction.ts`：
  - `shouldCompact(modelView, config): boolean`
  - `selectCompactionRange(modelView, retainTokens): {startIdx, endIdx} | null`（含工具配对回退）
  - `buildCompactionRequest(modelView, range): messages`（系统提示 + 区间 + 八节指令）
  - `COMPACTION_INSTRUCTION` 常量（八节模板全文，中英，含 `<compacted-summary>` 自合并规则）
  - `validateSummary(summary, originalTokens): boolean`（非空 + 未截断 + 收缩校验）
- `query.ts`：
  - 每次 `streamAgentCompletion` 前调用 `shouldCompact`，是则先跑压缩（压缩本身也是一次 `streamAgentCompletion`，无 tools）
  - 捕获 context-overflow 错误 → 压缩一次 → 重试
- `session.ts`：压缩后在显示层追加 `context-summary` 消息
- `services/agent-service.ts`：把压缩相关的配置从 `config.settings` 传入

### 验收

- [ ] `pnpm test` 全绿。重点单测：
  - `selectCompactionRange` 的工具配对回退（构造切开 tool-call/result 的场景，断言边界前移）
  - 系统提示永不进压缩区
  - 摘要比原文长时放弃
  - 多轮压缩时 `<compacted-summary>` 自合并
- [ ] 手动验证：构造一个超长对话（可临时把 `conversationTokenLimit` 调很低，如 2000，加速触发），继续对话，确认：
  - 触发了压缩（显示层出现 context-summary 提示）
  - 模型仍能记住早期关键信息（通过摘要），对话不「失忆」
  - 会话 JSON 里 `modelView` 的旧区间被摘要替换，显示层 `messages` 完整
- [ ] 提交信息建议：`feat(agent): 上下文自动压缩（阈值触发 + 摘要检查点）`

---

## Stage 4：范围权限（替代 tool-policy，软化确认）

> **为什么现在做：** Stage 0 删了 regex tool-policy，Stage 1-3 没动确认流。这一步用 dsh 的「范围策略」补上「不每次都打断」的能力。

### 核心思想（照抄 dsh，见参考系切换文档 3.3）

- **用范围策略替代每次确认，而非「记住选择」**：写入目标在项目工作区内 → 静默放行；越界 → 弹一次确认
- **授权永远一次性**：没有 always-allow
- **策略外置**：工具不标危险等级，「要不要确认」由一条独立决策链判断

### 范围与动作

**4.1 定义权限决策**

- 新建 `packages/core/src/core/agent/permission.ts`：
  ```ts
  type PermissionDecision =
    | { kind: 'allow' }              // 工作区内，静默放行
    | { kind: 'ask'; reason: string } // 越界，需用户确认（仅此一次）
  ```
- `decideWritePermission(path, project): PermissionDecision`：规范化路径后判断是否落在项目工作区根内（canonicalize + 前缀判断）。NovAI 的所有写操作本就被限制在项目目录内（`tools/path.ts` 已有 `normalizeProjectPath` 禁绝对路径/`..`），所以**绝大多数写天然在工作区内**——这一步的实际效果是：**默认不再每次确认，只有异常路径才确认**。
- 决策词表照抄 dsh：`ApprovalOutcome = 'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'`

**4.2 改造确认流**

- `tool-execution.ts`：写工具的确认从「无条件 buildConfirmation」改为「先 `decideWritePermission`，`allow` 则跳过确认直接执行，`ask` 才走现有确认流程」
- **保留**现有 `WriteConfirmation` UI（WriteConfirmationCard）不动，只是它现在极少触发
- ⚠️ **决策点**：是否提供「项目内也完全确认」的严格模式开关？dsh 用 sandbox mode 三档。NovAI 可简化为两档：`workspace`（默认，项目内静默）/ `strict`（每次确认）。建议第一版**只做 workspace 默认 + 不加开关**，把「strict 模式」留作后续按需添加（避免过度设计）。如果你（执行者）认为需要开关，在此提出并说明理由。

**4.3 安全边界保持不变**

- `tools/path.ts` 的所有保护（禁 `.novel/`、禁 `novel.config.json`、chapters 命名规范）**全部保留**——这些是硬安全，与「是否确认」无关
- 回收站软删除保留

### 验收

- [ ] `pnpm test` 全绿（为 `decideWritePermission` 的路径判断补单测：工作区内/外、`.`/`..` 逃逸尝试）
- [ ] 手动验证：让 Agent 在项目内创建/编辑章节，**不再弹确认卡**，直接执行
- [ ] 手动验证：tool-policy 已删，确认疲劳消失，但 `.novel/` 等保护仍生效（让 Agent 尝试改 `novel.config.json` 应被拒）
- [ ] 提交信息建议：`feat(agent): 范围权限——项目工作区内写入静默放行`

---

## Stage 5：长文本 Spill（贴合小说场景）

> **为什么值得做：** 这是 dsh 有、claude-code 没有的能力，且直接命中「小说章节动辄几千字」的痛点。实现量小（约 200 行），收益明确。

### 核心思想

超长工具结果/生成内容在**进入上下文之前**就拦截：全文落盘，上下文里只放预览 + 引用路径。与 compaction 互补——spill 管「单次太大」，compaction 管「总量超标」。

### 范围与动作

- 新建 `packages/core/src/core/agent/spill.ts`：
  - `SPILL_THRESHOLD_CHARS`（默认 8192，可参照 dsh）
  - `maybeSpill(toolResult, project): ToolResult`——若工具结果文本超阈值：
    - 全文写入 `.novel/spill/<随机id>.txt`（新增目录，加入项目结构）
    - 返回替换后的结果：`头 4096 字符 + \n\n[... 中间 N 字符已省略，完整内容见 <path> ...]\n\n + 尾 1024 字符`
  - 集成点：`tool-execution.ts` 在工具 `run()` 成功后、结果进消息流之前调用
- ⚠️ **决策点**：spill 目录是否纳入 `.novel/`（受写保护）？建议放 `.novel/spill/`（与其他内部数据一致），但注意别让 Agent 的 ReadFile 能随意读到（避免 read→spill→read 循环，dsh 特意跳过 read 工具）。**建议：CreateFile/EditFile 的结果不 spill（它们的内容是模型自己写的，不需要），只对 ReadFile/RagSearch 的超长结果 spill。** 如果你有不同判断，在此提出。

### 验收

- [ ] 单测：超阈值结果被 spill，上下文里是预览+路径；未超阈值原样通过
- [ ] 手动验证：让 Agent 读一个超长文件，确认上下文里没有整段塞入，而是预览+引用
- [ ] 提交信息建议：`feat(agent): 超长工具结果 spill 落盘 + 预览引用`

---

## Stage 6：轮次上限可配置 + 优雅收尾（小改进）

### 范围

- `query.ts` 的 `DEFAULT_MAX_TURNS = 8` 改为从 `config.settings` 读（新增配置项 `agentMaxTurns`，默认 8，设置面板「项目设置」加一项）
- 超限收尾从「硬编码 assistant 话术」改为：发一条系统提示告知用户已达上限 + 保留当前上下文，让用户可以继续发消息续接（而不是中断）

### 验收

- [ ] 配置改动生效；超限时优雅提示而非死话术
- [ ] 提交信息建议：`feat(agent): 轮次上限可配置 + 超限优雅收尾`

---

## Stage 7（可选，二期）：todo_write 与 ask_user_question

> 这两个是 dsh 里对小说场景很有价值、但不阻塞主线的增强。放最后，前面都稳定了再做。

- `todo_write`：全量列表替换语义的待办工具（~100 行），让 Agent 在多步写作任务（如「写 5 章大纲」）中可追踪进度
- `ask_user_question`：模型主动向用户提问（选项 + 多选），适合「问用户选哪条剧情线」
- 参照 dsh `todo/tool-todo` 与 `interaction/tool-ask-user`，但用 NovAI 现有工具协议实现

---

## 给执行者的速查：每阶段的「文件动作」一览

| Stage | 主要新建 | 主要修改 | 主要删除 |
| :--- | :--- | :--- | :--- |
| 0 清理 | — | prompt.ts, SettingsModal.vue, types/project.ts, defaults.ts, 设置面板 | tool-policy.ts(+test), generation-service.ts, TreeNode.vue, 4 个僵尸配置 |
| 1 流式 | — | query.ts, session.ts, services/types.ts, agent-service.ts, stores/chat.ts, MessageItem.vue | — |
| 2 ModelView | agent/model-view.ts | session.ts, query.ts, types/chat.ts | （agentMessages 字段迁移） |
| 3 压缩 | agent/compaction.ts | query.ts, session.ts, agent-service.ts, ProjectSettingsPanel.vue | — |
| 4 权限 | agent/permission.ts | tool-execution.ts, （可选）设置面板 | — |
| 5 Spill | agent/spill.ts | tool-execution.ts, project-fs.ts（spill 目录） | — |
| 6 轮次 | — | query.ts, types/project.ts, defaults.ts, 设置面板 | — |
| 7 可选 | agent/tools 下新工具 | tools 注册表 | — |

## 风险与注意事项

1. **Stage 顺序不要乱**。1（流式）独立可先做；2（ModelView）是 3（压缩）的地基，必须在 3 前；4、5、6 相对独立但建议在 3 之后（压缩是最大的行为变化，先稳定）。
2. **每阶段都要跑全量测试**。core 的测试基础好（204+ 用例），删除/新增时同步维护。
3. **压缩是最有风险的一步**。它的单测必须覆盖工具配对回退和多轮压缩收敛，手动验收要确认「压缩后不失忆」。如果 Stage 3 手动验收发现「压缩后模型频繁失忆」，停下来，调整摘要指令模板或 retain 比例，不要硬交付。
4. **不要引入 dsh 的框架**。看到 dsh 的 Cordis/事件日志/瀑布事件觉得「很好」也不要抄——我们只抄形状（算法、词表、流程），不抄运行时。
5. **遇到计划与代码现状冲突**：以代码为准继续，但在交付说明里显式记录偏差点。


---

## 执行结果（2026-09-14）

Stage 0-6 全部落地并验收通过，228 个测试全绿，tsc / vue-tsc 无错。决策依据沉淀为 [0005 决策记录](../decisions/0005-参考系切换至DeepSeekHarness.md)。

### 阶段与提交映射

| 阶段 | commit | 内容 |
| :--- | :--- | :--- |
| Stage 0 | `abfcbb4` | 删 tool-policy 全链路（含计划外引用文件）、generation-service 双轨路径、TreeNode；settings 收敛 8 项（新增 `conversationTokenLimit=12000`、`compressionKeepRecentTurns=5`、`agentMaxTurns=8`） |
| Stage 1+2 | `f6b38ff`、`bf7d764` | `model-view.ts` ModelView 模型消息源与显示层分离、旧 `agentMessages` 会话自动迁移、CJK 感知 token 估算 |
| Stage 3 | `f3bde25` | `compaction.ts`：8 段检查点摘要、阈值+溢出双触发、tool 配对回退、摘要自合并、validateSummary |
| Stage 4 | `9cd2105` | `permission.ts`：工作区内静默允许、工作区外一次性确认 |
| Stage 5 | `3b7a981` | `spill.ts`：ReadFile/RagSearch >8192 字符落盘 `.novel/spill/`，上下文只留头 4096 + 省略标记 + 尾 1024 |
| Stage 6 | `323a7b0` | `agentMaxTurns` 可配置，超限优雅收尾（turn-limit-reached） |
| 计划外 | `1ed4ce7` | vite 开发代理逐块转发响应体（原 `arrayBuffer()` 把 SSE 整体缓冲，流式打字机失效——验收②发现的真 bug） |

### 手动验收（mock LLM + OPFS 项目实测，全部通过）

1. **读改全流程**：创建/编辑章节写回、无确认卡、文件树刷新，编辑只动目标行。
2. **流式与停止一致**：采样证明文本单调增长、光标常驻；中途停止后 UI 保留的前缀与会话 JSON 落盘正文逐字一致。
3. **压缩不失忆**（阈值临时调 2000 压测）：UI 出压缩提示；请求含 `<compacted-summary>`；早期原文被摘要替换、近期轮原文保留、显示层 76 条完整不动；多次压缩后摘要恒为 1 条（自合并）。
4. **权限**：项目内写章节零打扰；改 `novel.config.json` 被工具层读写双向拒绝（`不能指向项目配置或 .novel 内部文件`）。
5. **spill**：9007 字长文只进上下文 5186 字符（头+标记+尾），全文 27084 字节落盘。

### 偏差记录（计划 vs 实际）

1. **commit scope**：commitlint 只允许固定 scope 列表，`core` 不可用 → 用 `project`。
2. **`conversationTokenLimit` 语义**：从「上限」落地为「压缩触发阈值」（soft limit，dsh 本意），设置页文案已按真实语义改写。
3. **token 估算**：dsh 的 chars/4 低估中文约 4 倍 → CJK 区间 1 字 ≈ 1 token，其余 4:1。
4. **Stage 0 删除面**：tool-policy / generation-service 存在计划未列全的引用链，按「以代码为准」扩大清理并同步测试。
5. **已知残留**：三层系统提示词痕迹与 multimodal 引用（4 个文件）不在 Loop 主路径上，未清理，留待专项。
6. **配置文件保护比计划更严**：`assertMutableDocumentPath` 同时用于 ReadFile——配置与 `.novel/` 是读写双向保护，接受代码现状。

### 决策点记录

1. **不做严格权限模式**：本地单机 + File System Access API 已是沙箱，strict 模式只增摩擦；保护点放在工具层（路径校验）而非权限层。
2. **spill 白名单**：仅 ReadFile/RagSearch（唯一会产生超长结果的工具），不做通用 spill。
3. **压缩边界 tool 配对回退**：dsh 未明说，但悬空 tool 消息会被 API 拒绝——边界遇 tool 消息逐条回退到完整回合。
4. **压缩失败不致命**：摘要校验失败保留原上下文；溢出触发压缩后仅重试一次，避免死循环。
5. **轮次上限优雅收尾**：到上限不报错，提示「继续发送消息可接着完成」，上下文全保留。

### 观察（非缺陷）

`conversationTokenLimit` 低于保留区内容（`compressionKeepRecentTurns=5` 内含长文 spill 预览 ≈5000 token）时，每轮都会触发一次压缩——尽力而为的预期行为；默认阈值 12000 不受影响。真实模型（非 mock）的摘要质量需真实创作验证。
