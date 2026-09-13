# 参考系切换：Claude Code 转向 DeepSeek Harness

## 记录时间

2026-09-14

## 状态

已采纳，并已落地。Agent Loop 重构 Stage 0-6 全部完成（`abfcbb4` → `323a7b0`，另含开发代理流式修复 `1ed4ce7`），228 个测试全绿，五项手动验收通过。修订 [0002 Claude Code 借鉴与映射设计](0002-ClaudeCode借鉴与映射设计.md) 的实现映射部分。

## 背景

[0002](0002-ClaudeCode借鉴与映射设计.md) 确立了「对话驱动 AI 智能体 + 工具调用 + 文件落地」的产品方向，参照 `/Users/honlnk/project/claude-code`（泄露的反编译产物）做实现映射。执行下来暴露了参照系本身的问题：

1. **来源不稳**：泄露产物无法跟进上游演进，映射只能靠逆向猜。
2. **学了一半**：借了 Claude Code 的形（确认卡、tool-policy 用户约束解析、三条平行执行路径），没学到它的核（上下文压缩、超长结果处理、范围权限），反而沉淀出僵尸配置与双轨执行路径。详见 [core 功能梳理与冗余问题诊断](../plans/core功能梳理与冗余问题诊断.md)。
3. **技术栈错位**：Claude Code 的实现深度绑定其自有运行时，可迁移性差。

同期评估了开源的 DeepSeek Harness（dsh）：MIT 协议、持续更新、单包 TypeScript、浏览器友好的依赖面，与 NovAI 的技术栈高度贴合，且其 agent 内核（compaction / spill / 范围权限 / turn-step loop）正是 NovAI 缺的那一块。

## 决策

**借鉴参照系从 claude-code 切换为 DeepSeek Harness，原则是「抄内核形状，不抄框架与存储」。**

### 抄的形状（已全部重写进 Agent Loop）

| dsh 形状 | NovAI 落点 |
| :--- | :--- |
| 压力触发 + 溢出重试的上下文压缩 | `core/agent/compaction.ts`：8 段检查点摘要模板、保留近期轮、摘要自合并、压缩边界 tool 配对回退 |
| 模型消息序列与显示层分离 | `core/agent/model-view.ts`：ModelView 作为唯一模型消息源，旧 `agentMessages` 自动迁移 |
| 范围权限 | `core/agent/permission.ts`：工作区内静默允许，工作区外一次性确认 |
| 超长结果 spill | `core/agent/spill.ts`：ReadFile/RagSearch 超 8192 字符落盘 `.novel/spill/`，上下文只留头尾预览 |
| turn/step 循环与流式词汇 | `core/agent/query.ts` 重写为 view 驱动；StreamChunk 事件词汇对齐 dsh |
| 轮次上限 | `agentMaxTurns` 可配置，超限优雅收尾 |

### 明确不引入的（非目标）

- Cordis 框架（NovAI 继续 Pinia + 手写服务层）
- 事件溯源存储
- 多协议生成链路（LLM 协议仍走 openai 兼容单通道）
- subagent、plan mode

### 主动偏离 dsh 的两处

1. **token 估算 CJK 感知**：dsh 的 `chars/4` 针对英文，中文会低估约 4 倍导致压缩形同虚设；NovAI 按 CJK 区间 1 字 ≈ 1 token 估算。
2. **不设严格权限模式**：NovAI 是本地单机应用，File System Access API 本身就是沙箱，strict 模式只增加交互摩擦。

## 与 0002 的关系

0002 的**产品方向结论全部保留**（对话驱动、工具调用、文件落地、聊天流是协作界面而非存储）；其**实现映射部分被本决策取代**——实现层以 dsh 内核形状为准。`/Users/honlnk/project/claude-code` 仓库继续保留用于对照研究，不再作为实现映射依据。

## 执行去向

施工细节、阶段验收与偏差记录见 [Agent Loop 重构开发计划](../plans/Agent%20Loop%20重构开发计划.md) 文末「执行结果」一节。
