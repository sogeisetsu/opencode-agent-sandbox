# oh-my-opencode-slim（OMO slim）：它是什么、内置哪些角色

## 一句话

`oh-my-opencode-slim` 是挂在 OpenCode 上的**插件**，不是 OpenCode 本体的一部分。它注入自己的一整套 agent 编排层：改写/停用宿主自带的 agent、注册自己的角色、按"预设"给角色分模型。Council（多模型共识）就是它内置的功能之一。

## 怎么分清"插件"还是"宿主"

结论：这些 agent 与 hook 都来自插件，`opencode.exe` 里没有。

- 插件名与版本写在用户配置 `~/.config/opencode/oh-my-opencode-slim.json` 的插件列表里（`.jsonc` 也支持）。
- 在托管 `opencode.exe` 里 grep 这些关键词：`councillor` 零命中；`council` 只命中一次，且落在压缩字典词表里，不是 agent 定义。→ 因此 Council 不属于宿主。
- 可直接读的实际安装源码在
  `~/.cache/opencode/npm/oh-my-opencode-slim@<版本>/<hash>/node_modules/oh-my-opencode-slim/`
  其中 `dist/server/index.js` 是打包后的运行时（含全部角色提示词、权限、hook），`dist/**/*.d.ts` 是同名类型声明，`oh-my-opencode-slim.schema.json` 是配置 schema。
- **别信 `~/.local/state/opencode/plugin-meta.json` 里的 `target`**：它是历史加载记录，可能指向已经不在磁盘上的旧路径（实测指向 `packages/<插件>@<旧版本>/node_modules/...`，而实际存在的只有 `npm/<插件>@<新版本>/<hash>/node_modules/...`）。判断「当前跑的是哪个版本」用 `ls` 看磁盘，别用 meta。

> 验证方式（只读）：对上面目录里的 `dist/server/index.js` 直接 `Select-String` 关键词即可定位到定义处，代码里带 `// src/xxx.ts` 的原始路径注释，很好对号入座。

## 内置角色：9 个

`ALL_AGENT_NAMES = ["orchestrator", ...SUBAGENT_NAMES]`，`SUBAGENT_NAMES` 共 8 个，加主角 orchestrator 就是 9 个：

| 角色 | 类型 | 说明 |
|---|---|---|
| `orchestrator` | 主 agent | 编排者：规划、分派、监控、汇总；不是默认实现工 |
| `explorer` | 子 agent | 只读侦察：定位代码、并行搜索、给地图 |
| `librarian` | 子 agent | 外部文档/网页研究：易变 API、版本行为、官方示例 |
| `oracle` | 子 agent | 架构/风险/调试策略/评审；**是升级手段，不是默认验证步骤** |
| `designer` | 子 agent | 独占 UI/UX：布局、层级、间距、动效、响应式 |
| `fixer` | 子 agent | 有界实现：多文件、机械性改动、可并行的分片活 |
| `observer` | 子 agent | 视觉/媒体分析，隔离图片/PDF 字节，返回结构化文本 |
| `council` | 子 agent | 多模型共识合成器（synthesis-only） |
| `councillor` | 子 agent | Council 席位基类，运行时按席位展开成 `councillor-<席位名>` |

- `PROTECTED_AGENTS = new Set(["orchestrator", "councillor"])`：这两个**不能**被 `disabled_agents` 停用（写入会被过滤掉）。注意 `council` 本身**不在**保护名单里，可以被停用。
- 子 agent 工厂 `SUBAGENT_FACTORIES` 覆盖 8 个：explorer / librarian / oracle / designer / fixer / observer / council / councillor。
- `observer` 默认是关的：`DEFAULT_DISABLED_AGENTS = ["observer"]`。要让 `disabled_agents` 生效为"全开"，得显式把它设成 `[]`。

## 模型解析顺序（`getModelForAgent`）

给某个 agent 定最终模型时，按这个顺序（源码 `createAgents`）：

1. 该 agent 覆盖层里的 `model`（顶层 `agents.<name>.model` 与所选预设里的 `<name>.model` 会先合并成一份覆盖）。
2. 没有 `model` 时看 `inheritModelFrom`：
   - `inheritModelFrom: "orchestrator"` → 用 orchestrator 的模型；
   - `inheritModelFrom: "session"` → 用会话当前模型（`/model` 所选）。
3. 特例：`fixer` 未显式给 model 时，**回落到 librarian 的模型**（`librarian.model` 取数组则取第一个）。
4. 都没有 → 用 `primaryModel`（会话模型）；再没有才用各角色内置默认。

另外注册阶段还会用预设再施加一次 model/variant（`presetForAgent`）。因此"顶层 `agents` 覆盖"与"预设覆盖"谁最终胜出，取决于合并实现，**别只凭读代码下结论**——见下节"已验证/推测"。

## 角色分工要点（编排器提示词里的 routing block 原文缩编）

- explorer：只读侦察（glob/grep/AST）。分派：发现、并行搜索、汇总地图；不分派：已知路径 / 需要完整内容 / 马上要改。
- librarian：外部文档与网页研究。分派：易变 API、版本相关行为、官方示例、疑难 bug。
- oracle：架构、风险、调试策略、评审；**升级手段而非默认验证**。
- designer：独占所有 UI/UX；编排器不要自己做界面。它弱在文案，收尾时由编排器复核文案但不改视觉。
- fixer：有界、定义清晰的实现；多文件或机械改动、按区域并行。
- observer：分析图片/PDF/图，把媒体字节挡在上下文外；**永远传完整文件路径**。
- council：高风险决策的多模型共识；**从不是任务上唯一的 lane**，且不是例行工作。

## 随包附带的 skill

`CUSTOM_SKILLS` 里 **8 个**（v3.0.4 实测，源码目录 `src/skills/`）：
`clonedeps`、`codemap`、`deepwork`、`oh-my-opencode-slim`、`reflect`、`simplify`、`verification-planning`、`worktrees`。

- 早先版本还有第 9 个 `loop-engineering`，**3.0.4 里已不存在**（`grep -c 'loop-engineering' dist/server/index.js` → 0）。注意 `/loop` **命令**仍在 `DISABLED_COMMANDS_VALUES` 里——命令没删，skill 删了，别把两者混为一谈。
- 另有 1 个 **permission-only** skill：`requesting-code-review`（`allowedAgents: ["oracle"]`），不进 `CUSTOM_SKILLS`，只在授权计算时参与（`getDefaultGrantedSkillNames` 把两个数组拼起来）。
- 都按角色/预设授予（例如 `skills: { orchestrator: ["*"], oracle: ["simplify"] }` 这类写法）。

## 已验证 / 推测

- **已验证**：归属（插件非宿主）、9 个角色名、`PROTECTED_AGENTS`、`DEFAULT_DISABLED_AGENTS=["observer"]`、`AGENT_ALIASES`、`getModelForAgent` 顺序、fixer→librarian 特例、随包 skill 清单（**v3.0.4 复核：8 个，`loop-engineering` 已移除**）——全部对着 `dist/server/index.js` 读出来的。
- **推测**：当顶层 `agents.council.model` 与所选预设里的 `council.model` **同时**存在时的最终胜出者（合并/再施加两处逻辑并存），建议落地后实机确认生效模型，不要只凭推断。
