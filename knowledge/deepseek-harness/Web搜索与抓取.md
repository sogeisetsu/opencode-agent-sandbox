# DSH Web 能力：搜索要花钱，抓取是免费的

> 核对：2026-11-12。依据：2026-11-09～11-11 的会话实测（`searchReady` 行为、两条搜索接口、`web_fetch` 边界）与源码/配置阅读；本轮未重跑计费请求。DSH 为 nightly 预发布，provider 与计费口径会漂移；有效窗口约一个季度。

## 一句话

DSH 的 `ctx.web` 下挂两个 provider：**`web_search` 走 DeepSeek 付费接口**（默认按模型 token 计费，也可切按条计费），**`web_fetch` 是免费的本机直连 GET**。同一个账号「有的会话能搜、有的不能」，取决于**当前模型路由与凭据**，不是界面故障。

## 架构：统一的 `ctx.web` + 两个 provider

| 能力 | provider | 是否要钱 | 形态 |
|---|---|---|---|
| 搜索 `web_search` | `@deepseek-ai/dsh-web-search-deepseek`（内部 id `deepseek-official`） | **要**（DeepSeek 付费 API） | 服务端检索 + 模型输出 |
| 抓取 `web_fetch` | `dsh-web-fetch-http` | **不要** | 本机直连 HTTP GET；只允许公网 http(s)、拒绝重定向、逐个校验并固定连接 |

## `web_search` 什么时候才会出现：`searchReady` 两道门槛

`dsh-tool-web` 只有在 `ctx.web.searchReady(当前模型路由)` 为真时，才把 `web_search` 暴露给模型。它只认两种凭据（**源码级 + 会话实测**）：

1. 当前对话模型路由是 `deepseek-account`（官方账号登录）→ 用账号令牌；
2. 否则必须有 `DEEPSEEK_API_KEY` 凭据。

本机实况（2026-11-11）：默认模型走第三方路由（不在官方账号路由），凭据库只有第三方 key、**没有 `DEEPSEEK_API_KEY`** → `searchReady` 为 false → 工具被隐藏。这解释了「某个会话能搜、另一个会话不能搜」——**取决于当时用的模型路由与凭据**。

## 两条付费路径的实际差异

| | 默认（Messages 搜索） | 「启用新版 Web 搜索 API」后 |
|---|---|---|
| 请求 | `POST https://api.deepseek.com/anthropic/v1/messages` | `https://api.deepseek.com/v1/tools/search`（抓取换 `/tools/fetch`） |
| 形态 | 一次完整模型请求，带原生 `web_search_20250305` 工具，模型 `deepseek-v4-flash`，每次最多搜 5 次、输出上限 4096 token | 一次直接请求，`count` 1–50 条，默认 10 条 |
| 计费 | 模型 token（含输出）+ 服务端检索，都算钱 | 按搜索/抓取条数计费 |
| 认证 | 账号令牌 `x-dsh-auth-token`，或 API key（`x-api-key` + `Bearer`） | 同一个 `DEEPSEEK_API_KEY` / 账号令牌 |

一句话：**搜索不是抓 Google/Bing 的免费结果页，而是实打实调 DeepSeek 付费接口**；默认那条路每次搜索相当于额外跑一轮模型。抓取则完全免费。

## 想恢复搜索 / 想省钱

位置：侧栏 **插件 → 官方分组 → 网页搜索**（同页可调 API 地址与上限）。

- **办法 A（最省事）**：把对话模型切到 `deepseek-account` 路由下的 DeepSeek 官方模型 → 用账号登录鉴权，不用另填 key（仍消耗账号额度）。
- **办法 B**：在「网页搜索」页填 `DEEPSEEK_API_KEY` 并保存（key 走凭据库，不进设置文件）→ 任何模型都能搜。
- **省钱**：同页打开「启用新版 Web 搜索 API」→ 切到按条计费。
- **想彻底绕开费用**：`ctx.web` 支持 `registerSearchProvider` 注册任意搜索来源（SearXNG 自建 / Tavily / Brave 免费额度），但社区插件目录里基本没有现成的搜索 provider 插件（多是抓取类，如网页转 Markdown 的插件），等于要自己写一个本地插件。

三条落地路径的取舍：

| 路径 | 代价 | 适合 |
|---|---|---|
| A. 自写本地搜索 provider | 免 API 费，但要开发维护 | 有自建 SearXNG 或长期高频搜索 |
| B. 填 `DEEPSEEK_API_KEY` 恢复官方搜索 | 仍花钱，最省事 | 偶发搜索、要稳定质量 |
| C. 只用 `web_fetch`，用户给链接 | 零成本，但没有关键词全网检索 | 已知来源的读取 |

## `web_fetch` 的真实边界（省得重踩）

| 入口 | 直接 HTTP 抓取的结果 |
|---|---|
| 小红书 | 404 占位页（`error_code 300031`） |
| 知乎专栏 | 403 |
| 百度贴吧 | 「百度安全验证」 |
| 微信公众号 | 「环境异常，完成验证后继续访问」 |
| 夸克网盘分享页 | JS 渲染空壳 + 还要提取码 |
| 百度文库/豆丁/道客巴巴 | 需登录或付费 |

**能拿到标题与摘要，但拿不到正文**——足以证明资源存在（**已验证**：会话实测）。`web_fetch` 还有些通用坑：

- `fetch failed`：直接失败，别重试同一 URL，换源；
- 跨域重定向：`cross-origin redirect to ... is not followed automatically`，直接对目标 URL 再发一次；
- `unsupported content type "unknown"`：放弃该页。

## 未确认

- 两种搜索计费的单价与量级未核算；「社区没有现成搜索 provider 插件」只来自一次 catalog 扫描，可能已变化。
- 官方账号路由的额度与限速口径以服务端为准，本文不展开。

相关：[模型与 Provider](模型与provider配置.md) · [山东省考行政执法专业科目](../exam-prep/山东省考行政执法专业科目.md)
