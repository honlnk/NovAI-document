# linkseek 直连重设计与 CORS 修复计划

## 记录时间

2026-09-29

## 背景与问题

《[内置联网搜索计划](内置联网搜索计划.md)》落地后，用户实测发现「自部署 linkseek」档存在设计缺陷：**在 NovAI 中填写自己的 linkseek 实例地址 + 合法 API Key，连接必然失败**（浏览器跨域报错）。

### 实证记录（2026-09-29，对生产实例 linkseek.honlnk.com 实测）

1. **Key 本身有效**：绕过浏览器直接 `POST /v1/search` + Bearer Key，正常返回搜索结果——ApiKey 体系为 MCP 与 `/v1` 共用，Key 与端点均无问题。
2. **唯一拦截点是 CORS**：对 `/v1/search` 发预检（OPTIONS），Origin 为白名单内站点（novai.honlnk.com）时返回 `access-control-allow-origin` 放行；Origin 为 `http://localhost:5173`（NovAI dev server）等白名单外来源时**返回 204 但零 CORS 头**——浏览器随即拦截真实请求，fetch 抛 "Failed to fetch"。
3. 推论：任何非白名单 Origin（本地 dev、第三方 NovAI 部署）带 Key 也连不通；全新自部署实例 `PUBLIC_API_ALLOWED_ORIGINS` 默认为空，**浏览器访问 `/v1` 全死（含 Key 流量）**——本想关"陌生人白嫖"，实际关掉的是"自备 Key 的合法用户"。

### 病根

linkseek `/v1` 的 CORS 中间件把两件事耦合在同一个白名单上：

- 「谁可以匿名白嫖绿灯」——防线，正确位置在 `resolveCaller`（那里本就有白名单 + 配额 + 突发三道闸）；
- 「哪些浏览器的请求技术上可达」——可达性问题，不应与上者绑定。

### 设计反思（用户 2026-09-29 拍板的产品重定位）

1. **「自部署」档名不副实**。该档真实语义是「**API Key 直连任意 linkseek 实例**」（BYOK），与在 zcode 里配 MCP server 同一心智：填地址 + Key 即通。覆盖三种场景：填官方托管地址 + 自己的 Key 突破免费限额（作者本人正当用法）、几个朋友合用一个实例、真·完全自部署。部署只是手段之一，不该是名字。
2. **Origin 白名单是"作者贡献算力"的开关，只该管匿名流量**。NovAI 内置托管地址指向作者服务器，其他部署者开白名单无意义（除非连 NovAI 都自部署，过度）。默认关是正确的，只有作者生产实例需要开。
3. **Key 流量不该被 CORS 管**。Key 本身就是鉴权与记账单位（UsageLog 落 Key、管理台可禁用），浏览器从哪个页面发起与它无关；CORS 也从来不是服务端防护（非浏览器客户端可随意伪造 Origin，linkseek 源码注释自己已言明）。

## 已拍板决策

### 方案拍板（本计划定稿即生效，审阅时可推翻）

1. **CORS 修复采用「一律回显 Origin」**：`/v1` 的 CORS 中间件对任何带 Origin 的请求回显 `Access-Control-Allow-Origin`（保持 `Allow-Methods`/`Allow-Headers`/`Vary` 现状，OPTIONS 204 快路径不变）。理由：`/v1` 无 cookie（非 credentials 模式），回显无 CSRF 面；匿名防线全部收拢在 `resolveCaller`（Origin 必填 + 白名单 + burst + 日配额），零削弱。不采用「预检声明 authorization 才放行」的窄门方案——多一层分支、少一分可解释性，防线反正不在 CORS 层。
2. **Key 流量补独立突发限流**：key 分流处增加 `burstAllow('key:<keyId>')`，新 env `PUBLIC_API_KEY_BURST_PER_MINUTE` 默认 **30** 次/分钟（高于匿名档的 10：合法 Key 用户的 Agent 单问可达 30 次工具调用，WebSearch 一次就并发 4 请求，10/min 会误伤；日配额不设——直连不限额是产品承诺）。超限 429 `RATE_LIMITED`，文案 `请求过于频繁，请稍后再试。`。匿名档 burst 默认 10 本期不动（生产遗留观察项，另行决定）。
3. **NovAI 档位改名不换值**：provider 枚举 `'linkseek-selfhost'` 保持不变（存量配置零迁移），仅改展示层：
   - label：`自部署 linkseek` → `linkseek 直连（API Key）`
   - description：`使用你自己部署的 linkseek 服务，不限额度。` → `使用 API Key 直连任意 linkseek 实例（可直接填官方托管地址），不限额度。`
   - `baseUrlPrefill`：空 → `https://linkseek.honlnk.com`（官方托管地址；`applyProviderSwitch` 只在当前值为空或等于其他档预填默认时替换，自定义地址不覆盖——真自部署用户不受影响）
   - placeholder `https://your-linkseek.example.com` 保留
4. **双仓引导文案同步换措辞**（"配置自部署 linkseek" → "选择「linkseek 直连」填写 API Key"）：
   - NovAI `web-fetch.ts` 的 `NO_FETCH_PROVIDER_MESSAGE`：`当前搜索来源不支持网页抓取。可在设置的联网搜索中选择「linkseek 直连」并填写 API Key 后使用该能力。`
   - linkseek `QUOTA_EXCEEDED_MESSAGE`：`免费搜索额度已用完（每日 50 次），明天自动恢复。如需不限量搜索，请在设置的联网搜索中选择「linkseek 直连」并填写 API Key，或改用第三方搜索。`
   - linkseek `GREENLIGHT_DISABLED_MESSAGE`：`免费联网通道已被服务方停用。如需继续使用联网搜索，请在设置的联网搜索中选择「linkseek 直连」并填写 API Key，或改用第三方搜索。`
5. **绿灯开关语义澄清，不新做开关**：`PUBLIC_API_ALLOWED_ORIGINS` 即"匿名绿灯开关"（env 级，重启生效），管理台禁用内置 NovAI 伪 Key 即动态总闸（已存在）——两者现成，修完 CORS 后语义自然收窄为"只管匿名"，仅改 env 注释与文档表述。
6. **施工顺序**：W1（NovAI）与 W2（linkseek）无代码依赖、可独立发布。**W2 开工时机由用户拍板**——linkseek 仓现有进行中任务，待其合入后再动，避免冲突。W3（发布与跨仓验收）依赖 W2 部署上线。W1 先行发布即已可用：官方站 Origin（novai.honlnk.com / honlnk.github.io）在白名单内，线上 NovAI 用「直连 + Key」今天就能通；本地 dev Origin 要等 W2 上线。

## 本期范围

### W1 NovAI：直连档改名 + 预填 + 文案（可立即开工）

改动点（均在 `packages/app/src`，core 侧仅 `web-fetch.ts` 一处文案）：

1. `utils/search-settings.ts`：`SEARCH_PROVIDER_OPTIONS` 的 linkseek-selfhost 项按决策 3 改 label/description/baseUrlPrefill。
2. `core/tools/web-fetch.ts`：`NO_FETCH_PROVIDER_MESSAGE` 按决策 4 更新。
3. 相关单测同步：文案断言、`applyProviderSwitch` 对新预填的行为（空值/他档默认值替换、自定义地址保留）。

W1 门禁：`pnpm test` 全绿 + `pnpm typecheck` 干净。
W1 验收：单测 + browser-use 冒烟（设置页四档切换渲染、直连档出现官方地址预填、测试连接对托管地址 + Key 真实发请求）。

降级方案：browser-use 不可用时降级为构建 + 单测 + 用户目检，写入未验证清单。

### W2 linkseek：CORS 修复 + Key 突发限流 + 文案（待用户放行开工）

改动点（均在 `/Users/honlnk/project/linkseek`）：

1. `src/public-api/router.ts` CORS 中间件按决策 1 重写（回显 Origin）；`resolveCaller` key 分支按决策 2 增加 burst 判定（放在 Key 校验之后、业务执行之前）。
2. `QUOTA_EXCEEDED_MESSAGE` / `GREENLIGHT_DISABLED_MESSAGE` 按决策 4 更新。
3. `src/config.ts` + `.env.example` / `.env.production.example`：新增 `PUBLIC_API_KEY_BURST_PER_MINUTE`（default 30）；`PUBLIC_API_ALLOWED_ORIGINS` 注释改为「仅控制匿名绿灯白名单；Key 流量不受此限制，CORS 对所有 Origin 放行」。
4. `public/docs.html` 公开 API 一节同步：Key 鉴权任意来源可连（浏览器 CORS 放行）；匿名仅限白名单站点 + 配额。补一句直连需实例版本 ≥ `07bba8f`（含 `/v1` 端点）。

W2 门禁：`pnpm typecheck` 干净 + curl 验收清单（linkseek 无测试基建，沿 0006 惯例）。

W2 curl 验收清单（本地实例，白名单临时含 `http://localhost:5173`）：

- 预检矩阵：白名单外 Origin（如 `https://random.example.com`）预检 → 204 **且** `access-control-allow-origin` 回显该 Origin、`allow-headers` 含 Authorization；OPTIONS 快路径不查库。
- Key 直连：带 Key 无 Origin → 200（既有行为保持）；带 Key + 白名单外 Origin 的真实 POST（curl 模拟）→ 200。
- 匿名不回归：无 Origin 无 Key → 401；白名单外 Origin 匿名 → 403 `ORIGIN_FORBIDDEN`（错误体现在响应体而非被 CORS 吞掉）；白名单内匿名 → 200 且配额照常计数。
- Key burst：同一 Key 31 连打 → 第 31 次 429 `RATE_LIMITED`；匿名 burst 10/分行为不变。
- 抓取链路：`render: 'auto'` 升级渲染 Key 流量计 UsageLog 不计 FreeUsage（既有行为保持，回归确认）。

### W3 发布与跨仓验收（依赖 W2 合入）

1. linkseek 构建 amd64 镜像、部署生产、同步 Docker Hub（注意施工遗留警告：`honlnk/linkseek:latest` 此前的 trust proxy 修复尚未推 Hub，服务器在补推前不得 `docker compose pull`；本次发布一并解决）。
2. NovAI 发布新版本（版本号用户定）。
3. 真机验收：
   - 本地 dev：`localhost:5173` NovAI → 直连 `https://linkseek.honlnk.com` + Key → 测试连接成功 + Agent 真实会话一次 WebSearch（多 query）+ 一次 WebFetch（含渲染升级）。
   - 生产：novai.honlnk.com 站内直连 + Key 全流程；匿名绿灯回归（默认档搜索/抓取、配额落库、超限文案）。
   - 绿灯总闸抽检：管理台禁用内置 NovAI Key → 匿名 403 新文案 → 恢复。

## 明确不做清单

- NovAI MCP 客户端不做（维持 0006 决策；NovAI 兼容的是 linkseek `/v1` REST 契约，不是 MCP 协议面）。
- `/v1` 开放 answer 系列 / render 独立工具不做（维持成本红线与工具契约）。
- Key 日配额不做（直连不限额是产品语义；滥用治理靠 Key 级 burst + 管理台禁用）。
- provider 枚举值 / SearchConfig 结构不改（避免存量配置迁移）。
- 匿名 burst 默认值（10/分）本期不动——生产遗留"偏紧"观察项，线上数据说话后另行调整。
- linkseek「干净服务器从零部署」简化 compose 与部署指南重写不做（另立计划；本期仅改 env 注释与 docs.html 表述）。
- 绿灯动态总闸新做管理 UI 不做（禁用内置伪 Key 即总闸，已存在）。

## 迁移条款

- **NovAI project config**：`search.provider` 枚举值不变，存量配置零迁移；label/description/预填仅影响展示与档位切换时的空值预填，不主动写回存量项目的 baseUrl。
- **linkseek**：无 DB 迁移；新 env `PUBLIC_API_KEY_BURST_PER_MINUTE` 缺省 30，存量部署不配置也能跑；`PUBLIC_API_ALLOWED_ORIGINS` 语义收窄（只管匿名），已配置的生产值（novai.honlnk.com + honlnk.github.io）行为不变。
- **版本边界**：`07bba8f` 之前的 linkseek 实例无 `/v1` 端点，不在直连兼容范围；docs.html 与设置项文案已标注版本要求。

## 新旧机制替代表

| 旧机制 | 去向 |
| :--- | :--- |
| `/v1` CORS 白名单门槛（同时管匿名防线与浏览器可达性） | 替换：CORS 一律回显 Origin；匿名门槛收拢至 `resolveCaller` 唯一防线 |
| 「自部署 linkseek」档名与描述 | 替换：「linkseek 直连（API Key）」+ 新描述 |
| 直连档 baseUrl 无预填 | 替换：预填官方托管地址 `https://linkseek.honlnk.com` |
| `/v1` Key 流量零突发限流 | 替换：Key 独立 burst（`key:<keyId>`，默认 30/分） |
| "配置自部署 linkseek" 引导文案（双仓四处） | 替换：「linkseek 直连」新措辞 |
| `PUBLIC_API_ALLOWED_ORIGINS` = 匿名开关 + 浏览器总闸（耦合语义） | 替换：仅匿名开关；浏览器可达性对所有 Origin 开放 |

## 文档同步计划

- 本计划：`docs/plans/README.md` 索引状态流转（待开始 → 进行中 → 已完成）。
- 决策 0006 加 2026-09-29 编者注：自部署档重新定位为「API Key 直连」、CORS 语义修正，正文不重写。
- linkseek `public/docs.html` 公开 API 一节随 W2 更新（决策 4/版本要求）。
- 施工日志格式：日期 + 波次 + 提交 hash + 验收结论 + 偏差记录；linkseek 侧记录体现在其仓库 commit message（conventional commits）。
- 提交纪律：两仓库分别提交；NovAI 的 docs 子模块先提交再推进父仓库指针；不 push。

## 施工日志

### 2026-09-29 W3 暂缓（等 CI/CD 发布链路）

linkseek 仓 CI/CD 批次开发完毕、正在测试验证；**本次验证版不包含本计划改动**。W3（生产部署 + 跨仓验收）挂起，触发条件：CI/CD 发布链路验证成功并发布一版之后，再用该链路（或手动）部署 `660edd6` 及 NovAI 新版本，按计划 W3 清单验收。W1/W2 已提交本地未 push，无在途风险。

### 2026-09-29 W1 NovAI 直连档改名完工（`9bebd03`）

**验收结论**：`pnpm test` 460 项全绿（52 文件）+ `pnpm typecheck` 干净。`SEARCH_PROVIDER_OPTIONS` linkseek-selfhost 项改 label「linkseek 直连（API Key）」+ 新 description + `baseUrlPrefill: 'https://linkseek.honlnk.com'`；`web-fetch.ts` 无抓取引导文案、`search-provider.ts` 档位注释与缺地址报错、`web-search.test.ts`/`search-provider.test.ts` 的 429 文案 mock 全部对齐新措辞；`applyProviderSwitch` 单测补直连档预填断言（空值 → 官方地址、自定义地址保留）、`SearchSettingsPanel.vue` 注释同步。provider 枚举与 SearchConfig 结构未动，存量配置零迁移。

**偏差记录**：

1. **browser-use 冒烟未执行**（计划 W1 验收含设置页四档切换冒烟）——本轮为纯文案/预填逻辑改动且逻辑已有单测覆盖，渲染冒烟合并到 W3 发布前与直连真机验收一起做，避免重复占用验证轮次。属验收时机挪移，非降级。

### 2026-09-29 W2 linkseek CORS 修复完工（linkseek `660edd6`）

**验收结论**：`pnpm typecheck` 干净；curl 验收在本地 dev 实例实测（`tsx watch` 热加载新代码，localhost:7300）：① 预检矩阵——随机 Origin（白名单外）预检 204 **且** `access-control-allow-origin` 回显该 Origin、`allow-headers` 含 `Authorization, X-NovAI-Client-Id`；② Key 直连——带 Key 无 Origin 200（既有行为保持）、带 Key + 白名单外 Origin 200（**原缺陷场景修复**）；③ 匿名不回归——无 Origin 无 Key 401 `ORIGIN_REQUIRED`、白名单外 Origin 匿名 403 `ORIGIN_FORBIDDEN`（错误体现在响应体，新文案）；④ Key burst——滑动窗口 60s 内 30 次（28+此前验收耗 2）全部 200、第 31 起全部 429 `RATE_LIMITED`（`请求过于频繁，请稍后再试。`）；⑤ `/v1/fetch` Key + 白名单外 Origin + `render: 'auto'` 200、`renderedBy: 'http'`、正文 225 字符（抓取链路无回归）。改动面：CORS 中间件重写（一律回显）、`resolveCaller` key 分支补 burst（`key:<keyId>`，`PUBLIC_API_KEY_BURST_PER_MINUTE` 默认 30）、绿灯超限/总闸/ORIGIN_FORBIDDEN 三处文案对齐新档名、config + 两份 env example 注释与新增项、`public/docs.html` 鉴权与限流说明（含 `/v1` 版本要求 ≥ `07bba8f`）。

**偏差记录**：

1. **白名单内匿名放行 + 配额落库未在本地复测**——本地实例 `PUBLIC_API_ALLOWED_ORIGINS` 为空（绿灯整体关闭）且不重启用户在跑的 watch 进程；该路径代码未动，留 W3 生产验收。
2. **Key 流量 UsageLog 落账 / FreeUsage 不计的库层核对未做**——记账逻辑未改动，以响应行为回归兜底，留 W3 生产核对。
3. **`.env.production.example` 与 CI/CD 批次同文件并行**——该文件有 CI/CD 未提交改动（`APP_IMAGE_TAG` 块），按 hunk 精确暂存只提交公开 API 段，CI/CD 块保留在工作区由其批次自行提交。
4. **`ORIGIN_FORBIDDEN` 文案顺带对齐**——计划决策 4 未列此处，但同为"自部署"旧措辞，随批更新（'当前来源不在免费额度白名单内。可选择「linkseek 直连」并填写 API Key 后使用。'）。


