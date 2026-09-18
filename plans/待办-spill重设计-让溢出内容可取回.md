# Spill 重设计：让溢出内容可被取回（方向反转）

## 记录时间

2026-09-14

## 状态

**设计已定稿（方向与口径已与用户确认），待排期实施。** 方向：把现状「spill 文件禁读」反转为「可读回」。ReadFile 三道闸与口径整套照搬 dsh（字节判断），数值微调留待实测。基于第六次提交（`3b7a981`）review 会话（sess_6a541707）的结论：用户判断「spill 被设计为不可读，那么 spill 的意义就没有了」——本文档把正确方向设计出来。本文档即可直接作为施工依据；如需可再补一份分步施工计划。

## 一、现状错在哪（为什么方向反了）

当前 spill（`3b7a981`）的行为：工具结果 > 8192 字符 → 全文写入 `.novel/spill/<id>.txt`，上下文只留「头 4096 + `[...中间 N 字符省略，完整内容见 <path>...]` + 尾 1024」。同时 `read-file.ts` 把 `.novel/spill/` 列为**禁读**。

这造成一个**自相矛盾的闭环**：

1. 省略标记叫模型「去这个路径读完整内容」；
2. 但 `.novel/spill/` 被 ReadFile 禁读（`assertMutableDocumentPath`）；
3. 模型照着标记去读 → 撞墙报错「不能指向项目配置或 .novel 内部文件」；
4. 而系统提示词（`prompt.ts`）**只字未提 spill**，模型不知道这个路径是锁的，只会白白浪费一轮去撞墙。

**净效果：spill 只剩「省 token」这一半价值，丢了「能取回细节」这另一半——而后者才是 spill 的存在意义。** dsh 的原设计里 spill 文件就是给模型读回的；我们为了防 read→spill→read 循环，把读回能力一刀切砍了，等于「头疼砍头」。

### 为什么会走反（根因）

dsh 敢让 spill 可读，是因为它的 read 工具自带**三道闸**：offset/limit 行分页、单行 2000 字符截断、**总输出 50KB 字节封顶**——所以 read 的结果天然不会大到要 spill，read 被豁免在 spill 白名单之外。

我们的 ReadFile **缺了第三道闸**：它有 offset/limit 行分页（2000 行）、有 512KB 整读上限，但**单次返回没有字节/字符封顶**。于是 ReadFile 的结果可能很大 → 必须进 spill 白名单 → 为了不循环又只能禁读 spill 文件。**缺一道闸，逼出了整个反转设计。**

## 二、设计目标（反转后应成立的几条）

1. spill 文件**可被模型读回**，「能取回细节」恢复成立。
2. **不产生 read→spill→read 循环**（这是当初禁读的正当顾虑，反转后必须用别的方式解决，不能假装它不存在）。
3. RagSearch 的结果被 spill 后，省略区内容**有可达路径**（现状是死胡同）。
4. spill 文件**不再无限累积**。
5. 系统提示词让模型**知道 spill 存在、知道怎么取回**，不再撞墙。

## 三、设计（五个相互咬合的改动）

这五处是一整套，单独改任何一处都会重新引入矛盾，必须一起做。

### 改动 1：给 ReadFile 照搬 dsh 的「三道闸」（含字节封顶，地基）

**决策已定：ReadFile 的结构与数值整套照搬 dsh 的 read 工具，不另起字符口径，统一按字节判断。** 微调研制见「待定决策」。

dsh read 工具（`packages/fs/tool-fs/src/read.ts`、`read-render.ts`）的三道闸，全部照抄：

| 闸 | dsh 常量与数值 | NovAI 现状 | 动作 |
| :--- | :--- | :--- | :--- |
| 行分页 limit | `READ_LIMIT = 2000`（行） | ✅ 已有（2000 行） | 保留 |
| 单行截断 maxLineLength | `READ_MAX_LINE_LENGTH = 2000`（字符） | ❌ 无 | **新增** |
| **总字节封顶 maxBytes** | `READ_MAX_BYTES = 50 * 1024`（50KB **字节**） | ❌ 无（仅 512KB 整读上限） | **新增，本设计的地基** |

要点：

- **单位是字节不是字符**。50KB 是 UTF-8 字节数（50×1024 bytes），中文一字 ≈ 3 字节，故 ≈ 1.7 万中文字。实现用 `new TextEncoder().encode(text).length` 计字节（浏览器/Node/vitest 均支持），**不要用 `string.length`**（那是 UTF-16 码元数，口径错会差 3 倍）。
- 三闸配套后，任何一次 read 都不会超过 50KB → ReadFile 结果天然不会大到要 spill → **ReadFile 从 spill 白名单退出**（对齐 dsh「read 豁免」）。
- 超过 maxBytes 时按字节截断（注意不要切开 UTF-8 多字节字符的边界，可用 `TextDecoder` 流或 `TextEncoder` 逐段累加），并附「结果过大已截断，完整文件可用 offset/limit 分段读取」的提示。
- 单行截断（maxLineLength）与字节封顶（maxBytes）是两个独立闸：前者防「某一行几万字撑爆」，后者防「总行数太多撑爆」，都要加。
- 这是整个反转的**地基**：只有 read 不会再产生超长结果，才敢放心让 spill 文件可读而不怕循环。

### 改动 2：给 `.novel/spill/` 单独开一个 ReadFile 的口子

- 现在 `assertMutableDocumentPath` 把整个 `.novel/` 一刀切禁读。改为：`.novel/` 仍禁读，**唯独 `.novel/spill/` 放行给 ReadFile**。
- 实现：`assertMutableDocumentPath` 不接受「可放行子路径」参数，需要新增一个 read 专用的断言（如 `assertReadableDocumentPath`），逻辑 = 「禁 `.novel/` 但放行 `.novel/spill/`」；写路径的 `assertMutableDocumentPath`/`assertWritableDocumentPath` 保持原样（spill 仍**禁止写/改/删**，只能读）。
- 与 [[待办-写工具权限与novel防护一致性]] 的关系：那边是「CreateFile/EditFile 漏装 `.novel/` 写防护」，这边是「给 ReadFile 的 `.novel/` 读防护开 spill 口子」——**一个是补写防护，一个是开读口子，方向相反、互不冲突**，但改的是同一族断言函数，建议到时一起看，避免两边对 `assertMutableDocumentPath` 各改各的。

### 改动 3：spill 省略标记改成「可执行的取回提示」

- 现在的标记 `[...完整内容见 <path> ...]` 是死指针。改成带 **retrievalHint** 的指引，明确告诉模型怎么读，例如：
  `[... 中间 N 字符已省略；完整内容已存至 <path>，可用 ReadFile 对该路径以 offset/limit 分段取回 ...]`
- 因为改动 1 之后 spill 文件可读、且 spill 文件本身不会再被 spill（见改动 4），这条指引是**真实可走的**，不再是死指针。

### 改动 4：spill 文件自身豁免 spill + RagSearch 的可达性

- **spill 文件不再二次 spill**：读 `.novel/spill/` 文件的结果即使再大也不进 spill（否则标记里又指一个新 spill 路径，循环回来了）。用「路径前缀是 `.novel/spill/`」判定豁免，简单可靠。
- **RagSearch**：RagSearch 没有 offset 分页，被 spill 后省略区无法靠分页取回。但 RagSearch 结果里每条候选带 `sourcePath`，指引模型「要完整原文就 ReadFile 该 sourcePath」。同时 RagSearch 结果 spill 后存成 `.novel/spill/` 文件，模型也可分段读它。两条路都通，死胡同消除。
- **循环为什么不会回来**：改动 1 让 read 结果有字节封顶（不会再 spill）+ 改动 4 让 spill 文件豁免 spill。两道一合，read→spill→read 的链条两头都断了——**这就是「不用禁读也能防循环」的机制**。

### 改动 5：spill 文件清理（抄 dsh 的 cleanup）

- 现状：`.novel/spill/` 只增不减，无限累积。
- 抄 dsh 的 startup cleanup（保留 N 天清一次）。在打开项目/启动时扫 `.novel/spill/`，删除早于保留期的文件。
- **顺序约束**：cleanup 要在「spill 可读」之后才抄才有意义（不可读时清理与否无所谓，反正模型也读不到）。
- 保留期取值见「待定决策 2」。

### 配套：系统提示词补一句 spill 说明

`prompt.ts` 的工具规则里补一条，让模型知道：超长结果会被存成文件、路径在哪、怎么取回。否则前面所有改动对模型都是不可见的。

## 四、改动后的一次完整流程（对照现状）

**现状（断的）：** ReadFile 读 9000 字章节 → spill → 上下文留预览 + 死指针 → 模型想取细节 → 读死指针撞墙报错。

**改后（通的）：** ReadFile 读 9000 字章节 → 若超字节上限，结果内截断 + 提示用 offset/limit → 模型 offset/limit 分段读完。**只有真正巨大的结果**（比如 RagSearch 一次性回了一大坨）才走 spill → 上下文留预览 + 活的取回指引 → 模型 ReadFile 那个 spill 路径分段取回 → 不撞墙、不循环。

## 五、待定决策（数值微调，留待实测）

### ReadFile 三道闸的数值微调

**结构与口径已定为照抄 dsh（字节判断），此处只留「数值是否在小说场景下微调」的余地，不重新开放方向。**

- 默认照抄 dsh：行 2000 / 单行 2000 字符 / 总封顶 50KB（字节）。
- **微调的动机只有一个**：50KB ≈ 1.7 万中文字，而一章正文典型 2000~4000 字（≈ 6~12KB），所以 50KB 约等于一次读 1.5~2.5 章。正常读单章远够，但「一次读最近几章原文」这类操作可能触顶。
- **何时调、怎么调**：等真实写作测试时观察——若发现常规多章连读频繁被字节封顶切碎，可把 maxBytes 上调（如 64KB / 96KB）；若不频，保持 50KB。**调整的是数值，不是结构或口径。**
- 注意：spill 阈值（8192）和 ReadFile 总封顶（50KB）是两道独立的关卡，数值上不要互相贴着，否则会矛盾（见「明确不做的」）。

### spill 文件保留期

**已定：7 天。** 我们这边 spill 文件主要是「写作过程里被截断的长内容」，章节正文本身项目里就有原件，spill 文件只是缓存副本，写作迭代周期短，7 天足够回溯。到期在打开项目/启动时清理（对齐 dsh 的 startup cleanup，仅保留期从 30 天改为 7 天）。

## 六、明确不做的

- 不引入 dsh 的 spill 抽象层 / spill-local 存储提供者 / spill-policy 策略插件那套插件结构——只抄「字节封顶 + 可读 spill + 取回提示 + 定期清理」这个**行为形状**。
- 不改变 spill 的触发阈值（8192 字符）和头尾预览分法（4096/1024）的现有默认值。注意它与 ReadFile 总封顶（50KB 字节）是两套独立关卡、口径不同（字符 vs 字节），数值上不要互相贴近，否则会出现「read 截断线和 spill 触发线打架」的矛盾。
- 不动 compaction——spill 管单次太大、compaction 管总量超标，分工不变。

## 七、验收要点（将来实施时对照）

- 给一个超大 RagSearch 结果触发 spill，模型能照着标记用 ReadFile 分段取回省略区内容，**全程不报错**。
- 反复读 spill 文件**不会**再次 spill（无循环）。
- `.novel/spill/` 之外仍禁读、`.novel/spill/` 仍禁写/删。
- 打开项目时过期 spill 文件被清理，未过期的保留。
- prompt.ts 里有 spill 说明；模型不再因读 spill 路径撞墙。
- 读一章正常长度的章节（2000~4000 字）**不被** spill 或字节截断（回归保护，防止三道闸数值误伤常规场景；若被切碎则说明 maxBytes 需按「待定决策」上调）。

## 八、dsh 参考锚点（实施时照哪抄、抄多深）

> 总原则（所有待办通用）：**抄 dsh 的「逻辑形状与数值」，用我们自己的轻量代码实现同款；绝不引入 dsh 的 Cordis 插件框架 / 事件溯源 / 多包结构。** dsh 仓库位于 `/Users/honlnk/project/deepseek-harness`（本地已克隆）。

本篇是**三篇待办里最能 1:1 照搬的**——dsh 的 read 工具是独立文件、三道闸只是常量 + 截断逻辑、不依赖插件框架，可几乎逐行对照。

**打开这些文件对照：**

| dsh 文件 | 抄什么 |
| :--- | :--- |
| `packages/fs/tool-fs/src/read.ts` | 三道闸的常量与截断流程：`READ_LIMIT=2000`（行）、`READ_MAX_LINE_LENGTH=2000`（单行字符）、`READ_MAX_BYTES=50*1024`（总字节，在 `read-render.ts:14`）、按字节截断的提示文案 |
| `packages/fs/tool-fs/src/read-render.ts` | 字节截断的实现细节（如何不切开多字节字符）、`truncatedByBytes` 标记的输出形态 |
| `packages/spill/spill-local/src/cleanup.ts` | spill 文件的保留期清理逻辑（我们把保留期改为 7 天，结构照抄） |
| `packages/spill/spill-policy/` 与 `spill/`（看形状即可） | 「生产时拦截超长结果 + 落盘 + 预览带取回提示」的流程形状 |

**抄多深**：三道闸数值、截断流程、提示文案、cleanup 逻辑——都可对照实现。字节口径用 `TextEncoder().encode(text).length`。

**坚决不抄**：dsh 的 spill 抽象层 `spill/`、`spill-local` 存储提供者、`spill-policy` 策略插件这套插件化结构；事件订阅与作用域注册。我们只在 `agent/spill.ts`、`read-file.ts`、`common.ts` 这几个现有文件里改。

## 关联

- 来源：第六次提交 `3b7a981` 的 review 会话（sess_6a541707），用户判定原方向错误。
- 与同族断言函数的另一处问题并列但方向相反：[[待办-写工具权限与novel防护一致性]]（那边补「写」防护，这边开「读」口子），实施时建议合并评审 `common.ts`/`path.ts` 的断言族。
- 同属 review 待修清单：[[待办-文件改动追踪不可靠]]。
