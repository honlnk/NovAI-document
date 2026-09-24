# 思考强度选择器与 DeepSeek 思考回传修复计划

- 状态：已完成（2026-09-24 结项；W1 `cb4a76d` / W2 `a415e82` / W3 `6f05ed2`）
- 创建：2026-09-24
- 前置：生成链路多协议适配计划（已完成，W5 落了思考流 UI 与 reasoning 落盘）
- 参考实现：`/Users/honlnk/project/deepseek-harness`（dsh，`packages/llm/llm-deepseek`）

## 1. 背景与问题

两件事合并立项：

1. **现有真机缺陷**：openai 协议（Chat Completions）+ DeepSeek 思考模型 + agent 工具循环，
   从工具结果回传的下一轮起硬报 400。原因是 W5 定下的「reasoning 只收不发」策略：
   openai-chat 适配器构建历史消息时构造性剥除思考内容，而 DeepSeek 思考模式（无论显式开启还是默认开启）
   强制要求工具轮 assistant 消息回传 `reasoning_content`。
2. **缺失功能**：输入框没有思考强度选择器。DeepSeek 现行模型（deepseek-flash / deepseek-v4-pro）
   全部是多级思考强度的混合推理模型，`thinking` 开关 + `reasoning_effort` 三档真实可控；
   「选哪个模型=选思不思考」是 V3 时代旧认知，已过时。

## 2. 真机调研结论（2026-09-24 实测，api.deepseek.com，模型 deepseek-flash）

计划的全部事实声明来自本轮实测（curl 直打两个端点），不是文档转述。

### 2.1 openai 端点（/chat/completions）

| 探测项 | 结果 |
|---|---|
| 不带任何思考参数 | 默认思考开（116 字 reasoning）——即 NovAI 现状实际在思考 |
| `thinking: {"type":"disabled"}` | 思考关，reasoning 长度 0，开关有效 |
| `reasoning_effort: "low"` | 生效（194 字 reasoning） |
| `reasoning_effort: "max"` | 生效（225 字，思考行为明显变化），无需搭配 thinking 参数 |
| 工具轮不回传 `reasoning_content`（流式/非流式） | **HTTP 400**："The reasoning_content in the thinking mode must be passed back to the API"——NovAI 现状即此 |
| 工具轮回传 `reasoning_content` | 200 正常 |
| 思考关闭时不回传 | 200 正常 |

### 2.2 anthropic 端点（/anthropic/v1/messages）

| 探测项 | 结果 |
|---|---|
| 不带 thinking 参数 | 默认思考开，thinking block **带 signature** |
| `thinking: {"type":"disabled"}` | 有效，思考关、无 signature |
| `thinking: {"type":"enabled","budget_tokens":1024}` | 接受，思考开 |
| 工具轮不回传 thinking block | **200，宽容不报错**（与真 Anthropic 不同） |
| 工具轮回传真 thinking block + signature | 200 正常 |
| 回传 thinking block 但篡改 signature | 200，**不校验签名** |
| budget 数值是否真实约束思考长度 | 简单问题上测不出差异（160 vs 177 字），未定论 |

结论：**400 缺陷精确锁定在 openai 协议**；anthropic 协议接 DeepSeek 现状不炸。
但档位选择器要在 anthropic 协议上开思考，就必须补 thinking block 回传链路（为真 Anthropic 铺路，
DeepSeek 端点已验证回传两条路都通）。

## 3. 目标与范围

- 修复：openai-chat 适配器回传 `reasoning_content`，DeepSeek 思考模型 + 工具循环真机跑通。
- 功能：输入框思考强度选择器（照 PermissionPresetPicker 形态），档位持久化到项目配置，
  四协议适配器各自映射到 wire 参数。
- 修订设计原则：「reasoning 只收不发」退役 →「思考内容按协议要求回传」。

## 4. 设计决策（拍死）

### D1 档位枚举与语义

`'default' | 'off' | 'low' | 'high' | 'max'` 五档，取 DeepSeek 全集（用户主力后端）：

- `default`：不传任何思考参数（= 现状行为，provider 自决；DeepSeek 即默认思考开）
- `off`：显式关闭思考
- `low` / `high` / `max`：三档强度

存储：`config.llm.reasoningEffort?: ReasoningEffort`（types/project.ts，可选字段，缺省 default）。
请求层类型 `AgentLlmInput.reasoningEffort?: 'off' | 'low' | 'high' | 'max'`（default 不进请求层，字段缺省即 default）。

不引入 `medium`：DeepSeek 无此档，OpenAI 官方的 medium 无 key 验证，不做半吊子档位。

### D2 各协议 wire 映射表（实现与测试的唯一依据）

| 档位 | openai + DeepSeek 方言 | openai + 其他后端 | anthropic | gemini | openai-responses |
|---|---|---|---|---|---|
| default | 不传 | 不传 | 不传 | 不传 | 不传 |
| off | `thinking:{type:'disabled'}` | 不传（UI 隐藏该档） | `thinking:{type:'disabled'}` | `thinkingConfig.thinkingBudget:0` | 不传（UI 隐藏该档） |
| low | `thinking:{type:'enabled'}` + `reasoning_effort:'low'` | `reasoning_effort:'low'` | `thinking:{type:'enabled',budget_tokens:2048}` | `thinkingBudget:2048` | `reasoning:{effort:'low'}` |
| high | 同上 `'high'` | `reasoning_effort:'high'` | `budget_tokens:8192` | `thinkingBudget:8192` | `reasoning:{effort:'high'}` |
| max | 同上 `'max'` | `reasoning_effort:'high'`（降级） | `budget_tokens:16384` | `thinkingBudget:32768` | `reasoning:{effort:'high'}`（降级） |

- **DeepSeek 方言判定**：`baseUrl` 含 `deepseek`（core/llm/shared 工具函数）。
  只有方言内才允许发 `thinking` 顶层参数（DeepSeek 方言字段，OpenAI 官方等后端可能拒绝未知参数）；
  `reasoning_effort` 两边都认识，非方言直接发。
- anthropic / gemini 的 budget 数值（2048/8192/16384 与 2048/8192/32768）为拍定值：
  anthropic 最小合法 1024、gemini 上限 32768 的框架内取整档。约束效果未实测（见 §9 未验证），
  后续实测后可调表，调表不动 UI。
- max 档对 OpenAI 官方语义降级为 high：档位集合本身没有 max，映射表承担降级，UI 不标。

### D3 UI 形态与档位可见性

- 组件 `ReasoningEffortPicker.vue`，**照抄 PermissionPresetPicker 交互**：
  输入区按钮显示当前档位名，点击在按钮上方弹菜单，当前档带勾，Esc/点外关闭。
- 档位可见性按 `protocol × 是否 DeepSeek 方言` 静态表（app 层 constants）：
  - openai + DeepSeek 方言：五档全
  - openai + 其他：default / low / high / max（off 无法实现，不显示骗人的档）
  - anthropic：五档全；gemini：五档全（后两家 wire 单测覆盖，真机见 §9）
  - openai-responses：default / low / high / max
- 选中后走与权限档位相同的 updateConfig 写回链路（项目配置持久化）。
- 档位标签：默认 / 关闭 / 低 / 高 / 最高（中文标签，与权限档位命名风格一致）。
- **仅输入区一个入口**，设置页 LLM tab 不加（见 §8 不做清单）。

### D4 回传策略（「只收不发」退役）

- **openai-chat**：`toOpenAiMessage` 的 assistant 轮，`AgentMessage.reasoning` 存在即输出
  `reasoning_content` 字段（无条件回传，收到什么回传什么——与 dsh 一致；不产思考的模型
  天然没有 reasoning 可回传，自洽）。
- **anthropic**：assistant 轮 `reasoning` 存在时，在 content 首位回传 thinking block；
  有落盘的 signature 就带上，没有就不带（DeepSeek 端点两条路实测都 200；真 Anthropic 见 §9）。
- **gemini**：不动（thoughtSignature 已在 AgentToolCall 回传；thought part 不回传，见尾巴）。
- **openai-responses**：不动（加密 reasoning item 不回传，见尾巴）。
- 注释与测试同步：types/chat.ts 的「只收不发」注释改写为「按协议要求回传」；
  W5 写下的 session-driver「请求体不含 reasoning」断言**反转**为「含」。

### D5 anthropic signature 落盘链路

档位开思考后真 Anthropic 强制验签，本期把数据链做通（真 Anthropic 真机本身在 §9）：

- 流式解析：`signature_delta` 累积（与 thinking_delta 平行）。
- `AgentAssistantMessage.thinkingSignature?: string`、`AssistantTextMessage.thinkingSignature?: string`、
  `ChatMessageView` 同步透传（W5 的 reasoning 三层链路照抄一层）。
- 回传时构造 `{type:'thinking', thinking, signature}`。

### D6 辅助请求档位固定 off

`compaction.ts`（压缩摘要）与 `llm/client.ts` `streamChatCompletion`（要素提取等结构化输出）
显式传 `reasoningEffort:'off'`：DeepSeek 方言映射为 `thinking:disabled`（省 token，对齐 dsh
对 session-title 的处理），非方言映射为不传（= 现状，无风险）。

## 5. 实现波次

每波门禁：`pnpm test` 全绿 + `pnpm typecheck` 干净 + 文档随手同步 + 分波提交
（docs 子模块先提、父仓指针后提；只 commit 不 push）。

### W1 修复 openai 回传（先修 bug，最小爆炸半径）

- `openai-chat.ts` `toOpenAiMessage`：assistant 轮带 `reasoning_content`（D4）。
- `session-driver.test.ts`：W5 的 reasoning-only 工具轮测试断言反转（第二个请求体**包含**思考文本）。
- openai-chat wire 单测：assistant + reasoning + tool_calls 轮的回传断言。
- 真机验证：临时脚本（读测试项目 key，用后即删）跑 deepseek-flash 工具循环两轮，断言 200 且拿到正文。
- 验收：真机工具循环不再 400。

### W2 档位参数链路（core 层）

- 类型：`ReasoningEffort`（types/ai.ts）、`config.llm.reasoningEffort`（types/project.ts）、
  `AgentLlmInput.reasoningEffort`。
- `query.ts` 从 config 读取传入；compaction / streamChatCompletion 传 'off'（D6）。
- 四适配器按 D2 映射表实现（含 DeepSeek 方言判定函数，放 core/llm/shared）。
- anthropic：signature 解析 + 落盘三层 + thinking block 回传（D5）。
- wire 单测：D2 映射表逐格断言（含方言两分支、default 不传、max 降级）。
- 验收：映射表全覆盖的单测；现有测试不回归。

### W3 UI 选择器（app 层）

- `constants/reasoning-efforts.ts`：档位表（值/标签/描述/协议可见性）。
- `ReasoningEffortPicker.vue`（照 PermissionPresetPicker）+ ChatPanel 挂载 + updateConfig 写回。
- 组件单测（档位渲染/可见性过滤/选择写回）。
- 浏览器冒烟（browser-use，IAB 通道；devtools 遮罩已知问题沿用 page-context click 方案）：
  选择器交互、档位切换、config 落盘、发消息带档位请求。
- 验收：冒烟全过。

### W4 真机档位验证 + 结项

- 真机（openai 端点）：off → 无 reasoning；low/max → 思考长度可感差异；工具循环 + 档位组合 200。
- 真机（anthropic 端点）：disabled / enabled+budget / 工具轮回传 thinking block 组合 200。
- 文档：当前进度.md 条目、plans/README.md 索引流转、本计划施工日志与未验证清单定稿。
- **计划复审（用户点名要求，结项门禁）**：逐条核对本文档 §2 事实表与 §4 决策在实现后的真实表现，
  特别复核 DeepSeek 思考模型相关的每条声明（本轮追溯的教训：旧认知会过期，文档声明必须对得上实测）；
  偏差当场记录进施工日志，最终汇报走三节格式（未按计划实现 / 未验证清单 / 留下的尾巴）。

## 6. 迁移条款

零迁移：`reasoningEffort` 是可选字段，旧项目配置缺省即 default（不传参数，行为与现状完全一致）。
旧会话已落盘的 reasoning 在 W1 后自动参与回传，无需补数据。

## 7. 新旧机制替代表

| 旧机制 | 去向 |
|---|---|
| 「reasoning 只收不发」设计原则 | **删除**：注释改写为「按协议要求回传」，相关测试断言反转 |
| gemini thoughtSignature 回传（W3 旧有） | 保留，不动 |
| 输入框无思考控件 | 替换为 ReasoningEffortPicker（唯一入口） |
| 适配器不思考参数 | 替换为 D2 映射表（default 档 = 旧的「不传」，行为兼容） |

## 8. 不做清单（本期诱惑项）

1. 设置页 LLM tab 的档位入口（仅输入区，与权限档位入口形态一致）。
2. 会话级 / 单消息级档位覆盖（档位是项目级配置，一次设置全局生效）。
3. `medium` 档（DeepSeek 无此档；OpenAI 官方语义无 key 验证）。
4. gemini thought part（明文思考）回传（thoughtSignature 已够跑通）。
5. openai-responses 加密 reasoning item 回传（无 key，无法验证正确性）。
6. 真 Anthropic 端点真机验证（无 key；signature 链路铺好，行为列未验证）。
7. anthropic budget 数值的实测调优（拍定值先行，见 D2）。
8. 模型能力动态探测（拉 /models 或试错探测支持档位；本期静态映射表）。
9. 非流式路径的档位参数（NovAI 生成链路全流式；非流式仅存于适配器容错 fallback，
   该路径不带档位，保持现状）。

## 9. 风险与未验证清单（立项时已知）

| 项 | 风险 | 兜底 |
|---|---|---|
| OpenAI 官方后端对回传 `reasoning_content` 的容忍度 | 未知字段可能被拒（无 key 验证不了） | 「收到才回传」：官方 chat completions 不返回 reasoning_content，正常使用不会触发；列未验证 |
| anthropic budget 数值约束效果 | 拍定值可能形同虚设或过紧 | DeepSeek 端点参数接受已验证；数值可后续调表不动 UI |
| 真 Anthropic：无 signature 的 thinking block 可能 400 | D5 链路铺好但无 key 复核 | DeepSeek 端点已验证回传可行；列未验证，等 key |
| openai-responses / gemini 档位真机 | 无 key | wire 单测覆盖；列未验证 |
| `reasoning_effort` 单独发（不带 thinking）在 DeepSeek 方言外的兼容后端行为 | 方言判定漏掉自建 DeepSeek 兼容网关（baseUrl 不含 deepseek） | off/max 走降级映射仍可用；用户当前无此场景 |

## 10. 文档同步计划

- 本计划：施工日志逐波追加（日期 + 波次 + 提交 hash + 验收结论 + 偏差记录），状态流转待开始 → 进行中 → 已完成。
- `docs/plans/README.md`：索引行随状态流转。
- `docs/project/当前进度.md`：W4 结项时新增条目、移除待开始项。
- 生成链路多协议适配计划文档不改（历史文档），其「只收不发」表述由本文档 §7 记录替代表。

## 11. 施工日志

### 2026-09-24 W1 修复 openai 回传（`cb4a76d`）

- 改动：`openai-chat.ts` `toOpenAiMessage` assistant 轮输出 `reasoning_content`
  （`AgentMessage.reasoning` 存在即回传）；`session-driver.test.ts` W5「只收不发」断言反转为回传断言
  （测试名同步改为「思考随工具轮回传（DeepSeek 400 修复）」）；`openai-chat.test.ts` 新增 wire 单测
  （工具轮 assistant 带 `reasoning_content`、无思考轮不带该字段）。
- 偏差记录：无（与计划一致）。测试小坑：Response body 只能读一次，多轮请求的 fetch mock
  需用 `mockImplementation` 每次生成新流，`mockResolvedValue` 复用同一 Response 会触发非流式 fallback。
- 验收：421 测试全绿（49 文件）+ typecheck 干净；真机验证（临时测试文件，用后即删）：
  deepseek-flash 思考模型两轮工具循环，第一轮思考 + 工具调用，第二轮回传后 200 并拿到含工具结果的正文
  （修复前该组合第二轮必 400，§2.1 已实测）。

### 2026-09-24 W2 档位参数链路（`a415e82`）

- 类型：`ReasoningEffort` 五档（types/ai.ts）+ `config.llm.reasoningEffort` 可选（零迁移）+
  `AgentLlmInput.reasoningEffort` 四档（default 缺省不传）。
- 四适配器按 D2 映射表落地：openai-chat 方言分支（`isDeepSeekDialect` baseUrl 判定，放 ai/shared）；
  anthropic budget 拍定值 + budget 触顶时抬高 max_tokens（真 Anthropic 要求严格大于预算）；
  gemini thinkingBudget；responses effort（max 降级 high）。
- anthropic signature 三层落盘：`signature_delta` 流解析 → AgentAssistantMessage / AgentAssistantResponse /
  AssistantTextMessage / ChatMessageView 同名透传；续聊无需重建逻辑（ModelView 的 AgentMessage[]
  本就随会话落盘，thinkingSignature 自动跟随）。
- 调用点：query 读 config 下发；compaction 与 streamChatCompletion（要素提取）固定 'off'。
- 偏差记录：无（与计划一致）。gemini 测试断言小坑：default 档下 maxTokens 也未传时整个
  generationConfig 不出现，断言对象应为 body 而非 body.generationConfig。
- 验收：428 测试全绿（+7：D2 映射表逐格 ×4 协议、thinking block 回传、signature 解析、
  档位下发/压缩 off）+ typecheck 干净。

### 2026-09-24 W3 UI 选择器（`6f05ed2`）

- `constants/reasoning-efforts.ts`：五档表 + `reasoningEffortOptions` 协议×方言过滤
  （openai 非 DeepSeek 方言与 openai-responses 隐藏 off 档）；`ReasoningEffortPicker.vue`
  照 PermissionPresetPicker 形态；ChatPanel 工具行与权限选择器并排；
  store `changeReasoningEffort` 走 updateConfig（default 档写 undefined = 字段消失）。
- 配置面补齐：ProjectConfigView.llm / project-fs normalize 增 reasoningEffort 可选字段
  （非法值回退未配置）；`isDeepSeekDialect` 经 services/types re-export 进入 app 合法导入面
  （core package exports 只开 services/* 与 types/*）。
- 偏差记录：无计划级偏差。两个环境坑（W5 已知问题的变体，均已在冒烟中解决）：
  ① 该浏览器 backend 的 evaluate 把字符串按表达式求值（外层括号包裹），
  代码末尾带 `;` 即 SyntaxError「Unexpected token ';'」——注入脚本一律以表达式结尾；
  ② FS shim 的 handle 被 app 存 IndexedDB（recent projects），对象字面量的自有函数属性
  不可 structured-clone——方法必须挂 prototype（本次用 function+prototype 形态）。
- 验收：433 测试全绿（+5 档位表单测）+ typecheck 干净；browser-use 冒烟（FS shim 无头项目，
  预置 DeepSeek 配置）：选择器渲染「默认」档 → 菜单五档可见（方言判定生效）→ 选「最高」
  按钮/落盘 `reasoningEffort: "max"` → 切回「默认」字段从 novel.config.json 消失（零迁移语义）；
  截图视觉确认与权限选择器并排协调、无布局异常。

### 2026-09-24 W4 真机档位验证 + 计划复审 + 结项

- 真机验证（临时测试文件走 streamAgentCompletion 真实链路，用后即删，4 项全过）：
  - openai 端点：off 档 0 思考；low < max 思考长度单调可感（档位真实生效）；
  - openai 端点 max 档 + 工具循环两轮 200，正文含工具结果（回传 × 档位组合）；
  - anthropic 端点：off 档无思考；low 档思考开且 `thinkingSignature` 落盘非空；
  - anthropic 端点 high 档 + 工具循环：回传 thinking block + signature 后 200 拿到正文
    （含 max_tokens 抬高路径 budget 8192 → 12288 的真机确认）。
- **计划复审**（用户点名的结项门禁，逐条核对本文档声明与实现后真实表现）：
  - §2.1 五条事实（默认思考开 / disabled 有效 / low·max 生效 / 不回传 400 / 流式同罪）
    → W1 修复前实测 + W4 修复后对照，全部吻合；
  - §2.2 五条事实（默认思考开带 signature / disabled 有效 / budget 接受 / 不回传宽容 /
    篡改签名不校验）→ 立项实测 + W4 复测吻合；实现选择回传真 thinking block（不依赖宽容行为，
    真 Anthropic 语义下也成立）；
  - D2 映射表逐格 → 四协议 wire 单测全覆盖 + 真机抽验（openai off/low/max、anthropic off/low/high）；
  - D5 signature 三层落盘 → 单测 + W4 anthropic 真机全链（流解析 → 响应 → 回传 200）；
  - D6 辅助请求 off → query.test 断言（压缩 effort='off'、真实请求档位跟随 config）；
  - §6 零迁移 → 冒烟实证（default 档字段从 novel.config.json 消失、旧配置行为不变）。
  - 复审结论：**计划与实现零偏差**；本轮回顾性纠错（DeepSeek 思考模型旧认知）发生在立项前，
    全部真机事实已固化进 §2，施工期间无新发现的计划级错误。

## 12. 未验证清单（定稿）

| 项 | 原因 | 兜底 |
|---|---|---|
| OpenAI 官方后端：回传 `reasoning_content` 的容忍度、off 档隐藏的必要性 | 无 key | 「收到才回传」逻辑上不会触发（官方端点不返回该字段）；off 隐藏依官方文档语义 |
| 真 Anthropic 端点：thinking 验签回传 | 无 key | signature 已落盘并随 thinking block 回传；DeepSeek anthropic 端点全链验证回传可行 |
| gemini 真机：thinkingBudget 映射 | 无 key | wire 单测逐档断言 |
| openai-responses 真机：reasoning.effort 映射 | 无 key | wire 单测逐档断言（含 max 降级） |
| anthropic budget 拍定值的约束曲线（2048/8192/16384 是否恰好对应预期强度） | 简单问题测不出差异 | low/high 真机跑通未截断；调表不动 UI，实测后可改 D2 常量 |
| 生产浏览器会话的档位 UI 真人体验 | 冒烟用 FS shim 无头项目 | 冒烟已覆盖交互/落盘/视觉三面；真实项目路径待日常使用复核 |

## 13. 结项说明

- 四波全部按计划落地，门禁全过：W1 `cb4a76d`（真机 400 修复）、W2 `a415e82`（428 测试）、
  W3 `6f05ed2`（433 测试 + 冒烟四项）、W4（真机四项 + 计划复审零偏差）。
- 零迁移条款兑现：`config.llm.reasoningEffort` 可选字段，旧项目行为不变；normalize 对非法值回退未配置。
- 11 条不做清单全部遵守，无范围蔓延。
- 「只收不发」原则正式退役（§7 替代表兑现）：openai 回传 `reasoning_content`、
  anthropic 回传 thinking block + signature；gemini thoughtSignature（旧有）不动。
