# NovAI 文档中心

本文档目录按用途分为五类：

- `product/`：产品定位、功能需求和 UI 体验设计。
- `architecture/`：长期有效的技术架构、Agent Loop、工具协议、RAG 与 UI 接口契约。
- `decisions/`：已经形成的阶段性决策记录，记录关键取舍、选择理由和边界。
- `plans/`：阶段性功能计划和重构计划，记录目标、步骤、验收点和落地状态。
- `project/`：项目执行文档，记录路线图、MVP、当前进度和开发日志。
- `archive/`：已完成阶段使命、仅保留历史参考价值的文档。

## 推荐阅读顺序

1. [项目当前进度](project/当前进度.md)
2. [项目总览](project/项目总览.md)
3. [产品愿景](product/产品愿景.md)
4. [技术架构总览](architecture/技术架构设计.md)
5. [第一阶段 Agent Loop](architecture/Agent会话引擎与工具协议.md)
6. [工具系统设计](architecture/工具系统设计.md)

## Product

| 文档 | 说明 |
| :--- | :--- |
| [产品愿景](product/产品愿景.md) | 项目定位、问题背景、核心架构、功能模块和阶段建议 |
| [AI 功能需求](product/AI功能需求说明书.md) | AI 模块的输入输出、交互逻辑、边界条件和非功能需求；**目标需求说明书，含大量未实现远期功能，文首有实现状态警示** |
| [UI 设计](product/UI设计文档.md) | 页面结构、布局设计和核心交互流程；§3.3 设置模态框已对齐当前实现，§4 交互流程多为未实现目标设计（文首有警示） |

## Architecture

| 文档 | 说明 |
| :--- | :--- |
| [技术架构总览](architecture/技术架构设计.md) | 技术选型、存储架构、项目目录和关键实现要点；§四目录结构为 monorepo 重构前旧规划（文首有警示） |
| [开发前最小契约](architecture/开发前最小契约文档.md) | 最小数据结构、接口协议、文件格式、模块边界和首批接口；§4 配置契约已对齐当前 `defaults.ts`，目录结构为 monorepo 前旧规划（文首有警示） |
| [第一阶段 Agent Loop 与工具协议](architecture/Agent会话引擎与工具协议.md) | 会话引擎、消息模型、Agent Loop、默认目标和最小工具协议；§十四落地状态已补今日实况，工具清单含 GetFileChangeHistory |
| [工具系统设计](architecture/工具系统设计.md) | 受控项目文件工具体系、工具边界、使用顺序和后续优先级 |
| [Agent 自主 RAG 工具](architecture/Agent自主RAG工具设计.md) | `RagSearch` 工具、向量库与要素文件关系、Agent 使用策略；§八优先级已标注 Orama 落地实况 |
| [向量索引与重排序](architecture/向量索引与重排序设计.md) | Embedding 文本组装、召回、Rerank、索引失效与解释层；rerank.mode 当前固定 text |
| [UI 协作接口契约](architecture/UI协作接口契约设计.md) | core 与 UI 协作边界、services 接口层和 Agent UI 事件协议；§3.5 文件索引与 §12 待办已对齐今日实况 |

## Decisions

| 文档 | 说明 |
| :--- | :--- |
| [0001 v1 技术选型](decisions/0001-v1技术选型决策.md) | 第一阶段前端技术栈、样式策略、组件策略与开发顺序 |
| [0002 Claude Code 映射](decisions/0002-ClaudeCode借鉴与映射设计.md) | NovAI 从 Claude Code 借鉴什么、不借鉴什么以及如何映射 |
| [0003 项目总览索引](decisions/0003-项目总览索引决策记录.md) | 是否引入项目总览索引文件的阶段性记录（已被 0004 推翻） |
| [0004 引入 NovAI 项目总览](decisions/0004-引入NovAI项目总览.md) | 推翻 0003，引入 prompts/NovAI.md 项目级累积记忆 + /生成项目记忆 斜杠命令 |
| [0005 参考系切换至 DeepSeek Harness](decisions/0005-参考系切换至DeepSeekHarness.md) | Agent Loop 重构的参照系从 Claude Code 切换为 DeepSeek Harness，重写 compaction、spill、范围权限、turn/step 循环、StreamChunk 词汇 |

## Plans

| 文档 | 说明 |
| :--- | :--- |
| [计划索引](plans/README.md) | 按进行中、待开始、已完成、已归档维护功能计划和重构计划 |
| [Agent 控制能力补强计划](plans/Agent控制能力补强计划.md) | 写入确认、diff 预览、停止运行、工具约束、system prompt 刷新和结构化文件变更事件 |
| [章节格式调整计划](plans/章节格式调整计划.md) | 章节正文使用 `chapters/*.txt` 的适配计划、迁移步骤和兼容要求 |
| [Element 要素体系优化计划](plans/Element要素体系优化计划.md) | `elements/entities/`、要素模板和 Agent 行为规则的优化计划 |
| [UI 重构计划](plans/UI重构计划.md) | 主工作区从文件树重构为 Activity Bar + 分类导航 + 对话驱动，覆盖场景入口、设置模态框、编辑模式、指令体系和面板拖拽 |
| [章节命名规范与整理工具计划](plans/章节命名规范与整理工具计划.md) | 章节统一命名 `第NNN章-标题.txt`，工具层格式校验 + 重号检测，配套旧项目批量整理工具；含 AI 章节整理完整形态的分级升级路径 |
| [对话输入框 AI 补全计划](plans/对话输入框AI补全计划.md) | 对话输入框 Copilot 式 ghost text 补全：DeepSeek FIM 接口、独立可开关配置、防抖触发、Tab 逐段接受，只服务于提示词编写不碰正文 |
| [设置页优化计划](plans/设置页优化计划.md) | 设置页布局照抄 gpt-image-studio、模型配置借鉴 duet（协议选择 + 获取模型列表）、自动保存替代保存按钮、四面板统一测试连接 |
| [Agent Loop 重构开发计划](plans/Agent%20Loop%20重构开发计划.md) | 参考系从 Claude Code 切换至 DeepSeek Harness 的完整重构计划，Stage 0-6 覆盖 ModelView、压缩、spill、范围权限、轮次安全阀、流式打字机渲染 |
| [core 功能梳理与冗余问题诊断](plans/core功能梳理与冗余问题诊断.md) | 重构前的核心代码诊断：三条平行执行路径、僵尸配置、tool-policy 位置，给出删除清单与「先清理后重写」顺序 |
| [参考系切换：Claude Code 转向 DeepSeek Harness](plans/参考系切换-ClaudeCode转向DeepSeekHarness.md) | 切换论证、对 dsh 的核实、需要重新接地的文档锚点、dsh 借鉴清单（形状/来源/取舍） |
| [实施路线图：四份待办施工波次](plans/实施路线图-四份待办施工波次.md) | W0-W6 波次排序与门禁：W0 小毛病 → W1 权限+spill → W2 循环 core → W3 账本 core → W4 service/store 合并 → W5 UI 合并 → W6 历史工具+收尾 |
| [待办：写工具权限与 .novel/ 防护一致性](plans/待办-写工具权限与novel防护一致性.md) | 写工具权限五档 + `.novel/` 写防护补齐（含大小写绕过修复），W1 波次施工依据 |
| [待办：spill 重设计——让溢出内容可取回](plans/待办-spill重设计-让溢出内容可取回.md) | ReadFile 三道闸（行分页/单行字符/字节封顶）、`.novel/spill/` 可读化、spill 路径豁免二次 spill、7 天清理 |
| [待办：文件改动追踪与聊天区重设计](plans/待办-文件改动追踪与聊天区重设计.md) | 改动账本 changeLedger、写工具 output 带 diff、聊天区 dsh 风折叠 + diff 面板、GetFileChangeHistory 工具（W3/W4/W5/W6 施工依据） |
| [待办：Agent 循环升级——队列与插话](plans/待办-Agent循环升级-队列与插话.md) | 分层循环、双队列收件箱（next-turn 排队 / next-step 插话）、session driver 化、QueueDock、输入框解禁（W2/W4/W5/W6 施工依据） |

## Project

| 文档 | 说明 |
| :--- | :--- |
| [项目执行说明](project/README.md) | 执行类文档的维护规则和使用建议 |
| [项目总览](project/项目总览.md) | 当前阶段、目标和整体状态 |
| [开发路线图](project/开发路线图.md) | 阶段划分、里程碑和优先级 |
| [MVP 清单](project/MVP清单.md) | 第一阶段最小可用产品范围与执行顺序 |
| [当前进度](project/当前进度.md) | 已完成、进行中、待开始和风险 |
| [开发日志](project/开发日志.md) | 按时间记录关键进展、调整和说明 |
| [正式 UI 开发计划](project/UI开发计划.md) | 正式 UI 的阶段计划、页面设计细节和验收标准 |

## Archive

| 文档 | 说明 |
| :--- | :--- |
| [真实创作复盘](archive/真实创作复盘对照.md) | 早期真实创作复盘，核心结论已被 `plans/Element要素体系优化计划.md` 吸收 |

## 维护规则

- 当前事实优先更新 [当前进度](project/当前进度.md)。
- 长期产品方向优先更新 [产品愿景](product/产品愿景.md) 和仓库根目录 `AGENTS.md`。
- 技术方案稳定后放入 `architecture/`。
- 已做出的关键取舍放入 `decisions/`，不要混在进度文档里。
- 阶段性功能计划和重构计划放入 `plans/`。
- 执行状态、路线图、当前进度和开发日志放入 `project/`。
- 已被新文档吸收、但仍有回溯价值的材料放入 `archive/`。
