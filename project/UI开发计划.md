# NovAI 正式 UI 开发计划

最后更新：2026-05-31

## 一、项目背景

### 1.1 当前状态

NovAI 的核心能力已经验证通过：

- ✅ 项目创建/打开/恢复
- ✅ 文件读取/编辑/创建/删除
- ✅ Agent Loop + 工具调用
- ✅ 流式生成
- ✅ 日志系统
- ✅ services + stores 层已就绪

但前端只有两个测试页面（`TestLabView`、`SessionTestView`），缺少正式的用户界面。

### 1.2 目标

开发一个完整的正式 UI，实现 NovAI UI 设计文档中定义的页面结构：

1. **首页**（项目列表）
2. **主工作区**（三栏布局：文件树 + AI 对话 + 内容预览）
3. **设置页**（模型配置 + 项目参数）

### 1.3 参考项目

布局风格参考 `gpt-image-studio`：

- 左侧深色侧边栏（`bg-[#171717]`）
- 中间白色主区域（消息列表 + 固定底部输入框）
- 右侧可折叠面板（内容预览）
- 响应式设计（移动端覆盖式抽屉）

---

## 二、技术方案

### 2.1 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue 3 | ^3.5.13 | 前端框架 |
| TypeScript | ^6.0.2 | 类型安全 |
| Vite | ^6.2.0 | 构建工具 |
| Tailwind CSS | ^4.2.2 | 样式方案 |
| Pinia | ^3.0.4 | 状态管理 |
| Vue Router | ^5.0.4 | 路由管理 |

### 2.2 依赖的后端能力

| Service | 方法 | 用途 |
|---------|------|------|
| `projectService` | `createProject`、`openProject`、`restoreLastProject`、`closeProject` | 项目生命周期 |
| `fileService` | `listFiles`、`readFile`、`refreshFiles` | 文件树和预览 |
| `settingsService` | `getConfig`、`updateConfig`、`testLlm`、`testEmbedding` | 配置管理 |
| `agentService` | `runTurn`、`getSession`、`createSession` | Agent 对话 |
| `chatStore` | - | 消息列表、运行状态 |

### 2.3 目录结构

```
packages/app/src/
├── App.vue                          # 根组件
├── main.ts                          # 入口文件
├── app/
│   └── router.ts                    # 路由配置
├── stores/
│   ├── project.ts                   # 项目状态（已有）
│   ├── settings.ts                  # 设置状态（已有）
│   └── chat.ts                      # 对话状态（已有）
├── views/
│   ├── HomeView.vue                 # 首页（项目列表）
│   ├── ProjectView.vue              # 主工作区（三栏布局）
│   └── SettingsView.vue             # 设置页
├── components/
│   ├── layout/
│   │   ├── FileTreeSidebar.vue      # 左侧文件树
│   │   ├── ChatPanel.vue            # 中间对话面板
│   │   └── ContentPanel.vue         # 右侧内容区
│   ├── home/
│   │   ├── ProjectCard.vue          # 项目卡片
│   │   └── CreateProjectDialog.vue  # 新建项目对话框
│   ├── chat/
│   │   ├── MessageList.vue          # 消息列表
│   │   ├── MessageItem.vue          # 单条消息
│   │   ├── ChatComposer.vue         # 输入区域
│   │   └── ToolCallCard.vue         # 工具调用展示
│   ├── file-tree/
│   │   ├── TreeNode.vue             # 文件树节点
│   │   └── FilePreview.vue          # 文件预览
│   ├── content/
│   │   ├── FileContentView.vue      # 文件内容预览
│   │   ├── GenerationPreview.vue    # 生成预览
│   │   └── EmptyContent.vue         # 空状态
│   ├── settings/
│   │   ├── LlmSettingsPanel.vue     # LLM 配置面板
│   │   ├── EmbeddingSettingsPanel.vue # Embedding 配置面板
│   │   ├── RerankSettingsPanel.vue  # Rerank 配置面板
│   │   └── ProjectSettingsPanel.vue # 项目设置面板
│   └── ui/
│       ├── Button.vue               # 按钮组件
│       ├── Input.vue                # 输入框组件
│       ├── Dialog.vue               # 对话框组件
│       ├── Toast.vue                # 提示组件
│       └── Loading.vue              # 加载组件
└── styles/
    └── scrollbar.css                # 滚动条样式
```

---

## 三、实现步骤

### Phase 1：搭建页面骨架和路由

**目标**：建立基本的页面结构和路由，实现三栏布局的空壳。

**任务清单**：

- [x] 1.1 更新路由配置，添加三个新页面路由
- [x] 1.2 创建 `HomeView.vue` 首页骨架
- [x] 1.3 创建 `ProjectView.vue` 主工作区骨架（三栏布局）
- [x] 1.4 创建 `SettingsView.vue` 设置页骨架
- [x] 1.5 创建布局组件：`FileTreeSidebar.vue`、`ChatPanel.vue`、`ContentPanel.vue`
- [x] 1.6 实现响应式布局（移动端覆盖式抽屉）
- [x] 1.7 添加基础 UI 组件：`Button.vue`、`Input.vue`

**交付标准**：

- 可以在三个页面之间跳转
- 主工作区显示三栏布局（内容可以是占位符）
- 移动端侧边栏可以正常展开/折叠

---

### Phase 2：实现首页（HomeView）

**目标**：实现项目列表页面，支持创建和打开项目。

**任务清单**：

- [x] 2.1 实现 `ProjectCard.vue` 项目卡片组件
- [x] 2.2 实现 `CreateProjectDialog.vue` 新建项目对话框
- [x] 2.3 集成 `projectStore`，显示最近项目列表
- [x] 2.4 实现创建项目功能（调用 `projectService.createProject`）
- [x] 2.5 实现打开项目功能（调用 `projectService.openProject`）
- [x] 2.6 实现恢复上次项目功能（调用 `projectService.restoreLastProject`）
- [x] 2.7 实现空状态引导

**交付标准**：

- 首页显示项目卡片列表
- 可以新建项目、打开项目
- 刷新后可以恢复上次项目
- 空状态时显示引导提示

---

### Phase 3：实现左侧文件树（FileTreeSidebar）

**目标**：实现项目文件树展示和文件选择功能。

**任务清单**：

- [x] 3.1 实现 `TreeNode.vue` 文件树节点组件（支持递归渲染）
- [x] 3.2 集成 `fileService.listFiles`，加载项目文件树
- [x] 3.3 实现文件夹展开/折叠功能
- [x] 3.4 实现文件点击选择功能
- [x] 3.5 实现文件树刷新功能（AI 新建/修改文件后自动更新）
- [x] 3.6 添加功能入口按钮（校对、整理、版本管理、设置）
- [x] 3.7 深色主题样式（参考 gpt-image-studio）

**交付标准**：

- 左侧栏显示项目文件树
- 可以展开/折叠文件夹
- 点击文件可以选中
- 文件树实时反映文件变化

---

### Phase 4：实现中间对话面板（ChatPanel）

**目标**：实现 AI 对话面板，支持消息展示和输入。

**任务清单**：

- [x] 4.1 实现 `MessageList.vue` 消息列表组件
- [x] 4.2 实现 `MessageItem.vue` 单条消息组件（支持用户消息、AI 消息、工具调用）
- [x] 4.3 实现 `ChatComposer.vue` 输入区域组件
- [x] 4.4 实现 `ToolCallCard.vue` 工具调用展示组件
- [x] 4.5 集成 `chatStore`，显示消息列表
- [x] 4.6 实现发送消息功能（调用 `agentService.runTurn`）
- [x] 4.7 实现流式消息展示（监听 `AgentUiEvent`）
- [x] 4.8 实现消息自动滚动到底部
- [x] 4.9 实现 Token 用量指示器（预留）

**交付标准**：

- 中间区域显示消息列表
- 可以输入并发送消息
- AI 响应实时流式展示
- 工具调用过程可观察

---

### Phase 5：实现右侧面板（ContentPanel）

**目标**：实现内容预览面板，支持文件预览和生成预览。

**任务清单**：

- [x] 5.1 实现 `FileContentView.vue` 文件内容预览组件
- [x] 5.2 实现 `GenerationPreview.vue` 生成预览组件
- [x] 5.3 实现 `EmptyContent.vue` 空状态组件
- [x] 5.4 实现面板展开/折叠功能
- [x] 5.5 集成 `fileService.readFile`，加载文件内容
- [x] 5.6 实现 Markdown 渲染预览（可选，第一版可以用纯文本）
- [x] 5.7 实现"预览/原始"切换功能
- [x] 5.8 响应式设计（移动端覆盖式抽屉）

**交付标准**：

- 点击文件时右侧显示文件预览
- AI 生成时右侧显示生成预览
- 可以展开/折叠右侧面板
- 移动端以抽屉形式展示

---

### Phase 6：实现设置页（SettingsView）

**目标**：实现配置管理页面，支持模型配置和项目设置。

**任务清单**：

- [x] 6.1 实现选项卡布局
- [x] 6.2 实现 `LlmSettingsPanel.vue` LLM 配置面板
- [x] 6.3 实现 `EmbeddingSettingsPanel.vue` Embedding 配置面板
- [x] 6.4 实现 `RerankSettingsPanel.vue` Rerank 配置面板
- [x] 6.5 实现 `ProjectSettingsPanel.vue` 项目设置面板
- [x] 6.6 集成 `settingsStore`，加载和保存配置
- [x] 6.7 实现测试连接功能（调用 `settingsService.testLlm` 等）
- [x] 6.8 实现返回主工作区功能

**交付标准**：

- 设置页显示四个选项卡
- 可以配置 LLM、Embedding、Rerank
- 可以测试连接
- 配置保存后持久化到 `novel.config.json`

---

### Phase 7：集成和优化

**目标**：整合所有功能，优化用户体验。

**任务清单**：

- [x] 7.1 实现首次使用引导（内容区显示引导提示）
- [x] 7.2 实现错误处理和 Toast 提示
- [x] 7.3 实现加载状态展示
- [x] 7.4 优化滚动条样式（参考 gpt-image-studio）
- [x] 7.5 优化移动端适配
- [x] 7.6 实现文件树实时更新（AI 操作后刷新）
- [x] 7.7 实现右侧面板自动展开（AI 生成时）
- [x] 7.8 测试和修复 Bug

**交付标准**：

- 所有功能正常工作
- 用户体验流畅
- 移动端适配良好
- 无明显 Bug

---

## 四、页面设计细节

### 4.1 首页（HomeView）

```
┌──────────────────────────────────────────────────┐
│  NovAI（诺艾）                        [全局设置]    │  ← 顶栏
├──────────────────────────────────────────────────┤
│                                                  │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│   │ 项目名称  │  │ 项目名称  │  │           │       │
│   │ 最后编辑  │  │ 最后编辑  │  │  + 新建   │       │
│   │ 章节数    │  │ 章节数    │  │  项目     │       │
│   └──────────┘  └──────────┘  └──────────┘       │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 4.2 主工作区（ProjectView）

```
┌────────────┬─────────────────────────────┬──────────────┐
│            │                             │              │
│   侧边栏    │       AI 对话面板            │  内容区       │
│  (深色)     │      (白色)                 │  (白色)      │
│            │                             │              │
│  [文件树]   │  [消息列表]                  │  [文件预览]   │
│  [功能入口]  │                             │  [生成预览]   │
│            │  [输入区域]                   │              │
│            │                             │              │
└────────────┴─────────────────────────────┴──────────────┘
     256px           flex-1                    320px
```

### 4.3 设置页（SettingsView）

```
┌──────────────────────────────────────────────┐
│  ← 返回项目     项目设置                       │  ← 顶栏
├──────────────────────────────────────────────┤
│  [LLM 配置] [Embedding 配置] [Rerank] [项目]  │  ← 选项卡
├──────────────────────────────────────────────┤
│                                              │
│  （当前选项卡对应的配置表单）                    │
│                                              │
│  API 地址：[________________]                 │
│  API Key：[________________]                 │
│  模型名称：[________________]                 │
│                                              │
│  [测试连接]                    [保存]          │
│                                              │
└──────────────────────────────────────────────┘
```

---

## 五、样式规范

### 5.1 颜色方案

| 用途 | 颜色值 | 说明 |
|------|--------|------|
| 左侧栏背景 | `#171717` | 深灰色 |
| 左侧栏文字 | `text-gray-100` | 浅灰色 |
| 主区域背景 | `white` | 白色 |
| 主区域文字 | `text-gray-900` | 深灰色 |
| 边框 | `border-gray-200` | 浅灰色边框 |
| 悬停状态 | `hover:bg-white/10` | 半透明白色 |
| 选中状态 | `bg-white/10` | 半透明白色 |

### 5.2 间距规范

| 元素 | 间距 |
|------|------|
| 侧边栏内边距 | `px-3 py-3` |
| 主区域内边距 | `px-4 py-6` |
| 消息间距 | `mb-4` |
| 按钮内边距 | `px-3 py-2` |

### 5.3 滚动条样式

参考 gpt-image-studio 的自定义滚动条：

```css
* {
  scrollbar-width: thin;
  scrollbar-color: transparent transparent;
}

*:hover {
  scrollbar-color: rgba(155, 155, 155, 0.4) transparent;
}
```

---

## 六、开发优先级

### 必须完成（MVP 阻塞）

1. Phase 1：搭建页面骨架和路由
2. Phase 2：实现首页
3. Phase 3：实现左侧文件树
4. Phase 4：实现中间对话面板
5. Phase 5：实现右侧面板

### 可以延后

6. Phase 6：实现设置页（可以先复用测试页的配置功能）
7. Phase 7：集成和优化（持续改进）

---

## 七、风险和注意事项

### 7.1 技术风险

- **浏览器兼容性**：File System Access API 仅 Chromium 内核支持
- **性能问题**：大项目文件树可能需要虚拟滚动
- **Markdown 渲染**：第一版可以用纯文本，后续再引入渲染库

### 7.2 开发风险

- **范围蔓延**：严格按 Phase 顺序开发，不要跳步
- **样式细节**：先完成功能，再优化样式
- **测试覆盖**：每个 Phase 完成后进行手动测试

### 7.3 依赖项

- 需要 `@novai/core` 包的 services 层稳定
- 需要 `AgentUiEvent` 事件协议稳定
- 需要 `fileService.listFiles` 返回正确的文件树结构

---

## 八、验收标准

### 8.1 功能验收

- [x] 可以创建、打开、恢复项目
- [x] 可以浏览项目文件树
- [x] 可以与 AI 对话
- [x] 可以查看文件预览
- [x] 可以查看 AI 生成过程
- [x] 可以配置模型参数

### 8.2 体验验收

- [x] 页面加载流畅
- [x] 操作响应及时
- [x] 错误提示清晰
- [x] 移动端适配良好

### 8.3 代码验收

- [x] TypeScript 类型完整
- [x] 组件职责清晰
- [x] 样式规范统一
- [x] 无明显 Bug

---

## 九、后续演进

正式 UI 完成后，可以继续演进以下能力：

1. **写操作确认机制**：在 AI 修改文件前显示 diff 预览
2. **AbortController**：支持停止 Agent 运行
3. **上下文压缩**：对话 Token 用量指示器和手动压缩
4. **校对视图**：集成校对功能到右侧面板
5. **版本管理视图**：集成 Git 版本管理
6. **提示词编辑模式**：提示词调教的专用视图

---

## 十、附录

### 10.1 相关文档

- [UI 设计文档](../UI设计文档.md)
- [UI 协作接口契约设计](../UI协作接口契约设计.md)
- [当前进度](当前进度.md)
- [MVP 清单](MVP清单.md)

### 10.2 参考项目

- gpt-image-studio：布局风格参考
- Claude Code：交互理念参考
- Obsidian：文件树布局参考
