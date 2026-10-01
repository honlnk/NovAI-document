# 计划索引

这个目录用于维护阶段性功能计划和重构计划。

计划文档不使用连续编号，优先使用清晰的中文标题。编号更适合不可频繁改名的决策记录；计划会随着实现、拆分、合并和归档而调整，用状态分组更容易维护。

## 进行中

| 文档 | 状态 | 说明 |
| :-- | :-- | :-- |
| [Agent 提示词分层与越权修复计划](Agent提示词分层与越权修复计划.md) | 已完工（2026-10-02 四波全部落地；W4 真机回归另修复 config 旧字段残留与 v1 旧默认模板豁免两个旧项目兼容问题） | 依据[越权诊断](../project/Agent越权执行问题诊断.md)对主 Agent 提示词做分层重构：砍掉 system「工作原则」栏只留身份+结构事实（陈述句）、system.md 后置拼接且默认模板未修改不注入、NovAI.md 降为 user-role `<system-reminder>`（层级声明+digest）、大纲随最新 user 消息动态附带不持久化（W3 scenes→outlines 改名含迁移）、user 尾巴删除、工具描述中性化；不做动词白名单/审批状态机/审查 LLM。四波：W1 system 重构+模板+工具描述 → W2 注入机制 → W3 改名迁移 → W4 真机回归。前身草稿「上下文分层注入重构计划」未成稿即被本计划取代（其「运行时快照」想法未纳入） |
| [Element 要素体系优化计划](Element要素体系优化计划.md) | 部分落地，继续推进 | `elements/entities/`、`entity` 类型、实体提取、RAG metadata 和 UI 数量展示已落地；要素模板已由《要素模块缺陷补全计划》落地，剧情块/时间线拆分提取侧已强化、写入侧归并与 Agent 编写要素相关项由《要素园丁子代理设计》承接 |

## 待开始

| 文档 | 状态 | 说明 |
| :-- | :-- | :-- |

## 已完成

| 文档 | 状态 | 说明 |
| :-- | :-- | :-- |
| [要素整理参数化与提取要素退役计划](要素整理参数化与提取要素退役计划.md) | 已完成（2026-10-02 四波同日施工完毕，W1 `2e724ab` / W2 `a57d4ff` / W3 `72a316c` / W4 `0963206`，施工于 worktree 分支 `honlnk/gardener-params`，已合并回 `honlnk/dev`（`7bb09e2` 零冲突，合并后门禁 517 测试 + 双 typecheck 绿）；browser-use 交互冒烟被宿主输入管道故障阻断，运行时模块冒烟通过，交互项留用户复核，见施工日志后记） | 四波：W1 参数弹窗 GardenerTaskDialog（自由文本+快捷选项+章节双列表；斜杠/工具栏按钮→当前会话锚定弹层、侧栏按钮→新会话模态、双击直发；快捷键保持一键直发）→ W2 `.novai/gardener-presets.json` 项目级配置 + `.novai/` 写保护名单 + 设置页「要素整理」编辑区（initialTab 定位跳转）→ W3 斜杠命令落文本新范式 + 发送链路行首前缀拦截替换（GARDENER_TASK_PROMPT/INIT_NOVEL_PROMPT，裸命令不构成有效指令）→ W4 退役 `/提取要素` 全套（-1568 行；含 elementService 双出口、核心提取类型、园丁 persona 约束句改「按指示提取」）。templates.ts 保留为 ELEMENT.md 兜底资产；输入框富文本化留二期 |
| [要素园丁 Phase 2 施工计划](要素园丁Phase2施工计划.md) | 已完成（2026-10-01 三波同日施工完毕，W1 `cfaa0f4` / W2 `ac1f0a6` / W3 `2070c2d`；真机验收项见计划文末未验证清单，留用户日常复核） | 三件套：①园丁独立配置——`runGardener` 合成快照注入覆盖（收紧与放宽双向生效，confirm 闭包方案已否决）；`gardener?` 可选块零迁移（无块=按次动态继承主 Agent 当前配置，运行中定格）；档位裁三档（review/material/full）；思考档随覆盖块；设置页「园丁」tab。②快捷入口——工具栏一键按钮 + 全局 Ctrl/Cmd+Shift+G 与 `/整理要素` 三入口共用 sendMessage 链路，运行中自动排队，零 core 改动。③主 Agent 主动委派——`DelegateToGardener` 描述扩写触发信号/频率节制/先建议后委派（D6），工作原则加收尾回顾软约束（D7），硬节流闸不做视真机另批。各波门禁 487/488 测试 + 双 typecheck + 生产构建。browser-use 冒烟已于 2026-10-01 dev server 恢复后补做（园丁 tab / 双开关展开 / 三档下拉 / 按钮与快捷键触发 / 模态守卫，见计划文末补记）；未验证：test-novel 覆盖端点真机、W3 主动建议行为观察 |
| [知识库直达工具计划](知识库直达工具计划.md) | 已完成（2026-09-30 单波次实现，随 release-v1.4.0 于 2026-10-01 发布上线，工具 `20bc79a`） | 新增 `KnowledgeLookup` Agent 工具：平台映射常量（8 键：维基百科/英文维基/维基词典/维基文库/百度百科/萌娘百科/SCP 中文维基/MDN，URL 模板，扩展调研定稿）+ `provider.fetch` 直达条目全文，实体查询从两步并为一步。未命中最小判定（4xx/维基 Special: 停留）；零 linkseek 改动、零迁移。门禁 471 测试全绿 + 双包 typecheck；8 键生产链路全部实证（维基系五键 2026-10-01 mihomo 恢复后补测通过，en 维基首败为恢复期瞬时抖动、重试即过）；随版附带聊天链接新标签打开与 CI actions 升级消 Node 弃用警告（六 action 升级 + deploy-pages 单步静默，dispatch 重跑零警告）；剩 UI 冒烟待用户真机复核 |
| [linkseek 直连与 CORS 修复计划](linkseek直连与CORS修复计划.md) | 已完成（W1 NovAI `9bebd03` / W2 linkseek `660edd6`，2026-09-29 结项；linkseek 侧随 CI/CD v0.2.0/v0.2.1 上线，NovAI 侧 `release-v1.3.0` 已正式发布（tag 推送触发 Pages 工作流，线上 bundle 验证含新文案）） | 修复「自部署 linkseek」档设计缺陷：`/v1` CORS 白名单把带 Key 的浏览器流量一起拦掉（实证 2026-09-29，Key 与端点本身无问题）。linkseek 侧 CORS 一律回显 Origin、匿名防线收拢 `resolveCaller`、Key 独立突发限流（默认 30/分，不设日配额）；NovAI 侧档位改名「linkseek 直连（API Key）」（枚举值不变零迁移）、预填官方托管地址、双仓引导文案同步。产品重定位：该档真实语义是 BYOK 直连任意实例（含官方托管），Origin 白名单只是作者生产实例的匿名绿灯开关。本地 curl 九项 + 生产四项（原缺陷场景 Key+白名单外 Origin 200）+ test-novel 浏览器真机全过；遗留：管理台总闸抽检未做（不阻塞） |
| [要素园丁 Phase 1 施工计划](要素园丁Phase1施工计划.md) | 已完成（2026-09-25 结项，`3ac5dd2`/`05c1ea0`/`2311afa`） | 依据《要素园丁子代理设计》v3 落地 Phase 1 最小闭环：`prompts/ELEMENT.md` Schema 层（默认六节规范，createProject 初始化 / repairProject 缺失补齐不覆盖 / PromptList 第四组「要素规范」）；园丁子代理运行壳（persona + 要素规范注入 + 六工具白名单 + `elements/` 路径闸，独立 ModelView 复用同一 `query()` 零 Loop 改造，深度固定 1 层）；`DelegateToGardener` 委派工具挂主 Agent 工具面（内层写工具复用同一确认链、档位继承，确认卡与写回面板带「园丁」归属徽标）；`/整理要素` 斜杠命令 + 聊天区嵌套「🌿 园丁整理」任务组。门禁 460 测试 + core/app 双 typecheck + 生产构建；真实 LLM 全流程 2026-10-01 经用户真机复核通过（见计划文末补记） |
| [思考强度选择器与 DeepSeek 思考回传修复计划](思考强度选择器与DeepSeek思考回传计划.md) | 已完成（W1 `cb4a76d` / W2 `a415e82` / W3 `6f05ed2`，2026-09-24 结项） | 修复真机缺陷（openai 协议 DeepSeek 思考模型工具轮回传缺失 `reasoning_content` 即 400，「只收不发」退役为按协议回传：openai 回传 reasoning_content、anthropic 回传 thinking block+signature 且 signature 三层落盘）；输入框思考强度选择器（default/off/low/high/max 五档照 PermissionPresetPicker 形态，DeepSeek 方言判定控制 off 档可见性，四协议 wire 映射表逐格单测；压缩/要素提取固定关思考）。真机验证：双端点档位生效（off 零思考、low<max 单调）、双端点工具循环两轮 200、anthropic signature 全链回传；browser-use 冒烟（五档渲染/落盘 max/切回默认字段消失/视觉协调）；计划复审零偏差。OpenAI 官方 / 真 Anthropic / gemini / responses 无 key 未验证见计划 §12 |
| [生成链路多协议适配计划](生成链路多协议适配计划.md) | 已完成（W1 `773e1a5` / W2 `35d1d5e` / W3 `64b4a7f` / W4 `39407e3` / W5 `6af57cd`，2026-09-24 结项） | 参考 dsh llm 分层（中立词汇 + 协议适配器，pi-ai Node-only 不引入、线协议自写）：`core/llm/protocol/` 落地，openai / anthropic / gemini / openai-responses 四协议全部接入生成链路（query / compaction / 要素提取三调用点）；reasoning 思考流「只收不发」+ ReasoningCollapse 折叠 UI（流式自动展开、正文开始自动收起）。真机验证 DeepSeek 双协议（openai reasoning_content / anthropic thinking，含工具完整往返）与 deepseek-reasoner 思考流 UI（browser-use 冒烟）；gemini / openai-responses 无 key，wire mock 兜底进未验证清单 |
| [内置联网搜索计划](内置联网搜索计划.md) | 已完成（2026-09-24 结项，linkseek `07bba8f` / NovAI `8b1bbee`、`69f2e75`） | linkseek 原生集成（非 MCP）全链落地：linkseek REST 公开端点 `/v1/search`、`/v1/fetch` + 匿名绿灯（每身份 50 次/天、渲染加权计 2，clientId+IP 双闸 + 突发 10/分，仅授权站点 Origin；抓取带质量门控自动渲染升级）；NovAI WebSearch/WebFetch 内置工具（不可信数据前缀 + Sources 引用规范）+ 四档搜索来源配置（托管默认/自部署 linkseek/Exa/Perplexity，设置页「联网搜索」tab）+ 匿名 clientId 注入链。本地端到端实测：真实搜索/抓取、配额边界 48/50/51 与 429 文案透传、FreeUsage 落库核对全过。决策 0006 落档并修订 0002。遗留（见施工日志未验证清单）：托管 baseUrl 占位符待上线替换、生产域名绿灯真机验收、真实 LLM 会话中的工具调用与渲染待用户复核 |
| [要素模块缺陷补全计划](要素模块缺陷补全计划.md) | 已完成（2026-09-23 结项，`ece0b12`/`08e18c5`） | 五类要素模板落地 core 资产（templates.ts，worldbuilding 按计划不出模板）；提取 prompt 注入模板分节、plot 强制按事件拆分、timeline 必带所属阶段与故事内时间、tags 由模型产出 + 兜底合并。门禁 358 测试全绿 + typecheck；真实 LLM 提取效果按预定降级为 mock 结构验证，待日常使用复核 |
| [写回面板文件聚合与计数口径修正计划](写回面板文件聚合与计数口径修正计划.md) | 已完成（2026-09-22 结项，`e604360`/`f3d1811`/`1d3891f`；收尾 `e44e35a`、删除行数 `a82d1d3`） | 写回面板从操作流水改为按文件聚合（净状态徽章 + 片段 diff 列表 + 展开区限高 30 行），计数从净行数差修正为与 DiffLines 同源的 diffLines 真实增删（修 +0−0、+−2 负值、新建尾部换行多计），面板聚合行数与 GetFileChangeHistory 输出均按片段实时重算、历史账本旧口径数字自愈；created→deleted 呈现删除行而非丢弃；删除文件落账 linesRemoved 并进面板/历史工具计数。门禁 336 测试全绿 + typecheck + 生产构建；聚合形态与限高滚动已经用户目检确认 |
| [长会话历史滚动性能计划](长会话历史滚动性能计划.md) | 已完成（2026-09-22 结项，`bb825e9`~`3083294`；修复 `cb39ae7`） | 借鉴 dsh 分层防线全部落地：①core 分页 API（`getSession` 尾部窗口 + `loadOlderMessages` 下标游标，无持久化迁移）；②store 窗口状态（100 条/页、prepend 防重入、run-finish 切片保持窗口）；③ChatPanel 滚动到顶自动翻页 + scrollHeight 锚定；④markdown 共享单例 + 块级渲染（流式只更新尾部块 DOM，消除每 token 整篇重渲）；⑤折叠组 `hidden="until-found"` 可搜索展开。门禁 326 测试全绿 + 双 typecheck + 生产构建；真机验收（翻页锚定/流式滚动/搜索展开）2026-10-01 经用户复核通过 |
| [文件树与打开文件实时刷新计划](文件树与打开文件实时刷新计划.md) | 已完成（2026-09-22 结项，`3c2e73e`；修复批 `8cefe23`） | 两缺口全部收掉：①core 层 `file-changed` 从收尾批量改为运行中按账本高水位实时广播（run-error/停止轮已落账改动也广播）；②内容面板按路径自动重读（renamed 落新路径）。修复批：删除命中当前打开文件时清空面板并提示回收站去向（初版保留快照会成幽灵文件，编辑保存还会静默复活）。门禁 301 测试全绿 + typecheck 干净；真机验收 2026-10-01 经用户复核通过 |

| 文档 | 状态 | 说明 |
| :--- | :--- | :--- |
| [章节格式调整计划](章节格式调整计划.md) | 已完成（2026-09-21 结项） | 五个 Step 全部落地：`chapters/*.txt` 切换、读取/预览适配、写入链路、要素提取兼容、旧项目策略（读取/预览/RenameFile 迁移保留；计数兼容已随 `43275d9` 口径收窄撤销）。衍生产出两条独立线：①章节命名规范与整理工具（强制链路仍在用，机械整理工具已下线）；②生成结果结构化处理（计数刷新已收掉 `86af61a`，自动打开变更文件与章节级元数据留在《当前进度》待开始清单，未立计划） |
| [实施路线图：四份待办的施工波次](实施路线图-四份待办施工波次.md) | 已完成（W0-W6 全部完工，2026-09-19） | **总路由**：四份待办的施工顺序。W0 小毛病清扫 → W1 权限+spill → W2 循环 core → W3 账本 core → W4 service/store 合并 → W5 UI 合并 → W6 历史工具+收尾；每波门禁=测试全绿+文档验收要点+提交+索引状态流转。全程施工日志见文档末尾（含各波偏差记录） |
| [Agent 循环升级：队列与插话](待办-Agent循环升级-队列与插话.md) | 已完成（W2/W4/W5，`9ceb4ac`、`9510aa1`、`4dfbddd`；W6 设置文案 `a02ffe5`） | 照抄 dsh 分层循环全部落地：query `for`→`while(true)` + step 边界抽干点（先抽干后压缩）+ steer 续命；双队列收件箱（next-turn 排队/next-step 插话，纯函数 + 随会话落盘、刷新后不自动消费）；session 层 driver 化（入队先于唤醒、停止清闩锁保队列、pendingWake 兜底停止竞态）；`agentMaxTurns` 默认 0 不限退化为安全阀（设置面板文案「0 表示不限制；仅当模型反复打转时用它兜底」）；UI：输入框运行中解禁、Enter=排队 / Ctrl+Enter=插话（空草稿+插话键=全部逐条插话）、QueueDock（编辑/删除/立即插话/多条折叠）、steering 气泡与普通用户气泡零视觉区别且非组边界。inject/blocked/concludesTurn/事件溯源明确未抄。三场景门禁以全链路集成测试 + W5 browser-use 冒烟验收 |
| [文件改动追踪与聊天区重设计](待办-文件改动追踪与聊天区重设计.md) | 已完成（W3/W4/W5，`ac8006e`、`9510aa1`、`4dfbddd`；W6 历史工具 `a02ffe5`） | 四件事全部落地：①改动账本 changeLedger（事件驱动 append-only 挂会话、随 JSON 落盘，免疫压缩；修复压缩丢清单 + 总结只报一个文件两个 bug，删 `lastWrittenPath`）；②写工具 output 扩展留片段级 diff（EditFile/CreateFile 实际应用文本 + extractChangeDiff）；③UI：聊天区 dsh 风（去头像、工具行按 toolCallId 合一 24px 单行、任务组默认折叠成计数摘要行、steering 非组边界、纯问答轮无折叠）、文字版完成总结删除并改造为 zcode 风 diff 面板（TurnChangesPanel + DiffLines/jsdiff，最新轮展开历史轮折叠、diff 懒展开、change-summary 随会话持久化）、右下角「本会话共修改 N 个文件」；④GetFileChangeHistory 只读工具按需取回历史（新→旧清单 + path/runId/limit 过滤，账本 getter 透传链，不长期注入）。本期未做撤销按钮（账本 diff 已预留）。四画面经 browser-use 冒烟验收 |
| [写工具权限分级 + .novel/ 防护一致性](待办-写工具权限与novel防护一致性.md) | 已完成（W1，`32aac02`） | 五档权限落 `permission.ts`（按动作+路径判定，结构操作除完全访问档外一律弹卡），档位持久化 config + 设置面板「项目设置」下拉；CreateFile/EditFile 补 `.novel/` 写防护 + 大小写绕过修复（断言统一小写比较） |
| [待办：spill 重设计——让溢出内容可取回](待办-spill重设计-让溢出内容可取回.md) | 已完成（W1，`32aac02`） | ReadFile 三道闸（行分页 2000 + 单行 2000 字符 + 50KB 字节封顶，字节口径行粒度截断）→ 退出 spill 白名单；`.novel/spill/` 开 ReadFile 口子（仍禁写删）；省略标记改取回指引；spill 路径豁免二次 spill；7 天启动清理挂 activateProject；prompt.ts 补 spill 说明。三道闸数值照抄 dsh 未微调，留待真实写作测试 |
| [章节命名规范与整理工具计划](章节命名规范与整理工具计划.md) | 已完成（命名规范强制链路仍在用；机械整理工具 2026-09-21 已下线） | 统一章节命名为 `第NNN章-标题.txt`，工具层格式校验 + 重号检测，EditFile 对称强制。配套的旧项目批量整理工具已随 `43275d9` 下线（定位是未来 AI 整理模块的前置基建，该模块暂不开发；实现可回溯 `136a9b0`），ActivityBar「章节整理」回到「即将推出」灰态。本计划是原始「AI 章节整理模块」的重命名子集与前置基建，不涉及 AI 拆分/合并/字数感知，完整形态见文档末尾升级路径 |
| [Agent 控制能力补强计划](Agent控制能力补强计划.md) | 已完成 | 六个 Step 全部落地：Step 1 结构化文件变更（`949d540`）、Step 2/3 写入前确认+diff 预览+暂停/确认/拒绝+旧残留清理（`0707809`/`b0651e7`）、Step 4 停止运行（`8feb3a4`）、Step 5 用户即时工具约束（`e774d2c`）、Step 6 system prompt 同会话刷新（`71c2d42`） |
| [UI 重构计划](UI重构计划.md) | 已完成 | R1-R7 全部落地：Activity Bar + 分类面板、设置模态框、内容面板三态 + 编辑模式、`@场景` 指令、选中内容引用、`/提取要素` 斜杠命令菜单、面板拖拽改宽 |
| [对话输入框 AI 补全计划](对话输入框AI补全计划.md) | 已完成 | Step 1-7 一次提交全部落地（`2d60ecf`）：DeepSeek FIM 流式客户端 + completion 独立配置（仿 rerank 可开关）、`useInlineCompletion` 防抖 / abort / `Intl.Segmenter` 中文分词逐段接受、设置页「输入补全」tab、`GhostTextOverlay` 灰色建议覆盖层、ChatPanel 集成；计划外顺带完成输入框卡片式样式改造。FIM 字段兼容性与延迟体验 2026-10-01 经用户真实使用验证通过 |
| [设置页优化计划](设置页优化计划.md) | 已完成 | 布局照抄 gpt-image-studio（三段式 + 左侧竖排导航 + BaseModal）、模型配置借鉴 duet（协议选择 + 获取模型列表 + 用途过滤，单配置无价格）、自动保存（防抖 600ms + 开关立即 + 关闭 flush + footer 状态）、四个模型面板统一测试连接（core 新增 `models-client.ts` 多协议拉取，DashScope 自动改写 compatible-mode 修复百炼 rerank 拉列表 404）；LLM 协议本期仅配置层，anthropic/gemini 生成适配待另立计划 |
| [core 功能梳理与冗余问题诊断](core功能梳理与冗余问题诊断.md) | 已完成 | 诊断结论已全部执行：三条平行执行路径收敛为单 Loop、僵尸配置六件套清除、tool-policy（正则解析用户约束）删除、FIM 补全保留；三桶处置（真未完成/遗物删除/形状待定）与「先清理后重写」顺序作为 Agent Loop 重构（Stage 0）的输入落地 |
| [参考系切换：Claude Code 转向 DeepSeek Harness](参考系切换-ClaudeCode转向DeepSeekHarness.md) | 已完成 | 决策落地为 [0005 决策记录](../decisions/0005-参考系切换至DeepSeekHarness.md)；dsh 的 compaction、spill、范围权限、turn/step loop、StreamChunk 词汇已重写进 Agent Loop；Cordis/事件溯源/多包/多协议/subagent/plan mode 全程未引入 |
| [Agent Loop 重构开发计划](Agent%20Loop%20重构开发计划.md) | 已完成 | Stage 0-6 全部落地（`abfcbb4`→`323a7b0`，另含代理流式修复 `1ed4ce7`）：ModelView 模型消息源与显示层分离、CJK 感知 token 估算、8 段检查点压缩（阈值+溢出双触发、tool 配对回退、摘要自合并）、工作区范围权限、ReadFile/RagSearch spill、`agentMaxTurns` 可配置、流式打字机渲染；228 测试全绿 + 五项手动验收通过（读改全流程/流式与停止一致/低阈值压缩不失忆/权限拒绝/超长 spill），偏差与决策点见文末「执行结果」 |

## 已归档

暂无。已失去执行价值但仍有回溯意义的材料，应移动到 `../archive/`，并在这里保留归档去向。

## 维护规则

1. 新的大功能、重构或跨模块调整，先在这里建计划文档。
2. 计划正在执行但仍有未完成项，放入“进行中”。
3. 尚未开始、但已经明确要做的计划，放入“待开始”。
4. 所有验收点完成后，移动到“已完成”。
5. 被新文档吸收或不再需要执行的计划，移动到“已归档”，并说明归档位置。
