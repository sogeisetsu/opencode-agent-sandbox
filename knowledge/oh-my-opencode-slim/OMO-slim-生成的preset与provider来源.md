# OMO slim 会生成 provider 的 preset，但不定义 provider

## 一句话

`oh-my-opencode-slim` 的 `GENERATED_PRESETS` 会**生成** `openai` / `opencode-go` 两个预设，但插件**不定义 provider** —— 它对 `config.providers` 只读。

## 它生成了什么

- `GENERATED_PRESETS = ['openai', 'opencode-go']` → 配置里的 `presets/opencode-go/*` 是它生成的，内容与内置 `MODEL_MAPPINGS` 逐条一致。

## 但它不碰 provider 定义

- bundle 里 `baseURL`、`zen/go`、`opencode.ai` 出现 **0 次** —— 插件不含任何端点。
- `generateLiteConfig()` 只返回 `$schema` / `preset` / `presets` / `disabled_agents`，**不写 `provider`/`providers`**。
- 对 `config.provider` 只**读**，用途是特性探测：`hasKimi` / `hasAnthropic` / `hasCopilot` / `hasZaiPlan` / `hasChutes`。
- `doctor` 也读 `config.providers`（`hasKimi` / `hasCopilot` 等），**只读不写**。

结论：provider 从哪来跟 omoslim 无关，那是 opencode 内置 catalog 的事（见 `opencode-provider/provider三层来源与内置catalog.md`）。

## 引用面（哪些地方用到这些模型名）

- `oh-my-opencode-slim.json`：30 处（两个 preset 的全部 agent）。
- `settings.json`：`defaultModel` / `lastSelectedModel` / `smallModelOverride` / `walkthroughModelOverride`，以及 `favorites` / `recent` / `recentEfforts`。

## 现状（实测时点）

- 当前 preset 是 `jibei-factory`（**不是** `opencode-go`），其 8 个模型全部在线可用。
- `presets/opencode-go/*` 里那两个解析不到的模型见 `opencode-provider/配置块是覆盖层-删了会怎样.md` 末尾。

## 已验证 / 推测

- **已验证（他次会话实测，本文件未复验）**：`GENERATED_PRESETS` 取值与生成关系、bundle 里 0 处端点串、`generateLiteConfig()` 返回键集、对 `config.provider` 的只读探测用途、doctor 只读、引用面计数、当前 preset。
- **推测**：无。
- 时效提醒：预设名、引用面计数、模型名都会随插件与套餐更新变化。
