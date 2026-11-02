# 专家包与技能包：主从代理编排、Skill Bundle、V2 的真实发现规则

> 核对范围（**已验证**）：上游 `anomalyco/opencode` 的 `dev` 分支，commit `805a42f`（2026-11-02）；运行时装机版本 `opencode v2.2.0`（Windows）。结论来自三处证据：源码、`opencode debug agents`、以及一个临时项目里实际放技能后让模型列出的结果。文中没标 **推测** 的都是这三处至少有一处能对上。

## 一句话

OpenCode V2 没有一个叫「专家团」或「专家包」的预封装对象：能配的单位只有**单个 agent** 和**单个 Agent Skill**。所谓「团」是主 agent 在运行时用 `subagent` 工具派子 agent 产生的效果；所谓「技能包」在文件系统上只是一个**没有 `SKILL.md` 的父目录**，社区叫它 **Skill Bundle**，官方没有这个类型。

## 1. 「专家团」不是对象，是编排

`opencode debug agents` 的本机输出（节选）：

| id | mode | 说明 |
|---|---|---|
| `build` / `plan` / `orchestrator` | `primary` | 可直接交互的主 agent |
| `designer` / `explorer` / `fixer` / `observer` / `oracle` | `subagent` | 只能被主 agent 派出去 |
| `fast-responder` | `all` | 两种都能当 |

`subagent` 工具（源码 `packages/core/src/tool/plugin/subagent.ts`）的行为：

- 在**子会话**里跑，`sessions.create({ parentID })` 新建，工具描述明说 “New child sessions start with fresh context”。
- 支持前台（等结果）和 `background: true`（立即返回，完成时通知）；后台模式可以在继续别的工作时跑，因此能同时开多条。
- `primary` 模式的 agent **不能**当子 agent 用，会直接报 `Agent <id> cannot run as a subagent`。
- 嵌套深度由 `experimental.subagent_depth` 控制，**默认 1**，即子 agent 不能再派子 agent。
- 调用前会走 `permission.assert`，所以「能不能派」还受权限规则约束。

agent 的配置 schema 里只有 `model`、`system`、`description`、`mode`、`hidden`、`steps`、`permissions` 等字段，**没有 team / bundle / members** 之类的字段（`packages/schema/src/config/agent.ts`）。

所以 OMO Slim 那种「专家团」的正确理解是：`orchestrator` 是一个 `primary`，它把 explorer / oracle / fixer 等 `subagent` 当后台任务派发；团的效果来自 orchestrator 提示词里的路由规则，不是来自一个叫「团」的配置对象。角色名册和 Council 机制见 [OMO slim 定位与角色名册](../oh-my-opencode-slim/OMO-slim-定位与角色名册.md)、[Council 多模型共识](../oh-my-opencode-slim/OMO-slim-Council多模型共识.md)。

**推测**：其他工具里的「专家包」如果指的是「一个 agent + 一组技能 + 展示元数据」，在 OpenCode 里只能拆成 agent 文件、技能目录和插件/配置三块分别落地，没有单一 manifest 能表达。

## 2. 技能有三种落盘形态

技能的来源有三种形态。前两种由 `SkillScan.scan`（`packages/core/src/config/plugin/skill-scan.ts`）扫描，第三种由 `SkillDiscovery` 拉取：

| 形态 | 例子 | id |
|---|---|---|
| 目录技能 | `<source>/review/SKILL.md` | 目录名 `review` |
| 单文件技能 | `<source>/quick.md`（只在扫描根的**第一层**） | 文件名 `quick` |
| URL 技能源 | 配置 `skills: ["https://…/"]`，远端提供 `index.json` | 远端声明的 name |

第三种走 `SkillDiscovery`：`index.json` 里列 `{ name, version, files }`，文件里必须含 `SKILL.md` 或 `<name>.md`，下载到全局缓存 `skills/<hash>/<name>/`。日常自用基本用不到。

## 3. 扫描规则：分组目录可以嵌套，但一个 `SKILL.md` 会截断往下

`SkillScan.scan` 的循环逻辑：

1. 深度上限 **6**；每层都跳过点开头的条目，以及 `FSUtil.ignoredDirectories` 里的目录（监听器另用 `**/.*` 这类 glob 过滤）。
2. 如果当前目录里有 `SKILL.md`，把它记为一个技能，然后 **`continue`**——不再进这个目录的子目录。
3. 只有深度 0 时，才把当前目录第一层的所有 `.md` 当作单文件技能。
4. 其余子目录入队，深度 +1。

实测（临时项目，4 组文件）：

| 放的路径 | 是否被识别 |
|---|---|
| `.opencode/skills/group-a/inner-one/SKILL.md` | ✅ `inner-one` |
| `.opencode/skills/group-a/`（本身没有 SKILL.md） | ❌ 不作为技能出现 |
| `.opencode/skills/single.md` | ✅ `single` |
| `.opencode/skills/single-dir/SKILL.md` + `single-dir/nested/SKILL.md` | ✅ `single-dir`；❌ `nested-inside` 被忽略 |

两个直接后果：

- **bundle 根目录千万别放 `SKILL.md`**。一旦放了，它自己变成一个技能，底下所有子技能全部被跳过——看起来像「只加载了一个」。
- 分组纯粹是文件系统层面的，嵌套几层都行（≤6），不需要注册父目录。

## 4. Frontmatter：流传的「name 必填、必须等于目录名、匹配正则、description 1–1024」在 V2.2.0 不成立

`SkillFile.parse`（`packages/core/src/config/plugin/skill-file.ts`）只解四个字段：`name`、`description`、`metadata`、`disable-model-invocation`。`name` 和 `description` 都是 optional。

- **`name` 不参与 id**。id 由路径决定：目录技能取目录名，单文件技能取文件名去掉 `.md`。`name` 只是展示名，可以和目录名不一致。实测目录 `bad-name`、frontmatter `name: Bad_Name`，加载结果是 `id: bad-name` / `name: Bad_Name`，没有报错。
- **没有 `^[a-z0-9]+(-[a-z0-9]+)*$` 校验**。上面那个 `Bad_Name` 就加载成功了。源码里 `Skill.Name` 只是 brand 过的字符串。
- **没有 1–1024 长度校验**。`description` 原样进系统提示；写太长就是每轮都烧 token。这条限制更像上游 Agent Skills 规范的约束，不是 V2 运行时行为（**推测**：没去核对规范原文）。
- **`description` 实际上必须有**：schema 虽 optional，但 `SkillInstructions.load` 会把 `description === undefined` 的技能从模型可见列表里过滤掉。没 description 的技能仍在注册表里，理论上可以按 id 显式加载，但模型不会知道它存在。
- `disable-model-invocation: true` 或 `metadata.opencode/autoinvoke: false` 效果相同：不列进模型可见列表，但技能仍存在。
- `name` 与目录名不一致虽然能跑，但 `skill` 工具的入参是 **id**（目录名），模型看到的是 `name`。建议保持两者一致，省得调用时对不上。

## 5. 模型看到什么，`skill` 工具加载什么

`SkillInstructions.load` 给模型的列表只有三项：`id`、`name`、`description`，按 id 排序。正文、路径、文件清单都不在常驻上下文里。

调用 `skill` 后才返回：

- `SKILL.md` 的完整正文；
- `Base directory for this skill: <dir>`，正文里的相对路径按它解析；
- `<skill_files>`：该技能目录下的文件**名单**，最多 10 个，只有名字没有内容。

所以 `scripts/`、`references/`、`assets/` 不会自动进上下文；模型知道有这些文件，需要时再读。这也是技能可以带很多参考资料却不爆上下文的原因。

## 6. 发现根：单数 `skill/` 也会扫

对每个配置目录（系统级、`~/.config/opencode`、项目 `.opencode`），同时扫 `<dir>/skill/` 和 `<dir>/skills/`：

| 范围 | 路径 |
|---|---|
| 项目 | `<project>/.opencode/skill/`、`<project>/.opencode/skills/` |
| 全局 | `~/.config/opencode/skill/`、`~/.config/opencode/skills/` |
| 系统 | `%ProgramData%\opencode\skill(s)\`（Windows）、`/Library/Application Support/opencode/...`（macOS）、`/etc/opencode/...`（Linux） |
| 兼容 | `~/.claude/skills/`、`<project>/.claude/skills/`、`~/.agents/skills/`、`<project>/.agents/skills/` |
| 配置追加 | `opencode.json(c)` 的 `skills: ["./dir", "~/dir", "https://…/"]` |

两个容易漏的点：

- **单数 `skill/` 是有效的**，不是只认 `skills/`。启动日志里能看到两棵目录都被订阅。
- `.claude` / `.agents` 只扫其下的 `skills/`，且是单独的兼容插件处理的；不要以为它们会像 `.opencode` 一样连 `skill/` 也扫。

同名 id 的覆盖关系：技能以 id 为 key 写进 Map，后写入的覆盖先写入的；但「哪个来源算后」取决于插件加载顺序（**推测**：没有专门造同名技能去实测）。要避免歧义，bundle 里保持 id 全局唯一。

## 7. Skill Bundle：文件系统上的分组，不是运行时对象

V2 的 schema 里只有 `Skill.Info`（id/name/description/path/content/autoinvoke），没有 bundle 类型、清单文件或构建步骤。「一组技能」在文件系统上就是：

```text
.opencode/skills/
└── code-review-kit/                 # 分组目录：没有 SKILL.md，本身不是技能
    ├── security-audit/
    │   ├── SKILL.md
    │   └── references/owasp-checklist.md
    ├── performance-review/
    │   └── SKILL.md
    └── pr-style-check/
        ├── SKILL.md
        └── scripts/lint-commit.sh
```

这个布局下会得到 3 个独立技能：`security-audit`、`performance-review`、`pr-style-check`；`code-review-kit` 不会出现在技能列表里。

术语现状：

- **官方**只定义单个 **Agent Skill**，没有给「多个技能的集合」起名。
- **社区**常用 **Skill Bundle**。实测查到的用法：npm 包 `opencode-plugin-preload-skills` 的 README 把功能写成 “Skill Groups | Bundle skills together with `@group-name`”，配置项说明是 “`groups` … Define skill bundles”；另一个包 `maxsyncai-opencode-skills` 的简介直接写 “opencode skills bundle”。
- 也有工具叫 **Skill Group**（就是上面那个插件的 `@group-name`）。所以 “Skill Bundle” 是通用叫法，但不是官方术语，写文档时第一次出现最好带一句解释。

分发一个 bundle 就是把整个父目录拷到目标发现根（如 `.opencode/skills/`），子技能各自独立注册；没有安装脚本或 manifest 要跑。

关键边界：**bundle 没有运行时语义**。放进同一个父目录不会让它们一起加载，也不能用一次 `skill` 调用加载整包；每个技能仍然按自己的 id 单独被发现、单独被 `skill` 工具加载。除非装一个像 `opencode-plugin-preload-skills` 那样的插件，用 `groups` 配置把多个技能绑成一个 `@group` 预加载——那是插件行为，不是 V2 内置能力。

OMO Slim 里的 **Bundled Skills** 是另一个维度：指「由插件随包分发」的 11 个技能，不是说它们共享一个 bundle 结构。清单见 [OMO slim 随包 skill 全表](../oh-my-opencode-slim/OMO-slim-随包skill全表.md)。

## 8. 怎么复核（只读）

```bash
# 锁定上游修订
gh api repos/anomalyco/opencode/commits/dev --jq '.sha + " " + .commit.committer.date'

# 扫描与解析
gh api repos/anomalyco/opencode/contents/packages/core/src/config/plugin/skill-scan.ts?ref=<sha> --jq .content | base64 -d
gh api repos/anomalyco/opencode/contents/packages/core/src/config/plugin/skill-file.ts?ref=<sha> --jq .content | base64 -d
gh api repos/anomalyco/opencode/contents/packages/core/src/skill/instructions.ts?ref=<sha> --jq .content | base64 -d

# 本地 agent 模式（只读）
opencode debug agents
```

技能加载探针（临时项目，跑完即删；会真的调一次模型，产生 token）：

```powershell
# 在 <project>/.opencode/skills/ 下放：
#   group-a/inner-one/SKILL.md      name: inner-one
#   bad-name/SKILL.md               name: Bad_Name（故意与目录名不一致）
#   single.md                       name: single
#   single-dir/SKILL.md             name: single-dir
#   single-dir/nested/SKILL.md      name: nested-inside
opencode run --standalone --format json "List only the names of every skill available to you, one per line. No commentary."
```

实测返回的列表里有 `inner-one`、`Bad_Name`、`single`、`single-dir`，没有 `group-a`，也没有 `nested-inside`——这正是第 3、4 节的依据。探针结束后已删除临时目录。

## 9. 已验证 / 推测

- **已验证**：三种技能形态；深度 6、隐藏目录跳过、`SKILL.md` 截断子树、根层单文件；`name` 可不等于目录名且不校验正则；`description` 缺失会被列表过滤；列表只含 id/name/description；`skill` 工具返回正文+基目录+最多 10 个文件名；发现根含单数 `skill/` 与 `.claude`/`.agents` 的 `skills/`；agent 的 `mode` 三种取值、`primary` 不能当子 agent、`subagent_depth` 默认 1；agent schema 无 team/bundle 字段；npm 上 “skill bundle” 的社区用法。
- **推测**：同名 id 跨来源的精确覆盖顺序；把「专家包」映射到 OpenCode 三块配置的具体做法；1–1024 限制来自上游规范而非 V2。
- 相关：[指令注入的四种时机](../agent-config-layers/指令注入的四种时机-跨工具对照.md)（按需注入的上层视角）、[OMO slim 定位与角色名册](../oh-my-opencode-slim/OMO-slim-定位与角色名册.md)（主从代理的现成例子）。
