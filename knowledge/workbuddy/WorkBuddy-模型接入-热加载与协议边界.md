# WorkBuddy：自定义模型热加载、网关头与协议边界

> 核对：2026-11-03，本机 WorkBuddy Desktop 5.9.3 + CLI 2.176.0。配置文件路径用 `~` 代替；不记录任何 key。

## 1. 自定义模型写 JSON 即热加载（已验证）

两侧各有一份 `models.json`：

- 宿主：`~/.workbuddy/models.json`
- CLI：`~/.codebuddy/models.json`

选择逻辑在打包代码里：`getUserConfigPath()` 按产品名选 `.workbuddy` 或 `.codebuddy`；环境变量 `WORKBUDDY_CONFIG_DIR` / `CODEBUDDY_CONFIG_DIR` 可以整体改目录。**已验证**（`app.asar` 里能 grep 到这段函数）。

热加载也是代码级确认：

- 有 `watchFile()` 监听 `models.json`，变化后 debounce 再同步。
- 日志里能看到 `[CustomModelsProvider] models.json changed, triggering sync`。
- 启动时日志会打印 `[CustomModelsProductProvider] ... userConfigPath=<path>` 和 `Loaded custom models config from user: <path>`。

验证方法（只读）：

```powershell
rg -n "models.json changed|Loaded custom models config" ~/.workbuddy/logs
```

本机命中多条，说明写入后确实会重新加载，不需要重启。**不要**拿日志里的路径直接当配置模板，路径因产品名不同而不同。

模型条目的字段（从本机文件读出，隐藏值）：

```json
{
  "id": "<model-id>",
  "name": "<display-name>",
  "vendor": "Custom",
  "url": "https://<gateway>/v1/chat/completions",
  "apiKey": "<redacted>",
  "supportsToolCall": true,
  "supportsImages": false,
  "supportsReasoning": true,
  "onlyReasoning": true,
  "reasoning": { "defaultEffort": "medium", "supportedEfforts": ["low","medium","high","max"] },
  "maxInputTokens": 200000,
  "useCustomProtocol": false
}
```

重点字段是 `url` 和 `useCustomProtocol`，其余是能力声明。

## 2. 网关会话头：宿主自带的够不够

打包代码里，宿主对内部网关请求会自动加：

- `X-Conversation-ID`
- `X-Conversation-Request-ID`
- `X-Conversation-Message-ID`
- `X-Request-ID`

（**已验证**：`app.asar` 里多处构造 headers 时同时写入这几个字段。）

所以对接一个要求「会话标识头」的网关时，宿主自带的会话头往往能直接满足，不必另造入口。但这是**按网关而定的推测**：不同网关校验的 header 名和值格式不一样，先抓一次请求或看网关日志确认它到底认哪个头。

补充更正：原说法「无需额外的自定义头入口」不绝对。CLI 有 `-H/--header` 选项（`codebuddy --help` 原文：`Custom HTTP headers for model requests`），需要时可以显式加。宿主侧则依赖产品配置，不是所有头都能从 UI 加。

## 3. 协议族限制：改配置绕不过去

自定义模型的 URL 默认会被归一化成 Chat Completions：

```text
url: useCustomProtocol ? 原样 : normalizeChatCompletionsUrl(url)
```

（**已验证**：`app.asar` 里 `normalizeChatCompletionsUrl` 的实现。）

这意味着：

- `useCustomProtocol: false`（默认）时，给一个只支持 `messages` / `responses` 协议的地址，应用仍按 Chat Completions 发请求，接不上。
- 打开 `useCustomProtocol` 只是**跳过 URL 归一化**，不会把请求体翻译成另一种协议。它解决的是「路径被改写」，不是「协议不支持」。

所以「仅提供专属协议的模型无法接入只支持 Chat Completions 的通道」这个判断成立；但「靠改配置绕」具体能绕到什么程度，取决于通道是否真的支持该协议，标 **推测**。

接入前先确认协议族：看网关文档的 endpoint 是 `/chat/completions`、`/messages` 还是 `/responses`，不要先改 JSON 试。

## 验证记录

```powershell
rg -n "models.json|CustomModelsProductProvider" ~/.workbuddy/logs   # 加载与变更日志
rg -a "normalizeChatCompletionsUrl|useCustomProtocol" "<install>/resources/app.asar"
rg -a "X-Conversation-ID|X-Conversation-Request-ID" "<install>/resources/app.asar"
codebuddy --help   # -H/--header 存在
```

未验证：某个具体第三方网关是否接受宿主自带的会话头；`useCustomProtocol` 打开后对非 Chat Completions 协议的实际兼容度。
