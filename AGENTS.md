# AGENTS.md

## 这是什么

一次性测试沙箱，不是产品仓库。三件事：

1. **横向评测模型**：用 OpenChamber 的 multi-run 让多个模型跑同一提示词，产出收进 `evals/<任务名>/<model-slug>.html`。
2. **验证其他项目**：在干净子目录里跑别人的项目，看能不能装/跑起来。
3. **知识沉淀**：实测到的机制、命令、坑写进 `knowledge/`。

除 `.gitignore` 和 `.opencode/command/`（只放自定义命令）外没有任何源码、依赖、构建/测试/lint 配置或 CI。别去找构建命令，也别为了"把项目补完整"添加 manifest、脚手架、CI 或 README —— 除非用户明确要求；`knowledge/`、`evals/` 各有一份索引 README，属于约定的一部分。

仓库根只该有 `.gitignore`、`AGENTS.md`、`knowledge/`、`evals/`、`.opencode/`。演示产物一律进 `evals/`，别落在根目录（根目录曾躺过一个 `index.html`，最后靠查会话库才知道是哪个模型做的）。

## 分支纪律（最容易犯的错）

- `git branch --show-current` 不是 `main` 时，说明你在 multi-run 工作树里：**只 commit 到当前分支，绝不切到 `main`、绝不往 `main` 提交或推送，也不要跑到主仓库目录里改文件**。`evals/`、`knowledge/` 的收录同样留在本分支，等用户自己回收。
- 用户说"push"时先看当前分支：模型分支按约定不推，主仓库 `main` 的推送由用户决定。
- 在工作树里跑 `/knowledge-ingest` 时，第 6 步的 `git push origin main` 不适用 —— 只提交本分支。

## 环境

- Windows + PowerShell 7。**主仓库**路径含空格：`E:\opencode agents\test`（命令加引号或 `-LiteralPath`）；工作树路径 `~/.local/share/opencode/worktree/<repo-hash>/<dir>` 不含空格。
- remote 指向一个公开仓库（`git remote -v` 看）。没有 CI、没有 PR 流程。
- 推送可能 TLS 握手失败（`schannel: failed to receive handshake`）：本机走代理，加 `-c http.proxy=http://127.0.0.1:7897` 可过；仍然失败就别硬重试，报给用户。
- 中文文件名在 `git status` 里显示成八进制转义，正常，别当成乱码去修。

## multi-run / 工作树

- 每次 multi-run 建一个工作树，分支名形如 `<任务名>/<model-slug>`（目录名里 `/` 变 `-`）。
- 每个分支是某个模型对同一提示词的独立答卷：**只对比，不要合并、汇总或互相"修好"**。
- 清理由用户在会话结束后做（agent 自己的 cwd 就在该工作树里，删不掉）：`git worktree remove --force <path>`（有未提交产物时必须 `--force`，会丢文件）→ `git worktree prune`。报 "being used by another process" 说明还有会话/进程以该目录为工作目录。
- 清完自检：`git worktree list` 只剩主仓库、`git branch` 只剩 `main`。不查就会积压（漏过一整批，隔了两周才发现）。

## evals/（横向测评产物）

- **原样**存放各模型对同一提示词的产出，只用于对比。不合并、不改代码、不互相"修好"——哪怕某个模型写得明显更烂。
- 结构 `evals/<任务名>/<model-slug>.ext`：一个提示词一个子文件夹，一个模型一个文件；同一任务换提示词（比如"加动画版"）就另开子文件夹，别覆盖旧文件。
- `model-slug` 用**实际产出模型**。multi-run 的分支名不可信（有分支叫 `fusion-xxx`、产物其实是 DeepSeek 的），对不上以会话库为准。
- 登记 `evals/README.md`：提示词要点、模型、时间、产出会话 id 只能靠这张表。**提示词原文能存就抄进表里** —— 早期条目只有要点，事后没法确认几版是不是同一个提示词，对比结论就只能打折。
- 从工作树收录时只拷最终产物，别把工作树里的中间文件搬过去。

## knowledge/（知识沉淀）

- 存放**测试过程中实测到的知识**（任何机制、工具、流程、坑、验证过的命令），不是教程搬运。
- 一个类别一个子文件夹，文件名自解释；`knowledge/README.md` 是索引，新增类别必须同步目录表。
- 这个仓库是 public：**不写隐私** —— 账号名、邮箱、仓库 URL、本机绝对路径、token 及其任何派生物都不写；举例一律用 `<owner>/<repo>`、`<path>`、`~/.config/<app>`。每条结论标注 **已验证 / 推测**，破坏性命令必须写清风险与前置检查。
- 画草图用 **Excalidraw 扩展**：写 `.excalidraw` 场景文件放进对应子文件夹，md 里用相对链接指过去；不要自己拼 ASCII 框图或渲染 SVG/截图。
- 默认动作：会话里搞明白了什么，收尾前就写进去，没有合适类别就新建；发现分类不合理（该拆、该并、命名看不懂、索引过时）就直接调整。
- 入库流程固定用 `/knowledge-ingest`：去重归类 → 脱敏 → 登记索引 → 提交（是否推送见"分支纪律"）→ 贴真实输出。
- 查产出会话 id：别按 `project.worktree` 过滤 —— 工作树里跑的会话，库里可能仍记成主仓库路径，过滤会一条都查不到。直接按消息内容搜（`session_message.data LIKE '%<特征串>%'`），细节见 `knowledge/session-provenance/`。

## 测试其他项目

- 放在主仓库根下的子目录 `E:\opencode agents\test\<project>\`，并加进 `.gitignore`，让仓库本体保持干净。
- 子目录内按那个项目自己的规范来，不要套用本文件。

## 规则来源

全局硬约束与工作流在 `~/.config/opencode/AGENTS.md`（不泄密、高风险/不可逆先确认、临时产物自行清理等）。本文件只写仓库特有内容，别重复全局规则。
