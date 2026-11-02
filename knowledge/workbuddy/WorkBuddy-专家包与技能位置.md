# WorkBuddy：专家包结构、技能落盘位置

> 核对：2026-11-03，本机 WorkBuddy 5.9.3。路径用 `~` / `<path>` 代替，不记录作者信息。

## 1. 专家包就是一个 Agent 定义包（已验证）

本机专家包在：

```
~/.workbuddy/plugins/marketplaces/experts/plugins/<expert>/
├─ .codebuddy-plugin/plugin.json     # 元数据
├─ agents/<expert>.md                # Agent 正文（frontmatter + 系统提示）
├─ skills/<skill-name>/SKILL.md      # 可选，随包技能
├─ avatars/                          # 头像
└─ README.md
```

`plugin.json` 的关键字段：

| 字段 | 作用 |
|---|---|
| `agents[]` | 指向 Agent MD 文件 |
| `expertType` | 本机见到的都是 `agent` |
| `skills[]` | 随包技能目录，**可选** |
| `displayName` / `profession` / `avatar` / `categoryId` | 市场展示 |
| `defaultInitPrompt` / `quickPrompts` / `tags` | 首次对话与推荐 |

**已验证**：`senior-developer` 的 `plugin.json` 里有 `skills: ["./skills/browser-use", "./skills/frontend-dev", "./skills/fullstack-dev"]`，对应目录里也确有这三个 `SKILL.md`。

## 2. 技能声明有两个位置，别只看一处

原说法是「Agent MD 的 frontmatter 声明预加载技能」。本机实测是**两种都存在**：

| 位置 | 例子 | 说明 |
|---|---|---|
| `plugin.json` 的 `skills[]` | `senior-developer` | 包级声明 |
| Agent MD frontmatter 的 `skills:` | 某数据分析专家 | 角色级声明，列出 5 个技能名 |

（**已验证**：逐个读取 9 个专家包的 `plugin.json` 与 `agents/*.md` frontmatter，两种写法都出现。）

所以迁移/复制专家时，两块都要读：

1. `plugin.json`：`agents[]`、`skills[]`；
2. `agents/<name>.md`：frontmatter 的 `skills:`、`maxTurns`、`name`、`description`，以及正文里的领域知识。

不带技能的专家（本机 `file-organizer-zh`）没有 `skills/` 目录，领域知识直接写在 Agent 正文里。这类包复制时不会漏依赖，但也意味着它的能力不能按技能单独替换。

## 3. 技能落盘位置决定可调用范围

本机与打包代码里能确认的搜索位置：

| 范围 | 位置 | 生效范围 |
|---|---|---|
| 用户级（宿主） | `~/.workbuddy/skills/<name>/SKILL.md` | 跨项目，仅本机 |
| 用户级（CLI） | `~/.codebuddy/skills/<name>/SKILL.md` | 跨项目，仅本机 |
| 工作区级 | `<workspace>/.workbuddy/skills/` | 只在该项目被发现，可随项目分发 |
| 工作区级（兼容） | `<workspace>/.codebuddy/skills/`、`<workspace>/.workbuddy-ai/skills/` | 同上 |
| 内置 | `<install>/resources/app.asar` 内 + `~/.workbuddy/plugins/cache/workbuddy-builtin/` | 随应用，不能卸载 |

**已验证**：`app.asar` 里能 grep 到这组隐藏路径（`.workbuddy/skills/`、`.workbuddy-ai/skills/`、`.codebuddy/skills/`）；内置技能卸载时会被拒绝，提示 `Built-in skills are shipped with the app and cannot be uninstalled`。

实践建议（原说法成立）：需要被别的 Agent / 别人复用的能力放**工作区级**并随项目打包；纯自用放**用户级**。别在两个位置长期各放一份——改了一处忘了另一处，排查时会出现「明明改了却不生效」。

## 4. 纠正：「扫描路径是写死的固定几段」

不完全对。打包代码里技能列表是通过 `skillScanDirs` 参数传给后端 `get-skill-list` 的，而这个值来自产品配置（`_fetchProductConfiguration().skillScanDirs`），不是纯常量数组。

- 默认集合确实是上面那几段（**已验证**：默认路径出现在隐藏目录列表和安装包字符串里）。
- 但它是**可被产品配置覆盖的**，所以「写死」只能当经验说法，不能当契约（**推测**：云端配置具体下发什么没抓到）。

## 验证记录

```powershell
Get-ChildItem ~/.workbuddy/plugins/marketplaces/experts/plugins/<expert> -Recurse -Depth 2
Get-Content ~/.workbuddy/plugins/marketplaces/experts/plugins/<expert>/.codebuddy-plugin/plugin.json
Get-Content ~/.workbuddy/plugins/marketplaces/experts/plugins/<expert>/agents/<expert>.md -TotalCount 30
rg -a "skillScanDirs|get-skill-list|\.workbuddy/skills/" "<install>/resources/app.asar"
```

未验证：工作区级技能在 Teams / 云端项目里的同步行为；不同产品配置下 `skillScanDirs` 的实际值。
