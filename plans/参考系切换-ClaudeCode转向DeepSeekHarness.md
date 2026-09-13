# 参考系切换：从 Claude Code 转向 DeepSeek Harness（dsh）

## 记录时间

2026-09-14

## 文档目的

本文档记录一次关于**借鉴对象**的决策：NovAI 从「借鉴 claude-code 泄露源码」转向「借鉴 DeepSeek Harness（dsh）」。它回答三件事：为什么换、dsh 里有什么、换参考系要求 NovAI 做哪种程度的重构。

本文档是**分析决策记录**，不含可执行的开发步骤。它与 [[core功能梳理与冗余问题诊断]] 配套——那篇回答「现有代码哪些该删」，这篇回答「重写时往哪个形状写」。后续 Loop 重写的具体开发计划应另立文档，并回引这两篇。

## 一、背景：为什么现在提换参考系

- claude-code 是**泄露的闭源源码**，不公开、无演进、无注释、无法核实；且非开源，长期作为参考有合规风险。
- 0002《ClaudeCode借鉴与映射设计》§2.1 原本把 claude-code 定位为「一次性取灵感的参考实现，不是要跟踪演进的上游」。但实际开发中，每做一个机制（/init、readFileState、压缩、工具协议）都要回头扒它的行为——这实质上仍是一种跟随，而泄露源码让跟随不可持续。
- **DeepSeek Harness（`dsh`）**是 DeepSeek 官方开源的智能体 harness（MIT），持续更新，且每一层都有文档与决策记录解释「为什么」。它走 OpenAI 兼容模型，比走 Anthropic SDK 的 claude-code 更贴合 NovAI 现有技术栈。

## 二、对 dsh 的整体判断

> 研究方式：对 `/Users/honlnk/project/deepseek-harness`（约 50 个 packages）做四路并行源码分析，覆盖架构与 Loop、会话与上下文压缩、工具与权限安全、以及 NovAI 自身文档的借鉴意图对照。

**dsh 是「万物皆插件」的平台级工程**（基于 Cordis），其 Cordis 本体、fiber 树、可逆效应纪律、Profile/Bundle/Patch 三层配置、Typert RPC、50+ 包分包粒度——这些是为「第三方插件生态 + 热重载 + 多 profile 部署」设计的，**对单机小说应用是明显的过度工程，绝不引入。**

但埋在框架之下的几个**内核机制**是前沿且可剥离的——它们恰好是 NovAI 当初想从 claude-code 学、却因对方闭源而学不到的那一半。

## 三、关键对照：NovAI 缺什么 × dsh 有什么

| NovAI 的硬伤（见诊断文档） | dsh 的现成答案 | 重实现量级 |
| :--- | :--- | :--- |
| 不流式到 UI（`query.ts:84` 丢弃 delta） | StreamChunk 词汇表 + durable 事件 / live delta 双轨：delta 实时推 UI，持久化共用同一份流 | 接口照抄 |
| 无上下文压缩，历史无限膨胀 | 完整 compaction 算法（见下） | ~300 行 |
| 每次写都要确认（确认疲劳） | sandbox 范围策略替代「记住选择」（见下） | ~200 行 |
| 已决定删除的 regex tool-policy | 证明策略可完全外置为「决策链 + 路径归属围栏」，工具零标注 | 含在权限内 |
| 消息双结构手工同步 | 单一事件日志 + surface 投影，显示历史与模型历史靠语义拆分 | 见存储取舍 |
| 章节长内容塞爆上下文 | **spill 模式**：超长结果全文落盘 + 上下文放预览+locator | ~200 行 |
| 多步写作任务无状态 | todo_write 全量替换工具，模型靠自己历史回看 | ~100 行 |

### 3.1 Compaction（NovAI 最缺，dsh 最完整）

dsh 的 compaction 是目前公开实现里最完整的，可直接按其语义重写：

- **触发三条路**：每步请求前按阈值（contextWindow × 0.8）做 pressure 检查；provider 报 context-overflow 时强制压一次并重试；手动 `/compact`。
- **选区**：系统提示（surface 节点 0）永不进压缩区；从尾部保留约 16% token 的原文；若切线切开 tool-call/tool-result 配对则边界前移（绝不让模型看到悬空的工具结果）。
- **摘要调用是 KV-cache 友好的**：用会话自己的系统提示 + 被压区间原文，末尾追加一条压缩指令——整段前缀与上次真实请求一致，prefix cache 不失效。
- **固定八节 Markdown 检查点**模板（Primary Request / Key Technical Concepts / Files and Code / Errors and Fixes / Pending Jobs / Current Work / Next Step / Critical Context），空节写 `(none)`。
- **收缩校验**：摘要不比原文小就不提交；截断/空输出视为失败不落地。
- **`<compacted-summary>` 自合并规则**：输入中已有旧检查点时不照抄，保留仍为真的事实并合并——使多轮压缩收敛、summary 套 summary 不膨胀。

### 3.2 Spill（dsh 有、claude-code 无清晰对应）

compaction 管「总量超阈值后的历史浓缩」，spill 管「单次工具结果太大」。超长文本结果在生产时就拦截：全文落盘到 session 私有位置，上下文里只放「头一半 + 尾一半预览 + `(…N bytes omitted… Full result stored at: <locator>)`」。**对「AI 写了 4000 字章节」的场景，spill 比 compaction 更贴合**，且实现极简，强烈建议借鉴。

### 3.3 权限 / 软化确认（正好接上删 tool-policy 的决定）

dsh 把 claude-code 的「权限模式」拆成**两个独立旋钮**：SandboxMode（read-only / workspace-write / danger-full-access，执行层围栏）× ApprovalPolicy（ask / never）。核心启发：

- **策略完全外置**：工具不标 read-only/危险等级，「能不能写」由沙箱模式 + 「要不要问」由 pre-execute 决策链决定。这正是删掉 regex tool-policy 后该有的形态。
- **用范围策略替代每次确认，而非记住选择**：workspace-write 模式下，写入目标在项目工作区内 → 静默放行，越界 → 弹一次确认。dsh **没有 always-allow**，授权永远一次性（`allowed-once`）。这直接解决 NovAI 的确认疲劳。
- **模型自申请提升**：写工具参数带 `sandbox_permissions` + `justification`，越界被 `[sandbox: ...]` 拒绝后模型带这两个参数重试一次，触发单次审批。
- 决策词表可直接抄：`PreToolDecision = allow | {deny, reason} | {ask, reason}`、`ApprovalOutcome = allowed-once | rejected | cancelled | unavailable`。
- plan mode 在 dsh **不是权限模式**，而是「一段提示词策略 + `exit_plan_mode` 提交计划给用户审批」——适合 NovAI 未来的「大纲模式」。

### 3.4 其余值得借鉴的内核

- **Loop 形状**：turn/step 双层循环 + inbox 输入队列（followup/steer/inject 三档输入语义）+ per-phase AbortController + abort 时落半成品 assistant 消息（保留已生成内容）。steer（中途插话改方向）/inject（后台注入资料）对写作场景天然契合。
- **LLM 抽象**：`LlmAdapter` 只强制实现一个 `stream(options)` 方法，统一 StreamChunk 词汇，错误归一化为 finish chunk；重试做成 loop 外的事件监听者而非写死在循环里。
- **工具定义**：`defineTool` 强制 `output.schema` + `render`（机读结果与模型可读内容分离）+ `isConcurrencySafe` 并行分类。
- **todo_write**：全量列表替换语义，模型靠自己历史里的调用参数回看列表，零额外注入。
- **ask_user_question 工具**：模型主动向用户提问（选项 + 多选），是「问用户选哪条剧情线」的天然载体。

## 四、决策：只抄形状，不抄存储

dsh 的地基是**事件溯源存储**（append-only 事件日志 + JSONL 持久化 + 从事件投影出模型历史），compaction、流式、撤销、崩溃恢复都长在这块地基上。它最优雅，但对单机单会话的小说应用是重投入。

**决策：不引入事件溯源存储。**

- 保留现有 `ChatMessage[]` 作为**显示层**（人类可见的完整 transcript，不被压缩改动）。
- 新增一个**模型视图层**，支持「区间 → summary」替换：构建发给模型的请求时，把被压区间替换为一条 user 消息（preamble + `<compacted-summary>` 包裹）。
- compaction 靠这层「区间替换表」实现，而不靠 dsh 的 `SurfaceOp: replace` 事件原语。

**代价与缓解**：放弃事件溯源意味着撤销 / 分支 / 崩溃恢复 / 完整审计这些 dsh「免费获得」的能力需要另外想办法或放弃。对单机小说应用这是可接受的；会话仍持久化为 JSON（现有 `.novel/sessions/*.json`），但**模型视图与显示层分离**，避免当前「展示消息与模型消息双结构手工同步」的不一致风险。

**结论：dsh 提供形状（compaction 算法、spill、权限模型、Loop 结构、StreamChunk 词汇），NovAI 用自己的轻存储实现这些形状。**

## 五、换参考系对既有文档/决策的冲击

NovAI 现有设计的主体（见下）与 claude-code 无关，换参考系后**全部存活**：Agent Loop 为会话核心、文件为主存储、消息流=操作轨迹、第一阶段非目标清单、无 BashTool + 专用受控工具 + 工作区边界、RagSearch 由 Agent 自主调用、预览即默认目标、prompts 体系作为一等数据、全部技术选型。

需要在文档层**重新接地**的（设计成品不变，论证锚点从 claude-code 换成对 dsh 的核实结果）：

1. 0002 §9 与工具系统 §6 的两份「claude-code X → NovAI Y」映射/替换表——设计已自洽独立，只需重述理由。
2. `readFileState` 先读后写 + 过期校验、WriteFile 暂缓——机制是通用好实践，保留设计、更换论据。
3. `/生成项目记忆` 对齐 claude-code `/init` 的「成熟模式」论据——实现已独立，改论证不改代码。
4. 「RagSearch 是差异点」建立在「claude-code 只有文本搜索」的对比上——需对 dsh 重新核实差异化论证（dsh 有 grep/glob 但无语义检索，NovAI 的 RagSearch 价值论证仍独立成立）。
5. 0002 §4 的「不借鉴清单」是针对 claude-code 功能集写的，需对 dsh 的功能集重新推导（dsh 多了 compaction/spill/todo/subagent/skill 等，其中部分值得纳入借鉴而非拒绝）。

**待补一篇**：「dsh 借鉴与映射设计」（对位 0002），系统梳理 dsh 有哪些值得借鉴、哪些明确不借鉴——现有文档只有单向的「从 claude-code 取」，没有评估第二参考系的框架。

## 六、与诊断文档的关系

- 诊断文档说「主线 Loop 值得重写而非修补」，本文档给出**重写的目标形状**（dsh 内核）。
- 诊断文档决定删除的 regex tool-policy，本文档给出其替代形态（外置策略链 + 范围围栏）。
- 两篇共同的结论：**先清理（诊断文档的删除清单），再重写（本文档的形状），重写时兑现压缩/权限等承诺。**

## 七、给后续 Loop 重写的形状清单（索引用）

重写计划落地时，按此清单逐项设计，每项标注来源：

| 形状 | 来源（dsh） | NovAI 取舍 |
| :--- | :--- | :--- |
| turn/step 双层循环 + inbox 三档输入 | `core/agent-loop` | 抄形状，去 request-header/KV-cache 表面管理 |
| StreamChunk 词汇 + LlmAdapter 单 stream 方法 | `llm/llm` | 直接照抄接口 |
| durable 事件 / live delta 双轨 | `core/session` + `api/session-controller` | 只学协议形状，存储用轻方案（见第四节） |
| compaction 算法 | `compaction/*` | 按 3.1 重写，存储用区间替换表 |
| spill 模式 | `spill/spill-policy` | 强烈建议抄，~200 行 |
| 范围权限 + 单次授权 | `interaction/user-approval` + `sandbox` | 抄词表与流程，~200 行 |
| todo_write / ask_user_question | `todo/tool-todo`、`interaction/tool-ask-user` | 直接抄 |
| defineTool（schema + render 分离） | `core/tools` | 抄形态，校验用 zod |

**明确不抄**：Cordis 框架、事件溯源存储、Typert/WebSocket 传输层、多协议适配器体系、内核沙箱/E2B、subagent 多智能体（第一阶段非目标仍成立）。


---

## 执行状态（2026-09-14）

决策已落地为 [0005 决策记录](../decisions/0005-参考系切换至DeepSeekHarness.md)，形状清单中的 compaction / spill / 范围权限 / turn-step loop / StreamChunk 词汇已重写进 Agent Loop（Stage 0-6 全部完成并验收）。「明确不抄」清单全程未引入。todo_write / ask_user_question 与 defineTool 形态未在本期范围（对应原计划 Stage 7 可选项，暂缓）。执行细节见 [Agent Loop 重构开发计划](Agent%20Loop%20重构开发计划.md) 文末「执行结果」。
