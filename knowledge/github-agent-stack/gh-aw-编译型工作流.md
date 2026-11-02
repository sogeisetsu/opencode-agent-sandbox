# gh-aw：把 agent 工作流「编译」成 CI 配置

> 核对：2026-11-02，`github/gh-aw` 官方 README + 仓库内实际工作流文件。未在本机跑 `gh aw compile` 的部分标 **推测**。

## 它解决什么

普通 CI 是确定性脚本；agent 工作流需要判断、归纳、生成，写不成固定脚本。直接让模型在 Actions 里跑，又会把「可写权限 + 不可复现」带进 CI。

gh-aw 的做法：**人写自然语言源文件，工具编译成标准 Actions YAML**。人读的和机器跑的分开，后者不手改。

- 仓库：`github/gh-aw`，description `GitHub Agentic Workflows`，MIT（`gh api repos/github/gh-aw` 实测）。
- 安装：`gh extension install github/gh-aw`（README Quick Start 原文）。所以它是**扩展**，属于 `gh` CLI 层，不是 MCP。

## 双文件约定

README 原文：源文件由 YAML frontmatter（triggers、permissions、tools、engine）+ Markdown 正文组成；`gh aw compile` 校验后生成 `.lock.yml`，由 Actions 执行。

仓库里的实际结构（`gh api repos/github/gh-aw/contents/.github/workflows` 实测）：

```
<name>.md          # 源文件，人手改
<name>.lock.yml    # 编译产物，机器生成
```

`search/code?q=repo:github/gh-aw+extension:lock.yml` 返回 346 个文件，说明这不是示例，是仓库自己就在用的约定。

规则只有一条：**只改 `.md`，重编译，不手改 `.lock.yml`**。手改会在下次编译时被覆盖，且 review 时看不出差异来自源还是产物。

**已验证**：同一个名字的 `.md` 与 `.lock.yml` 成对出现（如 `agent-performance-analyzer.md` / `agent-performance-analyzer.lock.yml`）。

## 安全默认值

README 原文：

- agent job 默认 **只读 GitHub 访问 + 沙箱执行**。
- 配置的写操作走 `safe-outputs`：先缓冲、校验，再由**独立 job 以受限权限**执行。
- 网络访问、工具、权限都可配，所以作者必须自己 review，不是装上就安全。

`gh api` 读取实际源文件，能看到 frontmatter 里的 `safe-outputs:` 块和 `permissions:`；lock 文件里对应出现 `GH_AW_SAFE_OUTPUTS`、`generate_safe_outputs_tools` 等生成逻辑。**已验证。**

这层缓冲的意义：模型不能直接拿写 token 调 API，只能在缓冲区里「提议」，由确定性 job 决定要不要执行、执行几次。

## 引擎可插拔

README 列出的内置引擎：GitHub Copilot、Claude Code、OpenAI Codex、Google Gemini、Pi。

换引擎只改 frontmatter 的 `engine.id`，工作流结构不变（**已验证**：源文件里是 `engine:\n  id: ...` 形式）。这条对成本谈判有用：模型涨价/降智时不必重写工作流。

## 边界：什么时候别用

- 任务是确定性的（跑测试、构建、发版）→ 普通 CI。引入模型只会增加不确定性和成本。
- 需要严格可复现的发布门禁 → 普通 CI + exit code。
- 团队没有 review agent 输出的习惯 → 先别开写权限。

agentic 只该用在「需要判断/归纳/生成」的环节，比如 issue 分类、PR 摘要、依赖升级建议、失败日志归因。

## 高频坑

| 坑 | 后果 | 正确做法 |
|---|---|---|
| 手改 `.lock.yml` | 下次编译被覆盖，review 失真 | 改 `.md` 后重编译 |
| 给 agent job 写权限 | 绕过 safe-outputs 缓冲区 | 保持只读，写走缓冲区 |
| 在 workflow 里直接调 `gh` 写命令 | 同上，且绕过校验 | 用 safe-outputs 声明的动作 |
| 改了配置不重编译 | Actions 跑的还是旧 lock | `gh aw compile` 后一起提交 |
| 用错触发命令 | 工作流不启动 | 按 quickstart 的 `gh aw` 子命令来 |

## 成本是双层的

- Actions 分钟数（每个 job、每次触发都算）。
- 模型 token（agent job 的实际开销）。

周期任务别都定在整点：模糊调度（错开分钟）能避开 runner 排队高峰。**推测**：这条是通用调度经验，没在 gh-aw 上单独量过。

## 验证记录

```bash
gh api repos/github/gh-aw --jq '.description,.license.spdx_id,.stargazers_count'
gh api repos/github/gh-aw/contents/.github/workflows --jq '.[].name' | Select-Object -First 12
gh api repos/github/gh-aw/contents/README.md        # 双文件、引擎列表、safe-outputs 原文
gh api "search/code?q=repo:github/gh-aw+extension:lock.yml" --jq '.total_count'   # 346
gh api repos/github/gh-aw/contents/.github/workflows/agent-performance-analyzer.md   # frontmatter 里的 safe-outputs/engine
```

本机没装 gh-aw，`gh aw compile` 的实际输出未验证；编译行为以上游 README 与仓库产物为准。
