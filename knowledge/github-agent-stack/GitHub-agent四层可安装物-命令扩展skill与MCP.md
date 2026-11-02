# GitHub agent 的四层「可安装物」

> 核对：2026-11-02，本机 `gh` 2.100.0 + 官方仓库源码/文档。标 **推测** 的没实跑。

## 先分清四层

同一个「给 agent 加 GitHub 能力」的说法，实际落在四个完全不同的层。装错层的典型症状是：想改模型行为却去装扩展，想加接口却去写 `SKILL.md`。

| 层 | 本质 | 装不装 | 影响谁 | 入口 |
|---|---|---|---|---|
| 内置子命令 | `gh` 自带命令（`gh pr`、`gh skill`） | 不装，跟版本走 | CLI 行为 | `gh --version` |
| 扩展 | 给 `gh` 加**子命令** | 装 | CLI 行为 | `gh extension install` |
| skill | `SKILL.md`，给 agent 的操作说明书 | 装 | 模型行为 | `gh skill install` |
| MCP server | 给模型调用的**工具接口** | 装 + host 配置 | 模型能调用什么 | host 的 MCP 配置 |

**已验证**的判断方法：看入口是 `gh <name>` 还是模型侧工具列表。前者是 CLI 层，后者是 MCP 层；skill 夹在中间，只改模型「怎么做」，不新增可调用工具。

## 内置子命令：`gh skill` 的版本门槛

- `gh skill` 是 **内置命令**，不是扩展。v2.90.0 的 release notes 标题就是 `Manage agent skills with gh skill (Public Preview)`；本机 2.100.0 直接可用。
- 低于 2.90.0 会报 `unknown command`。脚本里用它之前先 `gh --version` 卡一下。
- 仍是 preview：`gh skill --help` 原文写着 `in preview and subject to change without notice`，别把输出格式当稳定契约。
- 子命令（本机实测）：`install` / `list` / `preview` / `publish` / `search` / `update`，别名 `gh skills`。

```bash
gh --version                     # 期望 >= 2.90.0
gh skill --help                  # 第一行会写 preview
```

## 扩展：命名规律与安全边界

`gh extension --help` 的原文同时给了两条关键信息：

1. 仓库名必须以 `gh-` 开头，且含同名可执行文件；`gh <extname>` 会把参数透传给 `gh-<extname>`。所以 `gh-stack` → `gh stack`。
2. `Extensions are not verified, signed, or endorsed by GitHub.` —— 装扩展等于信任发布者。

扩展不能覆盖核心命令；名字冲突时用 `gh extension exec <name>`。

## skill：装到哪个 host、能不能锁版本

`gh skill install --help` 实测到的行为：

- 支持 `--agent`（Copilot、Claude Code、Cursor、Codex、Gemini CLI、OpenCode 等）和 `--scope project|user`，默认 `project`。
- 项目级下多个 host 共用 `.agents/skills`（包括 OpenCode、Codex、Cursor）。这跟 OpenCode 官方文档的搜索路径一致（见 `agent-config-layers/`）。
- 锁版本：skill 名后加 `@VERSION` 或用 `--pin`，版本按 **git tag 或 commit SHA** 解析。
- 安装时会把来源元数据写进 skill 文件，文件被搬走也还能追溯来源（`install --help` 原文：`source tracking metadata injected`）。

```bash
gh skill search <keyword>                              # 只读
gh skill preview <owner>/<repo> <skill>                # 只读，不落盘
gh skill install <owner>/<repo> <skill> --pin <tag>    # 写入本地
```

**已验证**：`preview` 只 fetch 并渲染 `SKILL.md`，明确写 `without installing anything`。

## MCP：官方 server 的事实基线

- 仓库 `github/github-mcp-server`，Go，MIT，GitHub 官方维护（`gh api repos/github/github-mcp-server` 返回 `license.spdx_id=MIT`、`language=Go`）。
- 两种形态：托管远程 `https://api.githubcopilot.com/mcp/` 与本地容器/二进制；远程走 HTTP，本地可 stdio。
- **GHES 不支持远程托管**，README 原文：`GitHub Enterprise Server does not support remote server hosting.` 只能自托管本地 server。
- 默认工具集（README 原文）：`context`、`issues`、`pull_requests`、`repos`、`users`。不传配置就用这五个。

配置是**按维度叠加**的，不是一层覆盖另一层：

| 维度 | 远程 | 本地 |
|---|---|---|
| 工具集 | `X-MCP-Toolsets` 头或 `/x/{toolset}` 路径 | `--toolsets` 或 `GITHUB_TOOLSETS` |
| 单个工具 | `X-MCP-Tools` 头 | `--tools` 或 `GITHUB_TOOLS` |
| 排除工具 | `X-MCP-Exclude-Tools` 头 | `--exclude-tools` |
| 只读 | `X-MCP-Readonly: true` 或 `/readonly` 路径 | `--read-only` 或 `GITHUB_READ_ONLY` |
| Lockdown | `X-MCP-Lockdown` 头 | `--lockdown-mode` 或 `GITHUB_LOCKDOWN_MODE` |

只读优先级最高：即使显式用 `--tools` 点了写工具，`--read-only` 也会跳过它（README 原文 `Read-only mode takes priority`）。

## 官方渠道 ≠ 背书

- 扩展：明确写着不验证、不签名、不背书。
- skill：`gh skill` 只负责分发，不审内容；带脚本的 skill 以**调用者权限**运行。
- MCP：托管位置不改变 token 的实际权限（详见 `CLI与MCP-谁决策与权限边界.md`）。
- 共同动作：先 `preview`/读源码，再 `--pin`，优先选开了不可变发布的仓库。

## 验证记录

```bash
gh --version                          # 2.100.0
gh skill --help                       # 6 个子命令 + preview 声明
gh extension --help                   # gh- 命名规则 + 不背书原文
gh api repos/cli/cli/releases/tags/v2.90.0 --jq .body | Select-String skill
gh api repos/github/github-mcp-server --jq '.license.spdx_id,.language'
gh api repos/github/github-mcp-server/contents/README.md   # 默认工具集、GHES 段
gh api repos/github/github-mcp-server/contents/docs/server-configuration.md  # 配置矩阵
gh api repos/actions/runner-images/contents/images/ubuntu/Ubuntu2404-Readme.md  # GitHub CLI 2.103.2 预装
```

Actions 运行器预装 `gh`：Ubuntu 24.04 镜像 README 的 CLI Tools 段列的是 `GitHub CLI 2.103.2`，**已验证**。
