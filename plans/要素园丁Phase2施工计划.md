# 要素园丁 Phase 2 施工计划

## 记录时间

2026-10-01 立稿（依据《[要素园丁子代理设计](../architecture/要素园丁子代理设计.md)》v3 §8 Phase 2 范围，与用户立项讨论逐点拍板）。

## 状态

已完成（2026-10-01 施工完毕；W1-W3 三波提交 cfaa0f4/ac1f0a6/2070c2d，真机验收项见文末未验证清单）。

## 背景与范围

Phase 1（2026-09-25 结项）交付了委派式子代理最小闭环：`DelegateToGardener` 工具、园丁运行壳、`/整理要素` 命令、嵌套任务组 UI、`prompts/ELEMENT.md` 规范层；真实 LLM 全流程已于 2026-10-01 用户真机复核通过。当前园丁只能被动触发、运行配置无条件继承主 Agent。

Phase 2 三件套（设计 §8）：

1. **设置页园丁独立配置**（模型 + 确认档位，决策 3/4 的覆盖项）；
2. **快捷入口**（快捷键 / 一键按钮触发委派）；
3. **主 Agent 主动委派**（创作中判断何时联系园丁）。

施工顺序 **W1 配置 → W2 快捷入口 → W3 主动委派**：前两件可单测、低风险、改动面小，先落地拿确定性；W3 是行为改动（提示词层），放最后单独上，行为异常时可归因（排除配置问题干扰）。

## 已拍板决策（本计划定稿即生效，审阅时可推翻）

### D1 覆盖注入点 = `runGardener` 合成快照（否决 confirm 闭包方案）

确认链路的判定顺序是：`tool-execution.ts` 先用主档位做 allow/ask 判定 → 判 ask 才调 confirm 弹卡。若档位覆盖做在 confirm 闭包里（收到园丁请求时查园丁档位自动放行），**只能放宽、不能收紧**——主档位 allow 时根本不会走到 confirm，园丁设「仅审阅」也拦不住。这是链路深处的坑。

正确做法：`runGardener` 是园丁运行的唯一入口，在每次运行开始时把覆盖合成进 config / project 快照——

```ts
const effectiveConfig = mergeGardenerOverrides(input.config)
const effectiveProject = { ...input.project, config: effectiveConfig }
```

判定函数读 `project.config.settings.permissionPreset`、协议适配器读 `config.llm`，合成后全链路无感知地**双向生效**（放宽与收紧都正确）。`project.handle` 同引用、RAG 按 projectId 照常命中，零副作用；合成值仅在园丁运行内存中存在，不落盘。改动收敛在 `gardener.ts`，不碰 Loop、不碰 tool-execution、不碰 permission.ts。

### D2 配置形态 = 可选字段，无块即继承（零迁移）

```ts
// types/project.ts 新增可选块
gardener?: {
  /** 未定义 = 继承主 Agent 当前 llm 配置；整组覆盖（协议+地址+key+模型+思考档），不拆字段混搭 */
  llm?: { baseUrl: string; apiKey: string; model: string; protocol: ModelProtocol; reasoningEffort?: ReasoningEffort }
  /** 未定义 = 继承 settings.permissionPreset */
  permissionPreset?: GardenerPermissionPreset
}
```

不用 `enabled` 布尔开关：旧项目无此块 = 全继承，零迁移语义天然成立（与 `reasoningEffort` 缺省口径一致）；布尔开关多出「开了开关但没填全」的非法态。llm 整组覆盖是刻意的——防半覆盖拼出「anthropic 协议配 gemini 模型」类无效组合；normalize 对残缺 llm 块整体剔除、回退继承。设置页 UI 用开关呈现（开 = 写入整块，关 = 删除块）。

### D3 园丁档位裁剪三档（不用五档）

`type GardenerPermissionPreset = Extract<PermissionPreset, 'review' | 'material' | 'full'>`。理由：园丁被路径闸限制只能写 `elements/`，「章节档」对它效果等同「仅审阅」（elements 改动照样全弹卡），照搬五档会出现两个等价档的迷惑项。三档语义：**仅审阅**（全弹卡）/ **素材静默**（改 elements 正文静默，新建文件仍弹卡）/ **完全静默**。判定函数 `decideWriteToolPermission` 无需改动——合成后的 preset 落在三档值域内，现有规则对这三档的行为本来正确。normalize 校验只接受三值，UI 下拉只列三项。

### D4 思考档进覆盖块；缺省动态继承主 Agent 当前档

`reasoningEffort` 是覆盖块 llm 组的可选字段。写了用写的（例：园丁固定 max，主 Agent 平时 low）；不写就继承主 Agent **委派当时**的档位。设置页园丁面板带思考档选择器（照输入框四档选择器形态），外加「跟随主 Agent」缺省态。

**动机叙事（用户 2026-10-01 拍板纠正）**：要素整理是**高认知负荷任务**——同名/近名合并是语义判断、归位错了丢事实、跨章节归拢要理解前后文、「存疑不合并」的边界是高判断力活（类比：很多人整理笔记用最贵的 Claude 模型）。独立模型配置的意义是**让园丁可以配得比主 Agent 更强（或不同），独立控制**；不是「机械活省钱降级」——此前讨论中「整理配便宜模型」的预设是误区，本计划全文与此口径对齐。设计文档风险表 token 成本条随 W1 同步修正表述。

### D5 继承语义 = 按次取值、运行中定格

未覆盖字段在**每次委派时**取主配置当时值（动态跟随：今天切了模型，明天的园丁就跟着用新模型）；一次园丁运行**中途**改主配置不影响正在跑的这次（合成快照已定格），下一次委派才跟随新值。此「按次定格、跨次跟随」为正确行为（一次运行内配置一致性），非待修缺陷。设计文档决策 3/4 只写了「默认继承」，此精确口径由本计划补定。

### D6 主动委派形态 = 先建议、后委派（起步）

主 Agent 判断该整理时，**先在对话里建议**（例：「本章新增 3 个实体、〈人物〉状态有变，建议让园丁整理一下」），用户点头（回复确认或点击建议）后才委派。不直接静默委派。理由：行为可预期、符合「建议优先」的产品气质；园丁写文件虽有确认卡兜底，但委派本身也产生 token 花费，应经用户同意。直接委派形态作为后续放宽方向，真机观察建议链路的实际体验后另行决定。

### D7 防滥用 = 提示词软约束起步，硬闸本期不做

提示词教主 Agent「收尾时回顾本轮改动」（判断数据天然免费：本轮工具调用历史就在它自己的上下文里）：涉及要素的改动有明确信号（新增实体较多、人物状态大变、用户提到设定对不上）才建议；改动轻微或本会话已建议过不重复。session 层硬节流闸（记录上次委派时机、间隔不足工具直接返回提示回灌）**本期不做**，视 W3 真机表现另批。

## 本期范围（波次）

### W1 园丁独立配置（core + 设置页）

core 改动：

- `types/project.ts`：`gardener` 可选块 + `GardenerPermissionPreset` 类型；
- `project/defaults.ts`：normalizeProjectConfig 处理 gardener 块（缺省不动 / 完整保留 / 非法值剔除；llm 块残缺整体剔除回退继承）；
- `agent/gardener.ts`：`mergeGardenerOverrides` 纯函数（D1/D2/D5 语义）+ `runGardener` 改用合成结果。

app 改动：

- SettingsModal 新增「园丁」分组：继承态提示行（默认）+ 展开后模型面板（协议下拉复用自定义浮层组件 / 地址 / PasswordInput / 模型 / 获取列表 / 测试连接，复用 `models-client.ts`）+ 档位下拉（三档，带「继承主 Agent」项）+ 思考档选择器（四档 + 「跟随主 Agent」缺省态）。

测试：normalize 矩阵（无块 / 完整 / 非法 / 残缺 llm 块）；`mergeGardenerOverrides` 展开（未覆盖字段动态取主值、覆盖字段整组生效）；园丁端到端测试照现有 fetch stub 模式加「覆盖生效」用例（断言请求打到覆盖后的 baseUrl / model）；权限矩阵（三档 × EditFile / CreateFile × elements 路径：review 全弹、material 改正文静默新建弹、full 全静默）。

门禁：`pnpm test` 全绿 + core/app 双 typecheck。
验收：browser-use 冒烟（开关切换渲染 / 档位下拉 / 测试连接真实发请求）+ 真机 test-novel 给园丁配置独立模型跑 `/整理要素`，从请求日志确认走覆盖端点。

### W2 快捷入口（纯 app 层）

- 全局快捷键 + 工具栏一键按钮，触发即 `sendMessage(GARDENER_TASK_PROMPT)`，完全复用现有链路（零 core 改动）；
- Agent 运行中触发走现有 enqueue 排队机制；
- 一键触发任务范围为全库整理；「只整理最近章节」变体本期不做（用户用自然语言委派已可表达范围）。

门禁：全量测试 + 双 typecheck + 生产构建。
验收：browser-use 冒烟（快捷键与按钮触发、运行中排队）+ 真机。

### W3 主 Agent 主动委派（提示词层）

- `gardener.ts` 委派工具 description 改写：从「适合存量要素治理」的被动定位扩写为主动指引（触发信号 / 频率节制 / D6 建议形态）；
- `prompt.ts` 工作原则区加一条收尾回顾指引（D7 软约束）；
- 不加硬闸、不加 config 开关（D7）。

门禁：全量测试 + 双 typecheck + 生产构建（提示词改动无新单测面）。
验收：真机多轮写作会话观察——建议时机是否合理（有信号才建议、不重复建议、无信号沉默）；建议文案用户点头后委派链路正常。行为类验收，browser-use 只冒烟无回归。

## 明确不做清单

- session 层硬节流闸（视 W3 真机表现另批）；
- 直接委派（不经建议）形态——后续放宽方向另议；
- 五档完整档位（D3 已裁剪三档）；
- 园丁独立压缩阈值 / agentMaxTurns / 独立子代理模型注册表（继续继承与复用，无第二后端需求）；
- `send_message / interrupt_agent / list_agents` 子代理间通信（dsh 参考标「后期」，单园丁深度 1 用不上）;
- 「只整理最近章节」快捷变体（自然语言委派已可表达）；
- Phase 3 全部内容（链接解析渲染 / 反向链接 / 索引页与最近变更 / 每章建议整理触发评估）。

## 迁移条款

- **NovAI project config**：`gardener` 为新增可选块，旧项目无此块 = 全继承，零迁移；normalize 不主动写入，仅在用户在设置页开启覆盖时写入。
- **会话历史**：无迁移。委派工具 schema（task / scope）不变，W3 仅改 description 文案。
- **设计文档**：《要素园丁子代理设计》状态节随本计划立项更新；风险表 token 成本条表述随 W1 修正（D4 口径）。

## 新旧机制替代表

| 旧机制 | 去向 |
| :-- | :-- |
| `runGardener` 直通 `runtime.project.config`（模型/协议/思考档/档位全继承主 Agent） | 替换：`mergeGardenerOverrides` 合成快照（llm 与档位可覆盖；未覆盖字段每次委派时动态取主值，运行中定格） |
| 园丁确认档位 = 主 Agent 五档之一（无条件继承） | 替换：三档独立（review / material / full），缺省继承 |
| 委派工具描述「适合存量要素治理」被动定位 | 替换：主动指引（触发信号 / 频率节制 / 先建议后委派） |
| `/整理要素` 命令为唯一入口 | 保留并新增：快捷键 + 一键按钮（同一 sendMessage 链路） |

## 文档同步计划

- 本计划：`plans/README.md` 索引「待开始」立档，开工后随波次流转。
- 《要素园丁子代理设计》：状态节补 Phase 2 立项指引（本计划链接）；风险表 token 成本条随 W1 修正表述。
- `docs/project/当前进度.md`：Element 条目补 Phase 2 立项一句。
- 施工日志格式：日期 + 波次 + 提交 hash + 验收结论 + 偏差记录。

## 施工日志

### 2026-10-01 W1 园丁独立配置完工

**改动清单**：

- core `types/project.ts`：`gardener?` 可选块 + `GardenerPermissionPreset` 三档类型（D2/D3）；
- core `project/defaults.ts`：`GARDENER_PERMISSION_PRESETS` / `isGardenerPermissionPreset` / `MODEL_PROTOCOLS` / `isModelProtocol` 守卫；
- core `fs/project-fs.ts`：normalize 加 `normalizeGardenerConfig` / `normalizeGardenerLlm`——残缺 llm 块整组剔除（D2）、非三档 preset 剔除（D3）、非法 reasoningEffort 无键化（刻意不用显式 undefined：合成时展开覆盖会遮蔽主配置思考档）；
- core `agent/gardener.ts`：`mergeGardenerOverrides`（D1/D5）+ `runGardener` 改合成快照（config 与 project 双合成，判定链无感知双向生效）；
- services：`ProjectConfigView` / `ProjectConfigPatch` 加 gardener（patch 为**整组替换**语义：带键即替换，两项覆盖皆空 = 删块回退继承）；mappers 透传；`updateConfig` 加整组替换分支；
- app：`constants/permission-presets.ts` 加 `GARDENER_PRESET_OPTIONS` 三档文案；新增 `GardenerSettingsPanel.vue`（模型覆盖开关 + 复用 `ModelEndpointFields` purpose=llm + 思考档下拉含「跟随主 Agent」缺省 + 三档档位下拉）；`SettingsModal` 加「园丁」tab（第 7 个）、`gardenerForm` 回填 / watch / `buildGardenerPatch` 构造；
- 测试：`gardener.test.ts` 新增 9 条（merge 纯函数 4 + 端到端 5：覆盖端点生效 / material 静默 / review 收紧弹卡——confirm 闭包方案做不到的方向 / full 结构操作静默 / 无覆盖块继承基线），fetch stub 补 url/model 抓取；`project-fs.test.ts` 新增 4 条归一化矩阵。

**门禁**：54 文件 487 测试全绿 + core/app 双 typecheck + 生产构建通过（1.23s）。

**验收与降级**：browser-use 冒烟未执行——用户 dev server（5173）当前未运行，遵 AGENTS.md 不自起 dev server。兜底：新增逻辑全部有单测（权限矩阵双向 / 覆盖端点 / 归一化矩阵）+ 双 typecheck + 生产构建；设置页渲染冒烟（园丁 tab / 开关展开 / 档位下拉 / 思考档下拉）与真机验收（test-novel 配置园丁模型跑 `/整理要素` 看请求走覆盖端点）留待 dev server 恢复后补或随用户真机复核。

**偏差记录**：

1. **日期口径修正（计划外随批）**：此前立档与真机复核补记写的「2026-09-29」为臆断（实际对话日 2026-10-01），本次随批修正 8 份文档中的立档 / 复核 / 拍板日期；linkseek v1.3.0 发布等 git 历史事实日期保留不动。
2. **发现未修**：`services/types.ts` 的 `ProjectConfigView.llm.protocol` 注释仍写「生成链路当前仅实现 openai」——多协议计划结项时漏改的过时注释（四协议早已接入）。按纪律记日志不顺手改，留 W3 收尾或修复批。
3. **已知简化**：模型覆盖开关打开但四项未填全时，防抖保存写入的残缺块被 normalize 剔除，重开设置页开关回到关闭态——「不完整的覆盖 = 没有覆盖」行为自洽，面板文案已注明；不做「填一半保持开关」的暂态。

### 2026-10-01 W2 快捷入口完工

**改动清单**：

- app `composables/keyboard.ts`：`isGardenerShortcut`——Ctrl/Cmd+Shift+G 匹配（Shift 按下时浏览器报大写 'G'，比较前 toLowerCase 归一；带修饰键的组合不会处于输入法组合中，刻意不加 isComposing 排除）；
- app `ChatPanel.vue`：`triggerGardener` 共享触发函数——工具栏按钮 / 全局快捷键 / `/整理要素` 斜杠命令三入口合一（`followAndScrollToBottom` + sendMessage，斜杠分支改为复用）；window 级 keydown 监听（onMounted 挂 / onBeforeUnmount 摘，成对），`role="dialog"` 模态守卫（设置页等 BaseModal 弹窗打开时不抢键，避免弹窗背后触发委派）；输入区工具栏左组（权限档位旁）加「整理要素」一键按钮——照 PermissionPresetPicker 工具行样式与叶形图标，tooltip 注明快捷键与运行中排队语义；
- 测试：`keyboard.test.ts` 新增快捷键矩阵 1 条（大写/小写两形态命中、缺 Shift / 缺主修饰键 / 其他键位 / 无修饰键不误触、Ctrl+Cmd 同按无害命中）。

**门禁**：54 文件 488 测试全绿 + 双 typecheck + 生产构建（990ms）。

**验收与降级**：browser-use 冒烟未执行——dev server（5173）仍未运行，遵 AGENTS.md 不自起 dev server。兜底：快捷键匹配为纯函数且有矩阵单测；三入口共用同一触发函数，enqueue 行为完全复用既有 sendMessage 链路（chat store 既有已测行为）。冒烟项（按钮渲染与点击、快捷键触发、运行中触发进 QueueDock）与真机验收（触发后 Agent 确实委派园丁）留待 dev server 恢复后补。

**偏差记录**：

1. 无计划外偏差。快捷键选定 Ctrl/Cmd+Shift+G（G = 园丁 Gardener；Chrome 中该组合仅在查找栏已打开时为「查找上一个」，本应用 preventDefault 覆盖，无实际冲突）。

### 2026-10-01 W3 主 Agent 主动委派完工

**改动清单**：

- core `agent/gardener.ts`：`DelegateToGardener` description 重写——保留原被动定位要点（task 自包含 / 结构化汇报 / 不适合写作与章节修改），新增三段主动指引：**何时该委派**（触发信号：新增要素条目较多、人物状态或关系显著变化、设定前后矛盾、用户提到「对不上/重复/记混了」）、**频率节制**（改动轻微、要素无实质变化、本会话已整理过且无新改动 → 不委派）、**委派前先建议**（D6：说明观察到的信号、经用户同意再调用，不静默发起；用户明确要求整理含 `/整理要素` 时直接委派）；
- core `agent/prompt.ts`：工作原则 RagSearch 条后新增一条收尾回顾软约束（D7）——每轮收尾回顾本轮要素改动，有信号才建议、建议时说明信号、经同意再委派；改动轻微 / 无新信号 / 本会话已建议过不重复；
- 顺修：`services/types.ts` `LlmConfigView.llm.protocol` 过时注释（W1 偏差 #2 声明留 W3 收尾）——「生成链路当前仅实现 openai」改为「四协议生成链路均已接入」。

**门禁**：54 文件 488 测试全绿 + 双 typecheck + 生产构建（1.07s）。提示词改动无新单测面（照计划）。

**验收与降级**：W3 验收本身即真机行为观察项——多轮写作会话中建议时机是否合理（有信号才建议、不重复建议、无信号沉默）、点头后委派链路正常。行为类验收 browser-use 无从替代，dev server（5173）也未运行；留用户日常写作复核（见未验证清单）。

**偏差记录**：

1. 硬节流闸与 config 开关按 D7 拍板不做，无计划外行为偏差。
2. 结项提交时 commitlint 拒绝 `core` scope（允许清单无此项）——按历史惯例改用 `app`（Phase 1 园丁运行壳 `05c1ea0` 同例），仅影响提交信息的 scope 字段。

## 结项（2026-10-01）

W1-W3 三波全部完工，三笔代码提交：W1 `cfaa0f4`（园丁独立配置）、W2 `ac1f0a6`（快捷入口）、W3 `2070c2d`（主动委派）；对应 docs 子模块日志提交 `89d42a9` / `51a4f6f` / `b7c5ca2`。各波门禁（54 文件 487/488 测试 + core/app 双 typecheck + 生产构建）均绿。

### 未验证清单（真机复核项，留用户日常使用观察）

1. **W1 设置页面板真机**：设置页「园丁」tab 渲染与交互（开关展开 / 三档与思考档下拉 / 测试连接真实发请求）——两波施工时 dev server（5173）均未运行，冒烟未做；单测 + 双 typecheck + 生产构建兜底。
2. **W1 覆盖生效真机**：test-novel 给园丁配置独立模型跑 `/整理要素`，从请求日志确认打到覆盖端点（单测已覆盖合成逻辑本身）。
3. **W2 快捷入口真机**：工具栏按钮点击 / Ctrl/Cmd+Shift+G 触发 / Agent 运行中触发进 QueueDock 排队。
4. **W3 行为验收（本计划核心验收项）**：多轮写作会话中主动建议的时机与节制——有信号才建议、建议时说明信号、不重复建议、无信号沉默；用户点头后委派链路正常、转述园丁汇报。行为类验收只能真实模型多轮观察；硬节流闸是否需要，视此轮观察另批。

### 留下的尾巴

- session 层硬节流闸（D7 明确本期不做，视 W3 真机表现另批）；
- 「只整理最近章节」快捷变体（明确不做清单：自然语言委派已可表达范围）；
- 直接委派（不经建议）形态（D6 后续放宽方向，真机观察建议链路体验后另议）；
- Phase 3 全部内容（链接解析渲染 / 反向链接 / 索引页与最近变更 / 每章建议整理触发评估）。

### 2026-10-01 browser-use 冒烟补记（dev server 恢复后）

用户要求运行项目查看效果，dev server（5173）由本次会话启动后，对未验证清单 1/3 两项做了 browser-use 冒烟（项目：IAB 档案内既存「验收项目」，mock LLM 配置）：

- **W1 设置页面板**：设置弹窗「园丁」tab 存在（7 个分类第 6 位）；继承态提示行与两个开关默认关闭渲染正确；「独立模型配置」开 → 模型面板四件套展开（协议下拉 / 地址 / Key 显隐 / 模型 + 获取列表 + 测试连接），思考档下拉默认「跟随主 Agent」，openai 协议空地址下「关闭」档按 D3 规则隐藏（与主面板一致）；「独立确认档位」开 → 三档下拉（仅审阅 / 素材静默默认 / 完全静默，无章节档）；两开关拨回后恢复继承态（防抖保存「已自动保存」状态可见）。
- **W2 快捷入口**：工具栏「整理要素」按钮渲染于权限档位旁（叶形图标）；点击按钮 → 委派 prompt 用户气泡出现（sendMessage 链路正向验证）；window keydown 监听与匹配链路经页面内合成 KeyboardEvent（ctrlKey+shiftKey+G）派发验证通过（第二条委派气泡出现）；模态守卫验证——设置弹窗打开时同法派发不触发（消息数不增）。
- **冒烟未覆盖**：真实物理键盘组合（IAB 的 CUA 合成按键未把修饰键标志送达页面，故改用合成事件验证监听器本身——真实按键下的触发属真机体验项）；Agent 运行中触发排队（第一轮对 mock 端点快速失败，未构成 isRunning 窗口）；获取列表 / 测试连接真实发请求。

未验证清单 1/3 的「冒烟」部分就此闭环；剩余真机项（test-novel 覆盖端点、真实模型下 W3 行为观察）不变。
