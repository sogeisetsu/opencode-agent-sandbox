# provider 的三层来源：内置 catalog / 你的配置 / 覆盖结果

## 一句话

`opencode.jsonc` 里的 `provider` 块**不是定义，是覆盖层**——provider 的"定义"来自 opencode 内置的 catalog（从 `models.opencode.ai` 下载）；你写同名的块，只是在 catalog 之上改字段。

> 草图：[`provider来源三层.excalidraw`](./provider来源三层.excalidraw)（OpenChamber 的 Files 视图点开就是画布）

## 三层模型

```
opencode 内置 catalog  →  你的配置文件  →  合并后实际生效
   （定义来源）            （覆盖层）         （覆盖赢）
```

理解这个仓库里所有 provider 相关问题（凭据归谁、删配置安全吗、模型列表为什么是这个数），钥匙都是这一层。

## catalog 是什么（实测）

- 从 `https://models.opencode.ai/api.json` 下载（实测实物 ~5.8 MB、232 个 provider）。
- catalog 里的条目会**自动变成可用的 integration**：对照另一条 API，运行实例 `/api/integration` 返回 235 个。
- 二进制里能搜到对应实现与报错文案：`Failed to cache models.dev catalog`、`Failed to fetch models.dev`，引用 `@opencode/ModelsDev`。
- catalog 里 `opencode-go` 的完整条目（实测读出）：

  | 字段 | 值 |
  |---|---|
  | `id` | `opencode-go` |
  | `env` | `["OPENCODE_API_KEY"]` |
  | `npm` | `@ai-sdk/openai-compatible` |
  | `api` | `https://opencode.ai/inference/go/openai/v1` |
  | `name` | `OpenCode Go` |
  | 模型数 | 32（**时效**，随套餐变动） |

- `opencode` 条目同形：`api=https://opencode.ai/inference/openai/v1`、同一个 `env`、127 个模型。

## 关键对照实验：catalog 条目会自动生效

`opencode` 这个 provider **你从没在配置里写过**，它的 baseURL 完全来自 catalog —— 而你在正常使用它。

反推结论：**catalog 里的条目不需要任何配置就生效**。配置块的作用只剩"覆盖"。

## 已验证 / 推测

- **已验证（他次会话实测，本文件未复验）**：catalog 的下载地址与规模、`/api/integration` 条目数、二进制报错文案与 `@opencode/ModelsDev` 引用、`opencode-go` 与 `opencode` 的 catalog 条目字段、对照实验。
- **推测**：无（模型数量属**时效**信息，别记死）。
- 时效提醒：catalog 内容、模型数量、报错文案都会随版本变化，引用前重新抓一次 `api.json` / `/api/integration`。
