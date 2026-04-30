# NovAI 工具系统升级设计

最后更新：2026-04-30

## 一、文档目的

本文档记录 NovAI 工具系统下一阶段的设计共识。

当前 NovAI 已经具备第一版 Agent Loop，并接入了 `ReadFile / EditFile / CreateFile / RenameFile / DeleteFile / ListDirectory / FindFiles` 七个项目文件工具。但随着产品方向进一步明确，现有工具层仍有几个后续需要补齐的问题：

- `ReadFile` 基本可用，但还没有形成读后状态记录。
- `EditFile` 已强化精确替换能力，但仍缺少工具层强制“先读后改”和文件过期保护。
- `CreateFile` 明确保持窄语义，只新建文件，不覆盖已有文件。
- `RenameFile / DeleteFile` 已补齐基础整理能力，但还没有用户确认与撤销 UI。
- `RagSearch` 尚未作为正式 Agent tool 接入新 Agent Loop。

本设计目标不是把 NovAI 做成通用开发工具，而是借鉴 Claude Code 成熟的工具分工方式，重建一套更适合小说项目工作区的受控工具层。

## 二、核心原则

### 2.1 只操作工作区内部

NovAI 的工具系统必须坚持一个边界：

> AI 只能读写当前小说项目工作区内的文件，不能访问或修改工作区外的内容。

这不是能力缺陷，而是产品定位。

NovAI 面向小说创作，不是通用终端 Agent。用户授权的是一个小说项目目录，AI 的工作范围也应该局限在这个目录里。

### 2.2 借鉴 Claude Code 的工具设计，而不是照搬权限范围

Claude Code 值得借鉴的是：

- 文件读、改、写的清晰分工
- 先定位文件，再读取文件的工作流
- 写入前的读状态校验
- 工具结果结构化
- 通过专用工具替代通用 Shell
- 用工具协议约束模型行为，而不是只依赖 prompt

不应该照搬的是：

- 整机文件系统访问
- Bash / Shell 通用命令执行
- 系统级权限审批
- Git / LSP / MCP 等开发工具链能力

### 2.3 工具是小说项目执行层，不是聊天附属品

NovAI 的聊天界面是 Agent 控制面板，真正的故事资产应该落在文件系统中。

因此工具系统应当成为 NovAI 的核心执行层，负责：

- 查找项目文件
- 查看目录结构
- 读取章节、提示词、要素文件
- 修改已有故事文件
- 新建、重命名、移动或删除项目文件
- 未来通过 RAG 查找语义相关上下文

## 三、当前实现状态

当前 Agent 真正可调用的工具为：

| 工具 | 当前能力 | 主要问题 |
| :-- | :-- | :-- |
| `ListDirectory` | 查看项目内某个已存在目录的直接子项，可选择显示隐藏项 | 只负责目录浏览，不递归读取文件正文 |
| `FindFiles` | 按 glob 路径模式查找文件，支持隐藏项与数量上限 | 不是内容搜索工具，语义搜索未来由 `RagSearch` 承担 |
| `ReadFile` | 读取项目内 `.md / .json / .txt` 文件，支持行号、`offset / limit`、大文件完整读取限制 | 没有记录读状态，没有读取去重，没有文件版本校验 |
| `EditFile` | 精确替换已有文本片段，支持 `replaceAll`，强化重复文本消歧与错误提示 | 没有工具层强制先读后改，没有外部修改检测，diff 能力不足 |
| `CreateFile` | 创建新文本文件，父目录自动创建，存在则失败 | 明确不承担覆盖已有文件职责 |
| `RenameFile` | 重命名或移动单个文本文件，目标存在则失败 | 暂不支持目录移动 |
| `DeleteFile` | 将单个文本文件移入 `.novel/trash` 回收站 | 暂无恢复工具和确认 UI |

旧的 chat 工具体系中的 `Bash` 占位已经移除。`RagSearch` 仍存在旧链路相关实现，但尚未作为正式 Agent tool 接入当前 Agent schema。

## 四、现有问题分析

### 4.1 从全量文件清单改为工具导航

当前已移除向模型注入全量文件清单的做法，改为通过 `ListDirectory / FindFiles / ReadFile` 让模型按需探索。

此前全量文件清单方式存在明显限制：

- 大项目中文件列表会占用大量上下文。
- 模型只能看到路径，不知道目录层级的局部结构。
- 模型无法按需查看某个目录的直接子项。
- 模型无法按名称模式查找文件。
- 模型可能为了确认上下文而读取过多文件。

Claude Code 的方式不是每轮塞完整文件树，而是提供 `Glob / Grep / Read / Bash(ls)` 等工具，让模型先定位，再读取。

NovAI 当前已经开始采用这种“导航工具 + 文件读取”的模式。

### 4.2 `ReadFile` 只完成了读取，没有形成写入保护基础

在 Claude Code 中，`Read` 不只是把文件内容返回给模型，它还会记录读取状态，用于后续 `Edit / Write` 判断：

- 文件是否已经读过
- 读的是完整文件还是局部片段
- 文件读取时的修改时间
- 后续写入前文件是否被外部修改

NovAI 当前 `ReadFile` 没有写入 session-level `readFileState`，因此 `EditFile` 只能依靠 prompt 要求模型先读文件，工具层无法强制。

### 4.3 `EditFile` 缺少协作安全

当前 `EditFile` 的逻辑是：

1. 读取当前文件内容。
2. 查找 `oldText`。
3. 替换为 `newText`。
4. 写回文件。

这适合 MVP，但长期看存在风险：

- 模型没有读过文件也能直接改。
- 如果用户在模型读取后修改过文件，工具无法发现。
- 返回结果只是摘要，没有结构化 diff。
- 对整章级改写不友好。

后续应把 `EditFile` 升级为真正的“基于已读内容的精确替换工具”，在工具层接入 `readFileState` 与文件过期保护。

### 4.4 `WriteFile` 暂缓，不作为当前必需工具

Claude Code 的 `Write` 可以创建或完整覆盖文件，但它依赖“目标已存在时必须完整 Read、写前检查文件是否过期”等安全机制。

NovAI 当前选择暂缓实现 `WriteFile`：

- `CreateFile` 保持低风险新建语义，目标已存在时失败。
- 已有文件的修改优先通过 `ReadFile -> EditFile` 完成。
- 小文件或短章节的完整重写可以先用 `EditFile` 做整段替换。
- 等 `readFileState`、写前过期检查、用户确认和 diff 预览补齐后，再评估是否引入 `WriteFile`。

因此 `WriteFile` 不是当前工具系统的立即目标，而是未来高风险写入能力的候选。

### 4.5 删除与重命名能力已经补齐

由于 NovAI 不再提供 BashTool，Claude Code 中常见的 `rm / mv` 场景已经由专用工具替代：

- `RenameFile`：负责项目内单个文本文件的重命名或移动。
- `DeleteFile`：负责把项目内单个文本文件移入 `.novel/trash` 回收站。

这两个工具都只允许操作 `.md / .json / .txt`，且禁止操作 `novel.config.json` 与 `.novel/**` 内部文件。

## 五、当前目标工具集

### 5.1 `ReadFile`

用途：读取工作区内的文本文件。

建议能力：

- 只允许读取工作区内相对路径。
- 支持 `.md / .json / .txt`，后续可扩展更多文本格式。
- 支持 `offset / limit`。
- 返回带行号内容。
- 返回文件元信息，例如总行数、起止行、是否截断、更新时间。
- 写入会话级 `readFileState`。

设计重点：

- `ReadFile` 是后续 `EditFile` 以及未来可能的 `WriteFile` 的安全前置。
- 局部读取是否允许写入需要谨慎设计。保守方案是：只有完整读取过文件，才允许覆盖或编辑。

### 5.2 `EditFile`

用途：对已有文件做精确文本替换。

建议能力：

- 只允许修改工作区内文件。
- 要求目标文件已经通过 `ReadFile` 读取。
- 写入前检查文件是否在读取后被外部修改。
- `oldText` 必须能唯一匹配，除非显式启用 `replaceAll`。
- 返回结构化 diff 或至少返回可渲染 patch。
- 写入成功后更新 `readFileState`。

设计重点：

- `EditFile` 适合局部修改，不适合整文件重写。
- prompt 中继续要求 `oldText` 来自已读取内容，但工具层必须做强校验。

### 5.3 `CreateFile`

用途：创建工作区内不存在的新文本文件。

当前能力：

- 只允许写入工作区内相对路径。
- 只支持 `.md / .json / .txt`。
- 父目录不存在时自动创建。
- 如果目标文件已存在，直接失败。
- 返回字符数与新增行数。

设计重点：

- `CreateFile` 是低风险新建工具，不承担覆盖已有文件职责。
- 已有文件修改请使用 `ReadFile -> EditFile`。
- 未来是否新增 `WriteFile`，应等读状态、过期保护和确认 UI 补齐后再评估。

### 5.4 `ListDirectory`

用途：查看某个目录下的直接文件结构。

输入建议：

```ts
type ListDirectoryInput = {
  path?: string
  showHidden?: boolean
}
```

语义：

- 不传 `path` 时，查看工作区根目录。
- 传 `path` 时，只查看该目录的直接子项。
- `showHidden` 控制是否显示隐藏文件或隐藏目录。
- 默认不显示隐藏项。
- 返回文件和目录，并标记类型。

建议输出：

```ts
type ListDirectoryOutput = {
  path: string
  entries: Array<{
    name: string
    path: string
    kind: 'file' | 'directory'
    hidden: boolean
  }>
}
```

设计重点：

- 这是 NovAI 对 Claude Code 中 `Bash(ls)` 场景的替代。
- NovAI 不需要 BashTool，但需要安全的目录查看能力。
- 该工具只展示直接子项，不做递归，避免结果爆炸。

### 5.5 `FindFiles`

用途：按文件名或路径模式查找工作区内文件。

输入建议：

```ts
type FindFilesInput = {
  pattern: string
  path?: string
  includeHidden?: boolean
  limit?: number
}
```

语义：

- 类似 Claude Code 的 `Glob`，但命名更贴合 NovAI。
- 用于 `chapters/*.md`、`elements/**/*.md`、`prompts/**/*.md` 等路径匹配。
- 默认从工作区根目录查找。
- 返回匹配文件路径，不读取文件内容。
- 应设置默认结果上限。

设计重点：

- `FindFiles` 解决“根据名字搜索文件”的问题。
- 它不是目录浏览工具；目录浏览由 `ListDirectory` 负责。
- 它也不是内容搜索工具；内容语义搜索未来由 `RagSearch` 负责。

### 5.6 `RenameFile`

用途：重命名或移动工作区内单个文本文件。

输入建议：

```ts
type RenameFileInput = {
  fromPath: string
  toPath: string
}
```

语义：

- `fromPath` 必须存在。
- `toPath` 不能已存在。
- `toPath` 的父目录可以自动创建。
- 只处理 `.md / .json / .txt` 文件，不处理目录。
- 禁止操作 `novel.config.json` 和 `.novel/**`。

设计重点：

- 这是 NovAI 对 Claude Code 中 `mv` 场景的替代。
- 不要用 `CreateFile + DeleteFile` 模拟重命名或移动。

### 5.7 `DeleteFile`

用途：删除工作区内单个文本文件。

输入建议：

```ts
type DeleteFileInput = {
  path: string
}
```

语义：

- `path` 必须存在。
- 只处理 `.md / .json / .txt` 文件，不处理目录。
- 不直接永久删除，而是移动到 `.novel/trash/<timestamp>/<original-path>`。
- 禁止操作 `novel.config.json` 和 `.novel/**`。

设计重点：

- 这是 NovAI 对 Claude Code 中 `rm` 场景的替代。
- 删除必须建立在用户明确意图上。
- 后续可以考虑增加 `RestoreFile` 或回收站 UI。

### 5.8 `RagSearch`

用途：按语义查找与当前创作任务相关的故事上下文。

暂不在本轮实现，但保留为 NovAI 替代 Claude Code `Grep` 的核心工具。

未来 `RagSearch` 应主要面向：

- 相关人物设定
- 相关地点设定
- 相关伏笔与剧情线
- 相关世界观设定
- 近期章节摘要或片段

设计重点：

- Claude Code 的 `Grep` 解决代码中的文本定位。
- NovAI 的 `RagSearch` 应解决小说中的语义定位。
- 不急于实现，但工具协议应为其预留位置。

## 六、明确不做 BashTool

NovAI 不应开发通用 `BashTool`。

原因：

- 小说项目不需要通用 shell 权限。
- Bash 会显著扩大安全边界。
- 需要处理命令注入、路径逃逸、平台兼容、后台任务、输出爆炸等问题。
- 与“只操作工作区内部”的产品原则冲突。
- 用户对小说写作工具的信任来自可控文件操作，而不是系统级自动化。

Claude Code 中 BashTool 的部分使用场景，NovAI 应以专用工具替代：

| Claude Code 场景 | NovAI 替代 |
| :-- | :-- |
| `ls` 查看目录 | `ListDirectory` |
| `find` / `Glob` 找文件 | `FindFiles` |
| `grep` / `rg` 查内容 | 未来 `RagSearch` |
| `cat` / `head` / `tail` 读文件 | `ReadFile` |
| `sed` / `awk` 改文件 | `EditFile` |
| heredoc / echo 写新文件 | `CreateFile` |
| `mv` 重命名或移动文件 | `RenameFile` |
| `rm` 删除文件 | `DeleteFile`，移入 `.novel/trash` |

## 七、推荐工具使用顺序

下一版工具 prompt 应引导模型形成以下工作流：

### 7.1 用户给出明确文件

1. `ReadFile`
2. `EditFile`
3. 总结变更

### 7.2 用户给出目录

1. `ListDirectory`
2. 选择相关文件
3. `ReadFile`
4. 需要时写回

### 7.3 用户给出文件名线索

1. `FindFiles`
2. `ReadFile`
3. 需要时写回

### 7.4 用户给出语义意图

短期：

1. 优先使用当前预览目标
2. 必要时 `ListDirectory` / `FindFiles`
3. 再 `ReadFile`

长期：

1. `RagSearch`
2. `ReadFile` 读取最终需要确认的源文件
3. `EditFile`

### 7.5 用户要求整理文件

- 重命名或移动：使用 `RenameFile`。
- 删除：确认用户意图明确后使用 `DeleteFile`。
- 不确定路径时，先使用 `ListDirectory / FindFiles`。

## 八、与当前文件清单上下文的关系

当前系统已经不再把所有可读文件路径列入用户上下文。

新增 `ListDirectory / FindFiles` 后，推荐方向是“项目摘要 + 工具导航”。

可能的演进方向：

### 已放弃方案：保留全量文件清单

优点：

- 改动小。
- 小项目体验直接。
- 模型可立即知道有哪些文件。

缺点：

- 大项目上下文膨胀。
- 模型仍可能被长清单干扰。

### 当前方案：项目摘要 + 工具导航

用户上下文只提供：

- 当前项目名
- 默认目标
- 顶层目录
- 章节数、要素数等摘要

需要具体文件时，模型通过 `ListDirectory / FindFiles` 探索。

优点：

- 更接近 Claude Code 的工具化工作流。
- 更适合大项目。
- 能降低无意义上下文。

缺点：

- 对工具质量要求更高。
- 模型需要多一步探索。

当前实现已经进入该方向：模型需要具体文件时，应通过 `ListDirectory / FindFiles / ReadFile` 探索。

## 九、下一步实现优先级建议

优先级建议如下：

1. 为 `ReadFile` 增加 `readFileState`。
2. 升级 `EditFile`，强制先读后改，并检查文件过期。
3. 为写入类工具增加执行前确认与 diff / 摘要预览。
4. 评估是否需要 `RestoreFile` 或回收站 UI。
5. 后续接入 `RagSearch`，平替内容级搜索。
6. 等安全机制补齐后，再评估是否新增高风险 `WriteFile`。

## 十、总结

NovAI 工具系统的下一阶段，不是追求更高权限，而是追求更好的工具分工与更可靠的文件协作。

核心方向可以概括为：

> 借鉴 Claude Code 的工具工程性，坚持 NovAI 的工作区边界，用小说语义 RAG 替代代码文本搜索。

这一方向下，NovAI 应逐步形成自己的工具组：

- `ListDirectory`
- `FindFiles`
- `ReadFile`
- `EditFile`
- `CreateFile`
- `RenameFile`
- `DeleteFile`
- `RagSearch`

这些工具共同支撑的不是传统聊天生成，而是一个真正围绕小说项目文件工作的写作 Agent。
