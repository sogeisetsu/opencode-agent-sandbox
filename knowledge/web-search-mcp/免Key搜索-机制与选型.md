# 免 Key 搜索服务：机制、五家对比、改造清单

> 核对：2026-11-08。三家免 Key 端点**本机实测**（HTTP 状态码 + 返回体摘录见「实测」节）；其余结论来自各项目 README 与 npm/GitHub 元数据，未实跑。标 **推测** 的未验证。领域迭代快（有项目 2026-11 刚做破坏性改版），结论有效窗口约 4–8 周。

## 一句话

「免 Key」不是免费额度，而是**特定端点 + 特定请求头**的组合。没有开箱即用的开源轮换项目覆盖全部目标 provider；可行路径是 fork `search-rotation-mcp`（已覆盖 4/5）再补一个后端，改造量约 150 行。

## 机制：免 Key 是请求头，不是计费策略

| Provider | 免 Key 端点 | 免 Key 信号 | 实测结果 |
|---|---|---|---|
| Tavily | `POST https://api.tavily.com/search` | `X-Tavily-Access-Mode: keyless` | **200 + 真实结果**，不需要任何 key |
| TinyFish | `POST https://agent.tinyfish.ai/mcp` | `X-TinyFish-Access-Mode: keyless` | **200，仅返回 1 个工具**；不带该头 → **401** `Unauthorized: Valid OAuth Bearer token required` |
| Exa | `POST https://mcp.exa.ai/mcp` | 无需 key 参数 | **200**，`tools/list` 返回 `web_search_exa` |

两条直接推论：

1. **TinyFish 的 REST API 必须带 `X-API-Key`，免 Key 只能走 MCP 端点**（**推测**：REST 侧无 keyless 路径，据官方文档口径）。这是本次选型的核心分水岭——任何现成的 REST adapter 移植过来都拿不到免 Key。
2. **keyless 与带 Key 的工具集不同**：TinyFish 免 Key 时 `tools/list` 只给一个 `search`（描述写着「Authenticate with TinyFish to unlock higher limits and the full MCP tool set」）。所以 adapter 不能只写一套 schema，要按 `keyless / authed` 分两档暴露。

## 实测（可复现）

```bash
# Tavily：keyless 头，直接拿结果
curl -s -o /tmp/t.json -w "HTTP %{http_code}\n" --max-time 20 \
  -X POST https://api.tavily.com/search \
  -H "Content-Type: application/json" -H "X-Tavily-Access-Mode: keyless" \
  -d '{"query":"npm registry","max_results":1}'
# 本机：HTTP 200，results[0] 是真实页面

# Exa：MCP 端点，无 key
curl -s --max-time 20 -X POST https://mcp.exa.ai/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
# 本机：HTTP 200（SSE 帧），tools[0].name = web_search_exa

# TinyFish：带头与不带头各跑一次
body='{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
curl -s -o /tmp/a.json -w "with-header %{http_code}\n" --max-time 25 -X POST https://agent.tinyfish.ai/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -H "X-TinyFish-Access-Mode: keyless" -d "$body"
curl -s -o /tmp/b.json -w "no-header   %{http_code}\n" --max-time 25 -X POST https://agent.tinyfish.ai/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -d "$body"
# 本机：with-header 200（tools 只有 search）；no-header 401 Valid OAuth Bearer token required
```

## 候选对比（2026-11-07 元数据）

| 项目 | 5 家覆盖 | 免 Key | 回退 | 冷却 | 传输 | fetch |
|---|---|---|---|---|---|---|
| `search-rotation-mcp`（MIT，TS，Node ≥ 20.3） | 4/5，缺 TinyFish | 支持 | 配额感知 | 有 | stdio + 带鉴权的 Streamable HTTP | 有 |
| `search2ai`（MIT，成熟度最高） | 4/5 | **不支持** | 链式含 429 | 无 | stdio + HTTP | 有 |
| `anysearch` | **5/5** | 仅 DDG | 顺序 | 无 | 仅 stdio | 无 |
| `web-researcher-mcp` | 2/5 | 不支持 | 支持 | 未核实 | stdio + HTTP | 有 |
| `mcp-omnisearch` | 3/5 | 不支持 | 无 | 无 | stdio | 已移除 |
| `dsh-websearch-keyless` | 3/5 | 完全对标 | 支持 | 未核实 | **不是 MCP Server** | 无 |

淘汰要点：`dsh-websearch-keyless` 实现方式正确但形态是 DeepSeek Harness 插件，不能被 Claude Code / Cursor 当标准 MCP 接入；`anysearch` 唯一 5/5，但无 fetch、无冷却、仅 stdio，且它的 TinyFish 走带 Key 的 REST；`search2ai` 成熟但无免 Key 路径。

## 推荐方案与改造清单

基线选 `search-rotation-mcp`：Provider 已含 Tavily · Firecrawl · Parallel · Exa · You.com · Google PSE · Jina Reader · DuckDuckGo HTML；工具为 `search_web` · `fetch_url` · `get_engine_status` · `update_engine_config` · `open_dashboard`；传输 stdio（`npx -y search-rotation`）与带鉴权 Streamable HTTP。

| 优先级 | 改造项 | 工作量 |
|---|---|---|
| P0 | TinyFish 后端（免 Key 走 MCP 端点，`tools/call` 用 `search`） | 约 120 行 + 单测 |
| P0 | 工具别名 `web_search` → `search_web` | 约 10 行 |
| P1 | 可选 `web_fetch` → `fetch_url` 别名 | 约 10 行 |
| P1 | 中文配置文档 | 1 页 |
| P2 | TinyFish 带 Key 的 REST 回退路径 | 约 40 行 |

验收标准（缺一不算完成）：

- 只配 Exa Key → `search_web` 可用，`get_engine_status` 把 Tavily / TinyFish 显示为 keyless 可用；
- 清空所有 Key → TinyFish 免 Key 兜底仍返回结果；
- 连续触发 429 → 该引擎进冷却、自动切下一个、状态可见；
- 5 家全不可用 → 明确报错并列出各家失败原因，**不得静默返回空**；
- Claude Code / Cursor / OpenCode 各跑一次 `tools/list` + `tools/call`。

## 风险

- **主要**：基线仓库单人维护、v0.x、star 极少。缓解＝fork 自维护，代码量小且有 CI 与单测，5 个工具契约稳定。
- **次级**：上游 README 提到 `web_search` / `engine_status` 已改名 `search_web` / `get_engine_status`，客户端若配了工具名白名单需同步改。
- 版本号不一致：README 标 v0.6.0，npm 页面 0.4.10 → 以 `npm view search-rotation version` 为准。

## 未核实项

- TinyFish MCP 端点 `tools/call` 的入参字段（本次只验到 `tools/list` 与工具名 `search`）；
- 上游对标产品 V2 的 provider 加载逻辑（源码级）；
- `search-rotation` npm 实际已发布版本号。

相关：DSH 侧的搜索插件选型（另一种绕过原生付费搜索的路径）见 [DSH 插件生态](../deepseek-harness/插件生态-原生短板与包名核实.md)。
