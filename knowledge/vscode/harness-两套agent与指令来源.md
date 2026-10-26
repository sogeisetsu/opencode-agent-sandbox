# harness：同一个模型，两套 agent 与两套指令来源

> 来源：VS Code 1.143 官方文档 + 内置 `copilot-chat` 扩展代码查证 + 本机实测。标 **推测** 的未复核。

## harness 是什么

官方定义：运行 agent 会话的软件层——把模型接到上下文和工具上、转 agent loop、维护会话状态。模型负责"想"，harness 负责让"想"变成动作。

关键推论：**换 harness ≠ 换模型。** 工具集、权限档位、agent 列表、会话生命周期都跟着 harness 走。所以"换了更强的模型还是一样笨"有时是换错了层。

## 两个主要 harness 的差别（查证）

| | Copilot harness | Local harness（界面叫"本地"） |
|---|---|---|
| 跑在哪 | Agent Host，独立进程 | 编辑器扩展宿主 |
| 运行时 | Copilot SDK，与 Copilot CLI / Copilot app 共享 | 扩展宿主 |
| 关窗后 | 继续跑 | 停 |
| 会话目标 | 本机 / 连接的主机 / Dev Container / Cloud | 本机 |

Session Target 控件里还有 Cloud / Claude / Codex。**Cloud 不是 harness**，是一类执行目标的归类，选完还要再挑具体云 agent。切 harness 时可以从 Local 会话把对话历史和上下文带过去。

## 各自的 agent（查证：内置 copilot 扩展代码）

查到的内置 agent 定义（原文字符串）：

| agent | 代码里的自我描述 |
|---|---|
| Agent | 默认模式，读/改/跑完整循环 |
| Ask | `Answers questions without making changes` |
| Plan | `Researches and outlines multi-step plans` |
| Explore | `Fast read-only codebase exploration and Q&A subagent. Prefer over manually chaining multiple search and file-reading operations... Specify thoroughness: quick, medium, or thorough.` |

Explore 是子 agent，不占下拉位置，由主 agent 派出去；在 `handoffs` 里通过 `agents:["Explore"]` 挂上。

官方文档补一句关键：**Plan agent 不在 agent 下拉里**，要用 `/plan` 唤出，且只在 Copilot 或 Claude harness 的会话里可用。

## 真正的差别是"命令面"，不是名字（查证）

- `/init`、`/create-instructions` —— 只有**选中 Local** 时可用。
- `/plan` —— 只在 **Copilot 或 Claude** harness 的会话里可用。
- 子 agent 支持 —— 不是每个 harness 都有。

所以准确表述：**每个 harness 决定自己的斜杠命令、工具集、权限档和读哪些指令文件。** 名字看着一样，能做的事不一样。

## 在哪儿切、在哪儿配

- 切换：Chat 输入框旁的 **Session Target** 控件。
- 设置（已在本机 settings.json / 源码核对默认值）：
  - `chat.defaultToCopilotHarness`（默认 `false`）——新会话默认走 Copilot harness
  - `chat.editor.preferCopilotHarness`（默认 `false`）——编辑器内聊天优先 Copilot harness
  - `chat.editor.localAgent.enabled`（默认 `true`）——本地 agent 开关
- 企业策略可隐藏 Local harness。源码日志原文：`Local harness hidden, new chats use the Agent Host Copilot SDK`。公司机器上找不到"本地"选项，是策略不是 bug。

## 指令文件：位置按 harness 分（官方文档）

| harness | 项目级 | 定向 | 用户级 |
|---|---|---|---|
| Copilot | `.github/copilot-instructions.md` | `.github/instructions/**/*.instructions.md` | `~/.copilot/copilot-instructions.md`、`~/.copilot/instructions/*.instructions.md` |
| Claude | `CLAUDE.md`、`.claude/CLAUDE.md` | `.claude/rules` | `~/.claude/CLAUDE.md`、`~/.claude/rules` |
| Local | `AGENTS.md`（开关控制） | `.github/instructions` 或 `.claude/rules` | VS Code profile 存储 |

- `AGENTS.md` 是跨 agent 格式，Copilot / Codex / Local 都支持。
- Local 专属开关：`chat.useAgentsMdFile`（默认 **true**）、`chat.useNestedAgentsMdFiles`（默认 **false**，即只读仓库根目录那份）。
- `applyTo`（Claude rules 用 `paths`）是 frontmatter 里的 glob，agent 创建/修改匹配文件时自动附加。
- **指令之间没有优先级**。官方原话：applicable sources 是叠加的，不要依赖顺序解决冲突，回源文件改。
- **指令不影响行内补全**，只管 chat / agent。
- 用户级指令放在 harness 目录下（`~/.copilot/instructions`、`~/.claude/rules`）**不随 Settings Sync 漫游**。

## 管理界面

命令面板 → `Chat: Open Customizations`（界面上叫"智能体自定义"）。左侧：Overview / 插件 / MCP 服务器 / 技能 / 指令 / 智能体 / 钩子 / 提示。

标题栏里会写当前 harness（形如"智能体自定义（本地 · <机器名>）"）——**面板内容跟着 harness 变**。指令栏分"用户"和"工作区"两个作用域，加号新建，`Chat: New Instructions File` 是等价命令。

诊断：`Developer: Open Agent Debug Logs`，或 Chat 视图省略号菜单里的 Show Agent Debug Logs。
