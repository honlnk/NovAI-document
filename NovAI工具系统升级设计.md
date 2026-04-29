# NovAI 工具系统升级设计

最后更新：2026-04-29

## 一、文档目的

本文档记录 NovAI 工具系统下一阶段的设计共识。

当前 NovAI 已经具备第一版 Agent Loop，并接入了 `ReadFile / EditFile / CreateFile` 三个项目文件工具。但随着产品方向进一步明确，现有工具层已经暴露出几个问题：

- AI 能看到项目文件列表，但缺少可控的文件导航工具。
- `ReadFile` 基本可用，但还没有形成读后状态记录。
- `EditFile` 仍是 MVP 版本，缺少“先读后改”和文件过期保护。
- `CreateFile` 只能新建文件，不能承担完整的工作区文件写入语义。
- 目前没有按名称或路径模式查找文件的能力。
- 目前没有查看某个目录直接结构的工具。

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
- 新建或完整重写文件
- 未来通过 RAG 查找语义相关上下文

## 三、当前实现状态

当前 Agent 真正可调用的工具为：

| 工具 | 当前能力 | 主要问题 |
| :-- | :-- | :-- |
| `ReadFile` | 读取项目内 `.md / .json / .txt` 文件，支持 `offset / limit` | 没有记录读状态，没有读取去重，没有文件版本校验 |
| `EditFile` | 精确替换已有文本片段，支持 `replaceAll` | 没有强制先读后改，没有外部修改检测，diff 能力不足 |
| `CreateFile` | 创建新文本文件，存在则失败 | 不能覆盖已有文件，不适合整章重写或完整重建文件 |

此外，旧的 chat 工具体系中存在 `RagSearch` 和 `Bash` 占位，但它们没有进入当前 Agent schema。

## 四、现有问题分析

### 4.1 AI 有文件清单，但没有导航工具

当前 `buildAgentUserContext` 会把项目中的所有可读文本文件路径列给模型。

这意味着 AI 并不是完全不知道工作区有哪些文件。但这种方式有明显限制：

- 大项目中文件列表会占用大量上下文。
- 模型只能看到路径，不知道目录层级的局部结构。
- 模型无法按需查看某个目录的直接子项。
- 模型无法按名称模式查找文件。
- 模型可能为了确认上下文而读取过多文件。

Claude Code 的方式不是每轮塞完整文件树，而是提供 `Glob / Grep / Read / Bash(ls)` 等工具，让模型先定位，再读取。

NovAI 应借鉴这种“导航工具 + 文件读取”的模式。

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

下一版应把 `EditFile` 升级为真正的“基于已读内容的精确替换工具”。

### 4.4 缺少完整写入工具

`CreateFile` 只能创建新文件，不能覆盖已有文件。

但小说写作中经常会有以下场景：

- 重写某一章全文
- 重建某个角色设定文件
- 覆盖某个 scene prompt
- 根据整理结果生成新的结构化文档

这些不适合全部用 `EditFile` 的 `oldText -> newText` 承担。

应新增 `WriteFile`，用于创建或完整覆盖工作区内文件。但覆盖已有文件时必须要求先 `ReadFile`，并做文件过期保护。

## 五、目标工具集

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

- `ReadFile` 是后续 `EditFile / WriteFile` 的安全前置。
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

### 5.3 `WriteFile`

用途：创建或完整覆盖工作区内文件。

建议能力：

- 只允许写入工作区内相对路径。
- 如果目标不存在，可以直接创建。
- 如果目标已存在，必须先 `ReadFile`。
- 写入前检查目标文件是否在读取后被外部修改。
- 返回 `create / update` 类型。
- 更新已有文件时返回 diff。
- 写入成功后更新 `readFileState`。

设计重点：

- `WriteFile` 取代当前 `CreateFile` 的长期位置。
- `CreateFile` 可以保留为更窄语义，也可以逐步并入 `WriteFile`。
- 对小说场景，`WriteFile` 是“整章重写 / 完整生成要素文件”的关键工具。

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

### 5.6 `RagSearch`

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
| heredoc / echo 写文件 | `WriteFile` |

## 七、推荐工具使用顺序

下一版工具 prompt 应引导模型形成以下工作流：

### 7.1 用户给出明确文件

1. `ReadFile`
2. `EditFile` 或 `WriteFile`
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
3. `EditFile` / `WriteFile`

## 八、与当前文件清单上下文的关系

当前系统会把所有可读文件路径列入用户上下文。

新增 `ListDirectory / FindFiles` 后，应重新评估这件事。

可能的演进方向：

### 方案 A：短期保留全量文件清单

优点：

- 改动小。
- 小项目体验直接。
- 模型可立即知道有哪些文件。

缺点：

- 大项目上下文膨胀。
- 模型仍可能被长清单干扰。

### 方案 B：改为项目摘要 + 工具导航

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

建议：

第一轮实现工具时可以先采用方案 A，待工具稳定后再逐步过渡到方案 B。

## 九、下一步实现优先级建议

优先级建议如下：

1. 新增 `ListDirectory`，替代 Bash `ls` 场景。
2. 新增 `FindFiles`，补齐按名字/路径模式查找文件能力。
3. 为 `ReadFile` 增加 `readFileState`。
4. 升级 `EditFile`，强制先读后改，并检查文件过期。
5. 新增 `WriteFile`，支持创建和完整覆盖。
6. 为 `EditFile / WriteFile` 增加 diff 输出。
7. 重新设计 Agent user context，减少全量文件清单依赖。
8. 后续接入 `RagSearch`，平替内容级搜索。

## 十、总结

NovAI 工具系统的下一阶段，不是追求更高权限，而是追求更好的工具分工与更可靠的文件协作。

核心方向可以概括为：

> 借鉴 Claude Code 的工具工程性，坚持 NovAI 的工作区边界，用小说语义 RAG 替代代码文本搜索。

这一方向下，NovAI 应逐步形成自己的工具组：

- `ListDirectory`
- `FindFiles`
- `ReadFile`
- `EditFile`
- `WriteFile`
- `RagSearch`

这些工具共同支撑的不是传统聊天生成，而是一个真正围绕小说项目文件工作的写作 Agent。
