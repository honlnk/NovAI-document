# Agent 提示词分层与越权修复计划

## 记录时间

2026-10-01

## 状态

已完工（2026-10-02 四波全部落地；W4 真机回归抓到两个旧项目兼容问题并当场修复，见 §八施工日志）。

## 文档目的

依据诊断文档 [docs/project/Agent越权执行问题诊断.md](../project/Agent越权执行问题诊断.md)（下称「诊断」），对主 Agent 的提示词体系做分层重构，修复「查一下 李世民身世」被升级为「改人物卡」这类越权执行问题。本计划是施工依据，波次门禁见 §三，定稿检查单见 §五，未实现项见 §六。

诊断结论一句话：越权是「提示词把 AI 教得太主动」的结果（执行层「不拦」是用户的选择，不是要解决的问题）；dsh 已用同模型对照实验证明病灶在 harness 不在模型。本计划不动 dsh 那套完整 sandbox/approval 状态机，只做三件事：**提示词分层（四级 md 注入）+ system 固定段瘦身（砍原则话、只留身份与项目结构）+ 工具描述中性化**。

## 一、核心设计决策：四级 md 提示词文件如何分层注入

这是本计划与 dsh 最大的差异点，也是最容易做错的地方。dsh 只有 AGENTS.md 一个工作区指令文件，降级为 user-role 建议层即可；**NovAI 有四个层级、性质不同的 md 文件，不能一刀切**。逐一定性后再定注入层：

| 文件 | 性质 | 当前注入 | 目标注入 | 理由 |
| :--- | :--- | :--- | :--- | :--- |
| `prompts/system.md` | 用户自定义的基调与**硬性约束**（内容红线/AI 惯用词/格式/续写节奏）——作者定的最外层规矩 | system 头部 | **system 固定段之后**（后置拼接）；**内容仍为默认模板（未被修改）时不注入** | 一旦写下整个创作过程基本不改，属最外层限制，放末尾即可；「用户没改 = 不需要」，默认占位文本不应白占上下文。注入机制见 §1.1.1。 |
| `prompts/NovAI.md` | 项目级累积记忆（写到哪了/有哪些人物/伏笔）——**等同 dsh 的 AGENTS.md** | system 头部 | **降为 user-role `<system-reminder>`** + 层级声明 + digest 按需注入；**非空即注入，空/缺失不注入** | 「项目正在维护李世民卡片」是**背景**，不是**命令**。它必须能被用户当轮的「查一下」压过去。这是本次越权最关键的一环。不给默认模板——后期 `/init` 指令一键生成（记入未实现项）。 |
| `prompts/outlines/*.md`（现 `prompts/scenes/`，改名，见 W3） | **大纲**——动笔前列的创作大纲 | system 头部 | **随最新一条 user 消息动态附带**（`<system-reminder>` 包裹），**不持久化进会话历史**；切换大纲时追加一条持久化切换记录；未选择/关闭/仍为默认模板时不附 | 一个会话通常按一份大纲写到底，附在最新消息上 = 「手边一直放着大纲」；不持久化 → 不累积 token、中途换 LLM 无感知；切换留痕防模型懵圈。注入机制见 §1.2。 |
| `prompts/ELEMENT.md` | 园丁子代理专属工作守则 | 只进园丁 system | **不动** | 与主 Agent 无关，园丁链路是独立的。 |

### 1.1 目标 system prompt 结构（主 Agent）——砍掉「工作原则」栏

**设计立场（用户拍板）**：学 dsh 的克制——system 只讲「身份 + 这个世界是什么」，**不讲「你该在什么场景做什么事」的原则话**。原来「工作原则」栏整个砍掉，工具纪律全部下沉到「工具使用规则」栏和工具 description。理由：

- dsh 通篇没有一句「编辑原则」，全部是工具使用方法，模型反而能根据用户指令自由判断只读/写。预写「协作模式/写操作边界」是把判断路径写死，给模型一个可错误套用的模板，反而误判（用户对 plan-mode 话术的直觉判断推广到了整条原则话）。
- NovAI 与代码开发的本质区别：**项目结构是固定的、由我们定义的**。代码项目结构任意，dsh 只能把结构知识丢给 AGENTS.md；NovAI 的 `chapters/`/`elements/`/`prompts/` 永远固定，这份「世界的事实」放在工具 description 里塞不下，正好放 system 固定段——它是陈述事实，不是下达指令。

重构后的 system 结构是**三块：一句身份 + 一份项目结构说明 + system.md（已修改时，后置）**：

```
你是 NovAI，一个使用 {model} 模型驱动的小说创作 AI 智能体。

本项目是一个小说工程，目录结构固定：

- chapters/：章节正文。一章一个 .txt 文件，纯文本，文件名形如 第NNN章-标题.txt（编号至少 3 位补零）。
- elements/：要素库，承载人物、设定等长期信息，按六类分目录：
  - characters/：人物，一人一文件。
  - locations/：地点，一地一文件。
  - entities/：具体实体——武器、武功、坐骑、丹药、信物、法器、书籍等会被点名、持有或发生状态变化的东西。
  - plots/：剧情事件，一个独立事件一文件。
  - timeline/：时间线阶段，一个故事阶段一文件。
  - worldbuilding/：世界观——抽象规则与体系（力量体系、社会制度、种族设定、地理总览等）。
  要素文件用 .md，文件名即要素名。
- prompts/：提示词文件。system.md 是全局基调与硬性约束，outlines/ 是大纲，NovAI.md 是项目级记忆，ELEMENT.md 是要素库整理规范。
- .novel/：应用内部数据（会话、日志、回收站、RAG 索引），不要直接读写。

---
（prompts/system.md 的内容，仅在用户修改过默认模板时拼接——见 §1.1.1）
```

身份一句把 NovAI 名与模型名合在一句话里（dsh 分两行是其「核心/部署层各自注入」的实现痕迹，我们没有那层拆分，一句更自然）。`{model}` 从 `config.llm.model` 拼接，**只写 LLM 主模型**；embedding/rerank 模型不写入。

**关键措辞约束**：结构说明一律用**陈述句**（「characters/：人物，一人一文件」），不用**指令句**（「人物要写入 elements/characters」）——模型读到「李世民是人物」自然知道去 characters/ 找，不需要被命令。`.novel/` 那句「不要直接读写」是唯一保留的红线（保护应用数据，非原则话）。

### 1.1.1 system.md 的默认模板与「未修改不注入」机制

**注入条件**：读到的文件内容 `!==` `DEFAULT_SYSTEM_PROMPT` 常量（逐字节比对）才注入；仍为默认模板（用户没动过）则视为「用户不需要」，整段不拼。用户删掉模板里的机制注释句、或改动任何内容，都算「已修改」，即注入。

**新默认模板草案**（替换现有 `DEFAULT_SYSTEM_PROMPT`，陈述句为主、内容克制；因默认不注入，长度不占上下文成本）：

```
注：本文件当前为提示词模板，保持原样不修改时不会应用到对话上下文；删掉本句即可让以下基调与硬性约束生效（建议用陈述句书写，内容不宜过多）：

- 内容：禁止血腥暴力描写；允许暧昧，禁止色情；禁止未成年恋爱。
- 用词：避免「指尖泛白」「喉咙一滚」「这不是……，这是……」等 AI 惯用词。
- 格式：正文使用纯文本，不使用 *、#、--- 等 Markdown 符号。
- 节奏：每次续写约 2000 字，保持节奏可控（快慢按个人习惯调整）。
```

**明确删除的内容**（原「工作原则」栏的病灶，诊断 §三、§8.2）：
- 「你的工作不是把所有正文都堆到聊天里，而是……使用工具读取、修改或创建项目文件」（诊断 §3.1 任务预设）。
- 「用户的小说、章节、设定、提示词和素材都应保存在项目文件系统中」（诊断 §三 A 类，「该落盘」推力）。
- 「每轮收尾时回顾本轮对要素的改动……建议整理要素库」（诊断 §8.2.1，每轮持续的「该有改动」压力源）。**删除后无信息缺口**：「有这么个子智能体、干什么、何时值得委派、需用户同意」已全部在 DelegateToGardener 工具描述里（gardener.ts:245-249，描述式写法），工具描述不动。dsh 对照：subagent 工具的说明只在工具 description 里做能力声明+描述性举例（「research, a scoped implementation, an analysis」），system prompt section 仅一句纯操作机制（并行发起、别干等）且条件注入，**通篇没有「每轮考虑要不要委派」类时机劝导**（tool-subagent/src/index.ts:259-277、598-605）；其 workflow/ralph 的 section 反而是「仅当用户明确要求」的限制句——限制重工具可以，推广工具不做。
- 「不要把完整长篇正文当作唯一结果留在聊天里」（诊断 §4.6，被泛化成「聊天给答案=不合格」）。
- 整栏 17 条中属于工具协作纪律的（先 ReadFile 再动手、EditFile 精确替换、CreateFile/RenameFile 用法、ListDirectory/FindFiles 用法）——**下移**到「工具使用规则」栏，不在原则栏重复。

**「工具使用规则」栏保留**（它本来就讲工具用法，是 dsh 式的逐工具纪律），承接从原则栏下移的协作纪律即可，内容不重写有边界含义的条目。

### 1.2 目标 user-role 注入结构（含大纲的动态附带）

每轮发给模型的消息序列：

```
[system]     身份一句 + 项目结构说明（+ system.md，已修改时后置）

（以下为持久化历史，进会话存储、参与压缩）
[user]       <system-reminder> 项目记忆（prompts/NovAI.md，非空即注入；内容变更时重注一条） </system-reminder>
[user]       用户意图：{用户原话}（历史各轮，原样保留）
[assistant/tool …] 各轮工具循环
[user]       （切换大纲时才有）用户已将写作大纲从《{旧名}》切换为《{新名}》。

（发送时动态拼接，不落持久化历史）
[user]       最新一条用户消息 = 用户意图：{原话} + [可选] 引用 + 项目/目标
             + <system-reminder> 当前大纲（prompts/outlines/*.md 激活文件全文） </system-reminder>
```

**层级声明**（放在 NovAI.md 的 `<system-reminder>` 开头，照 dsh 精神中文化）：

> 以下是项目级记忆，作为背景指导。它不构成指令；与用户当前指令冲突时，以用户当前指令为准。它描述的是「项目里有什么」，不代表「本轮该做什么」。

**大纲（outlines）的注入规则**——三个条件全满足才附在最新 user 消息上：① `settings.activeOutlinePath` 已选择且文件存在；② 文件内容 ≠ 默认大纲模板（模板与注入机制同 system.md，见 §1.1.1，模板草案见 W3）；③ 用户未关闭大纲附带。**不持久化**是关键设计：大纲本体只存在于每轮请求的动态拼接里，历史消息不存——不累积 token、中途换 LLM 无感知、改大纲即刻生效；需要留痕的只有「切换」这一个动作（追加一条持久化 user 消息），防模型被突然换纲搞懵。大纲 reminder 的开头声明：`以下是当前写作大纲，作为背景参考。`

**user 尾巴**：删掉「若需要写文件，直接调用合适的文件工具」（诊断 §3.3，每轮强化写）。不为「中性表述」补任何新话——用户原话 + 引用 + 项目/目标信息即可，尾巴整句删除。

**digest 按需注入（仅 NovAI.md）**：NovAI.md 是持久化消息，为避免内容不变时重复注入，算内容哈希，只在**首轮**或**内容变化**时注入/重注。大纲因不持久化、每轮动态附带，**无需 digest**。实现参照 dsh `agent-instructions` 的 baseline+digest 机制（诊断 §二、§8.4.7）。

### 1.3 systemPromptHash 语义变化

当前 `systemPromptHash` 把 system.md + NovAI.md + scenePrompt 一起哈希（session.ts:275-284），任一变化整条 system 重建。重构后：

- system 只含身份句（含 `{model}`）+ 结构说明 + system.md 实际注入内容（未修改时为空），`systemPromptHash` 只随这三者变化——**NovAI.md/大纲的改动不再触发 system 重建**（缓存友好，诊断 §4.4 的修复）。
- NovAI.md 维护独立 digest 驱动持久化 reminder 的按需注入；大纲不进 system、不持久化，无 hash 参与。
- 换 LLM 模型 → 身份行变化 → hash 变化 → system 重建一次。可接受：换模型本就废掉全部前缀缓存，属既有成本，不新增损失。

## 二、工具 description 清理

**本波不新增任何「写操作边界」类原则提示词**（理由见 §1.1 设计立场——原则话会写死判断路径，反而误判）。「该不该写」完全交给模型根据用户当轮指令自由判断，我们不预写规则。

唯一要做的是**清掉工具 description 里既有的引导性措辞**（诊断 §8.2.2），把「往编辑方向推」的话改成中性的「这个工具是什么」：

- `RagSearch`（agent/tools.ts:324）：「适合在**写作、改稿**或回答设定问题前召回背景上下文」——「写作、改稿」排在前，隐含行动预设。改为中性：「从项目要素索引语义检索人物、地点、实体、剧情、时间线、世界观设定」。
- `CreateFile`（:150）：「已有文件请用 EditFile 修改」——把「修改」引回 EditFile。改为中性说明用途与「目标已存在时失败」的事实。
- `GetFileChangeHistory`（:406）：「**当你需要回忆或核对之前的改动时使用**」——暗示「你应该改过一些东西」。改为中性：「查询本会话内对项目文件的修改历史」。
- `DeleteFile`（:212）：补上「仅在用户明确要求删除时使用」——这是唯一保留的授权约束（删除不可逆，与 prompt.ts 的删除红线对齐），属红线非原则话。

**不做**：本波不引入 dsh auto-review 那种独立审查 LLM，也不做 ask_user 工具，更不写「声明-确认」提示词协议——先靠「砍原则话 + 项目记忆降级 + 工具描述中性化」跑一段时间观察，若仍越权再立项（记入「未实现项」）。

## 三、施工步骤（分波次）

### 波次 W1：system 固定段重构 + 新默认模板（prompt.ts + defaults.ts + tools.ts）

1. 重写 `buildAgentSystemPrompt` 固定段：砍掉「工作原则」栏，改为「身份一句 + 项目结构说明」（§1.1）；`{model}` 从 config 传入（签名加 model 参数）；「工具使用规则」栏承接从原则栏下移的协作纪律。
2. system.md 改为**后置拼接**，并实现「未修改不注入」：读到的内容与 `DEFAULT_SYSTEM_PROMPT` 逐字节相等则不拼（§1.1.1）。
3. 替换 `DEFAULT_SYSTEM_PROMPT` 为新模板草案（§1.1.1，含机制注释句与四类硬约束示例）。
4. NovAI.md / scenePrompt 从 `buildAgentSystemPrompt` 入参中移除。
5. 改 `buildAgentUserContext`：删除尾巴整句。
6. 同步清理 agent/tools.ts 的工具 description 引导性措辞（§二）。

门禁：`pnpm test`（prompt.test.ts 同步更新断言，含「默认模板不注入」用例）+ `pnpm typecheck` 绿。

### 波次 W2：注入机制（session.ts + prompt.ts + agent-service.ts）

1. NovAI.md 改为持久化 user-role `<system-reminder>`（带层级声明）+ digest 按需注入：session 状态记 digest，首轮或变化时注入/重注。
2. 大纲（此波仍用 `prompts/scenes/` 路径）改为**随最新 user 消息动态附带、不落持久化历史**：发送前对最新 user 消息做动态包装；附带的条件与措辞见 §1.2。
3. 切换留痕：session 状态记 `lastOutlinePath`，检测到激活大纲变化时向持久化历史追加一条「用户已将写作大纲从《旧》切换为《新》」。
4. `systemPromptHash` 收窄为「身份一句（含模型名）+ 结构 + system.md 实际注入内容」。
5. 与 compaction 的交互确认：NovAI.md reminder 参与压缩时的重注语义（参照 dsh「session resume 后重新注入」）；大纲因不持久化天然不受压缩影响（验证即可）。

门禁：`pnpm test`（新增 digest 注入与大纲动态附着的单测）+ `pnpm typecheck` 绿。

### 波次 W3：scenes→outlines 改名与大纲模板（project-fs.ts + types + defaults.ts + app UI）

1. 目录常量 `prompts/scenes` → `prompts/outlines`；默认文件 `scene-001.md` → `outline-001.md`；`DEFAULT_SCENE_PROMPT` → `DEFAULT_OUTLINE_TEMPLATE`（新模板含机制注释句：未修改不注入，草案同 §1.1.1 思路，内容为梗概/主线/关键节点三节骨架）。
2. `settings.activeScenePromptPath` → `settings.activeOutlinePath`；config 归一化时旧字段名读入映射，不丢用户选择。
3. 旧项目迁移：打开旧项目时检测 `prompts/scenes/` 存在且 `prompts/outlines/` 不存在 → 目录整体改名 + config 里大纲路径前缀同步替换；用户文件只改名不动内容。
4. app 端「场景」选择器与文案改为「大纲」。

门禁：`pnpm test`（迁移逻辑单测）+ `pnpm typecheck` 绿。

### 波次 W4：真机回归 + 文档同步

1. 复现「查一下 李世民身世」：Agent 全程只读、聊天里直接给出完整信息、不碰人物卡。
2. 写作场景回归：「帮我写第 1 章」正常 CreateFile；「把这段改进 xxx」正常 EditFile。
3. 大纲链路：选大纲 → 请求里最新 user 消息带大纲；切大纲 → 历史出现切换记录；关闭 → 不再附带。
4. system.md 链路：新项目默认模板不注入；删掉注释句后注入生效。
5. 更新 docs/project/当前进度.md。

门禁：真机四类行为符合预期 + 文档同步。

## 四、影响面与风险

- **改动文件**：`packages/core/src/core/agent/prompt.ts`（主）、`packages/core/src/core/chat/session.ts`（注入与哈希）、`packages/core/src/core/agent/tools.ts`（工具描述）、`packages/core/src/core/project/defaults.ts`（新模板）、`packages/core/src/core/fs/project-fs.ts` 与 `packages/core/src/types/project.ts`（改名迁移与 activeOutlinePath）、app 设置页/大纲选择组件、对应 `*.test.ts`。全部是共享/核心文件——按工作纪律，开工前需用户确认本计划。
- **行为风险**：W2 改变项目记忆与大纲的注入位置，可能影响依赖「每轮都看到完整 NovAI.md/场景」的现有会话行为；digest 注入在超长会话 + compaction 下的正确性需要 W2 门禁专门验证。
- **迁移风险（W3）**：旧项目 `prompts/scenes/` 改名为 `outlines/` 只动目录名不动文件内容；迁移需幂等（重复打开不重复迁移）；`activeScenePromptPath`→`activeOutlinePath` 的旧字段映射不能丢用户已选大纲。
- **模板比对风险**：「未修改不注入」依赖逐字节比对，新项目创建写入的模板必须与常量完全一致（同一常量写入与比对，天然满足）；用户把文件改回与默认逐字相同的内容会被判为「未修改」，属可接受的边界。
- **兼容性**：session 持久化文件里的旧 `systemPromptHash` 与新语义不兼容，旧会话首次打开会触发一次 system 重建——可接受（一次性），但要在提交说明里写明。

## 五、定稿检查单

- [x] 四级 md 文件逐一定性，注入层逐一明确（§一）—— 这是用户特别提醒的点，dsh 单文件方案不能照抄。
- [x] 每条修复都能回链到诊断文档的具体小节。
- [x] 不做动词白名单（用户明确否决过）。
- [x] **砍掉「工作原则」栏，不新增任何原则话**——system 只讲「身份 + 项目结构事实」，「该不该写」交给模型自由判断（用户拍板的设计立场，诊断 §8.4 plan-mode 话术同此理由不采用）。
- [x] 不照搬 dsh plan-mode 的「计划/授权」话术进默认提示词（用户直觉判断可能更差，留作后备）。
- [x] 不引入 dsh 完整 sandbox/approval 状态机（过重，诊断 §七结论）。
- [x] 项目结构说明用陈述句不用指令句（「characters/：人物」而非「人物要写入 characters/」）。
- [x] 身份一句合并 NovAI 名与 `{model}`（仅 LLM 主模型，embedding/rerank 不写入）。
- [x] system.md 后置拼接；默认模板未修改不注入（机制注释句方案，模板含内容红线/AI 惯用词/格式/续写节奏四类示例）。
- [x] 大纲 = 随最新 user 消息动态附带、不持久化、切换留痕；scenes → outlines 改名含旧项目幂等迁移。
- [x] NovAI.md 等同 dsh AGENTS.md 待遇：非空即注入 + digest，无默认模板（/init 后续立项）。
- [x] **删除「配置归位」方向**（用户裁定：「不拦」是用户的选择，要解决的是提示词把 LLM 教乱的问题，不是拦截问题）。
- [x] 门禁明确：W1/W2/W3 测试+类型检查，W4 真机回归。

## 六、未实现项（须在完工汇报中口头说明）

- **`/init` 指令**（扫描全项目一键生成 prompts/NovAI.md 项目记忆）：本波不做；NovAI.md 在此之前无默认模板（新项目创建时留空占位，非空即注入）。
- 独立审查 LLM（auto-review 移植）：本波不做，若越权问题仍在再立项。
- ask_user 工具（给模型显式「问用户」通道）：本波不做，靠模型自由判断。
- **一切「写操作边界 / 声明-确认」类原则提示词**：本版**刻意不写**——用户判断原则话会写死判断路径、反而误判（诊断 §8.4 的 plan-mode 话术「祈使句=计划不是执行」「对话确认不授权」「先声明方案再动手」同属此类，一并留作后备）。**若后续工作中仍出现越权或边界问题，再回来评估是否补这类更明确的边界提示词**（后备出处：诊断 §8.4.1、dsh `bundle/base/cordis.patch.yml:322-336`）。
- system/user/tool 正文的完整日志落盘（诊断 §4.5）：属可调试性改进，与本计划正交，另行立项。

## 七、关联文档

- 诊断：docs/project/Agent越权执行问题诊断.md
- 决策：docs/decisions/0005-参考系切换至DeepSeekHarness.md
- 参考：/Users/honlnk/project/deepseek-harness（agent-instructions / system-prompt / plan-mode）

## 八、施工日志

### W1（2026-10-02）：system 固定段重构 + 新默认模板 + 工具描述中性化

改动文件：`core/agent/prompt.ts`、`core/project/defaults.ts`、`core/agent/tools.ts`、`core/chat/session.ts`、`types/chat.ts`、`services/agent-service.ts`、`core/agent/prompt.test.ts`。

按计划落地：工作原则栏整栏砍掉（含「每轮收尾建议整理要素库」「不要把正文堆到聊天里」等病灶话术）；system 改为「身份一句（含 `{model}`，仅 LLM 主模型）+ 项目结构陈述句 + 工具使用规则（承接原则栏下移的协作纪律）」；system.md 后置拼接 + 未修改不注入（逐字节比对）；新默认模板（机制注释句 + 四类硬约束示例）；`buildAgentUserContext` 尾巴整句删除；工具描述中性化四处（RagSearch/CreateFile/GetFileChangeHistory 中性化，DeleteFile 补「仅在用户明确要求删除时使用」）。

偏差与补充（均已在汇报中说明）：

1. **旧默认模板双比对**：计划只写与 `DEFAULT_SYSTEM_PROMPT` 比对；实现另存 `LEGACY_DEFAULT_SYSTEM_PROMPT`（旧版占位模板），旧项目从未改过的 system.md 同样视为未修改、不注入——否则旧项目会把旧占位文本当硬约束注入。
2. **prompts/ 结构说明暂写 `scenes/`**：计划 §1.1 模板是终态（`outlines/`）；W1 时目录仍叫 `prompts/scenes/`（W3 才改名），先写与实际目录一致的 `scenes/`，W3 随改名同步更新。
3. **「保持中文输出」随原则栏整栏砍掉**：计划拍板不加原则话；中文输出交给用户消息语言驱动，真机回归（W4）关注是否出现语言漂移。
4. **「章节首行写标题」并入结构事实**：原原则栏该条是创建指令句，改为 chapters/ 结构说明里的一句陈述事实（`首行写章节标题`），信息不丢。
5. **RagSearch 提示词侧同步中性化**：计划 §二只点名工具 description；system 工具规则栏里 RagSearch 那条同样带「写新章节、续写、改稿时先用」的行动预设，一并改为中性能力描述。

索引修正：`plans/README.md` 原有「上下文分层注入重构计划」索引行（本计划的前身草稿名，文件从未落盘），随本计划立稿取代——旧稿描述中的「运行时快照」想法未纳入本计划。

### W2（2026-10-02）：注入机制——NovAI.md reminder + 大纲动态附带

改动文件：`core/agent/prompt.ts`（reminder 构建、digest 同步、大纲附着与切换留痕，新增注入函数群）、`core/agent/model-view.ts`（`insertAfterSystem`/`replaceMessage` 原语）、`core/agent/query.ts`（`outlineReminder` 请求级附着，初始与溢出重试两处）、`core/chat/session.ts`（每 turn 接线）、`types/chat.ts`（会话状态 `novaiOverviewDigest`/`lastOutlinePath`、入参 `novaiOverview`/`outline`）、`services/agent-service.ts`（读取与传递）、`core/fs/project-fs.ts` + `core/project/defaults.ts`（NovAI.md 空占位、骨架退役为 `LEGACY_DEFAULT_NOVAI_OVERVIEW`）、`core/agent/prompt.test.ts`。

按计划落地：NovAI.md 非空即注入为持久化 user-role `<system-reminder>`（层级声明四句），digest 按需注入（首轮/内容变化时注入或重注）；大纲（此波仍 `prompts/scenes/` 路径）发送时动态附到最新 user 消息尾部、不落持久化历史、不进压缩请求（三条件过滤：已选路径+内容非空+非未动默认模板）；切换留痕（旧→新追加持久化 user 消息）；`systemPromptHash` 收窄（W1 已随签名改造达成）。压缩交互：reminder 可能被压缩吞掉（选区从 index 1 起），下一 turn 检测 marker 缺失即重插 index 1——对位 dsh「session resume 后重新注入」；大纲不持久化天然不受压缩影响（附着函数不改视图，单测覆盖）。

偏差与补充（均已在汇报中说明）：

1. **reminder 原地替换而非追加**：变化时在原位替换、恒为视图唯一一条，新旧不共存——故不需要 dsh 的 baseline 替换声明（「取代此前任何版本」一句话保留在更新版文案里，语义等价且更简）。
2. **切换留痕补「关闭」文案**：计划只写旧→新；A→null（关闭大纲）同样让模型可感，补「用户已关闭写作大纲（此前为《A》）」。首次选择（null→A）不留痕，大纲随消息附带模型自然看到。
3. **NovAI.md 未动骨架不注入**：计划只说「不给默认模板」，未提旧项目存量；实现把旧骨架存为 `LEGACY_DEFAULT_NOVAI_OVERVIEW` 参与比对，否则旧项目「待补充」骨架每轮作为 reminder 灌进上下文。新项目/修复补齐一律写空占位文件。
4. **大纲附着不参与 token 估算**：`estimateTokens` 只看视图，动态附着的大纲不计入压缩阈值——阈值估算略乐观，大纲量级通常小，可接受。
5. **debug 日志不含动态大纲**：`agent_messages_debug` 记录的是视图消息；大纲在请求层附加，调试时须知。
6. **新字段用终态命名**：ChatTurnInput 新字段名 `outline`（值仍为 scenes/ 路径），W3 只改路径值与 settings 字段名，避免 W2/W3 两度改名。

门禁：503 测试全绿（prompt.test.ts 22 例，含 digest 注入/同内容跳过/变化重注/压缩吞掉重插/骨架与空内容不注入/大纲三条件过滤/附着最后一条 user 且不改原数组/切换留痕文案）+ typecheck 干净。

### W3（2026-10-02）：scenes→outlines 改名与大纲模板

改动文件：core 侧 `core/project/defaults.ts`（`DEFAULT_OUTLINE_TEMPLATE` 新三节模板〔梗概/主线/关键节点〕+ `LEGACY_DEFAULT_OUTLINE_TEMPLATE` + settings 字段改名）、`types/project.ts`（`activeOutlinePath`、`missing-prompts-outlines`）、`core/fs/project-fs.ts`（目录常量、createProject 写 `outline-001.md`、`readOutlinePrompt`、`migrateScenesToOutlines` 迁移、config 旧字段与前缀映射）、`types/chat.ts` + `core/chat/target.ts`（`prompt-outline`）、`core/agent/prompt.ts`（双模板比对、结构说明改 outlines/）、`services/agent-service.ts`、`services/types.ts`；app 侧 `PromptList`/`CategoryPanel`/`ProjectView`/`ChatPanel`/`IndexStatusBar`、`OutlineCommandPopover`（原 `SceneCommandPopover` 改名）、`stores/project.ts`（`changeActiveOutlinePath`）。

按计划落地：目录/默认文件/模板常量/settings 字段四件套改名；旧项目打开时自动迁移（幂等）；config 归一化读入旧字段名与前缀映射（不丢用户选择）；app 端「场景」文案与选择器全部改「大纲」（@大纲 弹层、大纲 chip、状态栏、提示词面板）。

偏差与补充（均已在汇报中说明）：

1. **inspectProject 兼容旧目录**：检测改为 outlines/ 或 scenes/ 任一存在即通过——否则未迁移的旧项目会被误判损坏进修复流程（计划未提）。
2. **迁移为逐文件搬移而非目录改名**：浏览器 FSA 没有目录 rename 原语；同名冲突文件保留原处不覆盖（内容优先保护已存在版本），scenes/ 目录仅在搬空后删除，仍有残留则保留（计划只写「目录整体改名」，冲突语义是实现补充）。
3. **config 写回时机**：归一化始终在读时映射（旧字段名/scenes 前缀随读随治），迁移发生或 config 带旧字段时把结果写回盘——双保险。
4. **toast 文案纠错**：切换/关闭大纲从「新建会话后生效」改为直接确认——W2 动态附带后当轮即生效，旧文案已失真（W2 遗留的联动文案，随本波更正）。
5. **旧默认模板双比对**：迁移过来的旧项目未动过的 scene-001.md（旧 `# Scene Prompt` 模板）同样不附加——机制同 system.md/NovAI.md 的 legacy 比对（计划只写「模板与注入机制同 §1.1.1」，此为必要推论）。
6. **内部标识同步改名**：组件 `SceneCommandPopover`→`OutlineCommandPopover`、事件 `changeScene`→`changeOutline`、目标类型 `prompt-scene`→`prompt-outline` 等（计划只点名「选择器与文案」，内部名一并改防长期漂移；`其他场景`等无关语义的措辞不动）。

门禁：507 测试全绿（新增 4 条迁移用例：打开迁移+内容不动+config 映射、幂等、同名冲突保留、归一化边界）+ core/app 双 typecheck 干净。

### W4（2026-10-02）：真机回归

**验证方式偏差（最重要的一条）**：浏览器 IAB 的合成输入不构成 FSA 要求的「用户激活」，`requestPermission`/`showDirectoryPicker` 全被浏览器安全模型挡住——恢复项目、打开项目、新建项目三条入口都无法在 IAB 里自动化。真机回归改走 **Node 侧适配器挂真实项目**：`~/note/test-novel` 通过 Node-FS 版 DirectoryHandle 适配器挂进 `repairProject` + `setRuntimeProject` + `enqueueMessage`（app 调用的同一 services 链路），真实 DeepSeek 调用、真实文件读写、会话与日志真实落 `.novel/`。三阶段全过（跑完已把测试项目恢复原状；迁移幂等，用户下次打开自动重跑）。**app UI 层（@大纲 弹层、toast、状态栏 chip、提示词面板文案）未做浏览器走查**——由 vue-tsc + 单测兜底，列入未验证清单。

真机回归抓到并当场修复两个问题（均已补单测）：

1. **config 旧字段未随写回丢弃**：`normalizeProjectConfig` 的 `...config.settings` 展开把 `activeScenePromptPath` 原样带回写回结果；且写回触发条件用 `Boolean(旧字段值)`，本项目旧字段为 `null`（falsy）根本不触发写回。修：析构丢弃旧字段；触发条件改为「字段存在即写回」（`!== undefined`）。W3 施工日志第 3 条声称「旧字段名被丢弃」与事实不符，以此为准。
2. **v1 旧默认 system.md 模板未豁免**：monorepo 重构前出厂的默认模板只有标题+占位句（比 `LEGACY_DEFAULT_SYSTEM_PROMPT` 更早一代），真机项目的 system.md 正是它——字节比对不中，占位文本被拼进 system。修：新增 `LEGACY_DEFAULT_SYSTEM_PROMPT_V1` 一并豁免。scene 模板无此问题（真机字节与常量一致）。

三阶段结论（证据存 `/tmp/novai-w4/evidence-phase*.json`，会话/日志随验证轮落测试项目后已随恢复清除）：

- **阶段一（迁移 + 越权复现）**：scenes→outlines 真机迁移正确（scene-001.md 保留文件名移入 outlines/、scenes/ 删除、config 写回）。「查一下 李世民身世」全程只读（FindFiles/RagSearch/KnowledgeLookup/ReadFile 六次调用，零写工具），elements/ 与 chapters/ 内容哈希不变，聊天直接给出完整回答并**主动提议**「是否把家世补进人物卡」而非擅动——正是计划目标行为。注入分层核对：system=身份+结构+工具规则（旧模板不注入）、index1=user-role NovAI.md reminder（真实内容）、未选大纲不附带。
- **阶段二（写作链路）**：「帮我写第 1 章」→ CreateFile `chapters/第001章-城墙下的名字.txt`（965 字，真实落盘）；「把开头一段改成倒叙」→ EditFile 修改该章（哈希变化）。
- **阶段三（大纲 + system.md 链路）**：选中 main.md → 模型能原样引用大纲「关键节点」（请求级动态附着端到端生效）；切换 → modelView 出现「用户已将写作大纲从《main.md》切换为《outline-002.md》。」留痕；关闭 → 出现关闭留痕、模型确认无大纲。**留痕只落 modelView（持久化），不进显示层**——留痕面向模型（计划原意「防模型懵圈」），用户侧反馈由 toast+状态栏 chip 承担。system.md 改为真实内容后：模型复述出「短句白话」、modelView[0] 含该段——修改即注入生效。计划第 4 项「新项目默认模板不注入」由单测覆盖（新建项目入口同样被 FSA 墙挡住，未做真机新建）。

门禁：517 测试全绿 + core/app 双 typecheck 干净 + 文档同步。

### W4 补记（2026-10-02，浏览器走查与样例落档）

用户在 IAB 手动授权 FSA（真人点击「恢复项目」+ 权限条选「编辑文件」）后，补做此前列入未验证清单的浏览器 UI 走查：

已真机验证：

- 恢复项目链路：迁移与两处回归修复在真实浏览器下生效（config 读回仅剩 `activeOutlinePath`，旧字段消失）。
- @大纲弹层：输入 `@` 弹出、列出 `prompts/outlines/` 文件、搜索提示文案正常。
- 选中大纲：chip 落位（📍+文件名+「关闭大纲」按钮）、toast「已切换写作大纲」、输入框自动清空。
- 选择落盘：`novel.config.json` 写入 `activeOutlinePath`（无旧字段残留）——config 写回修复的真实流程验证。
- 消息发送链路正常（输入框清空、回合启动、状态栏「Agent 正在执行」）。

仍未验证（模型服务商故障阻断，移交用户自查）：

- 「查一下 李世民身世」的浏览器行为观察（Node 侧已验证等价场景，见上文阶段一）。
- 关闭 chip 的 toast「已关闭写作大纲」与 `activeOutlinePath: null` 落盘（单测覆盖；`null` 写回即 W4 修复点）。

走查中沉淀的 IAB 操作特性（后续真机验证须知）：

- **webview 后台回收**：IAB 面板退后台一段时间后 webview 被回收，重新激活时加载「最后一次完整导航的 URL」——SPA 项目路由与 FSA readwrite 授权随旧文档一起失效，表现为页面回首页、项目需重新恢复授权。真机走查需全程保持面板前台。
- **失败静默**：回合仅在本轮 `query_done` 后落盘会话；卡死/秒败（如模型 API 不可用）的回合不写盘、不留日志。排查法：`tail -f .novel/logs/agent.log.jsonl`，发消息后数秒内无 `model_start` 即模型链路故障。

交叉验证：另一项目（gpt-6.1-sol 经 openai-responses 中转）一轮完整会话的 modelView 分层同样正确——身份行随模型切换为 gpt-6.1-sol、index 1 reminder、用户包装第三行为「默认目标：〈该项目配置的默认章节文件〉」、WebSearch tool 结果自带不可信数据声明。

拼接结果样例（新旧 system 全文、reminder、包装、大纲附带格式、9/28 旧格式对照、跨项目样本）落档 [architecture/Agent提示词拼接样例-新旧对照.md](../architecture/Agent提示词拼接样例-新旧对照.md)（原 `/tmp/novai-w4` 产物，防重启丢失）。
