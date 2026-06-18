# UI 重构计划

最后更新：2026-06-18

## 一、背景与动机

### 1.1 现状

NovAI 正式 UI 已完成第一轮开发（见 [UI 开发计划](../project/UI开发计划.md)），采用「文件树 + AI 对话 + 内容预览」的三栏布局。这轮开发面向的是 IDE 式的工作流，对开发者友好但对小说创作者陌生：

- 左侧直接展示完整文件树，创作者第一时间无从下手
- 设置通过整页路由跳转，割裂感强
- 场景 prompt 注入功能已实现但无任何 UI 入口（只能手改 `novel.config.json`）
- 要素提取按钮藏在内容面板角落，可发现性差，且存在「提取即落盘、写入按钮形同虚设」的交互缺陷
- 输入框只能发纯文本，无法携带上下文引用

### 1.2 目标

把主工作区从「IDE 式文件树」重构为「创作者友好的分类导航 + 对话驱动」模式，贴合 AGENTS.md 定义的产品方向：对话是控制面，文件是存储，UI 是 agent 的操作界面。

重构后用户的核心动线：

```
左侧 Activity Bar 选分类 → 分类列表选目标 → 中间对话框下指令（含 @ 引用 / / 动作）
                                            → 右侧面板查看/编辑/引用内容
```

### 1.3 非目标（本轮明确不做）

- 文件树底部「校对 / 章节整理 / 版本管理」三个假按钮——功能本身暂不做
- 对话历史持久化存储——本轮「对话」分类面板先占位，存储能力后置
- 多章节批量要素提取——本轮保持单章节能力，仅预留扩展
- 富文本编辑器——本轮只提供纯 textarea 编辑
- 移动端的 Activity Bar 布局——移动端交互单独一轮处理

---

## 二、设计决策（已与产品方确认）

| 编号 | 决策点 | 结论 |
|:-----|:-------|:-----|
| D1 | Activity Bar 图标顺序 | 对话优先：对话 → 章节 → 要素 → 提示词 → 设置 |
| D2 | 要素二级结构 | 可折叠分组列表（想法二），6 个分组 UI 写死中文名 + 小图标，不依赖原始英文目录名 |
| D3 | 设置入口 | 移除 `/project/:id/settings` 整页路由，改为模态框（SettingsModal）|
| D4 | 场景 prompt 语义 | `@场景` 选择 = 持久化写入 `activeScenePromptPath`，作为项目级默认，手动切换或关闭前一直生效 |
| D5 | 指令体系规划范围 | 本轮只规划要落地的 `@场景` 和 `/提取要素`，校对/整理等留给对应功能计划 |
| D6 | 指令前缀约定 | `@` = 引用，`/` = 动作。本轮实现：`@场景`（引用）、`/提取要素`（动作）|
| D7 | 可编辑文件范围 | 章节、提示词、要素三类内容可编辑（纯 textarea）。`.novel` 配置和原始 JSON 不进入用户视图 |
| D8 | 选中引用持久性 | 选中内容绑定当前文件，切换文件清空。先做单段 |
| D9 | 面板宽度持久化 | 存 localStorage（界面偏好），不进 config（项目数据）|

---

## 三、整体布局

### 3.1 新布局结构

```
┌────┬──────────────┬─────────────────────────────┬──────────────┐
│ A  │              │                             │              │
│ c  │  分类面板     │       AI 对话面板（主区域）    │   内容面板    │
│ t  │  (随 Bar 切换) │                             │   (预览/编辑)  │
│ i  │              │  [当前场景chip] [选中引用chip] │              │
│ v  │  ──────────   │                             │  [预览|编辑|原始]│
│ i  │  分类列表      │  [消息列表]                  │              │
│ t  │  内容         │                             │  内容区       │
│ y  │              │  [输入框 @场景 /提取要素]      │              │
│    │              │                             │ ◄拖拽手柄     │
│ B  │              │                             │              │
│ a  │              │                             │              │
│ r  │              │                             │              │
└────┴──────────────┴─────────────────────────────┴──────────────┘
 56px    240px              flex-1                320px (可拖拽)
```

### 3.2 Activity Bar（最左竖条，56px）

垂直排列的图标按钮，每个图标 hover 显示中文 tooltip：

| 顺序 | 图标 | tooltip | 点击后分类面板显示 |
|:-----|:-----|:---------|:-------------------|
| 1 | 💬（对话气泡） | 对话 | 对话历史列表（本轮占位：显示「对话历史功能开发中」）|
| 2 | 📖（书本） | 章节 | 章节列表（平铺，读文件结构）|
| 3 | 🏷️（标签） | 要素 | 要素分组折叠列表（6 组写死中文）|
| 4 | 📝（便签） | 提示词 | 系统提示词 + 场景提示词列表（拉平）|
| 5 | ⚙️（齿轮） | 设置 | 打开设置模态框（不切换分类面板）|
| 底部 | 🏠（房屋） | 返回首页 | 关闭当前项目回首页（保留现有行为）|

**行为**：
- 点击图标高亮当前选中项（底部「返回首页」除外，不持久高亮）
- 设置图标是动作型入口，点击打开模态框，不改变分类面板选中态
- 移动端暂沿用现有抽屉方案，Activity Bar 适配留到移动端专项

### 3.3 分类面板（Activity Bar 右侧，240px）

根据 Activity Bar 选中项切换内容：

#### 对话分类（本轮占位）

```
┌──────────────────────┐
│ 对话                  │
├──────────────────────┤
│                      │
│  对话历史功能         │
│  开发中…              │
│                      │
│  （后续将显示历史会话  │
│   列表，支持切换）     │
│                      │
└──────────────────────┘
```

#### 章节分类

```
┌──────────────────────┐
│ 章节            [刷新] │
├──────────────────────┤
│  第001章-初入江湖.txt  │  ← 直接读 chapters/ 目录结构渲染
│  第002章-遇险.txt      │     不加图标，纯文件名
│  第003章-密境遇险.txt  │     点击 → 内容面板打开该文件
│  ...                  │
└──────────────────────┘
```
- 读 `chapters/` 下文件结构，平铺渲染（支持目录折叠，应对「卷/部」结构）
- 选中项高亮，与内容面板当前打开文件联动

#### 要素分类（写死 6 组中文 + 小图标）

```
┌──────────────────────┐
│ 要素            [刷新] │
├──────────────────────┤
│  ▼ 👤 人物 (3)         │  ← elements/characters/
│     林远.md            │
│     苏婉.md            │
│  ▶ 📍 地点 (5)         │  ← elements/locations/
│  ▶ 🔶 其他实体 (2)     │  ← elements/entities/
│  ▶ 📅 时间线 (4)       │  ← elements/timeline/
│  ▶ 📌 情节 (1)         │  ← elements/plots/
│  ▶ 🌐 设定 (6)         │  ← elements/worldbuilding/
└──────────────────────┘
```
- 6 个分组在 UI 写死，标题固定中文 + 图标，括号显示文件数
- 默认折叠，点击展开看该组下的要素文件
- 分组与磁盘目录的映射关系在代码常量里维护（不依赖目录名做中文翻译）

#### 提示词分类（拉平）

```
┌──────────────────────┐
│ 提示词                │
├──────────────────────┤
│  📄 系统提示词          │  ← prompts/system.md
│    （当前激活）         │
│  ─ 场景提示词 ─        │
│  🎬 武侠场景 ●          │  ← prompts/scenes/wuxia.md（●=当前激活）
│  🎬 都市场景            │  ← prompts/scenes/urban.md
└──────────────────────┘
```
- 系统提示词单独一项，恒为激活
- 场景提示词拉平列出 `prompts/scenes/*.md`，当前激活的（`activeScenePromptPath`）打 ● 标记
- 点击场景项 = 切换激活场景（写回 config，提示需新建会话生效）

---

## 四、核心交互设计

### 4.1 指令体系（本轮实现）

输入框支持两类指令前缀，互不冲突：

| 前缀 | 语义 | 本轮指令 | 触发效果 |
|:-----|:-----|:---------|:---------|
| `@` | 引用（带出内容） | `@场景` | 弹出场景选择框，选中后写回 `activeScenePromptPath` 并显示 chip |
| `/` | 动作（执行操作） | `/提取要素` | 弹出章节选择框，选中后触发提取流程 |

#### 4.1.1 `@场景` 场景选择

```
输入框：@场景|
        ┌─────────────────────────┐
        │ 选择场景提示词            │
        ├─────────────────────────┤
        │ 🔍 [筛选场景…]            │
        │  武侠场景      prompts/scenes/wuxia.md │
        │  都市场景      prompts/scenes/urban.md │
        │  ─────────              │
        │  ✓ 不使用场景（关闭）      │
        └─────────────────────────┘
```
- 输入 `@场景` 触发，支持「@场景」后继续输入关键字筛选
- 选中场景后：
  1. 写回 `novel.config.json` 的 `settings.activeScenePromptPath`（持久化）
  2. 输入框上方出现 chip `📍 武侠场景 ×`
  3. 输入框内的 `@场景` 文本被消费清除
  4. 检测当前已有会话时提示「场景已更新，新建会话后生效」
- chip 点 × = 清空 `activeScenePromptPath`（关闭场景）

#### 4.1.2 `/提取要素` 要素提取

```
输入框：/提取要素|
        ┌─────────────────────────┐
        │ 选择要提取的章节（可多选）   │
        ├─────────────────────────┤
        │ 🔍 [筛选章节…]            │
        │  ☐ 第001章-初入江湖       │
        │  ☑ 第002章-遇险           │
        │  ☑ 第003章-密境遇险       │
        │            [取消] [确认提取]│
        └─────────────────────────┘
```
- 输入 `/提取要素` 触发，弹出章节多选框（支持搜索筛选）
- **本轮实现限制**：UI 允许多选，但实际处理先按单章节能力执行（逐章提取，不合并）。文档标注此处为「第一阶段」，多章节合并去重留后续
- 确认后：
  1. 输入框 `/提取要素` 文本作为一条 user 消息发送（带「提取了哪些章节」元信息）
  2. 对话流展示提取进度（如「正在提取第 2/3 章…」）
  3. 提取完成，对话流展示候选要素分组预览
  4. 用户在对话流确认后写入 `elements/*.md`
- **解决已知遗留**：这一步顺带消除了「提取即落盘、写入按钮形同虚设」——候选预览和确认全部在对话流完成，内容面板的提取按钮移除

### 4.2 场景激活可视化

输入框上方常驻显示当前激活场景 chip：

```
┌──────────────────────────────────────────┐
│ 📍 武侠场景 ×          ← 当前激活，点×关闭  │
├──────────────────────────────────────────┤
│ [输入指令... @场景 /提取要素]    [发送]    │
└──────────────────────────────────────────┘
```
- 无激活场景时不显示 chip
- chip 与左侧「提示词分类」列表的 ● 标记双向联动

### 4.3 内容面板三态：预览 / 编辑 / 原始

现有 toggle 按钮改为分段控件：

```
┌──────────────────────────────────────────┐
│ 第003章-密境遇险.txt   [预览|编辑|原始]   │
├──────────────────────────────────────────┤
```

#### 预览模式（现有）
- Markdown 渲染或纯文本只读

#### 编辑模式（新增）
- 纯 `<textarea>`，无富文本
- 适用范围：`chapters/*`、`prompts/*`、`elements/*.md`
- 保存机制：Ctrl+S 手动保存 + 顶部「未保存」标记；失焦不自动保存（避免误触发和性能问题）
- 保存调用 File System Access API 的 writeFile
- 进入编辑模式时以磁盘当前内容初始化，编辑过程中实时标记 dirty

#### 原始模式（现有）
- 等宽字体显示源码

### 4.4 选中内容引用

用户在右侧内容面板用光标选中文本：

```
┌──────────────────────────────────────────┐
│ 第003章-密境遇险.txt   [预览|编辑|原始]   │
├──────────────────────────────────────────┤
│  密境的入口藏在一面断崖之后...            │
│  他推开门，[看见一尊石像]← 用户选中此段    │
│  ...                                      │
├──────────────────────────────────────────┤
│ 📄 第003章 · "看见一尊石像" ×  ← 引用chip │
├──────────────────────────────────────────┤
│ [输入指令...]                  [发送]     │
└──────────────────────────────────────────┘
```

- 监听内容面板的 `selectionchange` / `mouseup`，取选中文本（非空时记录），单段上限 500 字符（超出截断加 `…`），仅预览模式触发
- 输入框上方出现引用 chip：`📄 第003章 · "看见一尊石像" ×`
- 发送消息时，引用作为独立 `quote` 字段透传（不再拼到 `instruction` 前缀），core 层在 `buildAgentUserContext` 注入为独立段：
  ```
  用户意图：{实际输入}

  用户引用的内容：
  {选中文本}
  ```
- 用户消息气泡上方以独立引用块展示（区别于拼进正文的旧方案）
- chip 点 × 清除引用
- **切换文件清空引用**（绑定当前文件，避免跨文件引用混乱）
- 本轮只做单段，多段引用后置

### 4.5 面板拖拽改宽

- 内容面板左侧加拖拽手柄（仅 `lg:` 断点以上显示）
- 拖拽范围：240px ~ 720px
- 宽度持久化到 localStorage（key 如 `novai:panel-width`），不进 config
- 移动端不显示手柄，沿用浮层全屏

### 4.6 设置模态框

- 移除 `/project/:id/settings` 路由
- 现有 `SettingsView.vue` 内容迁移为 `SettingsModal.vue`（模态框组件）
- 由 Activity Bar 的设置图标触发，在主工作区顶层挂载
- 模态框内保留现有四个 tab（LLM / Embedding / Rerank / 项目设置）
- 关闭模态框 = 返回主工作区，无需路由跳转

---

## 五、受影响的现有文件

### 5.1 新增文件

| 文件 | 职责 |
|:-----|:-----|
| `packages/app/src/components/layout/ActivityBar.vue` | 最左竖条导航 |
| `packages/app/src/components/layout/CategoryPanel.vue` | 分类面板容器，按 Activity Bar 选中项切换 |
| `packages/app/src/components/category/ChapterList.vue` | 章节分类列表 |
| `packages/app/src/components/category/ElementList.vue` | 要素分类分组折叠列表 |
| `packages/app/src/components/category/PromptList.vue` | 提示词分类列表 |
| `packages/app/src/components/category/ConversationList.vue` | 对话分类列表（本轮占位）|
| `packages/app/src/components/chat/ComposerCommand.vue` | 输入框指令弹层（@场景 / /提取要素）|
| `packages/app/src/components/chat/SceneChip.vue` | 当前激活场景 chip |
| `packages/app/src/components/chat/SelectionChip.vue` | 选中内容引用 chip |
| `packages/app/src/components/chat/ChapterPicker.vue` | 章节多选弹框 |
| `packages/app/src/components/settings/SettingsModal.vue` | 设置模态框（迁移自 SettingsView）|

### 5.2 重构文件

| 文件 | 改动 |
|:-----|:-----|
| `packages/app/src/views/ProjectView.vue` | 用 ActivityBar + CategoryPanel 替换 FileTreeSidebar；挂载 SettingsModal；管理拖拽宽度状态 |
| `packages/app/src/components/layout/ChatPanel.vue` | 输入框接入 ComposerCommand；头部/输入区接入 SceneChip、SelectionChip |
| `packages/app/src/components/layout/ContentPanel.vue` | 移除要素提取按钮区；预览/原始 toggle 改三态分段控件；新增编辑模式；接入选中引用；接入拖拽手柄 |
| `packages/app/src/app/router.ts` | 移除 `/project/:id/settings` 路由和 SettingsView import |

### 5.3 删除文件

| 文件 | 原因 |
|:-----|:-----|
| `packages/app/src/views/SettingsView.vue` | 内容迁移到 SettingsModal.vue 后删除 |
| `packages/app/src/components/layout/FileTreeSidebar.vue` | 被 ActivityBar + CategoryPanel 取代 |

### 5.4 可复用保留

| 文件 | 说明 |
|:-----|:-----|
| `packages/app/src/components/file-tree/TreeNode.vue` | 章节分类列表如需目录折叠可复用其递归渲染逻辑 |

### 5.5 core 层契约影响

本轮重构**以 app 层为主**，但 R5（选中内容引用）实施时发现 core 层也需要配合改动，原计划「不动 core 的 `buildAgentMessages`」的判断已修正：

- `previewElementExtraction` 的「提取即落盘」行为需拆分（配合 4.1.2 的「预览-确认-写入」流程）。具体是改 core 还是 app 层做二次封装，实施时再定
- **R5 已落地的 core 改动**：选中引用并非在 app 层拼到 `instruction`，而是作为独立 `quote` 字段从 app 一路透传到 core：
  - `core/types/chat.ts`：`UserTextMessage` 与 `ChatTurnInput` 新增可选 `quote?: string`
  - `core/services/types.ts`：`ChatMessageView` 的 user 分支、`RunAgentTurnInput` 新增 `quote?`
  - `core/services/agent-service.ts`：`toChatMessageView` 透传 quote，`runTurn` 在重组 `ChatTurnInput` 时透传 `quote: input.quote`
  - `core/core/chat/session.ts`：`createUserMessage` 接收 quote 并写入消息，`runChatTurn`、`buildAgentMessages` 透传
  - `core/core/agent/prompt.ts`：`buildAgentUserContext` 将引用作为独立段注入（「用户引用的内容：」）
  - 这样做的好处：引用与用户意图在消息模型上解耦，展示层（用户气泡上方的引用块）和模型输入层（prompt 注入）可以独立演化
- 场景 prompt 注入仍由 app 层在调用 agent 前写入 config 的 `activeScenePromptPath`，core 通过读取 config 获取，无需额外透传字段

---

## 六、实施阶段（建议拆分提交）

每个阶段独立可提交、可验证。顺序按依赖关系排列。

### Phase R1：Activity Bar + 分类面板骨架

**目标**：用 ActivityBar + CategoryPanel 替换 FileTreeSidebar，分类切换可工作，数据源复用现有 fileService。

**任务**：
- [x] 新建 ActivityBar.vue（5 图标 + 返回首页）
- [x] 新建 CategoryPanel.vue 容器
- [x] 新建 ChapterList.vue（读 chapters/ 渲染）
- [x] 新建 ElementList.vue（6 组写死中文 + 折叠）
- [x] 新建 PromptList.vue（系统 + 场景拉平，场景激活态联动 config）
- [x] 新建 ConversationList.vue（占位）
- [x] ProjectView 接入新结构，移除 FileTreeSidebar
- [x] 文件选中仍走现有 `projectStore.openFile`，内容面板联动不变

**验收**：左侧可切换 5 个分类，章节/要素/提示词列表正确渲染，点击文件能在右侧打开。

### Phase R2：设置改模态框

**目标**：移除设置路由，改为模态框。

**任务**：
- [x] SettingsView 内容迁移为 SettingsModal.vue
- [x] ActivityBar 设置图标打开模态框
- [x] router.ts 移除 settings 路由
- [x] FirstTimeGuide 的「去配置」改为打开模态框

**验收**：点设置图标弹出模态框，四个 tab 功能完整，关闭返回主工作区，无路由跳转。

### Phase R3：内容面板三态 + 编辑模式

**目标**：预览/原始 toggle 升级为三态分段控件，新增编辑模式。

**任务**：
- [x] ContentPanel toggle 改分段控件
- [x] 新增编辑模式 textarea
- [x] Ctrl+S 保存 + dirty 标记
- [x] 接入 File System Access writeFile
- [x] 可编辑范围校验（章节/提示词/要素）

**验收**：可在编辑模式下修改三类文件并 Ctrl+S 保存到磁盘。

### Phase R4：输入框指令体系（@场景）

**目标**：输入框支持 `@场景` 选择并持久化激活场景。

**任务**：
- [ ] ComposerCommand.vue 指令弹层
- [ ] SceneChip.vue 激活场景可视化
- [ ] `@场景` 触发 → 选择 → 写回 `activeScenePromptPath` → 显示 chip
- [ ] 会话首轮注入限制的切换提示

**验收**：`@场景` 弹层可选，选中后 chip 显示且 config 更新，新建会话生效。

### Phase R5：选中内容引用

**目标**：右侧选中文字可携带为引用。

**任务**：
- [x] ContentPanel 监听 selection
- [x] SelectionChip.vue 引用可视化
- [x] 发送消息时拼接引用前缀
- [x] 切换文件清空引用

**验收**：选中文字后输入框出现引用 chip，发送时消息携带引用，切文件后 chip 清除。

### Phase R6：要素提取斜杠指令

**目标**：`/提取要素` 替代内容面板的提取按钮，流程改为「预览-确认-写入」。

**任务**：
- [ ] ChapterPicker.vue 章节多选弹框
- [ ] `/提取要素` 触发流程
- [ ] 对话流展示进度 + 候选预览
- [ ] 用户确认后写入
- [ ] 拆分 `previewElementExtraction` 的落盘行为（core 契约调整）
- [ ] ContentPanel 移除原要素提取按钮区

**验收**：`/提取要素` 走完整对话流，候选可预览，确认后才落盘，原按钮区已移除。

### Phase R7：面板拖拽改宽

**目标**：内容面板宽度可拖拽调整并持久化。

**任务**：
- [ ] 拖拽手柄组件
- [ ] 240~720px 范围约束
- [ ] localStorage 持久化
- [ ] 移动端不显示手柄

**验收**：拖拽改变宽度，刷新后恢复，移动端无手柄。

---

## 七、验收清单（整体）

### 功能验收
- [x] Activity Bar 5 个分类可切换，返回首页可用
- [x] 章节/要素/提示词列表正确渲染，要素分组显示中文名
- [ ] 场景激活态在 Activity Bar 提示词列表、输入框 chip 两处联动（提示词列表已联动，输入框 chip 待 R4）
- [x] 设置模态框四个 tab 功能完整，无路由跳转
- [x] 内容面板三态切换正常，编辑模式可保存
- [ ] `@场景` 指令完整工作流（R4 待实施）
- [x] 选中引用可携带发送，切文件清空
- [ ] `/提取要素` 完整工作流（预览-确认-写入）（R6 待实施）
- [ ] 面板拖拽改宽并持久化（R7 待实施）

### 体验验收
- [ ] 创作者无需理解文件树即可上手
- [ ] 场景 prompt 功能可见可操作（不再是隐藏功能）
- [ ] 要素提取流程清晰（不再是角落按钮）
- [ ] 无死按钮、无隐藏功能入口

### 代码验收
- [ ] TypeScript 类型完整
- [ ] router 无废弃路由
- [ ] 旧组件（FileTreeSidebar、SettingsView）已清理
- [ ] typecheck / build / test 通过

---

## 八、风险与注意事项

### 8.1 兼容性
- **旧项目 config**：`activeScenePromptPath` 已通过 `normalizeProjectConfig` 自动补全（内存层），磁盘 config 下次保存时补全。本轮无需额外迁移
- **File System Access**：编辑模式依赖 writeFile，仅 Chromium 支持，与现有能力一致

### 8.2 范围控制
- 对话历史存储、多章节批量提取、富文本编辑、移动端 Activity Bar 均明确排除在本轮
- 指令体系只实现 `@场景` 和 `/提取要素`，不提前实现校对/整理指令

### 8.3 依赖
- `previewElementExtraction` 落盘行为拆分（Phase R6）可能涉及 core 契约，实施时需评估影响面
- 场景 chip 与 config 的联动需要 settingsStore 或 projectStore 提供「更新单字段 config」的能力，确认现有 store 是否支持

---

## 九、与现有文档的关系

- 本计划是对 [UI 设计文档](../product/UI设计文档.md) 的**迭代修订**。UI 设计文档描述的「文件树 + 设置页」结构将被本计划的「Activity Bar + 模态框」取代
- 实施完成后，需同步更新 UI 设计文档（`product/UI设计文档.md`）和 UI 开发计划（`project/UI开发计划.md`）中与本次重构冲突的章节
- 本计划不改动 [Agent 控制能力补强计划](Agent控制能力补强计划.md) 和 [Element 要素体系优化计划](Element要素体系优化计划.md) 的范围，但 `/提取要素` 的预览-确认流程会与 Element 计划的「要素模板」有交集，实施时对齐

---

## 十、实施记录

记录各 Phase 实际落地情况、与计划的偏差和后续注意事项。按 Phase 顺序追加，最新在上。

### 2026-06-16：Phase R1 / R2 / R3 / R5 完成

四个 Phase 已合并提交落地（主仓库 `honlnk/dev` 分支）：

| Phase | 主仓库提交 | 摘要 |
|:------|:----------|:-----|
| R1 | `b87270f` | `refactor(layout): 用 Activity Bar 和分类面板替换文件树` |
| R2 | `39a0899` | `refactor(app): 设置入口改为模态框并增强密码输入框` |
| R3 | `4b16168` + `82320ed` | `feat(app): 内容面板新增编辑模式并统一 Tooltip 交互` + Tooltip 显示延迟修复 |
| R5 | `49fde65` | `feat(app): 选中内容引用支持独立引用块展示与模型注入` |

**与计划的偏差和补充决策**：

1. **R2 额外增强**：所有 API Key 输入框（LLM / Embedding / Rerank）统一替换为可复用的 `PasswordInput.vue`，右侧增加眼睛图标切换显隐。这是计划外的体验增强，属于 R2「设置模态框」的顺带改进
2. **R3 三态控件改图标**：预览/编辑/原始三态按钮由文字改为图标（eye / pencil / code），并附 Tooltip（300ms 延迟）。原因是文字标签占用空间过大，图标更紧凑
3. **R5 core 层改动（重要偏差）**：原计划 5.5 节判断「不动 core 的 `buildAgentMessages`」，实施时推翻此判断。引用采用独立 `quote` 字段从 app 透传到 core，理由是让展示层与模型输入层解耦，避免把引用硬拼进 `instruction`。具体改动见 5.5 节
4. **R5 调试遗留**：实施中曾因 core service 层 `runTurn` 漏传 `quote` 导致引用未送达模型，已修复。调试期间在 `agent_run_start` 日志 data 中补了 `quote` 字段便于排查，此增强保留

**待实施 Phase**：R4（`@场景` 指令）、R6（`/提取要素` 斜杠指令）、R7（面板拖拽改宽）。建议顺序 R7 → R4 → R6，先清轻量项再做复杂指令。
