# Copilot / BYOK：能换什么、不能换什么

> 来源：本机 VS Code 1.143 实测记录。标 **待确认** 的是推断，未复核。

## 市场里的旧条目别再装

Copilot 已合并为**单个内置扩展**：包名 `copilot-chat`，显示名 GitHub Copilot。市场里 `GitHub.copilot` 是 2025 年就停更的旧条目。

## BYOK 覆盖范围（关键）

覆盖：chat、agent、utility。
**不覆盖**：补全、语义搜索、embedding —— 这些仍然走 Copilot 官方服务。

所以"BYOK 接了自己模型但补全没变"是正常现象。

## 开关的组合

| 目标 | 设置 |
|---|---|
| 整个内置 AI 功能（聊天 + 内联建议） | `chat.disableAIFeatures: true` |
| 只关补全 | `github.copilot.enable: { "*": false }` |
| 只关 NES | `github.copilot.nextEditSuggestions.enabled: false` |

`chat.disableAIFeatures: true` 会让 BYOK 一起不可用；第三方扩展（如 OpenChamber）不受影响。

## harness 是运行时，不是模型

可选 harness：Copilot（Agent Host，关窗仍跑）、Local（扩展宿主，关窗即停）、Cloud、Claude、Codex。切 harness 会换掉 agent 列表、工具集和会话生命周期 —— 模型没变但行为会变，原因就在这。

## 模型默认值分几处写

- `chat.defaultModel`、`chat.planAgent.defaultModel`、`chat.exploreAgent.defaultModel` → 用**限定名**（vendor/modelId）
- `chat.utilityModel`、`chat.utilitySmallModel` → 用 `vendor/modelId` 形式
- 嫌分开配麻烦：`chat.byokUtilityModelDefault: "mainAgent"`，utility 直接复用主 agent 的模型

## 指令文件

- 用户级全量：`~/.copilot/copilot-instructions.md`
- 用户级按范围匹配：`~/.copilot/instructions/*.instructions.md`（按 `applyTo`）
- 项目级：`AGENTS.md` 或 `.github/copilot-instructions.md`
- **不影响行内补全**，只管 chat / agent。
- 提交信息语言由 `github.copilot.chat.commitMessageGeneration.instructions` 控制，与 utility 模型无关。
