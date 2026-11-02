# 待办：Agent 循环升级——收件箱队列与插话（照抄 dsh 分层循环）

## 记录时间

2026-09-19

## 状态

**已完成（W2/W4/W5/W6，`9ceb4ac` / `9510aa1` / `4dfbddd` / `a02ffe5`，2026-09-19；W6 设置文案 `a02ffe5`）。** 分层循环、双队列收件箱、session driver 化、QueueDock、输入框解禁全部落地；`agentMaxTurns` 默认 0 不限（安全阀语义）。验收记录见 [实施路线图-四份待办施工波次](实施路线图-四份待办施工波次.md) 施工日志。

## 背景：问题与 dsh 的答案

**我们的现状**：

- `query.ts:73` 是 `for (turn = 0; turn < maxTurns; turn++)` 固定上限循环（默认 8，Stage 6 起可配置），循环内用户无法插话；硬顶是唯一失控保护。
- 一次用户发送 = `runServiceTurn` await 一整轮（`stores/chat.ts:187-227`）；运行中输入框被禁用，发送被静默丢弃。
- 停止按钮已有（`chat.ts:230-238` abort 控制器），但停止后想继续说只能重新输入。

**dsh 的答案**（代码级核实，锚点见文末）：

- **三层循环**：driver（`kick()`：`while (await turn())`，靠收件箱判活）→ turn（`while(true)` 跑 step，结局 + next-step 队列空才收）→ step（`while(true)` 仅请求重试）。
- **没有轮次上限**（官方 README 明示 "No built-in turn budget"）。防失控靠数据驱动：显式 cancel、`agent/turn-stopping` 拦截点、流式全程可见。
- **双队列收件箱**：`next-turn`（followup，排队消息，每条独占一个未来 turn）+ `next-step`（steer，插话，下一个 step 边界整批生效；inject 静默注入我们不抄，见「明确不做」）。
- **入队永远先于唤醒**：任何发送都先耐久入队再唤醒 driver，没有"空闲直发"旁路。
- **运行中 UI 全可用**：输入框不禁用；Enter=排队、Ctrl/Cmd+Enter=插话（发送按钮文案随模式变）；输入框上方 QueueDock 列出排队消息，可逐条编辑/删除/立即插话；停止按钮**不清队列**；插话成功的消息与普通用户气泡**零视觉区别**。

## 总体设计

### 1. 收件箱（core，数据层）

双队列挂会话、随会话 JSON 落盘（**不引入事件溯源**，普通字段即可）：

```ts
// packages/core/src/types/chat.ts
export type QueuedMessage = {
  id: string
  text: string
  quote?: string           // 排队时快照的引用内容
  at: string               // ISO 时间
}

export type InboxState = {
  /** followup：排队，每条独占一个未来 turn */
  nextTurn: QueuedMessage[]
  /** steer：插话，下一个 step 边界整批生效 */
  nextStep: QueuedMessage[]
}

// ChatSessionState 新增：
//   inbox?: InboxState   —— 可选字段，旧会话视为空队列，无需迁移
```

- 纯函数操作集（新文件 `core/src/core/chat/inbox.ts`）：`enqueue(state, target, msg)` / `claim(state, target)` / `remove(state, id)` / `replace(state, id, text)` / `clear(state)` / `hasPending(state)`。全部不可变更新，便于测试。
- **claim 语义照抄 dsh**：next-step 永远整队抽干；next-turn 每次 turn 恰取 1 条；批次内顺序 = 先全部 next-step、后那 1 条 next-turn；**抽干不回滚**（claim 即消费）。
- **持久化与恢复**：随 `saveSession` 落盘。**刷新/重开后队列还在但不自动消费**（dsh 崩溃恢复语义）——QueueDock 显示出来，等用户下一次发送时顺带带走。新建/删除会话时队列随之消失（dsh 优雅退出丢弃语义的自然结果）。

### 2. query.ts：for 循环 → 分层循环（core，核心改造）

dsh 三层映射到我们：

- **driver 层** = §3 的 session 层改造；
- **turn 层** = `query()` 的 `for` 改 `while(true)`；
- **step 层** = 现有的溢出重试 `while(true)`（`query.ts:102-151`）**已经是这个形状，保留不动**。

turn 层改造点：

```
query() 主循环（伪代码）：
  step = 0
  loop:
    signal 检查（现有 :77）
    ① 抽干点：claimed = claim(inbox, 'next-step')       ← 新增
       claimed 非空 → 逐条 appendUserMessage(view, 包装后的 user context)
                     + pushMessage 显示层（kind:'steering'，见 §4）
                     + 发 queue-updated 事件
    ② compactIfOverThreshold()（现有 :83，顺序：先抽干再压缩判定，与 dsh 一致——claim 在前、压缩钩子在后）
    请求 → 流式 → 工具（现有逻辑不动）
    step += 1
    结局判定（现有：无 toolCalls = 完成；aborted；error）
    ③ steer 续命：结局已定但 inbox.nextStep 非空 → 不 break，继续 loop   ← 新增（dsh turn 只在 next-step 抽干后才闭合）
    maxTurns 安全阀：agentMaxTurns > 0 且 step 达到上限 → 发 turn-limit-reached → break
```

- **maxTurns 变为可关闭的安全阀**：`agentMaxTurns` 新增 `0 = 不限` 语义，**默认值改为 0**（照抄 dsh 无上限）。设置面板文案改为「0 表示不限制；仅当模型反复打转时用它兜底」。`turn-limit-reached` 事件与现有优雅收尾保留不变。
- **max-tokens 结局（简单版）**：`finishReason === 'max_tokens'` 时发 `model-finish` 后正常收尾该 turn（现有逻辑已自然如此：半截 tool-call 解析失败 → toolCallCount=0 → parse warning → 无工具调用 → done）。本期只在结局事件里透传 finishReason，不额外做半截块丢弃。
- 现有 abort 语义不动：流式中断保留部分内容为 assistant 消息（`query.ts:123-134`）。

### 3. session 层：runChatTurn → driver（core）

`session.ts` 的 `runChatTurn` 改造为 **driver**：一次唤醒连续跑多个 turn，直到收件箱抽干。

- **入口统一走收件箱**（照抄 dsh「入队永远先于唤醒」）：用户发送不再直接开跑，而是 `enqueue(inbox, 'next-turn', msg)` 后 `wakeDriver()`。driver 活着则不重复唤醒（它会在下一抽干点自己取）；driver 空闲则启动。
- **driver 循环**：`while (true)`：claim 本 turn 首批（next-step 全量 + next-turn 1 条）→ 首批为空且无在跑 → 收工；否则逐条 pushMessage 显示层（首条 followup = 普通 user 气泡，是任务组边界；steer 批次 = steering 气泡，不是边界）+ appendUserMessage(modelView) → 跑 query（§2）→ turn 收尾（change-summary 等现有逻辑）→ 检查 `hasPending`，有则开下一 turn。
- **turn 间状态**：`runId` 每 turn 一个新 id（改动账本的 runId 语义不变，每个 turn 各自一张 change-summary 面板）；标题首句截断逻辑（`agent-service.ts:292-294`）保持在第一个含用户消息的 turn 触发。
- **停止语义**：store 的停止 = abort 当前 turn 的控制器 + **清除唤醒闩锁**——当前 turn 结束后 driver 收敛，**队列保留不自动续跑**（dsh `cancel` 后 pending 保留等下次唤醒）。下次用户发送时入队 + 唤醒，旧队列随首批被带走。
- **每 turn 的两条 context-summary**（`session.ts:80-91`）保持每 turn push；steer 续 turn 不新开 turn，不产生新 context-summary。

### 4. service / store 接口

- `runAgentTurn` 拆为两个语义：
  - `enqueueMessage({ projectId, sessionId, text, quote, mode: 'queue' | 'steer' })` → 入队 + 唤醒 driver；**立即返回**（不再 await 整轮）。
  - `updateQueuedMessage({ sessionId, id, action })`：`action = { kind:'edit', text } | { kind:'remove' } | { kind:'steer' }`（把排队消息升级为插话；仅运行中可用，否则报错由 UI toast）。实现映射到 inbox 纯函数 + 唤醒。
  - `stopAgentRun(sessionId)`：abort + 清闩锁（§3）。
- 新事件 `queue-updated { queue: QueuedMessageView[] }`：队列任何变化（入队/抽干/编辑/删除）时广播全量快照（队列很短，全量即可）。view 带 `placement: 'queued' | 'steering'`（next-turn → queued；next-step 中用户消息 → steering）。
- chat store：
  - 新增 `queue: ref<QueuedMessageView[]>`，由 `queue-updated` 同步；切换会话时从 session.inbox 重建。
  - `sendMessage` 改为调 `enqueueMessage`（运行中 mode 由 UI 传入）；`isRunning` 改为由事件驱动（turn 开始/结束），不再由单个 await 的 finally 控制。
  - `abortRun` → `stopAgentRun`，runStatus 文案保留。
  - **发送失败草稿恢复**：enqueue 失败时把文本填回输入框（dsh 的 restoreFailedDrafts 对应物，简单版即可）。

### 5. UI 跟随修改（app）

照抄 dsh 交互（括号内为 dsh 锚点行为）：

1. **输入框运行中解禁**：删 `handleSend` 的 `isSending` 拦截（`ChatPanel.vue:86`）与 `isSending` computed（:58）。运行中可正常打字、可发送。
2. **发送模式**（dsh `resolveSubmitMode`）：
   - 空闲：Enter = 直接发送（走 queue 入队 + 唤醒，效果等同现状）；
   - 运行中：**Enter = 排队**（进 QueueDock），**Ctrl/Cmd+Enter = 插话**（下一 step 边界生效）；Shift+Enter 换行（现状保留）。
   - 发送按钮文案跟随模式：运行中且草稿非空时显示「排队发送」或「插话发送」（dsh InputBar 的模式名按钮）；停止按钮保持独立位置不变。
3. **QueueDock 组件**（新，`components/chat/QueueDock.vue`；dsh QueueDock）：渲染在输入框正上方。
   - 1 条直接显示行；多条默认折叠成「N 条排队消息」计数头 + 展开箭头；空队列不渲染。
   - 每行三个操作：**编辑**（行内 input，Enter 存 Esc 取消）、**删除**（撤回）、**立即插话**（仅运行中可用）。
   - 增强项（dsh 有，顺手做）：空草稿 + Ctrl/Cmd+Enter = 把全部排队消息逐条插话。
4. **steering 消息**：显示层 `UserTextMessage` 新增可选 `steered?: boolean`（或并列新 kind，实施时二选一）；**渲染与普通用户气泡完全相同**（dsh：零视觉区别），仅作为数据标记供分组规则使用——在 [[待办-文件改动追踪与聊天区重设计]] 的任务组折叠里，**steering 消息不是组边界，是组内常显成员（不折叠）**。
5. **运行状态指示**：runStatus 保留；可选增强：运行超过 15 秒追加计时（dsh TurnStatus）。
6. **turn 边界**：followup 被抽干后作为新 turn 开头的普通用户气泡出现，天然分界，不加分隔线（dsh 同）。

### 6. 与既有功能的交互（实施时注意）

- **压缩**：抽干点与压缩判定同处 step 边界，顺序固定为**先抽干（新消息进 modelView）→ 再 `compactIfOverThreshold`**。压缩本身逻辑不动。
- **写权限确认**：确认卡片等待期（awaiting-confirmation）loop 停在 confirm await；此时发送的消息入队，steer 也要等当前 step 结束才生效——语义自洽，UI 无需特判。
- **改动账本/面板**：driver 多 turn 连续时，每 turn 独立 runId、独立 change-summary 面板，与 [[待办-文件改动追踪与聊天区重设计]] 天然兼容；该文档 S5（聊天区改版）的组件与本文档 §5 在同一批文件上施工，**排期上两份文档的 UI 阶段应相邻或合并**，避免 ChatPanel 两次返工。
- **agentEvents 死数组**：`stores/chat.ts:32` 的 `agentEvents` 只累积不消费（review 遗留小毛病），本文档 S4 施工时顺手删除。

## 实施阶段

| 阶段 | 内容 | 涉及层 |
| :--- | :--- | :--- |
| **S1 收件箱数据层** | `QueuedMessage`/`InboxState` 类型、`ChatSessionState.inbox`、`inbox.ts` 纯函数集（enqueue/claim/remove/replace/clear/hasPending）、持久化、单元测试 | core |
| **S2 query 循环改造** | for→while(true)、抽干点（先抽干后压缩）、steer 续命、`agentMaxTurns` 0=不限且默认 0、finishReason 透传 | core |
| **S3 session driver 化** | 入口统一入队+wake、driver 循环、首批 claim（顺序：steer 全量+followup 1 条）、turn 间 runId/闩锁/停止语义 | core |
| **S4 service/store 接口** | `enqueueMessage`/`updateQueuedMessage`/`stopAgentRun`、`queue-updated` 事件、store queue ref 与 isRunning 事件化、失败草稿恢复、删 agentEvents | service + app/store |
| **S5 UI** | 输入框解禁、Enter/Ctrl+Enter 模式与按钮文案、`QueueDock.vue`、steering 气泡标记、（可选）运行计时 | app |
| **S6 收尾** | 设置面板 `agentMaxTurns` 文案与默认值、全文案检查、手动验收 | app + core |

每阶段结束跑 `pnpm test`（基线 228 全绿）并补对应测试：S1 收件箱纯函数全量测；S2 steer 续命/上限安全阀；S3 followup 接续、停止保队列、首批顺序。

## 明确不做

- **inject（静默注入）**：dsh 里它的生产者是插件（plan-mode、host-runner）；我们没有插件系统，没有生产者，不做。
- **blocked 结局 / pre-step reject 钩子 / turn-stopping 插件拦截点**：都是 dsh 的插件扩展位，我们没有插件系统；防失控用「可关闭的 maxTurns 安全阀 + 停止按钮 + 插话」三件套替代。
- **interrupted 结局（崩溃修复补写）**：dsh 靠事件溯源发现未闭合 turn；我们无事件溯源，会话恢复语义维持现状。
- **事件溯源收件箱**（`agent/inbox/spliced` 事件日志）：普通字段随会话 JSON 落盘即可。
- **busyEnter 设置项**（dsh 可配置 Enter 繁忙行为）：固定 Enter=排队、Ctrl/Cmd+Enter=插话；有用户反馈再加。
- **工具 `concludesTurn` 标记**：我们的工具集没有「任务完成」语义的工具（无 todo_write/goal），没有生产者，不做。
- **TurnNavigator 左侧导轨、TurnTail token 用量页脚**：dsh 有但无对应需求。
- 多条 steer 合并：dsh 不合并，各成独立 user 消息，照抄。

## 验收要点

- 运行中输入框可打字可发送：Enter → QueueDock 出现排队行；Ctrl/Cmd+Enter → 下一 step 边界生效（模型后续回复体现插话内容）。
- 排队消息可编辑、可删除、可「立即插话」；多条折叠为计数头；空队列不渲染。
- followup 接续：上一轮跑完后，QueueDock 第一条自动成为新 turn 开头的用户气泡，Agent 继续执行。
- 停止：当前 turn 优雅停止（部分生成内容保留），队列保留且**不自动续跑**；下次发送消息时旧队列随首批被带走。
- 刷新页面：队列仍在 QueueDock 中（持久化），不自动消费。
- 长任务不再被 8 轮打断（默认不限）；设置里把 `agentMaxTurns` 改为有限值后，`turn-limit-reached` 提示仍正常出现。
- 插话/排队与压缩同帧时顺序正确（先抽干后压缩）；确认卡片等待期发送的消息在确认后继续生效。
- `pnpm test` 全绿（含新增测试）。

## dsh 参考锚点（实施时对照阅读）

仓库：`/Users/honlnk/project/deepseek-harness`

- 三层循环主体：`packages/core/agent-loop/src/agent.ts`——driver `kick()` :225-238、turn() :269-350、step() :352-498、抽干点 `preStep`→`inbox.claim` :240-259、steer 续命/turn-stopping :315-320、跨 turn 重置 AbortController :345、followup/steer/inject 入口 :128-147、cancel :149-155
- 收件箱实现：`packages/core/agent-loop/src/inbox.ts`——`ReactLoopInbox`、claim :111-116（next-step 全量+next-turn 1 条）、remove/replace/clear :100-159；契约 `packages/core/agent/src/runtime-types.ts:48-100, 215-241`
- 六种 turn 结局：`packages/core/session/src/types.ts:200-221`（`TurnEndReasonMap`）；"No built-in turn budget"：`packages/core/agent-loop/README.md:200`
- UI 提交策略：`packages/client/ui-conversation/src/client/input/submission-policy.ts:30-39`（resolveSubmitMode）
- 输入框运行中解禁与按钮模式文案：`packages/client/ui-conversation/src/client/skeleton/InputBar.tsx`（:124-128 disabled 逻辑、:354 primaryStops、:358-364 模式 label、:299-301 空草稿 Ctrl+Enter 全部插话）
- QueueDock（编辑/删除/插话三操作、多条折叠、空不渲染）：`packages/client/ui-conversation/src/client/queue/QueueDock.tsx:90-363`
- steering 消息分类（claim 集合交叉判定）与零视觉区别渲染：`packages/client/ui-chat/src/client/conversation-nodes/message.ts:44-90`、`register-node-renderers.ts:20-21`（steering 与 user 共用 `UserMessageNodeView`）
- 行为参考 e2e：`apps/web/tests/steering.e2e.ts`、`live-interactions.e2e.ts`

## 关联

- 排期相邻：[[待办-文件改动追踪与聊天区重设计]]（S5 聊天区改版与本文档 §5 在同一批 UI 文件施工；steering 消息的分组规则已在该文档 §4 白名单思路中预留）
- 同批待修：[[待办-写工具权限与novel防护一致性]]、[[待办-spill重设计-让溢出内容可取回]]
- 前置历史：[[Agent Loop 重构开发计划]]（Stage 6 引入 `agentMaxTurns`，本文档将其改为可关闭的安全阀）
