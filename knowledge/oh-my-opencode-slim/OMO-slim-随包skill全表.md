# OMO slim 随包 skill 全表（v3.0.5，11 个）

## 一句话

`CUSTOM_SKILLS` 里共 11 个 skill，全部按 agent 授予。除 `simplify` 只给 oracle，其余 10 个都只给 orchestrator；子 agent（explorer / fixer / designer / observer / council）拿不到任何一个。

## 全表

| skill | 类型 | 谁能加载 | 干什么 |
|---|---|---|---|
| `simplify` | 流程 | **oracle** | 在不改行为的前提下简化代码、提升可读性 |
| `codemap` | 工具 | orchestrator | 给陌生仓库生成分层 codemap（很贵，只在明确要求时用） |
| `clonedeps` | 工具 | orchestrator | 把关键依赖的源码克隆到被忽略的本地目录，直接读内部实现 |
| `deepwork` | 流程 | orchestrator | 大型高风险多阶段编码的编排流程，带依赖和评审闸门 |
| `verification-planning` | 流程 | orchestrator | 动手前先规划一条可信、合比例的验证路径 |
| `ai-estimates` | 行为约束 | orchestrator | 按"人 + AI 协作"估工期，把 AI 快的部分与人工、外部瓶颈分开算 |
| `deanchor` | 行为约束 | orchestrator | 有示例时先提炼背后的原则，产出同级但原创的东西 |
| `step-back` | 行为约束 | orchestrator | 补丁越叠越多时停下重估方向，再决定继续还是换方案 |
| `reflect` | 行为约束 | orchestrator | 复盘重复出现的流程，提可复用的 skill / 命令 / 配置改进 |
| `oh-my-opencode-slim` | 工具 | orchestrator | 配置、调优 OMO slim 自身（agent、模型、提示词、MCP） |
| `worktrees` | 工具 | orchestrator | 用 git worktree 做隔离的编码车道，跑复杂 / 风险 / 并行活 |

## 分成四类看

- **工具**（有脚本、会碰外部系统，产出文件或数据）：`codemap`、`clonedeps`、`worktrees`、`oh-my-opencode-slim`。
- **流程**（规定先做什么后做什么，产出计划或评审顺序）：`deepwork`、`verification-planning`、`simplify`。
- **行为约束**（纯文本，只改"该不该这么干"的判断，不改可调用的能力）：`ai-estimates`、`deanchor`、`step-back`、`reflect`。
- 三类之间没有开关意义上的差别——都是靠 frontmatter 的 `description` 匹配触发，用户不必点名。

## 两个容易踩的点

- `simplify` 的 `allowedAgents` 是 `["oracle"]`，主 agent 加载不了它；要简化代码得派给 oracle。
- 另有 1 个 **permission-only** 的 `requesting-code-review`（同样只给 oracle），它不进 `CUSTOM_SKILLS`。算授权时它参与，算"随包 skill"时不算 —— 所以上面是 11 个而不是 12 个。

## 怎么自查（只读）

```powershell
# 1) 装了哪个版本
Get-ChildItem "$env:USERPROFILE\.cache\opencode\npm" -Filter 'oh-my-opencode-slim*' -Directory |
  Select-Object -ExpandProperty Name

# 2) 读权威清单（取最新版本目录下的 dist\server\index.js）
$root = (Get-ChildItem "$env:USERPROFILE\.cache\opencode\npm" -Filter 'oh-my-opencode-slim*' -Directory |
         Sort-Object Name | Select-Object -Last 1).FullName
$idx  = Get-ChildItem $root -Recurse -Filter index.js |
        Where-Object { $_.FullName -like '*dist\server*' } | Select-Object -First 1
Get-Content -LiteralPath $idx.FullName | Select-String -Pattern 'var CUSTOM_SKILLS' -Context 0,45
```

- 每个 skill 的原文在 `node_modules/oh-my-opencode-slim/src/skills/<name>/SKILL.md`，随包源码可直读。
- 判断某 skill 在不在：看 `CUSTOM_SKILLS` 数组，**别只看 `src/skills/` 有没有文件夹**（`loop-engineering` 的文件仍在，但没注册）。

## 已验证 / 推测

- **已验证**：11 个 skill 的名字、description、`allowedAgents`、`requesting-code-review` 不在数组内、`loop-engineering` 源码残留。
- 相关内容：[三个行为约束 skill 的细节](OMO-slim-三个行为约束skill-估工防照抄与收手.md)、[角色名册与模型解析](OMO-slim-定位与角色名册.md)。
