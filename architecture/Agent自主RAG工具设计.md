# NovAI Agent 自主 RAG 工具设计

最后更新：2026-06-07

## 一、文档目的

本文档记录 NovAI 下一阶段 RAG 工具链的产品与技术设计共识。

当前 NovAI 已经完成第一版文件工具 Agent Loop，模型可以通过 `ReadFile / EditFile / CreateFile / RenameFile / DeleteFile / ListDirectory / FindFiles` 读写小说项目文件。RAG 相关代码已经具备向量索引构建、Embedding 检索、Rerank 调试等基础能力，并且 `RagSearch` 已作为 Agent 可主动调用的正式工具进入主工作流。

本阶段要解决的问题是：

> RAG 不应该只是系统每轮自动塞给模型的背景材料，而应该成为 Agent 可以主动选择、主动调用、主动追问的小说记忆工具。

因此，本文档明确采用“方案 A”：**由 Agent 自主决定什么时候调用 `RagSearch`，而不是系统在每一轮对话开始前固定自动检索。**

截至 2026-06-07，该方向已完成 MVP 基础接入：正式设置页可查看并手动重建 RAG 索引，新的 Agent Loop 已能暴露 `RagSearch` 工具。后续重点转为验证真实创作任务中的触发率、召回质量与上下文使用策略。

## 二、核心决策

### 2.1 采用 Agent 主动工具调用

NovAI 的 RAG 工具设计采用以下模式：

1. 用户用自然语言表达创作意图。
2. Agent 根据任务判断是否需要历史设定、人物状态、地点信息、伏笔、时间线等上下文。
3. 如果需要，Agent 主动调用 `RagSearch`。
4. `RagSearch` 返回语义相关的要素摘要与来源路径。
5. Agent 必要时继续调用 `ReadFile` 读取完整要素文件。
6. Agent 基于检索结果和项目文件执行生成、修改或整理。

也就是说，`RagSearch` 在 NovAI 中不是后台自动补料机制，而是和 `ReadFile`、`FindFiles` 一样的正式工具。

### 2.2 不采用默认自动预检索

暂不采用“每轮用户输入后，系统自动先跑一次 RAG，再把结果注入模型上下文”的方案。

原因如下：

- NovAI 的产品方向是 Claude Code / Vibe Coding 风格的 Agent 工作流，而不是传统聊天增强。
- 自动预检索会让模型被动接收上下文，削弱“自主判断何时用工具”的能力。
- 自动注入容易把无关要素塞进上下文，干扰小说创作判断。
- 用户很难区分哪些内容是 Agent 主动查到的，哪些内容是系统隐式塞入的。
- 自动检索会增加每轮调用成本，即使任务并不需要 RAG。

后续可以保留自动预检索作为高级模式或兜底策略，但不作为当前 MVP 主路线。

### 2.3 RAG 是小说记忆工具，不是普通文本搜索

`RagSearch` 的职责不是替代 `FindFiles` 或全文搜索。

工具分工如下：

| 工具            | 作用                                                       |
| :-------------- | :--------------------------------------------------------- |
| `FindFiles`     | 按路径和文件名模式查找项目文件                             |
| `ListDirectory` | 查看目录结构                                               |
| `ReadFile`      | 读取明确路径的文件内容                                     |
| `RagSearch`     | 根据语义查找相关人物、实体、地点、情节、时间线、世界观要素 |

`RagSearch` 解决的是小说创作中的语义问题，例如：

- “这个人物上次出现时是什么状态？”
- “黑风谷相关伏笔有哪些？”
- “这条时间线前面埋过什么冲突？”
- “续写这一章前需要回忆哪些设定？”

## 三、目标体验

理想情况下，用户可以这样输入：

```text
续写下一章，重点延续林远和黑风谷的伏笔，语气保持压抑一点。
```

Agent 应该能够自主执行类似流程：

```text
1. ListDirectory(elements)
2. RagSearch("林远 黑风谷 伏笔 当前状态")
3. ReadFile(elements/characters/林远.md)
4. ReadFile(elements/locations/黑风谷.md)
5. ReadFile(elements/plots/黑风谷伏笔.md)
6. CreateFile(chapters/第002章.txt)
```

用户看到的不是一段凭空生成的回答，而是一次可追踪的文件化创作过程：

- Agent 查了哪些故事要素
- 使用了哪些文件
- 写入或修改了哪些章节
- 后续哪些要素需要更新

## 四、`RagSearch` 工具设计

### 4.1 工具定位

`RagSearch` 是 Agent 的语义检索工具。

它不会直接把 `elements/**/*.md` 全部读入模型上下文，而是查询由这些要素文件生成的向量库。向量库中的每条记录包含 embedding 向量和元数据，例如 `sourcePath / type / name / summary / tags / contentHash`。

真实数据源始终是 `elements/` 下的 Markdown 文件；向量库只是可重建的检索缓存。当要素文件变化、Embedding 模型变化或检索文本模板变化时，对应向量记录需要失效并重建。

### 4.2 输入结构

当前 MVP 输入保持克制：

```ts
type RagSearchInput = {
  query: string;
  topK?: number;
  finalLimit?: number;
  filters?: {
    type?: Array<
      | "character"
      | "entity"
      | "location"
      | "timeline"
      | "plot"
      | "worldbuilding"
    >;
    tags?: string[];
    lastUpdatedChapter?: string;
  };
};
```

字段说明：

| 字段                         | 说明                                                  |
| :--------------------------- | :---------------------------------------------------- |
| `query`                      | 必填，语义检索查询，应该描述当前创作任务需要回忆什么  |
| `topK`                       | 可选，最多召回多少条候选，默认使用项目配置            |
| `finalLimit`                 | 可选，最终返回给 Agent 的上下文条数，默认使用项目配置 |
| `filters.type`               | 可选，限定要素类型                                    |
| `filters.tags`               | 可选，限定标签                                        |
| `filters.lastUpdatedChapter` | 可选，限定最后关联章节                                |

### 4.3 输出结构

工具返回应兼顾模型可用性和 UI 可解释性。当前实现采用以下结构：

```ts
type RagSearchOutput = {
  query: string;
  recalledCount: number;
  returnedCount: number;
  usedRerank: boolean;
  candidates: Array<{
    id: string;
    sourcePath: string;
    type:
      | "character"
      | "entity"
      | "location"
      | "timeline"
      | "plot"
      | "worldbuilding";
    name: string;
    summary: string;
    retrievalText: string;
    tags: string[];
    lastUpdatedChapter: string;
    relatedChapters: string[];
    score?: number;
    rerankScore?: number;
  }>;
};
```

其中 `sourcePath` 非常重要。Agent 拿到摘要和 `retrievalText` 后，如果需要完整上下文，应该继续调用 `ReadFile(sourcePath)`。

字段说明：

| 字段            | 所属类型             | 说明                                                                                                                                                                        |
| :-------------- | :------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `query`         | `RagSearchOutput`    | 本次实际用于检索的查询文本。它可能来自用户原始指令，也可能是 Agent 为了更好召回要素而改写后的检索 query。保留该字段是为了让 UI、日志和调试面板解释 Agent 到底查了什么。     |
| `recalledCount` | `RagSearchOutput`    | 粗召回候选数量。                                                                                                                                                            |
| `returnedCount` | `RagSearchOutput`    | 最终返回给 Agent 的上下文数量。                                                                                                                                             |
| `usedRerank`    | `RagSearchOutput`    | 本次是否启用了 Rerank 链路。                                                                                                                                                |
| `candidates`    | `RagSearchOutput`    | 本次返回给 Agent 的命中要素列表。每一项都是一条从向量库召回的要素记录，可能已经经过 Rerank 精排和最终上下文筛选。                                                           |
| `id`            | `RagSearchCandidate` | 要素在向量库中的稳定 ID，用于去重、更新、追踪和调试。它不替代真实文件路径，真实内容仍以 `sourcePath` 指向的 Markdown 文件为准。                                             |
| `sourcePath`    | `RagSearchCandidate` | 命中要素对应的项目内 Markdown 文件路径，例如 `elements/characters/林远.md`。这是从检索结果回到真实文件的桥。Agent 如果需要完整上下文，应该继续调用 `ReadFile(sourcePath)`。 |
| `type`          | `RagSearchCandidate` | 要素类型，用于告诉 Agent 和 UI 这条命中属于人物、地点、时间线、情节还是世界观设定。它也可以用于检索过滤和结果分组。                                                         |
| `name`          | `RagSearchCandidate` | 要素名称，例如人物名、地点名、事件名或设定名。用于展示，也方便 Agent 快速判断命中对象。                                                                                     |
| `summary`       | `RagSearchCandidate` | 要素摘要，通常来自要素文件 frontmatter，或由要素正文生成。它用于让 Agent 在不读取完整文件的情况下快速判断这条命中是否有价值。                                               |
| `tags`          | `RagSearchCandidate` | 要素标签，例如 `主角 / 黑风谷 / 伏笔 / 旧王朝`。标签可用于筛选、解释和后续 UI 分组。                                                                                        |
| `score`         | `RagSearchCandidate` | 向量召回分数，表示 query embedding 与要素 embedding 的相似程度。它主要用于排序、调试和解释，不应直接展示成百分比准确率。                                                    |
| `rerankScore`   | `RagSearchCandidate` | Rerank 精排分数。开启 Rerank 时，该字段表示精排模型重新判断后的相关性；未开启或精排失败时可以为空。                                                                         |
| `retrievalText` | `RagSearchCandidate` | 实际参与检索的要素文本，用于解释为什么召回这条要素。它不是完整文件内容，但足够帮助 Agent 判断是否需要进一步 `ReadFile`。                                                    |

### 4.4 工具结果格式

返回给模型的文本不应过长。当前返回格式接近：

```text
RAG 召回并重排 3 条，返回 3 条上下文
query: 林远 黑风谷 伏笔

#1 林远 (character)
path: elements/characters/林远.md
score: 0.8200
tags: 主角, 黑风谷
lastUpdatedChapter: chapters/第001章.txt
relatedChapters: chapters/第001章.txt
summary: 年轻修士，正在调查黑风谷异变。
retrievalText: 林远在上一章发现黑风谷入口处的石碑裂纹...
```

设计重点：

- 摘要用于快速判断。
- `path` 用于后续 `ReadFile`。
- `retrievalText` 用于让模型确认为什么召回。
- 不直接把完整要素正文全部塞给模型。

## 五、向量库与要素文件关系

### 5.1 真实数据来源

NovAI 必须坚持：

> `elements/**/*.md` 是故事要素的真实数据来源，向量库只是由这些文件生成的可重建检索缓存。

因此：

- 用户可以通过外部编辑器或文件管理器直接编辑要素文件；NovAI 负责读取和响应这些文件变化，但本站不提供内置文本编辑器能力。
- Agent 可以通过文件工具修改要素文件。
- 向量库记录可以删除、重建、失效、迁移。
- 不能把向量库中的 `vector + metadata` 当成不可替代的数据源。

### 5.2 向量库范围

MVP 阶段 `RagSearch` 只检索要素文件：

```text
elements/characters/*.md
elements/entities/*.md
elements/locations/*.md
elements/timeline/*.md
elements/plots/*.md
elements/worldbuilding/*.md
```

暂不直接向量化章节全文。

原因：

- 章节全文太长，直接召回容易噪声过大。
- 要素文件是对章节事实的结构化沉淀，更适合长期记忆。
- 这能迫使 NovAI 把“生成内容 -> 提取要素 -> 复用要素”闭环做实。

后续可以增加章节片段向量化，但不应取代要素向量库。

### 5.3 向量库记录结构

MVP 阶段，每一个可检索要素在向量库中对应一条记录。记录由两部分组成：

1. `vector`：由 Embedding 模型生成的向量。
2. `metadata`：用于过滤、展示、回溯真实文件和判断是否过期的元数据。

建议结构：

```ts
type VectorStoreRecord = {
  id: string;
  projectId: string;
  vector: number[];
  metadata: {
    sourcePath: string;
    type:
      | "character"
      | "entity"
      | "location"
      | "timeline"
      | "plot"
      | "worldbuilding";
    name: string;
    summary: string;
    tags: string[];
    retrievalText: string;
    lastUpdatedChapter?: string;
    relatedChapters: string[];
    contentHash: string;
    sourceModifiedAt: string;
    embeddedAt: string;
    embeddingProvider: string;
    embeddingModel: string;
    embeddingDim: number;
    embeddingTextVersion: number;
  };
};
```

字段说明：

| 字段                                                | 说明                                                                            |
| :-------------------------------------------------- | :------------------------------------------------------------------------------ |
| `id`                                                | 向量库记录 ID，应该稳定可复用，用于更新已有记录而不是重复插入。                 |
| `projectId`                                         | 项目隔离 ID，避免不同小说项目的要素互相召回。                                   |
| `vector`                                            | Embedding 模型生成的向量数据，用于相似度计算。                                  |
| `sourcePath`                                        | 对应的真实要素文件路径，例如 `elements/characters/林远.md`。                    |
| `type / name / summary / tags`                      | 要素基础元数据，用于过滤、展示和返回给 Agent。                                  |
| `retrievalText`                                     | 实际送入 Embedding 模型的检索文本，通常由类型、名称、摘要、标签和正文拼接而成。 |
| `lastUpdatedChapter / relatedChapters`              | 与章节相关的回溯信息，用于后续新鲜度排序、解释和筛选。                          |
| `contentHash`                                       | 要素文件内容哈希，用于判断向量记录是否已经过期。                                |
| `sourceModifiedAt`                                  | 要素文件读取时的修改时间，用于辅助判断外部编辑。                                |
| `embeddedAt`                                        | 这条向量记录生成时间。                                                          |
| `embeddingProvider / embeddingModel / embeddingDim` | 生成该向量所使用的 Embedding 配置。配置变化时，旧记录需要重建。                 |
| `embeddingTextVersion`                              | 检索文本拼装模板版本。模板变化时，即使原文件没变，也需要重新生成向量。          |

MVP 默认使用 Orama 作为向量检索引擎，并通过 IndexedDB 做浏览器本地持久化。Orama 负责维护可检索的数据结构与执行检索，IndexedDB 负责把这份数据保存到本地。后续如果迁移到 LanceDB、SQLite vec、Tauri 本地数据库或远程向量数据库，逻辑结构仍应保持一致。

### 5.4 向量库写入流程

向量库写入不直接发生在用户对话里，而是由要素文件变化触发。

推荐流程：

```text
elements/**/*.md
  -> 解析 frontmatter 和正文
  -> 生成 retrievalText
  -> 调用 Embedding 模型生成 vector
  -> 组合 vector + metadata
  -> upsert 到 Orama 向量库
  -> 更新项目级 vectorStoreStatus
```

关键规则：

- 同一个 `sourcePath` 对应的要素应更新已有向量记录，而不是重复写入。
- 如果要素文件被删除，对应向量记录也应删除或标记不可用。
- 如果 `contentHash` 未变化，可以跳过重新 Embedding。
- 如果 Embedding 模型或 `embeddingTextVersion` 变化，即使文件内容没变，也应该重建。
- 向量库写入失败不能破坏原始要素文件，因为真实数据源始终是 `elements/**/*.md`。

### 5.5 向量召回流程

`RagSearch` 的召回流程建议如下：

```text
Agent 构造 query
  -> 调用 Embedding 模型生成 queryVector
  -> 调用 Orama 在当前 projectId 下执行向量召回
  -> 根据 type / tags 等 metadata 做可选过滤
  -> 取 topK 候选
  -> 做轻量后处理
  -> 可选进入 Rerank 精排
  -> 返回 RagSearchOutput
```

MVP 阶段默认使用 Orama 负责粗召回。Orama 在这里承担三件事：

- 维护可检索的向量文档结构。
- 根据 query vector 做相似度召回。
- 根据 `projectId / type / tags` 等 metadata 做过滤。

向量数据的浏览器本地持久化由 IndexedDB 承担。Orama 可以通过持久化插件把自身数据保存到 IndexedDB，但 IndexedDB 才是实际落盘介质。

具体召回链路的落地状态以 [当前进度](../project/当前进度.md) 和 [开发日志](../project/开发日志.md) 为准。本文只记录目标方案：正式召回层以 Orama 为主。

召回结果中的 `score` 来自向量相似度。开启 Rerank 后，`rerankScore` 来自精排模型。最终返回给 Agent 的 `candidates` 应保留 `sourcePath`，这样 Agent 可以继续通过 `ReadFile(sourcePath)` 读取真实要素文件。

这个流程的重点是：

- 不把所有 `elements/**/*.md` 直接塞进模型上下文。
- 不让向量库取代原始 Markdown 文件。
- 先用向量库找到少量可能相关的要素。
- 再由 Agent 判断是否需要读取真实文件。

## 六、Agent 使用策略

### 6.1 何时应该调用 `RagSearch`

系统提示词应引导 Agent 在以下场景主动调用：

- 用户要求续写、改写、整理已有故事。
- 用户提到具体人物、地点、组织、伏笔、事件。
- 用户要求保持前后设定一致。
- 用户要求检查冲突、补全时间线、延续悬念。
- Agent 发现当前任务依赖历史上下文，但只凭当前对话无法确认。

### 6.2 何时不应该调用 `RagSearch`

以下场景可以不调用：

- 用户只是要求创建空项目文件。
- 用户明确要求写一个完全无关的新草稿。
- 用户要求修改当前已打开文件中的局部文字，且上下文已经由 `ReadFile` 足够提供。
- 向量库为空或未配置 Embedding，此时应说明无法检索，并使用文件工具继续工作。

### 6.3 与文件工具的配合

`RagSearch` 不应该代替 `ReadFile`。

推荐工作流：

```text
RagSearch -> 判断候选 -> ReadFile(sourcePath) -> 生成/修改文件
```

也就是说：

- `RagSearch` 负责发现相关要素。
- `ReadFile` 负责读取真实文件。
- `EditFile / CreateFile` 负责落地修改。

如果 Agent 要基于某个要素做重要剧情决策，应该优先读取对应文件，而不是只依赖 `RagSearch` 的摘要。

## 七、开发计划

### 阶段 1：要素提取与写入

目标：先让 RAG 有稳定的数据来源。

如果向量库里没有要素数据，`RagSearch` 即使接入 Agent Loop，也只能返回空结果。因此第一步应该先打通“章节内容 -> 要素文件”的链路，让 `elements/` 下有可向量化、可召回、可追溯的故事资料。

当前状态：

- 第一版规则型 `extractElementsFromChapter()` 已落地，可以从章节内容中提取人物、实体、地点、情节、时间线与世界观候选。
- `writeExtractedElements()` 已支持将候选写入 `elements/characters`、`elements/entities`、`elements/locations`、`elements/plots`、`elements/timeline`、`elements/worldbuilding`。
- Test Lab 与正式右侧内容面板已提供“提取要素 / 写入 elements”入口。
- 写入要素后已能标记 RAG 索引为 `stale`。
- 该实现用于打通数据链路，后续仍需升级为 LLM 结构化提取。

任务：

- [x] 实现第一版 `extractElementsFromChapter()`。
- [ ] 让 LLM 从章节内容中提取人物、实体、地点、情节要素。
- [ ] 输出稳定结构化 JSON。
- [x] 支持提取预览。
- [x] 用户确认后写入 `elements/`。
- [x] 写入后标记向量库过期或触发重建。
- [ ] 支持已有要素的去重、合并与覆盖确认。

验收标准：

- [x] 新章节能提取出最少三类要素。
- [x] 要素文件格式具备第一版稳定 frontmatter。
- [x] 写入的要素文件可以被向量化。
- [ ] 手动重建向量库后 `RagSearch` 可以召回新要素。

### 阶段 2：完善向量库构建与状态处理

目标：让 Agent 和用户知道向量库是否可用。

当前状态：

- 正式设置页已提供 RAG 索引状态查看入口。
- 正式设置页已提供手动重建向量索引入口。
- 索引元信息已展示状态、文档数、向量维度、Embedding 模型和最近构建时间。
- 要素写入后已能标记索引为 `stale`。

任务：

- [ ] `RagSearch` 返回更明确的 `vectorStoreStatus`。
- [ ] 向量库为空、过期、错误时返回更完整说明。
- [x] UI 显示向量库状态。
- [x] 设置页或测试页提供重建向量库入口。
- [ ] 自动增量重建与索引过期提醒。

验收标准：

- [x] 没有 Embedding 配置时不会静默失败。
- [x] 向量库为空时用户可看到当前索引记录。
- [ ] 向量库为空或过期时 Agent 能收到更结构化的状态提示。
- [ ] 向量库过期时用户可看到更明显的提醒。

### 阶段 3：接入 Agent 工具

目标：让模型可以主动调用 `RagSearch`。

当前状态：

- `RagSearch` 已接入新的 Agent tool name 类型。
- `RagSearch` 已在 `src/core/agent/tools.ts` 中暴露给模型。
- 工具结果已能回灌模型。
- 结果包含 `sourcePath`，Agent 可继续 `ReadFile` 精读。

任务：

- [x] 在 `src/core/agent/tools.ts` 中新增 `RagSearch` schema。
- [x] 在 Agent tool name 类型中加入 `RagSearch`。
- [x] 复用现有 `searchRagCandidates()` 作为核心实现。
- [x] 补充 `RagSearch` 的输入校验与结果格式化。
- [x] 在 UI tool message 中展示检索 query、命中数量和主要路径。
- [ ] 继续观察模型在真实写作任务中是否会稳定主动调用。

验收标准：

- [x] Agent 可以在一轮 Query Loop 中主动调用 `RagSearch`。
- [x] 工具结果可以回灌模型。
- [ ] 向量库为空时有更清晰错误或提示。
- [x] 命中结果包含 `sourcePath`，Agent 可以继续 `ReadFile`。

### 阶段 4：Rerank 与解释层

目标：提升召回质量，并让用户理解 Agent 查到了什么。

当前状态：

- `RagSearch` 内部已接入已有 Rerank 逻辑。
- Rerank 失败时当前会在 `rerankRetrievalCandidates()` 内降级为粗召回候选。
- 工具结果已保留 `score / rerankScore`。

任务：

- [x] 在 `RagSearch` 内部接入已有 Rerank 逻辑。
- [x] Rerank 失败时降级为粗召回。
- [x] 工具结果中保留 `score / rerankScore`。
- [ ] UI 展示召回结果、重排结果、最终使用结果。

验收标准：

- Rerank 开启时结果排序更符合 query。
- Rerank 服务不可用时不阻塞 Agent 工作。
- 用户可以看到 Agent 为什么使用某些要素。

### 阶段 5：后续增强

后续再考虑：

- 章节片段向量化。
- 增量向量更新。
- 删除文件后的向量记录清理。
- 检索 query 改写。
- 多轮检索记忆。
- 针对不同任务类型的检索策略。

## 八、当前优先级

当前最优先做的是：

> 在正式项目中验证“要素文件 -> 向量库 -> Agent 主动 RagSearch -> ReadFile(sourcePath) -> 写作/改稿”的真实创作链路。

原因：

- `elements/**/*.md` 已经可以由章节内容产生，但提取质量仍需 LLM 结构化方案补强。
- 设置页已能手动重建索引，`RagSearch` 已接入 Agent Loop。
- 下一步需要验证模型是否会在真实创作任务中主动检索，并在必要时继续读取要素源文件。
- 当前检索实现仍是过渡方案，正式 Orama 召回、自动增量索引和更好的状态提示仍需补齐。

下一步建议优先在正式项目里做端到端测试，并根据日志观察结果决定先优化触发提示词、索引状态提示，还是先替换为正式 Orama 召回。
