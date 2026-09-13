# core 功能梳理与冗余问题诊断

## 记录时间

2026-09-14

## 文档目的

在决定重构 Agent Loop 之前，对 `packages/core` 与 `packages/app` 做了一次全量功能梳理（方法：三路并行代码探索覆盖 app 层 / core 引擎层 / 服务层与文档，配置项消费状态用全仓 grep 逐个验证）。本文记录梳理发现的问题与讨论达成的处置共识，作为后续「清理提交」与「Loop 重写」两步工作的依据。

## 一、总体判断

core 的问题不是零散垃圾的堆积，而是三个结构性问题叠加：

1. **三条平行的模型调用路径，只有一条是产品主线。** Agent Loop（主线）、要素提取管线、FIM 输入补全各自有独立的 LLM 客户端、独立配置、独立前端编排，互不相通。
2. **Claude Code 的借鉴「学了一半」。** 学对的部分（文件工具、/init 式项目记忆、安全层方向）都是资产；但只学了骨架没学内脏（流式渲染、上下文压缩、权限分级、单一消息流全部缺位），同时还混入了 Claude Code 根本没有的自创冗余（tool-policy 正则解析约束）。
3. **配置面远大于实现面。** 大量「为未来预建」的配置与机制，实际消费点为零或远超当前规模。

## 二、问题清单

### 2.1 三条平行的执行路径

| 路径 | core 代码量 | 现状 |
| :--- | :--- | :--- |
| Agent Loop（主线） | `core/agent` 2085 行 + `core/chat` 811 行 + `agent-service` 623 行 | 唯一符合产品方向（对话驱动、工具落盘）的路径 |
| 要素提取管线 | `core/elements` 444 行 + `core/llm/client.ts` 211 行 + `element-service` | 完全绕开 Agent（`useElementExtraction` 明确注释「不进 Agent 对话」），并为此维护了一个与 `agent/llm.ts` 功能重复的独立 LLM 客户端 |
| FIM 输入补全 | `completion-client.ts` 332 行 + completion-service + 前端约 500 行 | 独立链路，定位为输入辅助（见处置共识：保留） |

（RAG 的 embedding/rerank 客户端算第四条调用路径，但属产品核心能力，保留。）

### 2.2 Claude Code 影子对照

**学对的（保留）**：

- Read/Edit/Create 文件工具 + ReadFileState 读后写哈希校验（对应 Claude Code 的 Read/Edit/Write 语义）。
- `/生成项目记忆` → `prompts/NovAI.md`（对应 /init → CLAUDE.md，ADR-0004 决策）。
- 软删除回收站、写前确认的安全层方向。

**学了骨架没学内脏的（Loop 不好用的真因）**：

- **流式渲染缺失**：`agent/query.ts:84` 调 `streamAgentCompletion` 时流式回调传了 `() => {}`，delta 全部丢弃，用户要等整段生成完才能看到文字。
- **上下文压缩缺失**：`buildAgentMessages` 只做 `[...previousMessages, nextUserMessage]` 拼接，`agentMessages` 跨轮全量累积、全量重发、全量持久化。`conversationTokenLimit` / `compressionKeepRecentTurns` 零消费。
- **权限分级缺失**：每次写操作都强制确认，无「本轮不再询问」类信任衰减，一章改多处即多次打断。
- **消息双结构**：7 类 ChatMessage（展示层）与 4 类 AgentMessage（模型层）靠事件流手工同步，存在不一致风险。
- **缺内容搜索（Grep）**：`ListDirectory` 与 `FindFiles` 两个文件名搜索工具功能重叠，而小说场景最需要的正文内容搜索（「某人物在哪几章出现」）缺位。

**自创且冗余的（删除）**：`tool-policy.ts` 正则解析自然语言约束（详见 2.4）。

### 2.3 僵尸配置与死代码

**僵尸配置六件套**（全仓 grep 验证：除类型定义、默认值、设置 UI 外无任何消费点）：

| 配置项 | UI 上的承诺 | 实际 |
| :--- | :--- | :--- |
| `conversationTokenLimit` | 「接近上限触发上下文压缩」 | 压缩零实现 |
| `compressionKeepRecentTurns` | 「压缩保留轮数」 | 同上 |
| `generationRecentChapters` | 「生成时携带最近 N 章原文」 | 近章注入零实现 |
| `proofreadDefaultChapters` | 校对默认章节数 | 校对功能是占位 |
| `organizeDefaultChapters` | 整理默认章节数 | 整理工具实际全量扫描，不用此值 |
| `enableBackgroundIndexing` | 自动后台索引 | 零实现，重建靠手点状态栏 |

**死代码 / 遗物**：

- `services/generation-service.ts`：固定流程生成的遗骸，**不是未完成，是被 Agent Loop 取代后已完成且过时**，app 零调用。
- `components/file-tree/TreeNode.vue`：分类面板改平铺方案后未被引用。
- 系统提示词编辑：`stores/settings.ts` 的 read/writeSystemPrompt 能力残留，设置弹窗已无对应入口（FirstTimeGuide 仍引导用户去设一个不存在入口的项）。
- `rerank.mode: 'multimodal'`：死枚举值，无分支逻辑，也无任何文档规划。
- 默认值漂移：`SettingsModal.vue` 硬编码回退值 `conversationTokenLimit: 8000` 与 `defaults.ts` 的 12000 不一致。
- `llm.protocol` 多协议（openai-responses/anthropic/gemini）：生成链路仅实现 openai 兼容，其余协议只能拉模型列表与测试连接，UI 靠警告文案兜底——半成品（处置见第三节的形状待定桶）。

### 2.4 tool-policy：已决定删除

- 实现：否定词表 + 读/写动作词表 + 「否定词后 6 字符窗口」正则匹配解析用户指令（`tool-policy.ts:109-129`），再经三层执行（prompt 软声明 → 工具可见性过滤 → 执行层硬拦截）。
- 问题：用正则猜自然语言意图，而这恰恰是模型本身的能力；「约束写操作」的硬需求已由写前确认覆盖；「不要读文件」类软约束模型自己能听懂。误判率高、维护成本大、渗透进 prompt/query/execution 三层，**几乎一点用都没有**。
- 删除后无防护真空：写前确认、路径白名单、`.novel/` 写保护、回收站全部保留。
- 保留一条认知：「路径级写锁」（只改当前文件）作为能力是成立的，Claude Code 的权限系统有 path 级 allow/deny；但将来若需要，应做成**显式的权限开关/模式选择**，而不是解析输入框文本。
- 删除范围：`tool-policy.ts` 及其测试、`query.ts` 的 `filterAvailableTools` 调用、`tool-execution.ts` 的拦截分支、`prompt.ts` 的 `describeActivePolicy` 注入、`agent-service.runTurn` 的 `parseToolPolicy` 调用、`ChatTurnInput.toolPolicy` 字段。

### 2.5 Loop 本体硬伤（重写对象）

- 上下文无限膨胀（见 2.2）；且 `agentMessages` 持久化时含每次 ReadFile 正文与 RagSearch 全文候选，会话 JSON 随轮次线性膨胀。
- `DEFAULT_MAX_TURNS = 8` 写死不可配；超限收尾话术硬编码为 assistant 消息，模型无法续接。
- 工具串行执行（`runAgentTools` 逐个 await），模型调用失败无重试/退避（仅有流式→非流式一次 fallback）。
- 场景切换「新建会话后生效」（UI toast）与 system prompt hash 同会话 in-place 刷新两套语义并存。
- `/提取要素`、`/生成项目记忆`（复用 sendMessage 换 instruction）、普通对话三条路径体验割裂。
- 展示消息与模型消息双结构手工同步（见 2.2）。

### 2.6 为未来预建的机器（记录在案，本轮不动）

RAG 的增量短路四元组、`embeddingTextVersion` 版本化、tags 召归放大 4 倍 hack、explain 三层解释、`runRagDebug`、索引事件总线与六态状态机、会话 LRU 缓存——服务的当前量级是几十个要素文件。属「超配」而非「冗余」，暂不处理，待真实创作验证后再收敛。

## 三、处置共识

讨论达成的三桶分类：**不是所有未生效的配置都按「待完成」保留——要区分「真未完成」「被推翻的遗物」和「形状待定」**，否则会把错误形状的东西 freeze 进重构。

**第一桶：真未完成，保留占位**

上下文压缩（Loop 重写时兑现）、近期章节注入、AI 校对、Git 版本管理、自动后台索引——均为产品文档与进度文档中的明确待办，配置先在、功能后补是合理策略。

**第二桶：被后续设计推翻的遗物，删除**

`generation-service`、`organizeDefaultChapters` 配置项、`rerank.mode: 'multimodal'`、`TreeNode`、系统提示词编辑残留、默认值漂移修复，加上 tool-policy 全链路（2.4）。

**第三桶：形状待定，重构设计时显式做决定（不要顺手「继续做」）**

- **多协议**：建议收缩为仅 OpenAI 兼容（浏览器直连 anthropic/gemini 有 CORS 与 key 暴露硬伤；OpenAI 兼容已覆盖 DeepSeek/百炼/Kimi/OpenRouter 等主流选择）。除非明确有上 Gemini 的计划，否则删除协议选择器与 `models-client.ts` 的三套协议分支。
- **要素提取管线**：保持独立命令，还是 Agent 化（作为工具或纯 prompt 驱动）？两种都成立，决定影响 `core/llm/client.ts` 与 `core/elements` 的去留，与 Loop 重写一起定。

**FIM 输入补全：保留。** 定位是辅助用户输入提示词，不占 Agent 的道；且工程上自包含（默认关、独立配置、独立客户端、不碰 chatStore），不构成主线负担。「自包含的外围功能」与「渗进主线的冗余」是两类问题，前者不参与清理。

## 四、行动顺序

1. **清理提交（不改行为）**：删 tool-policy 全链路 + 第二桶全部 + 多协议按第三桶的决定收缩或保留。预计净删 1500 行左右，主线能力零损失。
2. **Loop 重写**：流式渲染到 UI、上下文压缩（兑现两个压缩配置项或重新设计其语义）、轮次可配、消息结构合一、写确认的信任衰减。
3. 重构设计期间对第三桶逐项做决定并记录。


---

## 执行状态（2026-09-14）

诊断结论已全部执行：tool-policy 全链路删除、僵尸配置清除、三条平行执行路径收敛为单 Agent Loop、FIM 补全保留未动。「先清理（Stage 0）后重写（Stage 1-6）」的顺序被采纳，执行细节见 [Agent Loop 重构开发计划](Agent%20Loop%20重构开发计划.md) 文末「执行结果」。
