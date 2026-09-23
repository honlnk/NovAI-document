# 内置联网搜索与 linkseek 集成

## 记录时间

2026-09-24

## 状态

已采纳。W1（linkseek 侧公开 REST 端点与匿名绿灯配额）已完工（linkseek `07bba8f`），NovAI 侧按《[内置联网搜索计划](../plans/内置联网搜索计划.md)》W2-W4 推进。修订 [0002](0002-ClaudeCode借鉴与映射设计.md)「明确不照搬 Web Search / Web Fetch」条目。

## 背景

0002 曾把 Web Search / Web Fetch 列入「明确不照搬」清单（第一阶段最小工具集）。现在产品阶段推进到需要联网能力：写作 Agent 查资料、用户问外部信息都需要搜索 + 抓取闭环。

不接第三方聚合 API，也不走 MCP——自托管的 linkseek（SearXNG 聚合 + HTTP 抓取 + browserless 渲染）以**原生内置工具**形态集成：默认零配置走托管服务的匿名免费绿灯，高级用户可自部署或换别家。

## 决策

### 1. 搜索后端：自托管 linkseek 为默认，四档可配

| 档位 | 鉴权 | 抓取能力 |
| :--- | :--- | :--- |
| linkseek 托管（默认） | 匿名绿灯（Origin 白名单 + clientId） | 有（含自动渲染升级） |
| linkseek 自部署 | API Key（Bearer） | 有 |
| Exa | API Key | 无（浏览器直连被 CORS 挡死，抓取必须经服务端） |
| Perplexity | API Key | 无 |

工具契约照 dsh web capability seam 裁剪：**换后端不动模型契约**。provider 只实现 `{ search, fetch? }`，工具 schema 恒定。

### 2. linkseek 绿灯的边界与防线

- **范围**：只开放搜索与抓取。AI 工具（answer 系列）不进绿灯——烧宿主 LLM token 是成本红线；render 不独立暴露，只作为抓取的质量门控自动升级路径。
- **身份**：匿名 clientId（localStorage UUID）+ IP 双闸，不做浏览器指纹（概率性、可伪造、复杂度不配防刷强度）。
- **配额**：每身份 50 加权次/天（渲染计 2）、IP 总闸 200/天、突发 10 次/分钟。Origin 白名单只挡误用，配额才是防线（Origin 可被非浏览器客户端伪造）。
- **本地不算绿灯**：只认部署域名 Origin；本地开发用自部署档。

### 3. 工具契约（模型可见面保持最小）

- `WebSearch`：唯一必填 `queries`（1-4 条），多 query 并发 + URL 去重合并，cap 8 条；条数上限不暴露给模型。
- `WebFetch`：唯一必填 `url`，确定性检索不做 LLM 摘要；服务端 `render:auto` 对模型透明（输出带 `renderedBy`），超时 200s。
- 结果前缀不可信内容提示；要求模型以 markdown 链接引用来源。

### 4. 部署红线

browserless 容器网络必须隔离（不通内网、不可达云元数据端点）——render 路径 SSRF 防护弱于普通抓取（DNS 由容器解析），匿名绿灯开放后容器隔离是唯一兜底。上线前确认。

## 与 0002 的关系

0002 的最小工具集决策是其阶段的正确产物；本决策在产品进入下一阶段时**收回**其中 Web Search / Web Fetch 一条，其余不照搬条目（多 agent、MCP 生态、LSP、Task 系统等）维持不变。参考实现取自 DeepSeek Harness 的 `packages/web/`（tool-web 的工具契约与 search/fetch 裁剪、provider seam 形状），细节取舍见计划文档「已拍板决策」。
