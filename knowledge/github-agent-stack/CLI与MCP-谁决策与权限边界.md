# CLI 与 MCP：谁决定下一次调用

> 核对：2026-11-02。结论来自官方文档/源码原文 + 本机 `gh` 实测；没跑过的标 **推测**。

## 判据只有一条

不是「新旧」，也不是「谁更强」，而是：**下一次调用由谁决定**。

- 人/脚本决定 → CLI（`gh`、`gh api`）。
- 模型决定 → MCP。
- 模型决定「做什么」、脚本决定「怎么做」→ 读写分离：读走 MCP，写走 CLI。

同一个平台两套接口并存是常态，不是过渡期。GitHub 官方 MCP 和 `gh` 各自有明确缺口，谁也替不了谁。

## 能力对照

| | CLI（`gh`） | MCP |
|---|---|---|
| 输出 | 确定性，`--json` + `--jq` | 由 server 决定，结构化但不保证稳定 |
| 门禁 | exit code 可直接当 CI 判据 | 无 exit code 语义，只能看 `isError` |
| 覆盖范围 | `gh api` 能打任意 REST/GraphQL | 只能等官方加工具 |
| 离线 | 二进制 + 本地 git 可用 | 远程端点要联网 |
| 本地 git 语义 | checkout/diff/credential helper 都在 | **没有**（工具面是 API，不是仓库） |
| 审计 | 命令行可留痕、可复现 | 调用在模型上下文里，需 host 侧记日志 |
| 适合 | 脚本、CI、批量、需要确定性 | 探索、跨系统组合、非工程角色 |

`gh api` 是 CLI 的抗遗漏能力：官方 MCP 没封装的端点，它一条命令就能到。反过来，MCP 的工具集裁剪、内容过滤、自然语言组合，CLI 也没有等价物。

**已验证**：`gh extension --help` 说明扩展只是加子命令；MCP README 的工具集列表里没有本地 git 操作（`repos` 是远程仓库 API）。

## 只读开关不是授权边界

这是最容易误判的一条。

- `--read-only` / `X-MCP-Readonly` / `/readonly` 只做一件事：**跳过写工具**。它不缩小 token 的权限。
- 官方文档原话：只读模式是「安全过滤器」，agent **仍然能读 token 允许它读的私有数据**。
- Lockdown 是另一回事：过滤公开仓库里无 push 权限用户产生的内容，防注入，不是限权。
- 真正限权的地方只有一处：token/身份本身的 scope 与仓库权限。

所以「已开启只读」不能作为「这个 agent 碰不到生产数据」的理由。要限权就换只读 token。

## 工具注解不可信

MCP 规范对 `annotations`（含 `readOnlyHint`）的原文是：

> clients **MUST** consider tool annotations to be untrusted unless they come from trusted servers.

即：注解是**建议**，协议不强制 server 说实话，client 也不该拿它当门禁。已有工具把写操作标成只读的披露，属于这类风险的实际案例。

规避方式：host 侧对写工具加 deny/ask 兜底，不依赖 `readOnlyHint`。OpenCode 可以在 agent 的 `tools` 里按名字/通配关掉 MCP 工具（官方示例 `"mymcp_*": false`），这是可落地的兜底点（**已验证**：`opencode.ai/docs/agents` 的 Tools 段）。

## 间接提示注入是结构性的

- issue/PR 正文、评论、仓库文件会原样进上下文；任何能给公开仓库写 issue 的人都是潜在注入源。
- 这不是「模型不够聪明」，是数据通道本身不可信。
- 官方缓解：Lockdown 模式 + 默认开启的内容清洗（changelog 原文：`comprehensive content sanitization is now enabled by default`）。
- 但清洗不等于免疫，仍需 host 审批。

分层防御，按成本从低到高：

1. 只读 token / 只读端点先兜底。
2. 工具集最小化（`X-MCP-Toolsets` / `--toolsets`），别开 `all`。
3. Lockdown + 内容清洗。
4. host 侧对写操作 `ask`/`deny`。
5. 读写分离：读的 agent 和写的 agent 不共享凭据。

## 凭据单一来源

多个工具接同一平台时，token 应该派生自同一处（同一个 PAT、同一个 GitHub App 安装、同一个 credential helper）。否则会出现「CLI 能推、MCP 403」这类权限漂移，排查时还以为是工具坏了。

反例：`gh auth` 一份、Git Credential Manager 另一份、MCP 配置里再手写一个 PAT —— 三份独立过期、独立 scope。

**已验证**：`gh auth status` 与 `git push` 的凭据本来就是分开的，这一点在 `github-cli/gh-命令速查与排查.md` 里有实测记录。

## CLI 侧的真实风险不是权限

是 **agent 拼 shell 参数**。模型把 issue 标题、分支名、文件路径直接拼进命令行，就多了一个注入面：

```bash
# 危险：变量来自外部内容
gh issue create --title "$TITLE" --body "$BODY"
```

- 优先用支持 stdin/文件输入的命令，或 `--json` 读结构化数据。
- 不要在 `bash -c` 里拼接未转义的外部字符串。
- 破坏性命令（`gh repo delete`、`gh pr merge`）在 agent 配置里设成 `ask`，别设 `allow`。

## 反模式

- 把 MCP 当 CI 脚本用：没有 exit code、输出不稳定、还要维持一个常驻连接。
- 给 agent 一个可写 token，又指望「它会小心」。
- 用 `readOnlyHint` 当唯一防线。
- 为了「统一」把所有操作都塞进 MCP，结果丢掉本地 git 能力。
- 读写共用一个高权限 token。

## 一句话版本

读用 MCP 省上下文，写用 CLI 保确定性；判断交模型，执行交脚本；授权永远回到 token，不回到开关。
