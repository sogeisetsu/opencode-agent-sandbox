# 端点：`/zen/*` 与 `/inference/*` 是同一个网关

## 一句话

`/zen/*` 和 `/inference/*` 是同一个网关的**别名**（错误响应逐字相同），chat 端点是 `…/go/openai/v1/chat/completions`；但**模型列表在另一个路径**，拼到 chat base 后面会 404。

## 端点清单（实测）

| 用途 | 路径 | 结果 |
|---|---|---|
| 官方文档 / catalog 的 base | `https://opencode.ai/inference/go/openai/v1` | 正常 |
| 同网关别名 | `https://opencode.ai/zen/go/v1` | 正常，错误响应与上面**逐字相同** |
| 模型列表 | `https://opencode.ai/inference/go/v1/models`（**不带 `/openai`**） | `200`，37 个模型 |
| chat | `…/go/openai/v1/chat/completions` | `200` + 真补全 |

## 坑：模型列表不能拼在 chat base 后面

- `https://opencode.ai/inference/go/openai/v1/models` → **404**。
- 原因：那是**可选路由**，opencode 客户端根本不用它 —— 它的模型列表从 catalog 拿（见 `provider三层来源与内置catalog.md`）。
- 教训：别用"往 baseURL 后面拼 `/models`"的通用套路去探 OpenCode 的模型，直接查 catalog 或 `/api/integration`。

## 请求头

- `chat/completions` **要求** `x-opencode-session`。
- `x-opencode-org-id` 可选。
- 鉴权用 `OPENCODE_API_KEY`（占位符，别把真值写进文件）。

只读探法（把 `$TOKEN` 换成环境变量，别内联真 key）：

```bash
# Windows / pwsh 也行；只读，不改任何东西
curl -sS "https://opencode.ai/inference/go/v1/models" -H "Authorization: Bearer $TOKEN"
```

## 两条 base 都做过端到端实测

用真 OAuth token + `deepseek-v4.1-flash`、`max_tokens=4`：

- `https://opencode.ai/inference/go/openai/v1/chat/completions` → `200` + 真补全
- `https://opencode.ai/zen/go/v1/chat/completions` → `200` + 真补全

即 baseURL 用哪一个都是同一个后端，这也解释了为什么配置块把 baseURL 从 `/inference/go/openai/v1` 换成 `/zen/go/v1` 之后一切照常。

## 已验证 / 推测

- **已验证（他次会话实测，本文件未复验）**：两条 base 都是 200 且返回真补全、`/zen` 与 `/inference` 错误响应逐字相同、`/go/v1/models` 返回 37 个、`/go/openai/v1/models` 404、`x-opencode-session` 必需。
- **推测**：无（`x-opencode-org-id` 只确认"可选"，未逐一验证其影响）。
- 时效提醒：路径与模型数量属产品面，会变。
