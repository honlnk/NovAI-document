# 引入 NovAI 项目总览（prompts/NovAI.md）

## 记录时间

2026-06-20

## 状态

已采纳。推翻 [0003 项目总览索引决策记录](0003-项目总览索引决策记录.md) 的「暂不实现」结论。

## 背景

0003 决策（2026-06-09）曾评估为每个小说项目增加一个项目级总览索引文件，作用类似 Claude Code 的 `CLAUDE.md`，用来汇总：当前写到哪一章、已有哪些人物/剧情块/时间线、哪些伏笔还没回收、最近更新了什么。当时的结论是**暂不实现**，理由是项目刚起步、基础能力还不齐全。

此后基础能力已经基本就位：

- 主工作区 UI 重构（R1-R7）全部完成。
- Agent 控制能力补强计划六个 Step 全部完成（结构化文件变更、写入前确认 + diff 预览、停止运行、用户即时工具约束、system prompt 同会话刷新）。
- 文件工具 Agent Loop、LLM 要素提取、过渡版 RAG 与 `RagSearch` 已打通。
- 会话持久化已落地。

[真实创作复盘](../archive/真实创作复盘对照.md) 也记录了真实使用中暴露的痛点：用户需要反复问 Agent「我们都写了什么」「第一章讲了什么」「有哪些内容」，这正是项目总览要解决的问题。该复盘明确建议「后续可评估 `NovAI.md` 作为项目状态地图」。

到这个阶段，0003 的「基础能力还不齐全」前提已经不成立，优先级应该重新放到这个直接提升长篇创作连贯性的能力上。

## 决策

为每个小说项目引入项目总览文件 `prompts/NovAI.md`，作为项目级累积记忆。具体约定：

### 职责边界

- `prompts/system.md` 继续承担「**怎么写**」：稳定的文风、叙事视角、格式约定。
- `prompts/NovAI.md` 承担「**写到哪了**」：章节进度、主要人物、世界观速览、伏笔追踪、风格约定。

### 注入机制

每轮 Agent 运行时，`readNovAiOverview` 读取 `prompts/NovAI.md` 内容，经 `buildAgentSystemPrompt` 注入到 system prompt 的「项目总览」段落。它复用 Step 6 已经建立的 system prompt hash 刷新机制：`prompts/NovAI.md` 内容变化会导致 system prompt hash 变化，下一轮自动刷新 system message，无需新增刷新逻辑。

### 维护机制：prompt 驱动的斜杠命令（对齐 Claude Code `/init`）

`prompts/NovAI.md` 的内容**不自动后台更新**，而是通过斜杠命令 `/生成项目记忆` 手动触发：

- 选中 `/生成项目记忆` 后，一段精心设计的驱动 prompt（`init-novel-prompt.ts`）作为用户意图发送给 Agent。
- Agent 复用现有工具（ListDirectory / FindFiles / ReadFile / RagSearch / CreateFile / EditFile）自主扫描项目并生成或更新 `prompts/NovAI.md`。
- 文件已存在时，prompt 约束 Agent 先 ReadFile 再 EditFile，不静默整篇覆盖，只更新有变化的段落。

这套设计直接借鉴 Claude Code 的 `/init`：斜杠命令本身不是扫描程序，而是「一段注入对话的宏指令 + 标准的 Agent 循环」。这是改动最小、最契合 NovAI 现有架构的方案。

### 初始化与修复

- 新建项目时 `createProject` 写入 `DEFAULT_NOVAI_OVERVIEW` 空骨架。
- 打开/修复旧项目时 `repairProject` 温和补齐缺失的 `prompts/NovAI.md`。
- `prompts/NovAI.md` 是可选文件：缺失时 `readNovAiOverview` 返回空串，Agent 仍能正常工作，因此**不纳入 `inspectProject` 的强制检测项**，避免给所有旧项目报「缺失」。

## 为什么选 prompt 驱动而非其他方案

考虑过三种维护机制：

| 方案 | 取舍 | 结论 |
|:--|:--|:--|
| **后台自动更新**（run-finish 后自动追加客观信息） | 不消耗 token、不占 Loop。但「后台偷偷改文件」可见性差，且 LLM 应用的经典陷阱——模型自己写的记忆被当事实、错误会累积放大——难以控制。 | 不采用。 |
| **新增 Agent 工具**（让模型每轮主动决定更新） | 可见、可确认。但改动面大（6 处类型 + 注册 + 实现），消耗 token，依赖模型自觉调用、可能遗忘。 | 不采用。 |
| **prompt 驱动的斜杠命令**（本次采纳） | 改动最小、完全复用现有 Agent Loop、手动触发零幻觉累积风险、对齐 Claude Code 成熟模式。唯一代价是要写一段高质量的驱动 prompt。 | 采用。 |

prompt 驱动还有额外好处：`/生成项目记忆` 的扫描过程对用户完全可见（每一轮工具调用都作为 ToolCallView 展示），用户能实时看到 Agent 读了哪些文件、怎么判断的，比后台静默更新更可信。

## 与要素系统的关系

`prompts/NovAI.md` 是项目级的横向状态地图；`elements/` 下的要素文件是按人物/地点/实体逐条的纵向档案。两者互补而非重叠：

- `prompts/NovAI.md` 的「主要人物」「伏笔追踪」段落可以从 `elements/` 的结构化内容汇总，但不应该简单复制全文。
- 未来要素模板 Step 3 落地后，要素文件会有稳定字段（`lastUpdatedChapter`、`relatedChapters` 等），届时 `/生成项目记忆` 可以更可靠地从要素文件汇总项目总览。

## 后续

- 验证 `/生成项目记忆` 在真实长篇项目中的扫描质量和生成结构稳定性。
- 后续可考虑在 UI 提示用户「项目尚未初始化总览」或为 `prompts/NovAI.md` 增加专门的入口。
- 与要素模板 Step 3 配合，让总览能从结构化要素汇总。
