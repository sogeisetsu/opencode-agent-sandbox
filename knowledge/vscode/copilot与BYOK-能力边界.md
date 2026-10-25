# Copilot / BYOK：能换什么、不能换什么

> 来源：本机 VS Code 1.143 实测记录。标 **待确认** 的是推断，未复核。

## 市场里的旧条目别再装

Copilot 已合并为**单个内置扩展**：包名 `copilot-chat`，显示名 GitHub Copilot。市场里 `GitHub.copilot` 是 2025 年就停更的旧条目。

## BYOK 覆盖范围（关键）

覆盖：chat、agent、utility（还是 utility 那档的模型选择）。
**不覆盖**：行内补全，以及语义搜索 —— 仍然走 Copilot 官方服务。

这里的 **embedding** 不是单独一项功能，而是语义搜索背后的机制：Copilot 把工作区里的文件切块、转成向量建索引（官方文档叫 workspace index / semantic index），`#codebase` 靠它按"意思"而不是关键词找代码。官方文档和 microsoft/vscode#330904 都说得很明确：inline suggestions 和 semantic search 仍需要 GitHub 账号 / Copilot 服务支持，BYOK 接的对话模型换不掉它们。

推论：
- "BYOK 接了自己模型但补全没变"是正常现象。
- "BYOK 后 `#codebase` 还是按 Copilot 的方式检索"同理 —— 想纯靠本地关键词搜，退回 grep / 文件搜索那条路。
- `chat.disableAIFeatures: true` 一关，这套索引也就没意义了。

## 语义索引由谁建（已查证）

官方文档《How Copilot understands your workspace》给的答案是**按工作区来源分三种**，不是简单的"客户端还是服务端"：

| 工作区来源 | 索引怎么来 | 首次可用 |
|---|---|---|
| GitHub 仓库（github.com / GitHub Enterprise Cloud） | GitHub 侧建，每个仓库建一次、复用 | 通常立刻可用 |
| Azure DevOps 仓库 | 自动建并维护，需在 VS Code 里登录 Microsoft 账号 | 看状态栏 |
| 其他（含本地文件夹） | Copilot 代建，建完在后台增量更新 | 首次要几分钟 |

补充：
- 索引可能**部分存在本机、部分来自远端**，官方明确"不需要你自己管这个区分"。
- 非 GitHub 工作区：个人账号默认开，组织/企业账号默认关，要靠策略打开。
- GitHub Enterprise Server **不支持**远程索引。
- 手动重建：命令面板 → `Build Codebase semantic index`（该命令随内置 `copilot-chat` 扩展发布，本机扩展里能搜到对应字符串）。
- 索引还没建好时不阻塞使用，agent 退回 text search / grep / 文件搜索。

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
