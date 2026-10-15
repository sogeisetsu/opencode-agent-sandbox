# OMO slim 的配置项与 schema

配置文件：`~/.config/opencode/oh-my-opencode-slim.json`（`.jsonc` 也认）。权威 schema 是包内的 `oh-my-opencode-slim.schema.json`，运行时校验用 zod（`RawPluginConfigSchema`）。

## 顶层键全集

| 键 | 作用 |
|---|---|
| `preset` | 当前启用的预设名 |
| `setDefaultAgent` | 是否把 orchestrator 设为默认 agent |
| `compactSidebar` | TUI 紧凑侧栏，默认 `true`，`false` 用展开布局 |
| `stripOrchestratorModel` | 为 `true` 时不把 orchestrator 的 model/variant 写进 SDK 配置，改用 `/model` 选的会话模型；预设若显式设置了则保留。默认 `false` |
| `routingBlock` | `false` 时从 v2 orchestrator 提示词里去掉 `<Agents>` 路由块（v1 宿主无效，会告警）。默认 `true` |
| `autoUpdate` | `false` 关掉插件自动更新。默认 `true` |
| `presets` | 命名预设表，每个预设定一套角色→模型 |
| `agents` | 顶层角色覆盖层（对所有预设生效） |
| `disabled_agents` | 要停用的 agent 名。**orchestrator / councillor 受保护、滤掉**；`observer` 默认就在停用列表里 |
| `image_routing` | `auto` / `direct`。省略 = 老行为（仅当 observer 启用时拦截附件）；`auto` 需 observer 开启并先落盘再提示委派；`direct` 原样交给 orchestrator |
| `deepworkGuardMode` | `shadow`（默认，只记不回阻塞）/ `enforce`（还会拦截无回执的完成写入） |
| `disabled_mcps` | 要停用的 MCP server 名 |
| `disabled_tools` | 要停用的工具名 |
| `disabled_skills` | 要停用的 skill 名 |
| `disabled_hooks` | 要停用的 hook（取值见下表），未知值会被剥掉并告警 |
| `disabled_commands` | 要停用的斜杠命令（见下表） |
| `multiplexer` | 多路复用终端配置（tmux/zellij/herdr/kitty/cmux-tui/none…） |
| `interview` | interview 工具配置 |
| `backgroundJobs` | 后台任务配置 |
| `fallback` | 失败回退配置 |
| `council` | Council 多模型共识配置（见 Council 那篇） |
| `companion` | companion 应用配置 |
| `webfetch` | webfetch 工具配置 |
| `acpAgents` | ACP agent 配置 |

## 预设的三种写法（互斥）

`normalizePreset` 会把三种外部语法归一成同一份内部结构，同一个预设里只能用一种：

1. **定义式**：带 `agents` 包装，`{ extends?: ..., agents: { <角色>: {...} } }`。
2. **继承式**：`{ extends: "<父预设名>" }`，深度优先解析，缺父/成环会报 `PresetResolutionError`。
3. **扁平 catchall 式**：直接把 `<角色>: {...}` 平铺在预设对象上（除 `extends` / `marketplace` 激活指令外的键都当角色覆盖）。作者文档给的示例预设就是这种扁平式。

- 预设层叠加：`mergePresetMaps` 按层合并，`inheritModelFrom` 是"显式清空下层 model"的信号，不是普通 merge。
- `master` 键在 council 里被**静默忽略**（历史遗留）；`multiplexer.zellij_pane_mode` 是已知的弃用键，会被忽略并告警。

## 单个角色覆盖的字段（`AgentOverrideConfigSchema`，`.strict()`）

`model`（字符串，或数组形式的兜底链：`string | { id, variant }`）、`inheritModelFrom`、`temperature`(0–2)、`variant`、`skills` / `skills_add` / `skills_remove` / `skills_include_local`、`mcps`、`prompt`、`orchestratorPrompt`、`options`、`displayName`、`color`(`#RRGGBB` 或主题色)、`description`、`permission`。

- **orchestrator 上禁止 `prompt` / `orchestratorPrompt`**（schema `superRefine` 直接拒），要改 orchestrator 提示词得用 md 覆盖文件。
- `skills_add`/`skills_remove` 在解析期折进 `skills`；`skills_remove` 在 `skills_add` 之后应用，**移除优先**。

## `disabled_hooks` 的 12 个合法值

`phase-reminder`、`foreground-fallback`、`deepwork-guard`、`deepwork-goal`、`chat-headers`、`cache-monitor`、`json-error-recovery`、`tool-loop-guard`、`search-path-guard`、`absolute-path-rescue`、`apply-patch`、`council-inject`。

- 语义：被禁用的 hook 干脆不注册（如 `phase-reminder`、`chat-headers`、`council-inject`），或只让实现变惰性（如 `foreground-fallback`）。
- 整份列表都是未知值时视为"未设置"，下层配置的同名列表仍生效。

## `disabled_commands` 的 5 个合法值

`implement`、`interview`、`deepwork`、`reflect`、`loop`。同理：未知值剥掉并告警；`reflect` 在 `disabled_skills` 里出现时，`/reflect` 命令也会一起停。

## 已验证 / 推测

- **已验证**：顶层键全集、`AgentOverrideConfigSchema` 全字段与 `.strict()`、orchestrator 禁止 prompt、预设三种语法与 `master` 忽略、`disabled_hooks`/`disabled_commands` 取值与语义、`image_routing`/`deepworkGuardMode` 描述——均对照 `dist/server/index.js` 与包内 schema 读出。
- **推测**：无。
