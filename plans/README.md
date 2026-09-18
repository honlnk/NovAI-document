# 计划索引

这个目录用于维护阶段性功能计划和重构计划。

计划文档不使用连续编号，优先使用清晰的中文标题。编号更适合不可频繁改名的决策记录；计划会随着实现、拆分、合并和归档而调整，用状态分组更容易维护。

## 进行中

| 文档 | 状态 | 说明 |
| :--- | :--- | :--- |
| [章节格式调整计划](章节格式调整计划.md) | 基本落地，仍有尾项 | `chapters/*.txt`、旧 `.md` 兼容和要素提取兼容已落地；生成结果结构化处理仍待补齐 |
| [Element 要素体系优化计划](Element要素体系优化计划.md) | 部分落地，继续推进 | `elements/entities/`、`entity` 类型、实体提取、RAG metadata 和 UI 数量展示已落地；要素模板、剧情块拆分、时间线拆分和整理指令仍待补齐 |
| [实施路线图：四份待办的施工波次](实施路线图-四份待办施工波次.md) | 施工中（W0、W1、W2 已完成） | **总路由**：下面四份待办的施工顺序。W0 小毛病清扫 → W1 权限+spill（`path.ts` 同批）→ W2 循环 core（高风险门禁）→ W3 账本 core → W4 service/store 合并 → W5 UI 合并 → W6 历史工具+收尾；排序由三处文件冲突决定（path.ts / session.ts / chat.ts+ChatPanel），每波门禁=测试全绿+文档验收要点+提交+索引状态流转。施工日志见文档末尾 |
| [Agent 循环升级：队列与插话](待办-Agent循环升级-队列与插话.md) | 进行中（W2 core 已完成，`9ceb4ac`） | 照抄 dsh 分层循环：query `for`→`while(true)` + 抽干点（先抽干后压缩）+ steer 续命；双队列收件箱（next-turn 排队/next-step 插话，纯函数 + 随会话落盘、刷新后不自动消费）；session 层 driver 化（入队先于唤醒、停止清闩锁保队列）；`agentMaxTurns` 默认 0 不限、退化为安全阀。UI 跟随：输入框运行中解禁、Enter=排队 / Ctrl+Enter=插话、QueueDock（编辑/删除/立即插话/多条折叠）、steering 气泡与普通用户气泡零视觉区别。inject/blocked/concludesTurn/事件溯源明确不抄。六阶段 S1-S6；UI 阶段与「文件改动追踪与聊天区重设计」的 S5 排期相邻 |

## 待开始

| 文档 | 状态 | 说明 |
| :--- | :--- | :--- |
| [文件改动追踪与聊天区重设计](待办-文件改动追踪与聊天区重设计.md) | 已定稿，待实施 | 四件事一次做掉：①改动账本 changeLedger（事件驱动 append-only 挂会话、随 JSON 落盘，免疫压缩；修复压缩丢清单 + 总结只报一个文件两个 bug，删 `lastWrittenPath`）；②写工具 output 扩展留片段级 diff（抄 dsh before/after 机制）；③UI 三件：聊天区 dsh 风（去头像、工具行按 callId 合一、任务组默认折叠成计数摘要行）、文字版完成总结删除并改造为 zcode 风 diff 面板（change-summary 结构化消息随会话持久化，两者不并存；本期不做撤销）、右下角改会话级总数；④`GetFileChangeHistory` 只读工具按需取回历史（不长期注入）。六个实施阶段 S1-S6，dsh 锚点齐全 |

## 已完成

| 文档 | 状态 | 说明 |
| :--- | :--- | :--- |
| [写工具权限分级 + .novel/ 防护一致性](待办-写工具权限与novel防护一致性.md) | 已完成（W1，`32aac02`） | 五档权限落 `permission.ts`（按动作+路径判定，结构操作除完全访问档外一律弹卡），档位持久化 config + 设置面板「项目设置」下拉；CreateFile/EditFile 补 `.novel/` 写防护 + 大小写绕过修复（断言统一小写比较） |
| [待办：spill 重设计——让溢出内容可取回](待办-spill重设计-让溢出内容可取回.md) | 已完成（W1，`32aac02`） | ReadFile 三道闸（行分页 2000 + 单行 2000 字符 + 50KB 字节封顶，字节口径行粒度截断）→ 退出 spill 白名单；`.novel/spill/` 开 ReadFile 口子（仍禁写删）；省略标记改取回指引；spill 路径豁免二次 spill；7 天启动清理挂 activateProject；prompt.ts 补 spill 说明。三道闸数值照抄 dsh 未微调，留待真实写作测试 |
| [章节命名规范与整理工具计划](章节命名规范与整理工具计划.md) | 已完成 | 统一章节命名为 `第NNN章-标题.txt`，工具层格式校验 + 重号检测，EditFile 对称强制，配套旧项目批量整理工具（预览确认 + 缺标题正文兜底）。注意：本计划是原始「AI 章节整理模块」的重命名子集与前置基建，不涉及 AI 拆分/合并/字数感知，完整形态见文档末尾升级路径 |
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
