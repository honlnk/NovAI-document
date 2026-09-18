# 写工具权限分级 + `.novel/` 防护一致性

## 记录时间

2026-09-14（定稿于同日）

## 状态

**已定稿，可交智能体实施。** 本文档含两件事：①「五档权限」是已定稿的功能设计；②「`.novel/` 防护一致性」是已定稿的一行修复。两者都改 `common.ts`/`path.ts`/`permission.ts` 这一族文件，合并实施。

## 一、五档权限（核心交付物）

### 1.1 档位定义（已定稿，照此实现）

用户可在五档中选择，控制「哪些修改不需要每次弹确认卡」：

| # | 档位名 | 免确认范围（这些直接做，不问） | 其余操作 |
| :--- | :--- | :--- | :--- |
| 1 | **仅审阅** | 无（只读） | 任何修改都弹卡问，批一次改一次 |
| 2 | **章节内容** | 修改 `chapters/` 下已有章节的正文（EditFile） | 其余都问 |
| 3 | **素材内容** | 修改 `elements/` 下已有要素文件（EditFile） | 其余都问 |
| 4 | **章节 + 素材** | 改 `chapters/` 和 `elements/` 已有文件正文 | 其余都问 |
| 5 | **完全访问** | 所有允许修改的内容（含 prompts/、NovAI.md 等） | 无 |

### 1.2 三条判定规则（已定稿）

1. **按「动作 + 路径」双重判定**：
   - **内容修改**（EditFile 改正文）：看路径——`chapters/` 走章节档、`elements/` 走素材档。
   - **结构操作**（新建 CreateFile、删除 DeleteFile、重命名/移动 RenameFile）：**一律弹卡问，除非选了「完全访问」**。它们和「改提示词」同级，不管操作对象是不是章节/素材。
   - 移动即改名：RenameFile 同时承担改名与移动（移动 = 改路径名），归结构操作。
2. **素材范围**：`elements/` 下全部子目录（characters/locations/entities/timeline/plots/worldbuilding 六类）都算素材。
3. **「仅审阅」不是拒绝**：弹卡询问，用户批了这一次就执行（区别于 dsh 的「仅可查看 = 直接拒绝」，故改名「仅审阅」区分）。

### 1.3 硬底线（任何档位都不例外）

- `.novel/` 内部目录、`novel.config.json` 项目配置：**永远禁止修改**，连确认卡都不弹，直接在工具校验层拒绝（这是 Stage 4 已有的语义，保留）。
- 所有写路径仍须通过 `normalizeProjectPath`（禁绝对路径 / `..` 逃逸）。

### 1.4 默认档与已知行为

- **默认档：「章节 + 素材」（第 4 档）**——日常写章、提要素不打断。
- **已知且接受的后果**：该档下 Agent 每次更新 `prompts/NovAI.md`（项目记忆）、改 `system.md` 等提示词，会弹卡问一次。用户已确认接受此默认（能盯着 Agent 不乱改记忆/设定）。若后续嫌烦，可考虑把 NovAI.md 挪入内容档——**本期不做**。

### 1.5 实现要点（供智能体）

- 现状抓手：写文件前 `tool-execution.ts:94-95` 调 `decideWriteToolPermission(toolName, validatedInput, project)` 返回 `allow | ask`。改为：先读**当前档位**（持久化，见下），再按 1.2 规则判定该工具调用是 allow 还是 ask。
- `permission.ts` 需扩展输入：除路径外，还要区分**动作类型**（内容修改 vs 结构操作）。可由工具名推出——`EditFile`=内容修改；`CreateFile/DeleteFile/RenameFile`=结构操作。
- 决策词表沿用：`PermissionDecision = allow | {ask, reason}`；`ApprovalOutcome = allowed-once | rejected | cancelled | unavailable`（无 always-allow）。
- **档位持久化**：存进 `novel.config.json` 的 `settings`（新增如 `permissionPreset: 'review' | 'chapter' | 'material' | 'chapter-material' | 'full'`，默认 `'chapter-material'`），并在 UI 提供切换入口（见 1.6）。读取时经 `normalizeProjectConfig` 回填默认值，旧配置无此字段自动补 `chapter-material`。
- 越界路径（无法规范化）的行为：结构/内容判定先基于规范化路径；无法规范化的路径在工具校验层已被拒，走不到权限判定（与现状一致）。

### 1.6 UI（供智能体）

- 一个档位选择控件（下拉，对照用户给的 dsh 截图样式：仅可查看/工作区内修改/完全权限），列五档。
- 位置建议：对话输入区附近（对照截图），或设置面板「项目设置」——**实现时任选一处并在交付说明里记录选了哪**；后续可再调。
- 切换档位即时生效（写回 config，下一轮起按新档判定），不需要重启。

### 1.7 验收（对照）

- 选「仅审阅」：任何修改（含改章节正文）都弹卡；批准后仅当次生效。
- 选「章节内容」：EditFile 改 `chapters/第003章.txt` 正文不问；改 `elements/characters/x.md` 问；CreateFile 新章问；DeleteFile 删章问。
- 选「素材内容」：改要素不问；改章节问。
- 选「章节 + 素材」：改章节、要素不问；改 `prompts/NovAI.md`、`system.md` 弹卡问；CreateFile/DeleteFile/RenameFile 一律问。
- 选「完全访问」：以上都不问，但写 `.novel/`、`novel.config.json` 仍被直接拒绝（不弹卡）。
- 档位切换后下一轮立即生效；重启/重开项目后档位保留。

## 二、`.novel/` 防护一致性（已定稿的一行修复）

### 事实（代码逐行核实）

`.novel/` 与 `novel.config.json` 的写保护不是统一开关，而是每个写工具各自决定是否查：

| 工具 | 校验函数 | 拦 `.novel/` |
| :--- | :--- | :--- |
| RenameFile | `assertWritableDocumentPath` | ✅ |
| DeleteFile | `assertMutableDocumentPath` | ✅ |
| **CreateFile** | 仅 `assertWritableTextFilePath` | ❌ |
| **EditFile** | 仅 `assertTextFilePath` | ❌ |

（前两个函数在 `core/tools/file-tools/common.ts:19-33`，含 `path.startsWith('.novel/')` 拦截；后两个在 `core/tools/path.ts:53-70`，不含。）

### 影响（准确口径）

- 非本次重构新引入：CreateFile/EditFile 一直没查 `.novel/`；Stage 4 移除「每次必确认」后人工兜底没了才暴露。
- CreateFile 可在 `.novel/` 下**新建**文件、静默无感；但因「已存在则失败」，不能覆盖 `novel.config.json` 等已有文件。
- EditFile 受 readFileState + ReadFile 禁读 `.novel/` 间接兜底，实际难够着，但属脆弱防护。

### 修复（已定稿）

- CreateFile `validateInput`：`normalizeTextFilePath` 后补 `assertWritableDocumentPath`。
- EditFile `validateInput`：补 `assertMutableDocumentPath`。
- **连带修大小写绕过**：`assertMutableDocumentPath`/`assertWritableDocumentPath` 现用大小写敏感的 `===`/`startsWith` 判 `novel.config.json`/`.novel/`，而扩展名检查却 `toLowerCase()`；macOS APFS 大小写不敏感，`Novel.Config.JSON` 可能命中真实配置。**断言前先对路径做 casefold/统一小写比较**。
- 补测试：CreateFile/EditFile 写 `.novel/` 路径应报错；大小写变体同样被拒。

## 三、关联

- 删除 tool-policy 的决策不变（正则解析用户意图那条路已废弃，不回退）。
- 同属 review 待修清单：[[待办-文件改动追踪不可靠]]、[[待办-spill重设计-让溢出内容可取回]]。
- 注意：[[待办-spill重设计-让溢出内容可取回]] 会给 ReadFile 的 `.novel/` 读防护开 `.novel/spill/` 口子——**那是改「读」断言，与本篇改「写」断言方向相反、互不冲突**，但都动 `common.ts`/`path.ts` 断言族，实施时合并评审避免各改各的。

## 四、dsh 参考锚点（照哪抄、抄多深）

> 总原则：**抄「逻辑形状」，用我们的轻骨架实现；不引 dsh 框架。** dsh 仓库位于 `/Users/honlnk/project/deepseek-harness`。

本篇**参考 dsh 的「预设（preset）」概念与 UI，不搬它的代码结构**。dsh 的权限是「sandbox 模式 × 审批策略」双旋钮 + 预设表打包 + 事件投影 + 作用域分层——那套对我们是过度工程，不抄。我们的 `permission.ts` 是 92 行单点判定函数，比 dsh 简单得多，五档判定直接在它里面实现即可。

**打开这些文件对照（只看概念与 UI）：**

| dsh 文件 | 参考什么 |
| :--- | :--- |
| `packages/interaction/permission-presets/src/types.ts` | 「预设档位」的类型形状：`PresetOption`（id/名称/描述）、预设表 + 当前值的投影结构——我们五档的「选项列表 + 当前选中」可对照设计 |
| `packages/interaction/permission-presets/src/index.ts` | 预设如何打包「哪些操作免确认」、如何切换——**只理解思路**，实现用我们自己的 `permission.ts` |
| 用户提供的 dsh 截图（仅可查看/工作区内修改/完全权限下拉） | UI 形态参考 |

**抄多深**：预设（几档可选 + 用户自选好）这个概念、下拉 UI 形态。

**坚决不抄**：双旋钮模型、审批策略包、`sandbox/*` 的事件投影与作用域、Cordis 注册方式。判定逻辑全部落在我们的 `permission.ts` + 五档规则表。
