# knowledge —— 测试过程中攒下来的各种知识

这个文件夹放的是**搞明白了**的知识：踩过的坑、验证过的命令、搞清楚的机制。**不限于 Git / GitHub** —— 任何领域、任何工具、任何流程都往这里收，成不成类只看有没有第二个同类文件。
不是教程搬运。

证据来源有两路：本机实测（默认），和外部文档（资料库/云文档里的调研、上游文档与包元数据、上游源码）。后者允许收，但**按文件标证据等级**，不按类目划例外：**已验证** > **源码级** > **文档级** > **推测**，未验证项在文末单列。整类非实测的目前只有 `reading-notes/`（用户要求归档的读书/史料笔记，按原文保留）。分级细则与复核要求见仓库根 `AGENTS.md` 的 `knowledge/` 一节。

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
| `web-search-mcp/` | 免 Key 搜索的真实形态（端点 + 请求头，三家已实测：Tavily 200 出结果、Exa MCP 返回 `web_search_exa`、TinyFish 带头 200 仅 1 个工具 / 不带头 401）、五个候选项目的覆盖与回退对比、fork `search-rotation-mcp` 补 TinyFish 后的改造清单与验收标准 |
| `browser-skill/` | BrowserSkill（v1.0.7，`~/.workbuddy/skills/browser-skill/`）扩展的五个快捷功能：视频录制 / 网站调试 / 长截图均有 CLI、操作录制为 `bsk record`、操作审计仅面板入口且文档无章节；附各自的上限与前置（视频 24h 过期、抓包只记开抓之后、长截图不支持嵌套滚动与虚拟列表）；以及为什么 agent 总新开窗口（Agent Window 隔离 + 默认后台新建 + 借用页签三连，确认框不可绕过）；daemon 篇：`--foreground` 为何必须套宿主后台托管、`BSK_AUTO_START=0` 复用探测、扩展"未连接"的排查顺序与四个误区 |
| `opencode-console/` | opencode.ai/console 的对象模型与页面位置：Member 只含自然人、Service account 是机器身份、Credentials/Active 的读法、`Legacy:` 账号的由来、`Restricted · 1 scope` 与 `All` 两类 key 的风险差异、Activity 四个标签分工；含打码截图 |
| `eval-comparison/` | 同一任务下多个模型产物的横向对比结论（先例：两个鹈鹕骑自行车动画的实现与性能；追加：Iceman 版"轮子绕画布中心公转"等三处硬伤与验证方法；再追加：混元 4 版的传动系自洽性、三层视差无缝与四版横向对照；再追加：MiMo V2.6 Pro 版实测与五版对比，含「周期比==画出齿比」勘误、两骨 IK 腿与无缝回绕） |
| `oh-my-opencode-slim/` | OMO slim 插件：角色名册、配置 schema、Council 多模型共识机制、生成的 preset 与 provider 来源、随包 11 个 skill 全表（含估工/防照抄/收手三个行为约束型） |
| `openchamber-storage/` | 本地库的存储机制与体积：`-wal`/`-shm` 是什么与为什么不能手删、删了为什么不缩（freelist/VACUUM/auto_vacuum）、收缩流程与风险、粘贴图片存文件还是 base64 落库 |
| `opencode-host/` | 托管 OpenCode CLI 怎么定位、有哪些子命令、PATH 里同名 V1 的辨识、托管配置文件 |
| `opencode-provider/` | provider 从哪来（内置 catalog 三层来源）、凭据归谁（opencode-go 与 opencode 同身份）、端点（zen/inference 同网关）、配置块作为覆盖层删了会怎样、OpenChamber 怎么管 provider |
| `opencode-v2/` | OpenCode V2 的分发（CLI/Desktop 两条线、Windows 装 Desktop 不带终端命令）、三端一服务架构、web UI 认证与端口、V1/V2 源码对照纠错、插件注册工具的实际 API（无工具文件形态、`ctx.tool.transform`、schema 三选一、发现路径与单文件限制）、技能发现与 Skill Bundle（嵌套分组、单文件技能、frontmatter 真实约束、主从代理编排）、Desktop 代码评论的落点（composer/note）与"聊天绑定 session"的定位、内置工具的实际注册名（`shell` / `patch`）与 V1 清单的差异（`bash` 会被路由到 `shell`、`apply_patch` 权限并入 `edit`）、LSP 的官方口径（保留配置、不跑服务器、不暴露工具、不产诊断，改用 lint/typecheck）、`todowrite` 默认关闭需 `experimental.todo`、`browser` 是 Code Mode 命名空间而非单个工具，以及与 Pi 对照 |
| `deepseek-harness/` | DSH（DeepSeek Harness）：配置是 bundle 默认层 → profile/home patch → 命令行 overlay 的分层叠加，覆盖条目是**整段替换 `config`**；默认工作区是首启登记记录、「未分组」是兜底分组、三种「消失」语义、zstd 多帧会话日志与「考古」方法；`AGENTS.md` 两个全局根与注入/预算/刷新机制；标题插件 64 token 静默失败与修复；Windows Low IL 沙箱（跨进程 IPC 断、Word COM 挂起、Edge 假成功）、danger-full-access/提权/环境过滤快照/pwsh 探测链；模型统一走 `llm-pi-ai`、内置模型目录是打包静态快照（`never refreshes`）、opencode-go 双 Base URL 四协议、子代理白名单；凭据解析优先级/明文存储/UI 三态与内置 `secret-mask`；插件 bundle/plugin/patch 三概念、三入口、安装生命周期与删除插件生态；插件安全评审的零执行扫描、26 条规则、判定表与「只给结论」输出契约；Web 搜索付费 vs fetch 边界；轨迹视图与 `run_code` 渲染。另两篇：PTC 绑 preset 的问题与 `run_code` 双通道改法；插件生态四块原生短板与公共 npm 包名存废表（上游文档给的作用域约一半 404） |
| `document-generation/` | Windows 生成中文 PDF 的两条路：Pillow 纯用户态位图（无文字层，5 类排版 bug、A4@200dpi 常量、看图验收）vs Word COM 文字层（需 `danger-full-access`，Low IL 下挂起）；PDF 页码/`ToUnicode` 字节级验收与两种「假成功」 |
| `exam-prep/` | 考试备考的事实与真伪校验：山东省考行政执法专业科目的考情、回忆版真题发布节奏与入口、小红书笔记 ID 时间戳反推、AI 假真题三处硬特征；方法可迁移，日期/题量/渠道以当年度公告为准 |
| `planning-capabilities/` | 从想法到计划的 14 个能力：`/weigh`、`/plan-feature`、`/craft-goal` 三个 OpenChamber 命令，加上 config / OMO 的 skill，按"卡在哪一步"分流；另附上游的 `enhance-prompt` 三种形态（命令型 skill / 可安装 skill 包 / 插件）判据 |
| `reading-notes/` | 读书/史料笔记（非实测）：《卫国那些事儿》与《左传》引文核对 |
| `review-commands/` | 三种审查命令的分工：`/review` 是引擎内置（模板在源码里，界面改不了），`/handoff-review`、`/workspace-review` 是宿主魔法提示词（可改）；附命令来源的核对方法；以及会话内让 agent 自查改动的三种提示词写法与四个必加约束 |
| `session-provenance/` | 从 `opencode.db` 回溯某文件是哪个模型、哪次会话产出的（相似度比对、multi-run worktree 的过滤坑） |
| `sensenova-skills/` | SenseNova-Skills 图像能力速查（读源码得出，非实跑）：五个 skill 分层、三个生图后端的端点与响应形态、配置优先级与 `.env` 查找顺序、六条坑（21:9 / 4K / 改图锁模型 / 空转参数 / 配额 / 水印），以及"Windows 不必上 WSL2"的本机实测纠正 |
| `vscode/` | VS Code 1.143：harness（Copilot/本地）两套 agent 与指令来源、Copilot Chat 内置包与旧条目边界、关掉内置 chat 扩展的联动与界面变化、BYOK 覆盖范围与语义索引来源、OpenCode 模型接入；扩展禁用/卸载与更新残留、键位覆盖与组合键陷阱、补全折叠与语言服务边界、GitHub 集成边界、光标字体 Profile 等设置坑 |
| `workbuddy/` | WorkBuddy 桌面与同源 CLI：两侧技能/记忆/连接器隔离、内置工具与技能的区别、场景路由与输出形态、自定义模型热加载与协议边界、专家包结构、会话目录编码与 JSONL 结构、`/compact` 与 `/总结` 三种触发路径；记忆机制三条互不同步的链路（云端画像 / 用户级 `~/.workbuddy/` / 项目级 `.workbuddy/memory/`）、谁才叫「自动」、设置面板三项各自的落点、注入优先级（指令→人设→记忆）的措辞依据、两套同名 `MEMORY.md` 的辨识 |
| `agent-harness-and-clients/` | 开源 Agent 工具的三层判据：自带运行时的 harness（ZCode/NorthCUA/localharness）、多引擎客户端（desktop-cc-gui）、纯前端（OpenCodeUI/官方 web）；未定位到仓库的名单单列，不写特性 |
| `code-review-tooling/` | 代码审查工具按"审查发生在哪一层"分层（会话/PR/CI/审计），含实测星数与许可证：open-code-review、code-review-graph、PR-Agent、reviewdog、brooks-lint；以及那批"只听到名字"的项目的复检结果（可用 / 极小 / 仍未定位） |
| `windows-shell-env/` | Windows 上的三层环境：PSReadLine 补全与预测的真实默认值（Tab 是逐项循环、Ctrl+Space 已是菜单补全）、Git Bash 是 MINGW64+MSYS2 而非 Linux、WSL 的 `ext4.vhdx` 与迁移三步、`wslc` 不是「WSL 3」 |
| `writing-prompts/` | 中文改写提示词：降 AI 检测清单（用户实测约 40%→0.x%，未独立复现）、它实际改了什么、已知副作用 |

## 建类判据

一次全库结构巡检（23 个分类逐个读完、再互相评边界）暴露出来的问题几乎都是同一类：**同一个事实在两个分类里各写了一遍，靠文末互链硬撑**。判断"该不该新建文件夹、该不该并"用这几条：

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
  - **`.excalidraw`**（任何支持该格式的编辑器都能打开，带 Excalidraw 扩展的 VS Code 即可）：需要自由布局、手绘标注、并排对比的草图。文件名带"草图"语义，并在对应 md 里用相对链接指过去。
- 纯目录树、命令、表格、清单这类**不是图**的内容照常写文本 —— 给它们套一张图只会多一层阅读成本。
- Mermaid 图里同样不写隐私：路径用 `<owner>/<repo>`、`<path>` 这类占位符。
- Mermaid 坑（实测，mermaid 11）：尖括号按图类型处理不同 —— **flowchart** 里裸写 `<name>` 会被当 HTML 标签整段吃掉，必须用 `&lt;` `&gt;`；**sequenceDiagram** 里反过来，`&lt;` 直接 Parse error，尖括号裸写即可；**stateDiagram** 的迁移标签（`s1 --> s2: …`）两种写法都会被搅乱（`&lt;` 错位、裸 `<` 连后半句一起吃），干脆别在这种标签上放尖括号。`#` 同理：sequenceDiagram 的消息和 Note 里会**截断后文**（`a ## b` 只剩 `a`），flowchart 里裸 `#` 无害，只有 `#35;` 这类实体码会被解码。长标签用 `<br/>` 手动折行，不然宽图在 GitHub 上会被缩到看不清。
- 既有的 `.excalidraw` 保持原样，不按新划分去迁移 —— 那批图本来就是按当时的口径画的。
- 命令优先给"只读"版本；破坏性命令必须同时写清风险和前置检查。
- 每条结论标证据等级：**已验证**（本机实跑或本机文件核对）> **源码级**（读上游源码/包内容，没跑）> **文档级**（官方文档、README、元数据口径）> **推测**。非实测为主的文件开头写清依据哪几路，文末单列未验证项；时效性强的补一句有效窗口。
- 外部文档入库前至少复核一条关键结论（`curl` 探端点、`npm view` 查包、在安装目录 grep），把实测与文档口径的差异写进文件——差异比原文更有用。
