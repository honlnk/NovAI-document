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
| [AI 功能需求](product/AI功能需求说明书.md) | AI 模块的输入输出、交互逻辑、边界条件和非功能需求 |
| [UI 设计](product/UI设计文档.md) | 页面结构、布局设计和核心交互流程 |

## Architecture

| 文档 | 说明 |
| :--- | :--- |
| [技术架构总览](architecture/技术架构设计.md) | 技术选型、存储架构、项目目录和关键实现要点 |
| [开发前最小契约](architecture/开发前最小契约文档.md) | 最小数据结构、接口协议、文件格式、模块边界和首批接口 |
| [第一阶段 Agent Loop 与工具协议](architecture/Agent会话引擎与工具协议.md) | 会话引擎、消息模型、Agent Loop、默认目标和最小工具协议 |
| [工具系统设计](architecture/工具系统设计.md) | 受控项目文件工具体系、工具边界、使用顺序和后续优先级 |
| [Agent 自主 RAG 工具](architecture/Agent自主RAG工具设计.md) | `RagSearch` 工具、向量库与要素文件关系、Agent 使用策略 |
| [向量索引与重排序](architecture/向量索引与重排序设计.md) | Embedding 文本组装、召回、Rerank、索引失效与解释层 |
| [UI 协作接口契约](architecture/UI协作接口契约设计.md) | core 与 UI 协作边界、services 接口层和 Agent UI 事件协议 |

## Decisions

| 文档 | 说明 |
| :--- | :--- |
| [0001 v1 技术选型](decisions/0001-v1技术选型决策.md) | 第一阶段前端技术栈、样式策略、组件策略与开发顺序 |
| [0002 Claude Code 映射](decisions/0002-ClaudeCode借鉴与映射设计.md) | NovAI 从 Claude Code 借鉴什么、不借鉴什么以及如何映射 |
| [0003 项目总览索引](decisions/0003-项目总览索引决策记录.md) | 是否引入项目总览索引文件的阶段性记录 |

## Plans

| 文档 | 说明 |
| :--- | :--- |
| [计划索引](plans/README.md) | 按进行中、待开始、已完成、已归档维护功能计划和重构计划 |
| [章节格式调整计划](plans/章节格式调整计划.md) | 章节正文使用 `chapters/*.txt` 的适配计划、迁移步骤和兼容要求 |
| [Element 要素体系优化计划](plans/Element要素体系优化计划.md) | `elements/entities/`、要素模板和 Agent 行为规则的优化计划 |

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
