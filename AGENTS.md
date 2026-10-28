# AGENTS.md

## 这是什么
一次性测试沙箱，不是产品仓库。两个用途：

1. **横向评测模型**：用 OpenChamber 的 multi-run 让多个模型跑同一提示词，对比结果。产出是单文件 SVG/CSS/JS 演示（早期的 `index.html`、`octopus-bike.html`），现在收进 `evals/<任务名>/<model-slug>.html`。
2. **验证其他项目**：在干净子目录里跑别人的项目，看能不能装/跑起来。

除 `.gitignore` 和 `.opencode/command/`（只放自定义命令，没有别的代码）外没有任何源码、依赖、构建/测试/lint 配置或 CI；`main` 只有一个 `chore: initial commit`。别去找构建命令，也别为了"把项目补完整"而添加 manifest、脚手架、CI 或 README —— 除非用户明确要求；`knowledge/`、`evals/` 各有一份索引 README，属于约定的一部分，不在此列。

## 环境
- Windows + PowerShell 7。仓库路径含空格：`E:\opencode agents\test`，命令里要加引号或用 `-LiteralPath`。
- remote：`origin` → `sogeisetsu/opencode-agent-sandbox`（public），只有 `main` 会推上去；multi-run 的模型分支从不推。所以删分支、删工作树、丢未提交改动基本不可恢复，动手前按全局硬约束先确认。

## multi-run / 工作树约定
- 每次 multi-run 会建一个工作树：`C:\Users\suyue\.local\share\opencode\worktree\<repo-hash>\<dir>`，分支名形如 `<任务名>/<model-slug>`（目录名里 `/` 变 `-`）。
- 每个分支是某个模型对同一提示词的独立答卷：**只对比，不要合并、汇总或互相"修好"**。
- 评测结束后顺手清理：删掉本次的分支和工作树，不长期保留。
- 删工作树用 `git worktree remove --force <path>`（有未提交产物时必须 `--force`），再 `git worktree prune`。报 "being used by another process" 说明还有会话/进程以该目录为工作目录，得先结束对应会话或重启 host，再删目录。
- 清完自检：`git worktree list` 只剩本仓库主目录，`git branch` 只剩 `main`。

## evals/（横向测评产物）
- 用途：**原样**存放各模型对同一提示词的产出，只用于对比。不合并、不改代码、不互相"修好"——哪怕某个模型写得明显更烂。
- 结构：`evals/<任务名>/<model-slug>.ext`。一个提示词一个子文件夹，一个模型一个文件；`model-slug` 用**实际产出模型**，通常与 multi-run 分支名 `<任务名>/<model-slug>` 的后半段一致，但分支名不一定可信（有分支叫 `fusion-xxx`、产物其实是 DeepSeek 的），对不上以会话库为准。同一任务换提示词（比如"加动画版"）就另开子文件夹，别覆盖旧文件。
- 登记：`evals/README.md` 是登记表，新增文件必须补一行 —— 文件名里只有模型，**提示词要点、模型、时间、产出会话 id**只能靠这张表。会话 id 从 `~/.local/share/opencode/opencode.db` 查（方法见 `knowledge/session-provenance/`），不记的话事后只能靠内容反推。
- 从工作树收录时只拷最终产物，分支和工作树按上面的流程清掉。

## knowledge/（知识沉淀）
- 用途：存放**测试过程中实测到的各种知识**（不限于 Git/GitHub —— 任何机制、工具、流程、踩过的坑、验证过的命令都收），不是教程搬运。当前只有 Git/GitHub 类别，只是因为这是目前碰到的。
- 结构约定：**一个类别一个子文件夹**，文件名要自解释（看名字就知道讲什么）；根目录 `knowledge/README.md` 是索引，新增类别时同步更新它的目录表。
- 写作纪律（因为这个仓库是 public，必须守）：**不写隐私** —— 不写账号名、邮箱、仓库 URL、本机绝对路径、token 或其任何派生物（哈希、前缀）；举例一律用 `<owner>/<repo>`、`<path>`、`~/.config/<app>` 占位。每条结论标注 **已验证 / 推测**，破坏性命令必须同时写风险与前置检查。
- 画草图：用 **Excalidraw 扩展** —— 写一个 `.excalidraw` 场景文件放进对应子文件夹，md 里用相对链接指过去；**不要**自己拼 ASCII 框图或渲染 SVG/截图。
- **主动补充（默认动作，别等人催）**：只要这次会话搞明白了某个机制、摸清了一个工具或流程、踩了坑、验证了一条命令，就在收尾前把它写进 `knowledge/` 对应类别；没有合适的子文件夹就新建一个（知识类型不受限，别只往 Git/GitHub 里塞）。
- **架构可调（别硬塞）**：补内容时如果发现现有分类不合理（该独立成一类的、命名看不懂的、索引表过时了），就直接调整 —— 拆/并子文件夹、改名、同步 `knowledge/README.md` 的目录表；架构服务于好找，不要为了"不动既有文件"把知识塞进不合适的地方。
- 入库流程固定，用 `/knowledge-ingest` 命令（`.opencode/command/knowledge-ingest.md`）：去重归类 → 脱敏 → 登记索引 → 提交推送 → 贴真实输出。

## 测试其他项目
- 放在仓库根下的子目录，如 `E:\opencode agents\test\<project>\`，并把这些子目录加进 `.gitignore`，让仓库本体保持干净。
- 子目录内按那个项目自己的规范来，不要套用本文件。

## 规则来源
全局硬约束与工作流在 `C:\Users\suyue\.config\opencode\AGENTS.md`（不泄密、高风险/不可逆操作先确认、自己产的临时产物自行清理等）。本文件只写仓库特有内容，别重复全局规则。
