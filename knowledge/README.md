# knowledge —— 测试过程中攒下来的各种知识

这个文件夹放的是**在沙箱里实测过**的知识：踩过的坑、验证过的命令、搞清楚的机制。**不限于 Git / GitHub** —— 任何领域、任何工具、任何流程都往这里收，成不成类只看有没有第二个同类文件。
不是教程搬运，每条结论都对应一次实际执行。

例外只有两类，都在文件开头写明核对范围：`reading-notes/`（用户要求归档的读书/史料笔记，按原文保留）和 `sensenova-skills/`（读上游源码得出的能力速查，标了 `[源码]` 的条目没在本机实跑）。

## 怎么用

- 一个类别一个子文件夹，先看下面的目录表再进去。
- 文件名自解释：看名字就知道里面讲什么，不用逐个点开。
- 每条结论尽量带：结论 → 命令 → 现象/输出 → 为什么。

## 目录

下面只是**当前已有**的类别，会随知识类型增长；新知识没地方放就新建子文件夹，并同步这张表。

| 子文件夹 | 讲什么 |
|---|---|
| `agent-config-layers/` | 指令的四种注入时机（常驻/条件/按需/角色）、Copilot 与 OpenCode 的路径对照、为什么「glob vs 目录」的说法不准确、叠加不等于覆盖 |
| `github-agent-stack/` | GitHub agent 的四层可安装物（内置命令/扩展/skill/MCP）、`gh skill` 版本门槛、官方 MCP 的工具集与只读开关、CLI 与 MCP 的选型判据、gh-aw 的编译式工作流与 safe-outputs |
| `git-worktree/` | 工作树的原理、删除、被占用排查 |
| `git-branch/` | 分支删除、安全边界（什么时候会真的丢东西） |
| `git-remote/` | 本地目录名与远程仓库名的关系、首次推送全流程 |
| `github-credentials/` | 一份凭据扩散到哪几处、怎么只读地查（只比长度 + 哈希前缀，不打印明文）、轮换顺序；本沙箱真实踩过的掩码事故 |
| `github-cli/` | `gh` 命令速查与排查 |
| `github-code-search/` | 公开代码搜索（grep.app 类工具）能干什么、不能干什么 |
| `chrome-devtools-mcp/` | Chrome DevTools MCP：抢前台与后台开页面、rAF 节流、用 trace 量动画每帧开销、SVG 局部/全局坐标互量 |
| `eval-comparison/` | 同一任务下多个模型产物的横向对比结论（先例：两个鹈鹕骑自行车动画的实现与性能；追加：Iceman 版"轮子绕画布中心公转"等三处硬伤与验证方法） |
| `oh-my-opencode-slim/` | OMO slim 插件：角色名册、配置 schema、Council 多模型共识机制、生成的 preset 与 provider 来源、随包 11 个 skill 全表（含估工/防照抄/收手三个行为约束型） |
| `openchamber-storage/` | 本地库的存储机制与体积：`-wal`/`-shm` 是什么与为什么不能手删、删了为什么不缩（freelist/VACUUM/auto_vacuum）、收缩流程与风险、粘贴图片存文件还是 base64 落库 |
| `opencode-host/` | 托管 OpenCode CLI 怎么定位、有哪些子命令、PATH 里同名 V1 的辨识、托管配置文件 |
| `opencode-provider/` | provider 从哪来（内置 catalog 三层来源）、凭据归谁（opencode-go 与 opencode 同身份）、端点（zen/inference 同网关）、配置块作为覆盖层删了会怎样、OpenChamber 怎么管 provider |
| `opencode-v2/` | OpenCode V2 的分发（CLI/Desktop 两条线、Windows 装 Desktop 不带终端命令）、三端一服务架构、web UI 认证与端口、V1/V2 源码对照纠错、插件注册工具的实际 API（无工具文件形态、`ctx.tool.transform`、schema 三选一、发现路径与单文件限制）、技能发现与 Skill Bundle（嵌套分组、单文件技能、frontmatter 真实约束、主从代理编排） |
| `planning-capabilities/` | 从想法到计划的 14 个能力：`/weigh`、`/plan-feature`、`/craft-goal` 三个 OpenChamber 命令，加上 config / OMO 的 skill，按"卡在哪一步"分流 |
| `reading-notes/` | 读书/史料笔记（非实测）：《卫国那些事儿》与《左传》引文核对 |
| `review-commands/` | 三种审查命令的分工：`/review` 是引擎内置（模板在源码里，界面改不了），`/handoff-review`、`/workspace-review` 是宿主魔法提示词（可改）；附命令来源的核对方法 |
| `session-provenance/` | 从 `opencode.db` 回溯某文件是哪个模型、哪次会话产出的（相似度比对、multi-run worktree 的过滤坑） |
| `sensenova-skills/` | SenseNova-Skills 图像能力速查（读源码得出，非实跑）：五个 skill 分层、三个生图后端的端点与响应形态、配置优先级与 `.env` 查找顺序、六条坑（21:9 / 4K / 改图锁模型 / 空转参数 / 配额 / 水印），以及"Windows 不必上 WSL2"的本机实测纠正 |
| `vscode/` | VS Code 1.143：harness（Copilot/本地）两套 agent 与指令来源、Copilot Chat 内置包与旧条目边界、关掉内置 chat 扩展的联动与界面变化、BYOK 覆盖范围与语义索引来源、OpenCode 模型接入；扩展禁用/卸载与更新残留、键位覆盖与组合键陷阱、补全折叠与语言服务边界、GitHub 集成边界、光标字体 Profile 等设置坑 |
| `workbuddy/` | WorkBuddy 桌面与同源 CLI：两侧技能/记忆/连接器隔离、内置工具与技能的区别、场景路由与输出形态、自定义模型热加载与协议边界、专家包结构、会话目录编码与 JSONL 结构、`/compact` 与 `/总结` 三种触发路径 |
| `writing-prompts/` | 中文改写提示词：降 AI 检测清单（用户实测约 40%→0.x%，未独立复现）、它实际改了什么、已知副作用 |

## 建类判据

这轮结构巡检（23 个分类逐个读完、再互相评边界）暴露出来的问题几乎都是同一类：**同一个事实在两个分类里各写了一遍，靠文末互链硬撑**。以后判断"该不该新建文件夹、该不该并"用这几条：

1. **一个答案只能有一个落点。** 先问"读者想查这件事，会翻哪一篇"，那篇就是权威，别处最多留一行引用。互链不算划界——两篇各写一遍、结尾各指对方一次，读者还是得读两遍。
2. **按"目的"分界，不按"提到了什么"分界。** 三篇都提到 `opencode.db` 不算重叠（一篇回溯产出、一篇讲体积收缩、一篇讲凭据残留，关切不同）；两篇都在回答"凭据在哪"才是重叠。
3. **说得出这个类收的第二篇长什么样，才建文件夹。** 说得出来就成类，单文件也行（`git-branch`、`sensenova-skills`、`session-provenance` 都是这么立的）；说不出来（一次性对照、一次性纠错记录）就写成某个分类里的一节。
4. **横切类只做索引，不复制事实。** 按概念或用途切的分类（`agent-config-layers`、`planning-capabilities`、`review-commands`）负责"卡在哪一步、该看谁"；具体路径和命令留在工具类里。
5. **目录名的宽窄要对得上内容。** 比内容宽（只写"删分支"却叫"分支"）会被误读，比内容窄（装进通用手段却只叫某个厂商）迟早放错东西。对不上时优先移篇或改名，别靠索引描述长期兜。

## 写作纪律（重要）

- **不写隐私**：不写账号名、邮箱、仓库 URL、本机绝对路径、token 或它的任何派生物（哈希、前缀）。
- 需要举例时用占位符：`<owner>/<repo>`、`<path>`、`~/.config/<app>`。
- **图按性质选画法，别自己拼 ASCII 框图或渲染图片**：
  - **Mermaid**（md 里直接写 ` ```mermaid ` 代码块）：时序（谁先谁后、谁并行）、判定分流（if/else 树）、状态流转、分层依赖、流程步骤。好处是 GitHub 直接渲染、能 diff、能复制。
  - **`.excalidraw`**（OpenChamber 已装 Excalidraw 扩展，Files 视图点开就是画布）：需要自由布局、手绘标注、并排对比的草图。文件名带"草图"语义，并在对应 md 里用相对链接指过去。
- 纯目录树、命令、表格、清单这类**不是图**的内容照常写文本 —— 给它们套一张图只会多一层阅读成本。
- Mermaid 图里同样不写隐私：路径用 `<owner>/<repo>`、`<path>` 这类占位符。
- Mermaid 坑（实测，mermaid 11）：尖括号按图类型处理不同 —— **flowchart** 里裸写 `<name>` 会被当 HTML 标签整段吃掉，必须用 `&lt;` `&gt;`；**sequenceDiagram** 里反过来，`&lt;` 直接 Parse error，尖括号裸写即可；**stateDiagram** 的迁移标签（`s1 --> s2: …`）两种写法都会被搅乱（`&lt;` 错位、裸 `<` 连后半句一起吃），干脆别在这种标签上放尖括号。`#` 同理：sequenceDiagram 的消息和 Note 里会**截断后文**（`a ## b` 只剩 `a`），flowchart 里裸 `#` 无害，只有 `#35;` 这类实体码会被解码。长标签用 `<br/>` 手动折行，不然宽图在 GitHub 上会被缩到看不清。
- 既有的 `.excalidraw` 保持原样，不按新划分去迁移 —— 那批图本来就是按当时的口径画的。
- 命令优先给"只读"版本；破坏性命令必须同时写清风险和前置检查。
- 区分 **已验证**（本机实测）与 **推测**（未验证），推测要标明。
