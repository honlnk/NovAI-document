# 文件预览 frontmatter 卡片化计划

> 状态：已完成（2026-11-23 落地，全量 1803 例 + typecheck 全绿 + OPFS 真机抽查全过）
> 来源：2026-11-22 真机预览缺陷 + 会话内方案对稿（HTML 原型 `frontmatter-card-样式方向.html`，设计审查两视口必修 0）

## 施工日志

- 2026-11-23 阶段一（utils：frontmatter 切分 + 块级识别）：`frontmatter-display.ts`（splitFrontmatter / parseFrontmatterDisplay）+ `markdown-blocks.ts` detectFrontmatter 可选项落地；新增 `frontmatter-display.test.ts`（13 例）与 `markdown-blocks.test.ts` 增补（5 例），26 例全绿，app typecheck 干净。偏差：空围栏 `---\n---` 按「首条内行非键形态」门槛**不识别**（照旧渲染两条分隔线），而非计划 Step 2 预设的「识别成功但卡片渲染为空」——统一走键形态门槛让识别规则只有一条，行为对用户无感知差异。
- 2026-11-23 阶段三（MarkdownRenderer + FilePanelBody 双路径接入）：`frontmatter` prop 默认关；常规预览走卡片、高亮回放路径以 markdown-block 包装层承载绿显/定位类（卡片根透明底，避免与 change-hl-green 的背景竞争，外观不变）；「正文 N 字」预览态排除 frontmatter（编辑模式口径不变，计划原文如此）。app 全量 589 例（48 文件）全绿 + typecheck 干净。
- 偏差备注：FilePanelBody 既有 `.spec.ts`（如 FilePanelBody.autosave.spec.ts）本就不被 pnpm test 收录，本阶段不为其补卡片化断言；若将来复活 spec 体系，需同步补 frontmatter 卡片的用例。
- 2026-11-23 阶段四（OPFS 真机抽查，计划验收点 3）：SKILL.md（无 tags→摘要回落 42 字 +「3 个字段」）、要素 characters（李世民/陈砚 tags chips 正确）、展开网格键值齐全、无 hr 残留、正文从 h1 开始全过；「正文 1,023 字」与 core `countContentChars` 对 SKILL.md 全文直算吻合——word-count 本就剥 frontmatter（`FRONTMATTER_PATTERN`），FilePanelBody 先经 splitFrontmatter 取 body 属双层剥离幂等，口径不回归；无 frontmatter 的 `references/自查表.md` 无卡片无 hr 渲染不变（迁移条款验证）；展开态跨文件记忆链（展开→切走不泄漏→切回保持）全过。**真机抓到缺陷（当场修复）：常规预览路径漏传 path，frontmatter 块 key 恒为 0 致 Vue 复用同一卡片实例，展开态泄漏到下一个文件——MarkdownRenderer 增 `path` prop 透传给 FrontmatterCard、FilePanelBody 常规路径传 `file.path`，与高亮路径对齐走按路径记忆；新增 `MarkdownRenderer.test.ts` 3 例（path 记忆、复用切换不泄漏切回恢复、缺省不识别）。复跑：全 workspace 1803 例（142 文件）全绿 + typecheck 干净。**清理与记录：cleanup-opfs.js 清记录 2 条 + OPFS 8 项、含密钥种子已删，使用记录已追加至 `docs/project/真机测试OPFS仿真目录方案.md`。种子清单 `seed-manifest.txt` 本轮补 `skills/*` 一行（验证技能文件需要）。
- 收尾说明：本轮「正文 N 字」验证确认 core word-count 一直会剥 frontmatter——计划目标形态第 3 条（「正文 N 字排除 frontmatter」）在旧实现里对标准围栏已成立；本次改动的实际价值是让字数口径与卡片识别**同一来源**（splitFrontmatter 键形态门槛），并覆盖「编辑模式之外所有预览态」的显式保证。

## 背景与问题

文件预览（FilePanelBody 预览模式）把 Markdown 文件原文直接喂给共享 markdown-it（`MarkdownRenderer` → `parseMarkdownBlocks`），不认识 YAML frontmatter：首行 `---` 渲染成 `<hr>`，元数据段落被闭合线变成 setext `<h2>`（加粗大字）。用户在真机先后抓到两类现场：

1. 技能文件 `skills/*/SKILL.md`（name / description / when_to_use）；
2. 要素文件 `elements/characters/*.md`（id / type / name / summary / tags / lastUpdatedChapter / relatedChapters / updatedAt）。

伏笔卡片（按 frontmatter.id 定位）同理。core 侧各领域解析器（`core/skills/frontmatter.ts`、`core/elements/parser.ts`、`core/foreshadowing/store.ts`）都会剥离 frontmatter，但预览渲染与它们无关——根因是预览渲染链路没有 frontmatter 概念，一处修复覆盖全部带 frontmatter 的 md 文件类型。

## 目标形态（用户已拍板）

- md 文件预览时，frontmatter 围栏不再进入正文渲染：折叠成一行摘要条（标题 + tags chips + 「N 个字段」），点击展开为完整属性网格（键值行、数组 chip 化、空值「—」占位）。
- 无 tags 的文件折叠条回落为「标题 + 首个长值截断 42 字」（如技能文件的 description）。
- 「正文 N 字」排除 frontmatter，只统计正文。
- AI 改动高亮回放：frontmatter 波及改动时整块（卡片）绿显，块级精度原则不变。
- 聊天渲染零变化（检测默认关闭）；编辑模式仍看原文（textarea），不动。

## 方案

### Step 1（app/utils）：frontmatter 切分与块级识别

- 新增 `packages/app/src/utils/frontmatter-display.ts`：
  - `splitFrontmatter(source): { hasFrontmatter, raw, body }`。识别规则：BOM/CRLF 归一后，首行匹配 `^---\s*$` 且存在闭合行（同样 `^---\s*$`，扫描上限 200 行）；第一条围栏内行必须匹配键形态 `^[\w.-]+\s*:`（防把「开头分隔线 + 短正文 + 分隔线」的场景分割线误判为 frontmatter）。`raw` = 围栏全文（含两行分隔线），`body` = 其余原文；任一条件不满足即不识别（照旧渲染）。
  - `parseFrontmatterDisplay(raw): Array<{ key: string, value: string | string[] }>`。展示级 dumb 解析：跳过分隔线；行内首个冒号切键值；缩进续行/无冒号行并入上一值（块标量 `>` 与 `|` 的多行天然被收进）；`[a, b]` 包裹的值按逗号切数组后去空格；不做 YAML 语义解析（嵌套缩进块按续行收拢，展示容错）。
- `utils/markdown-blocks.ts`：`parseMarkdownBlocks(source, options?: { detectFrontmatter?: boolean })`，默认关（聊天路径零变化）。开启且识别成功时：整篇 parse（token.map 保持全源行号）后丢弃起始行 ≤ 闭合线行号的顶层组（即 hr 与 setext h2 两组），在首位插入特殊块 `{ key: 0, html: '', kind: 'frontmatter', raw }`。正文块 key 仍取全源行号，高亮映射 `markdownBlockIndicesInRange` 无需改动（块起始行序列 = [0, 闭合线+1, …]，与 range 求交逻辑天然兼容）。
- `MarkdownBlock` 类型加可选 `kind?: 'frontmatter'` 与 `raw?: string`。
- 测试：`markdown-blocks.test.ts` 增补（识别 / 未闭合 / 空围栏 / CRLF / BOM / 无 frontmatter / 场景分割线误判防护 / key 稳定性与既有用例不回归）；新增 `frontmatter-display.test.ts`（切分与键值解析全形态）。

### Step 2（app/components/ui）：FrontmatterCard 组件

- 新增 `packages/app/src/components/ui/FrontmatterCard.vue`，props：`raw: string`（围栏全文）、`path?: string`（同路径展开态记忆键）。
- 折叠条（button，可点击）：chevron SVG（展开时旋转 90°）+ 标题（首个非空 name/id，兜底首个非空标量）+ tags chips（有 tags 时）+ 摘要截断 42 字（无 tags 时；取首个非空 summary/description/when_to_use）+ 「N 个字段」。展开态渲染属性网格：键 12px mono、值 14px、数组 chip 化、空值「—」。
- 空围栏（解析结果 0 个字段）不出卡片，渲染为空。
- 样式对齐原型：灰底（gray-50）圆角卡、应用现有灰阶；组件 scoped 样式，不动全局 CSS。
- 展开态记忆：组件内以 `Map<path, boolean>` 记忆（path 缺省时退化为本地 ref），切换文件返回后保持原状。
- 测试：组件 spec 覆盖折叠/展开切换、path 记忆、无 tags 回落、空围栏不渲染。

### Step 3（app）：MarkdownRenderer prop 与 FilePanelBody 双路径接入

- `MarkdownRenderer.vue`：新增 prop `frontmatter?: boolean`（默认 false）；true 时 parser 开 `detectFrontmatter`，`kind==='frontmatter'` 的块渲染 `FrontmatterCard`，其余块照旧。聊天消费方（MessageItem 等）不传该 prop，行为逐字节不变。
- `FilePanelBody.vue`：
  - 常规预览（~899 行）：`<MarkdownRenderer :content="file.content" frontmatter />`。
  - 高亮回放路径（mdBlocks v-for，~232 行与 ~900 行）：`parseMarkdownBlocks` 传 `detectFrontmatter: true`；`kind==='frontmatter'` 块渲染 `<FrontmatterCard :raw :path>`，`change-hl-green` / `change-hl-target` 类直接挂卡片根元素（绿显 = 块级精度，与既有原则一致）。
  - wordCount（~136 行）：markdown 格式且 `splitFrontmatter().hasFrontmatter` 时以 `body` 计数，其余格式与编辑模式口径不变。
- 编辑模式、JSON 预览、纯文本高亮路径不动。

### 验收点

1. `pnpm test` 全绿（新增 frontmatter-display.test.ts、FrontmatterCard spec、markdown-blocks.test.ts 增补 + 全量回归）。
2. `pnpm typecheck` 干净。
3. 真机抽查（常驻 5173 + ~/note/test-novel 项目）：技能 SKILL.md 与要素 `elements/characters/*.md` 预览出现折叠条（tags chips 正确、无 hr 残留、正文从 h1 开始）、点击展开网格、「正文 N 字」不含元数据；技能文件（无 tags）回落为摘要截断。浏览器环境用 IAB，按 `novai-opfs-testing` 技能种 OPFS。
4. 聊天回归抽查：任一含 AI 输出 markdown 的会话渲染无变化（单测层面由 flag 默认关保证 + 现有 chat 相关 spec 回归兜底）。

### 文档同步计划

- 本计划文档：状态流转（待开始 → 施工中 → 已完成）+ 施工日志（日期 + 阶段 + 验收结论 + 偏差记录），随阶段门禁写盘。
- `docs/project/当前进度.md`：落地后补一行。

## 已拍死的取舍（用户已确认）

1. 呈现方向 **C（折叠摘要）**：frontmatter 主要供 AI 与工具消费，用户平时不看，折叠让正文第一时间可见。A（属性网格）/ B（轻清单）/ D（档案头卡）落选。
2. 折叠条内容：**类型徽标不显示，tags chips 替代**（用户明确要求）；摘要只在无 tags 时兜底出现——一行装不下两样，保证折叠条恒为一行的简洁形态。
3. **字数排除 frontmatter**（用户拍板）：卡片化后「正文 N 字」与观感一致。
4. 检测放块级解析层（flag 控制、默认关），不在 FilePanelBody 预先剥离——保证高亮映射同源、聊天渲染逐字节不变；不用 markdown-it 插件方案（全局影响面不可控）。
5. 展开态按路径记忆，切换文件返回后保持原状。
6. 展示解析刻意 dumb（不引入 YAML 库、不复用 core 领域解析器）：预览是化妆不是语义；core 保持 UI 无关，`core/src/core/**` 本就不在 app 可导入范围。

## 迁移条款

无迁移。纯渲染层改动，不涉持久化格式、配置或旧数据；无 frontmatter 的 md 文件渲染输出与今天逐字节一致。

## 明确不做清单

- 聊天消息渲染不加 frontmatter 语义（消息开头 `---` 仍是分隔线/普通渲染）。
- 不做 chip 点击交互（标签过滤/跳转）。
- 不做 frontmatter 编辑/表单化（编辑模式保持原文 textarea）。
- 不做完整 YAML 解析（锚点、多文档、嵌套结构不做）。
- Plan/Rhythm/Foreshadowing 详情面板不接入（其正文来自结构化视图，无 frontmatter 现场问题；后续发现再立项）。
- 不支持 `...` 作为闭合线。

## 新旧机制替代

旧机制「md 预览整篇直渲染（frontmatter 变 hr + setext h2）」被 frontmatter 卡片**替换**，无并存。core 各领域 frontmatter 解析器不动（语义解析归 core，展示解析归 app，职责不重叠）。
