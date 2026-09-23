# 计划索引

这个目录用于维护阶段性功能计划和重构计划。

计划文档不使用连续编号，优先使用清晰的中文标题。编号更适合不可频繁改名的决策记录；计划会随着实现、拆分、合并和归档而调整，用状态分组更容易维护。

## 进行中

| 文档 | 状态 | 说明 |
| :--- | :--- | :--- |
| [Element 要素体系优化计划](Element要素体系优化计划.md) | 部分落地，继续推进 | `elements/entities/`、`entity` 类型、实体提取、RAG metadata 和 UI 数量展示已落地；要素模板已由《要素模块缺陷补全计划》落地，剧情块/时间线拆分提取侧已强化、写入侧归并与 Agent 编写要素相关项（行为约束、整理指令）随方案重估暂缓 |
| [生成链路多协议适配计划](生成链路多协议适配计划.md) | 进行中（W1 完工 `773e1a5`，2026-09-24） | 参考 dsh llm 分层（中立词汇 + 协议适配器，pi-ai Node-only 不引入、线协议自写）：W1 协议层地基已落地——`agent/llm.ts` 重构为协议分发（签名不变），openai 迁入适配器并补 reasoning_content 解析，协议链路经 query/compaction/要素提取三处接通，394 测试全绿 + DeepSeek 真机回归通过。余 W2 anthropic（真机经 `api.deepseek.com/anthropic`）、W3 gemini、W4 openai-responses（无 key 以 wire mock 兜底）、W5 思考流 UI（deepseek-reasoner 真机）、W6 收尾 |

## 待开始

暂无。

## 已完成

| 文档 | 状态 | 说明 |
| :--- | :--- | :--- |
| [内置联网搜索计划](内置联网搜索计划.md) | 已完成（2026-09-24 结项，linkseek `07bba8f` / NovAI `8b1bbee`、`69f2e75`） | linkseek 原生集成（非 MCP）全链落地：linkseek REST 公开端点 `/v1/search`、`/v1/fetch` + 匿名绿灯（每身份 50 次/天、渲染加权计 2，clientId+IP 双闸 + 突发 10/分，仅授权站点 Origin；抓取带质量门控自动渲染升级）；NovAI WebSearch/WebFetch 内置工具（不可信数据前缀 + Sources 引用规范）+ 四档搜索来源配置（托管默认/自部署 linkseek/Exa/Perplexity，设置页「联网搜索」tab）+ 匿名 clientId 注入链。本地端到端实测：真实搜索/抓取、配额边界 48/50/51 与 429 文案透传、FreeUsage 落库核对全过。决策 0006 落档并修订 0002。遗留（见施工日志未验证清单）：托管 baseUrl 占位符待上线替换、生产域名绿灯真机验收、真实 LLM 会话中的工具调用与渲染待用户复核 |
| [要素模块缺陷补全计划](要素模块缺陷补全计划.md) | 已完成（2026-09-23 结项，`ece0b12`/`08e18c5`） | 五类要素模板落地 core 资产（templates.ts，worldbuilding 按计划不出模板）；提取 prompt 注入模板分节、plot 强制按事件拆分、timeline 必带所属阶段与故事内时间、tags 由模型产出 + 兜底合并。门禁 358 测试全绿 + typecheck；真实 LLM 提取效果按预定降级为 mock 结构验证，待日常使用复核 |
| [写回面板文件聚合与计数口径修正计划](写回面板文件聚合与计数口径修正计划.md) | 已完成（2026-09-22 结项，`e604360`/`f3d1811`/`1d3891f`；收尾 `e44e35a`、删除行数 `a82d1d3`） | 写回面板从操作流水改为按文件聚合（净状态徽章 + 片段 diff 列表 + 展开区限高 30 行），计数从净行数差修正为与 DiffLines 同源的 diffLines 真实增删（修 +0−0、+−2 负值、新建尾部换行多计），面板聚合行数与 GetFileChangeHistory 输出均按片段实时重算、历史账本旧口径数字自愈；created→deleted 呈现删除行而非丢弃；删除文件落账 linesRemoved 并进面板/历史工具计数。门禁 336 测试全绿 + typecheck + 生产构建；聚合形态与限高滚动已经用户目检确认 |
| [长会话历史滚动性能计划](长会话历史滚动性能计划.md) | 已完成（2026-09-22 结项，`bb825e9`~`3083294`；修复 `cb39ae7`） | 借鉴 dsh 分层防线全部落地：①core 分页 API（`getSession` 尾部窗口 + `loadOlderMessages` 下标游标，无持久化迁移）；②store 窗口状态（100 条/页、prepend 防重入、run-finish 切片保持窗口）；③ChatPanel 滚动到顶自动翻页 + scrollHeight 锚定；④markdown 共享单例 + 块级渲染（流式只更新尾部块 DOM，消除每 token 整篇重渲）；⑤折叠组 `hidden="until-found"` 可搜索展开。门禁 326 测试全绿 + 双 typecheck + 生产构建；真机验收（翻页锚定/流式滚动/搜索展开）未执行，待用户复核 |
| [文件树与打开文件实时刷新计划](文件树与打开文件实时刷新计划.md) | 已完成（2026-09-22 结项，`3c2e73e`；修复批 `8cefe23`） | 两缺口全部收掉：①core 层 `file-changed` 从收尾批量改为运行中按账本高水位实时广播（run-error/停止轮已落账改动也广播）；②内容面板按路径自动重读（renamed 落新路径）。修复批：删除命中当前打开文件时清空面板并提示回收站去向（初版保留快照会成幽灵文件，编辑保存还会静默复活）。门禁 301 测试全绿 + typecheck 干净；真机验收未执行，待用户复核 |

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
| [对话输入框 AI 补全计划](对话输入框AI补全计划.md) | 已完成 | Step 1-7 一次提交全部落地（`2d60ecf`）：DeepSeek FIM 流式客户端 + completion 独立配置（仿 rerank 可开关）、`useInlineCompletion` 防抖 / abort / `Intl.Segmenter` 中文分词逐段接受、设置页「输入补全」tab、`GhostTextOverlay` 灰色建议覆盖层、ChatPanel 集成；计划外顺带完成输入框卡片式样式改造。FIM 字段兼容性与延迟体验待真实使用验证 |
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
