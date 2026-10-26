# Copilot / BYOK：能换什么、不能换什么

> 来源：本机 VS Code 1.143 实测记录。标 **待确认** 的是推断，未复核。

## 市场里的旧条目别再装

Copilot 已合并为**单个内置扩展**：包名 `copilot-chat`，显示名 GitHub Copilot。市场里 `GitHub.copilot` 是 2025 年就停更的旧条目。

已查证（2026-10，市场 gallery API + 本机内置目录）：

| 条目 | 版本 | 最后更新 | 说明 |
|---|---|---|---|
| `GitHub.copilot` | 1.388.0 | 2025-10-24 | 只有补全的旧条目，已停更 |
| `GitHub.copilot-chat`（市场） | 0.48.1 | 2026-05-15 | 独立条目，正常不用装 |
| 内置 `resources/app/extensions/copilot` | 0.71.0 | 随 VS Code | `name: copilot-chat`，`displayName: GitHub Copilot` |

内置扩展**不能卸载，只能禁用**；`code --list-extensions` 不列内置扩展，所以那里永远看不到 Copilot。

## 工作区级"禁用 AI 功能"写在哪（查证 + 实测）

扩展页下拉里的 **Disable AI Features (Workspace)** / 中文"仅在此工作区禁用"**不写任何 settings 文件**，`git status` 永远干净。它走的是扩展的 workspace enablement。

源码对照（1.143 bundle）：

- 全局版 `extensions.disableAIGlobally` → `configurationService.updateValue("chat.disableAIFeatures", true)`
- 工作区版 `extensions.disableAIInWorkspace` → `extensionsWorkbenchService.setEnablement(extension, 11)` + `updateRunningExtensions(...)`，**没有一次 updateValue**

落盘位置：

```
%APPDATA%\Code\User\workspaceStorage\<hash>\state.vscdb
```

键：`extensionsIdentifiers/disabled`，值形如 `[{"id":"github.copilot-chat","uuid":"..."}]`；启用回来则记在 `extensionsIdentifiers/enabled`。

后果：不跟仓库走、换机器/重新 clone 不带走、无法提交给同事。想让设置跟着仓库走只能手写 `.vscode/settings.json` 的 `chat.disableAIFeatures`。

## 关掉内置 Copilot 的联动（源码已验证）

禁用内置 `copilot-chat` 时，核心会自动把 `chat.disableAIFeatures` 写成 `true`：

```
Disabling AI features because builtin chat extension is disabled
```

反向也成立：`chat.disableAIFeatures === true` 且 chat 初始化未完成时，会尝试禁用内置 chat 扩展。两者是同一开关的两面。

`chat.disableAIFeatures` 的设置描述（nls 原文）：

> Disable and hide built-in AI features provided by GitHub Copilot, including chat and inline suggestions.

扩展视图里对应的右键/齿轮项：**Disable AI Features**、**Disable AI Features (Workspace)**（仅本工作区）。中文界面显示为 **禁用 AI 功能**，旁边有下拉可选工作区版本；扩展页标题写 `GitHub Copilot Chat`、标"内置"。

## 关掉之后界面上少了什么（实测）

- 右下角 Copilot 用量图标消失（原先点开看额度）
- 标题栏 `Try agents window` 按钮 + Copilot 聊天按钮消失
- 小灯泡（`Alt+Insert`）里那组 AI 重构动作消失：`提取到封闭范围中的 constant`、`提取到 inner function`、`提取到 module 范围中的 function`、`Modify`、`Review`；剩下的是语言服务给的（源代码操作、Format with Prettier、整理导入、修复所有可修复的 JS/TS 问题、删除未使用的代码、添加所有缺少的导入）
- 灰字（ghost text）消失，但 `editor.action.triggerSuggest`（`Ctrl+Space`）照常 —— **补全列表来自语言服务，与 Copilot 无关**

重载后 `chat.disableAIFeatures` 不再出现在 settings.json 里（重新启用时被清掉或回到默认）。

## 扩展的实验性补全会和 Copilot 的灰字混淆（已澄清）

`opencodego.inlineSuggestions` **默认 `false`**。打开后（需重载）扩展注册自己的 `InlineCompletionItemProvider`：

- 模型：`opencodego.inlineSuggestionsModel`，默认 `qwen3.6-plus`，强制 `enable_thinking=false`（思考模型即使关思考也烧 100+ 推理 token，不推荐）
- 调参：`inlineSuggestionsDebounceMs`(300) / `TimeoutMs`(3000) / `MaxTokens`(128) / `PrefixLines`(10) / `SuffixChars`(300)
- 另有 `inlineSuggestionsChatInput`（默认 false）：是否也在聊天输入框里出补全

开 / 关的实际差别：

- 关：没订 Copilot 就完全没有灰字；订了就是 Copilot 的灰字
- 开：不管有没有 Copilot 都出灰字，花的是 OpenCode 额度

**不是替换 Copilot 的补全，是加一个 provider。** 多 provider 同时注册时谁先返回显示谁，所以两个都开时分不清屏幕上的灰字是谁给的。判定法：关掉 Copilot 后仍出灰字，才是扩展在供货。

扩展自己的定位（readme 原文）：`This extension provides chat, agent tool calls, and utility-model access; it does not replace standard Copilot autocomplete.`

## 收残留 UI 的键位（已验证存在于 1.143 bundle）

| 目标 | 设置 |
|---|---|
| 标题栏 Copilot 登录按钮 | `chat.titleBar.signIn.enabled: false` |
| 侧边栏不被聊天视图占着 | `workbench.secondarySideBar.defaultVisibility: "hidden"`（等价：聊天视图标题栏 `...` → Show View by Default 取消勾选） |

`chat.commandCenter.enabled` **在 1.143 已不存在**（bundle 里 0 命中），别再照旧教程写它。

## OpenCode 接进 Copilot Chat（实测在用）

扩展：`ltmoerdani.opencode-copilot-chat`（市场名 "OpenCode for Copilot Chat: BYOK 30+ AI Models"，MIT，社区项目，非 OpenCode 官方）。

- 原理：把自己注册成 **Language Models 提供商**，OpenCode Go / Zen 的模型出现在原生模型选择器里；不需要 GitHub 登录或 Copilot 订阅。
- 接入：Copilot Chat 模型选择器 → **Add Models…** → 选 OpenCode Go / Zen → 粘贴 key（VS Code 存）。
- 它**自己注册 `InlineCompletionItemProvider`** 来提供补全（`opencodego.inlineSuggestions`，默认关，实验性）。这不是"换掉 Copilot 的补全"，是另一个补全来源——官方明确 BYOK 不覆盖补全，但开放了该 API 给扩展。
- 会改核心设置：`chat.agentHost.byokModels.enabled`、`extensions.supportAgentsWindow`（由 `opencodego.autoEnableAgentsWindow` 管，关掉会回滚）。
- 免费 Zen 模型自 2026-09-17 起对第三方客户端返回 403，扩展已自动隐藏。

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
