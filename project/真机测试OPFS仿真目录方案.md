# 真机测试 OPFS 仿真目录方案

## 元信息

- **性质**：测试环境配方（非施工计划）——记录「浏览器无法授予文件系统权限时如何照常真机测试」的完整做法
- **首次使用**：2026-10-20，[要素管理员提示词升级施工计划](../plans/要素管理员提示词升级施工计划.md) W4 真机回归（十验收点全过）
- **维护约定**：方案本身稳定不常改；每次使用后在文末「使用记录」追加一行

## 适用场景

NovAI 的真机测试需要打开一个真实项目目录（如 `~/note/test-novel`），而网页读写本地目录依赖 File System Access API 的授权面板。在以下环境该面板**弹不出来**，授权请求被自动拒绝并持久化为 `denied`，导致「恢复项目」「打开项目」全部失败：

- ZCode 内置浏览器（IAB）——无文件系统授权 UI（2026-10-20 实测）
- 任何嵌入式 WebView / 无头浏览器——同理（未逐一实测）

注意这是**环境限制，不是应用缺陷**。真实浏览器（Chrome 等）不受影响。

## 诊断步骤（先确诊再动手）

1. 正常路径点击「恢复项目」，页面无反应、无报错 → 怀疑权限层。
2. 在页面上下文探测存储句柄的真实权限状态（从 IndexedDB `novai-projects` 库 `recent-projects` 表取出记录的 handle）：
   - `handle.queryPermission({ mode: 'readwrite' })` 返回 `denied`
   - `handle.requestPermission({ mode: 'readwrite' })` 仍返回 `denied` 且无任何面板出现
3. 两条都 denied 即确诊：该环境无法授予本地目录权限，走本方案。

## 方案原理

OPFS（Origin Private File System，`navigator.storage.getDirectory()`）是浏览器标准 API，返回一个**真实的** `FileSystemDirectoryHandle`——`values()`、`getDirectoryHandle`、`getFileHandle`、`createWritable`、`removeEntry` 全部原生可用，且**天生免授权**（页面自己的沙箱）。

应用侧两个特性让「换柄」可以透明生效：

1. 最近项目记录存在 IndexedDB（`novai-projects` 库），每条记录的 `handle` 字段可整体替换；
2. core 文件层对句柄是鸭子类型调用（无 `instanceof` / `isSameEntry` 硬检查）。

因此：把项目文件副本种进 OPFS，再把记录的句柄换成 OPFS 根，应用的全链路（迁移逻辑、UI、真实 LLM 调用、工具执行、确认链）都在真实代码上运行，**只有存储传输层换了载体**。真实项目目录零改动。

被排除的替代方案（试过或评估过，勿重走）：

| 替代方案 | 不可行原因 |
| :-- | :-- |
| IndexedDB 里存自造的假句柄对象 | 结构化克隆会剥掉对象方法，句柄存不进去（读回是裸对象）；运行时也会 `DataCloneError` |
| 「打开项目」走目录选择器 | IAB 不支持原生文件选择对话框 |
| 等待权限面板 | 该环境根本渲染不出面板 |

## 操作配方

### 第一步：盘外构建种子（shell 侧）

从真实项目目录读出需要的文件（配置、提示词、要素、`.novai/` 下的预设等），打包成一个 JSON（`{ "files": { "<相对路径>": "<文本内容>" } }），放在 `/tmp` 下。二进制文件不进种子（本项目全是文本）。

需要测「存量迁移」时，在这一步向 `novel.config.json` 注入旧字段块（如 `gardener`）——种子是测试数据，可以自由做手脚。

注意：`novel.config.json` 含 API key，种子文件只放 `/tmp`，页面用完即弃；key 在页面内存中运行时使用，不写进任何仓库文件。

### 第二步：页面内播种 OPFS（浏览器自动化 evaluate）

把种子 JSON 作为 evaluate 参数传入页面，在页面上下文执行：

```js
const root = await navigator.storage.getDirectory();
// 1) 清空 OPFS 根
for await (const entry of root.values()) {
  await root.removeEntry(entry.name, { recursive: true }).catch(() => {});
}
// 2) 逐文件写入：按路径逐级建目录、建文件、写内容
for (const [path, content] of Object.entries(seed.files)) {
  const parts = path.split('/');
  const fileName = parts.pop();
  let dir = root;
  for (const part of parts) dir = await dir.getDirectoryHandle(part, { create: true });
  const fh = await dir.getFileHandle(fileName, { create: true });
  const w = await fh.createWritable();
  await w.write(new TextEncoder().encode(content));
  await w.close();
}
```

### 第三步：换柄并恢复

仍在页面上下文，打开 `novai-projects` 库，把 `recent-projects` 表所有记录的 `handle` 字段替换为 OPFS 根（`rootName` 一并置空串），`put` 写回。然后正常点击「恢复项目」——应用按原生流程打开「项目」，全程无感知。

### 附：该环境的输入通道差异

IAB 里 Playwright 合成点击（`locator.click()`，含 `force: true`）会超时**但无实际效果**。可用通道：

- `tab.cua.click({ x, y })` 坐标级真实输入（坐标先用 `getBoundingClientRect()` 量）
- `evaluate` 里 `element.click()` JS 派发（对 Vue `@click` 有效；足够触发应用逻辑，不产生原生授权所需的用户激活——本方案无需授权所以无碍）

## 验证与清理

**验证落盘**：从页面 evaluate 直接遍历 OPFS（或读 `novel.config.json`）核对文件状态，比 UI 目视更硬。

**清理**（测试遗留物三处，一次 evaluate 可全清）：

1. `recent-projects` 表里的测试记录（注意恢复操作会追加新 projectId 的记录，旧记录不会自动消失——会出现两条同名条目，属预期）；
2. OPFS 根的内容（`removeEntry` 逐项）；
3. 磁盘上的落点（供好奇/核对，不建议手工翻）：`~/Library/Application Support/ZCode/session/Partitions/zcode-embedded-browser/File System/`，文件以数字编号存储，人读无意义。

## 边界与注意事项

- 副本会带上测试产生的改动（重命名、写回、新增预设等）——它们只存在于 OPFS，与真实项目无关；
- 用户自己浏览器的站点存储独立存在，本方案不触碰；
- 涉及「写文件弹确认卡」「权限档位」类验收点时，子智能体在 OPFS 上的写操作与真实目录行为一致（走同一运行层路径闸与确认链），可正常验收；
- debug 日志若应用未落盘（如 `.novel/logs` 不存在），消息序列类验收点降级为单测断言。

## 使用记录

| 日期 | 任务 | 结果 |
| :-- | :-- | :-- |
| 2026-11-07 | 内部文件屏蔽单一来源与工具不可见化 W3 真机验收 | 全过；根目录列表只见 4 个目录（无 novel.config.json/.novel/.novai）、直接要求读配置撞教学文案后模型不再变体重试而用上下文作答（模型/项目/默认目标）并请用户自查、根目录内容搜 deepseek（配置内 4 处）零匹配零泄漏；本浏览器档案无历史记录可换柄（recordsUpdated: 0）——手工按 `LastProjectRecord` 结构插入 last-opened + project-{projectId}（projectId 取自种子 `.novel/manifest.json`）后恢复流程照常；用完即清（记录 2 条 + OPFS 7 项 + 种子删除） |
| 2026-10-20 | 要素管理员升级 W4 真机回归（十验收点） | 全过；验收点 5 按计划降级单测；种子注入 gardener 旧块验证迁移 |
| 2026-10-22 | 角色注册机制 W4 真机回归（三链路） | 全过；主写作/委派（双委派并发确认）/直聊+续聊/分组/确认徽标；真机暴露 session-created 列表刷新竞态并修复（`f968531`） |
| 2026-10-23 | 要素召回渐进式披露 W2–W4 真机验证（0007） | 全过；目录 reminder 落 messages[1]（`agent_messages_debug` 证据）、写场景先 LoadElements(4 要素)、索引重建仅 plots/timeline 2 文档、「魏征是谁」走 LoadElements / 绢商案走 RagSearch；真实项目 plots/timeline 为空，OPFS 副本补种各 1 文件；种子缺 `.novel/manifest.json` 致恢复卡住，手工补写 |
| 2026-11-03 | 创作计划板二/三阶段真机验收 + 四阶段回归 | 全过；看板手动操作全链路（建/编/拖/删/回收站/冲突/重开恢复/外部修改），真实模型会话五工具全走通（素材档 Create/Delete 弹卡、Update 免确认、拒绝路径），回归三分类无泄漏；真机抓到窄屏看板溢出（缺 `min-w-0`）当场修复；新环境 `recordsUpdated: 0` 需手工建 `recent-projects` 两条记录（key=last-opened 与 project-&lt;id&gt;，projectId 取 manifest）；本轮 AGENTS.md 已改为允许端口空时临时起 dev server |
| 2026-11-06 | 创作计划板交互优化第三阶段整链验收 | 全过；新建抽屉三态（提交/失败保留/成功转预览）、改名落盘与重开项目找回、分组高亮同步、已搁置筛选、保存→自动回预览（真机发现缺口当场修复）、删除两步确认进回收站、MD 主题回归；真机抓到两个问题当场修复：新建文件名缺可读计划名段（`identity.ts` 未接新格式）、保存成功不回预览态（`detail-mode.ts` 状态机）；窄视口长 URL 会把预览区撑出横向滚动（内容可滚读，记已知瑕疵）；HMR 编辑 core 源会整页重置视图（数据不丢）；`build-seed.sh` 缺 `plans/` 清单需手工补种 |
| 2026-11-06 | 计划页工作区布局统一计划第二阶段真机验收 | 全过；跨分类面板归属/恢复、看板三入口选中态一致、计划↔文件草稿往返不串稿、保存回预览、改名文件名同步、冲突双分支（重新读取/保留草稿）、两步删除进回收站、重开数据一致、四视口（1440/1280/1024/375）与键盘验收；真机抓到四个问题当场修复：文件草稿跨分类丢失（ProjectView v-if 改 v-show 保挂载）、改名与正文草稿并存时正文首行不同步（app saveCard 接 replacePlanTitleLine 组合）、移动端抽屉宽度不收敛致保存按钮屏外（ContentPanel 加 `max-lg:max-w-full`）、看板卡片键盘不可达（article 加 tabindex+Enter/Space）；发现 activeFile 一次无法复现的重置（疑激活异步竞态）；清理时确认 OPFS 与测试记录均可全部清除 |
| 2026-11-06 | 文件面板小改真机目视（删原始模式 + 预览/编辑卡片撑满高度） | 全过；上轮清理后 recent-projects 已空，按 2026-11-03 条目经验在播种 evaluate 里一并补建 last-opened 与 project-&lt;id&gt; 两条记录（chapterCount/elementCount 供首页统计）；章节 .txt 与要素 .md 两种格式均验证分段控件只剩预览/编辑、卡片撑满面板高度且滚动收进卡片内部、编辑模式 textarea 同样撑满；用完即清（记录 2 条 + OPFS 7 项 + 种子删除） |
| 2026-11-06 | 计划板顶栏统一真机目视（顶栏换 GitHub+面板开关、新建入口移侧栏） | 全过；侧栏新建按钮建卡→提交→详情全链路、顶栏开关收起/展开往返（有选中卡片时展开回详情）、默认收起态展开落到新建表单、`&lt;&lt;` 侧栏收起展开均验证；种子无 plans/ 目录（既有清单缺口）不阻碍——建卡走 UI 正常写盘；用完即清（记录 2 条 + OPFS 7 项 + 种子删除） |
| 2026-11-06 | 计划页布局统一施工 review 定点复核 | 按钮最终位置、白色看板、侧栏新建、跨分类草稿恢复、375px 详情底部操作可达均通过；复现新建面板窄屏关闭会切回已有详情而非收起（草稿未丢），与「关闭只影响可见性」约定不符；未保存测试草稿随副本清理，真实项目零改动；记录 3 条 + OPFS 6 项已清理 |
| 2026-11-06 | 计划页并发交互修复真机回归 | 375px 新建顶部关闭只收起、重新展开草稿保留；“取消”回原详情且新建草稿保留；普通保存与改名落盘通过（详情标题、正文首行、文件名一致）；在真实文件写入被暂停时同卡继续输入，保存完成后追加草稿仍在；A/B 两卡独立编辑保存不串稿。快速切卡、跨卡冲突与旧项目在途请求由 18 项受控事件序单测覆盖，未在真机人为制造；记录 2 条 + OPFS 7 项 + 临时种子已清理，真实项目零改动 |
| 2026-11-06 | 计划看板加载反馈修复真机验收（worktree 分支 `honlnk/plan-loading-feedback`，新规则下临时 5174 端口验证） | 全过；首载/手动刷新/HTML5 DnD 拖动/详情保存四场景连续采样（10ms 级），全程卡片数不变、无「正在读取计划…」白屏、无骨架屏闪现；手动刷新完成短暂显示「计划已更新」；拖动落列即时生效（松手即到「已完成」列）；保存追加段落实写 OPFS 落盘核对；本地目录小导致「正在刷新计划…」进行中态一闪而过（本地读取 <5ms，属预期快路径）；新建两条卡片走 UI 全链路真实写盘；用完即清（记录 3 条 + OPFS 7 项 + 种子删除 + 5174 服务关闭） |
| 2026-11-07 | 传统工具升级与内容搜索补齐阶段 4 真机验收 | 全过；五组对话：模型自主选 SearchFiles 搜人名（vs RagSearch 分工生效）、CreateFile 空文件 → EditFile oldText="" 填充 → 两次追加全免重读（OPFS 直读核验三行原文）、FindFiles mtime 降序 + 17 匹配跨 5 目录采样 + spill、SearchFiles 未超限分支（「他」3 处全列）与超限分支（「李承乾」limit 2 → 前 2 处 + spill）双覆盖；两个 spill 文件均 OPFS 直读核验内容；外部修改拦截按计划降级为单测覆盖、未真机直测；新环境 `recordsUpdated: 0` 沿 2026-11-03 经验手工建两条记录；Lexical 输入 execCommand insertText 在 focus + 300ms 停顿后可用（此前 2026-10-22 记录称必须 cua.type，本次复测有效，估计与停顿给 Lexical 焦点同步留时间有关）；AI 行内补全幽灵文本会残留在输入框，Cmd+A 全选删除后再插入；用完即清（记录 2 条 + OPFS 8 项 + 种子删除） |
| 2026-11-07 | 整体 UI/UX 升级调研：真机视觉与交互取证 | 部分完成；副本查看 1280px 空会话工作区、真实长章节、设置实际状态、计划空状态和 900px 稿件抽屉，收集截图供设计审计。单行真实键盘发送成功且未复现额外空行；真实编辑确认与主动提问未到达可验收状态，不能据此判断相关链路通过。仅副本设置档位变化，真实项目零改动；测试记录、OPFS 内容和临时种子已清理，首页恢复空项目状态 |
| 2026-11-07 | `/压缩上下文` 手动压缩斜杠命令真机验收 | 全过；斜杠菜单出现新命令与裸发提示、发送后本地拦截（无用户气泡、草稿清空）、状态栏「正在压缩上下文…→已压缩 2 条早期消息（约 944→799 token）」、聊天出现「已手动压缩」提示卡、真实 DeepSeek 产出八节检查点摘要、OPFS 直读核验 modelView 11→10 条且含 `<compacted-summary>`、压缩后续聊模型凭摘要准确复述此前进展；种子追加会话文件时以 JSON 对象塞 `files` 会被 `TextEncoder.encode` 写成 `[object Object]` 而被会话列表静默过滤——追加文件必须是字符串文本（本次用 evaluate 直接重写该文件纠正）；CUA 单键 keypress（Enter/Backspace）不可靠，发送用 evaluate 合成 `KeyboardEvent('keydown')`、输入用 `cua.type`；用完即清（记录 2 条 + OPFS 7 项 + 种子删除） |
| 2026-11-07 | 整体 UI/UX 升级调研：计划拖拽布局抖动受控复现 | 复现成功；副本建 6 张测试计划，CUA 坐标拖拽未生效，改用页面内原生 DragEvent 走真实拖拽处理、写盘与读回链；「正在同步计划…」横条高 29px，看板顶部 57→86→57px、高度 635→606→635px，约 12ms 闪入闪出，卡片始终 6 张；截图超时，证据取 DOM 状态与约 10ms 几何采样，真实鼠标设备拖拽尚未验证；用完即清（记录 2 条 + OPFS 7 项 + 种子删除，首页无测试项目，真实项目零改动） |
| 2026-11-07 | 交互体验优化阶段1真机验收（提示分级/计划板几何/输入矩阵/图标/设置页脚） | 全过；取消选目录黄色内联提醒（amber、absolute 零位移、3s 自动消失定时器确认触发）、真实失败仍红报错块；恢复项目 OPFS 原生流程打开；计划板同步横条元素已不存在、原地投放零落盘零位移、插入指示 absolute 拖完即清、真实重排写 frontmatter order 字段且刷新后保持；输入矩阵 Shift+Enter 不发送、组字期 Enter 无残段（preventDefault 生效）、尾窗鼠标发送被忽略、Enter 真实发送全链路（agent 真跑 1 次工具调用后回复）；设置页脚 idle/pending/saved 状态机 + 原型补丁模拟写盘失败走通红字重试/放弃并关闭/关闭拦截/丢弃生效；六分类、提示词、斜杠菜单、@ 弹层、状态栏图标全部命中受控注册表（lucide class 名核验）；注意 LLM 拦截桩要匹配 dev 代理相对路径 `/api-proxy` 而非 api.deepseek.com；发送失败回填未真机构造（入队成功后失败属轮次级错误、正确地不回填）；IAB 失焦标签 CSS transition 滞留使提醒 DOM 移除延迟（leave 类已出现证明逻辑触发，非应用缺陷）；新环境 `recordsUpdated: 0` 沿既有经验手工补建两条记录；用完即清（记录 2 条 + OPFS 7 项 + 种子删除） |
| 2026-11-07 | 交互体验优化阶段2真机验收（现场恢复/会话归档/命令显示分离/压缩状态行） | 全过；旧会话平铺兼容、现场恢复跨刷新与重开（含队列跨刷新保留）；归档运行中会话 toast 双动作、行即离列表且**归档后消息 26→36 持续增长（归档不停止运行实证）**、刷新后归档区恢复（updatedAt 未触碰）、连续归档两 toast 各自撤销互不覆盖；气泡/队列/自动标题全链路显示命令原文而非内部 prompt（主会话标题落 `/管理要素 整理地点`）；`/压缩上下文` noop 状态行（未执行徽标）+ 裸发提示 + 带参数琥珀提示且草稿保留（拼接草稿被守卫二次拦截属意外复验）；真机抓到「快照 waiting-user 当待回答兜底」徽标噪音当场修复（实时注册表为唯一来源）；页面重载杀驱动遗留孤儿 tool-call 令该会话后续运行持续被 API 拒绝（既有边界，记入计划遗留观察）； Lexical 输入确认「点击编辑器→Range 置光标末尾→execCommand insertText」稳定可用，会话切换后首次插入易丢失需校验重试，清空草稿用 beforeinput deleteContentBackward（execCommand delete 无效）；新环境 `recordsUpdated: 0` 沿既有经验手工补建两条记录；用完即清（记录 2 条 + OPFS 7 项 + 种子删除） |

## 已知限制追加（2026-10-23）

- **模态框判开关别用 `[role="dialog"]`**：BaseModal 没有该属性，会误判「已关闭」而实际浮层（`div.fixed.inset-0` z-50）仍压在输入框上，CUA 坐标点击全被吞。判遮挡用 `document.elementsFromPoint(x, y)` 看栈顶是否为浮层；关闭按钮是空 textContent 的图标按钮，按 aria-label 或在浮层内按位置找。
- **输入框被浮层挡住时的替代输入法**：`evaluate` 里 `editor.focus()`（程序化聚焦不受浮层影响）再 `cua.type`，Lexical 正常接收真实键盘事件，发送按钮随之启用；发送用 `evaluate` 里 `sendBtn.click()`。
- **种子清单必须含 `.novel/manifest.json`**：恢复项目的校验卡在 manifest，缺了会静默无反应（`recordsUpdated: 0` 且点击无响应）。`build-seed.sh` 的清单若遗漏需手工在 evaluate 里补写该文件（projectId 用真实项目的）。

## 已知限制追加（2026-10-22）

- **IAB 偶发整页重载会杀掉内存中的运行链**（原因未定位，怀疑渲染进程被回收）：表现为运行中会话停在「排队中」、待确认清空、runStatus 归零；磁盘状态完好，重发指令即可续跑（被杀轮次的排队消息在下一轮被消费，主智能体可能连发两次委派）。真机回归遇到「会话莫名没跑」先怀疑这个，别急着改产品代码。
- IAB 里 Playwright 合成 click 对 Vue 组件常不生效，用 `tab.playwright.evaluate` 里 `element.click()`；Lexical 输入框必须走 CUA 坐标点击 + `cua.type`（execCommand 无效）；ActivityBar 分类按钮的 `title` 属性由悬停 tooltip 机制填充（平时为空），稳定选择器是 `nav button[aria-label="对话"]`。
