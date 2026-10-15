# `gh` 命令速查与排查

## 先确认能不能用（只读）

```bash
gh --version                    # 有没有装、版本
gh auth status                  # 登录了哪个账号、走的什么协议、token 有哪些 scope
                                # 输出里 token 是掩码的（安全）
gh api rate_limit --jq '.rate.limit, .rate.remaining'   # 真发一次 API 调用，确认网络+凭据都通
```

判据：`auth status` 显示 `✓ Logged in`，且 `rate_limit` 能返回数字。

## 常用只读命令

```bash
gh repo list --limit 200 --json name,visibility,isEmpty    # 自己有哪些仓库（含私有）
gh repo list --visibility private                          # 只看私有
gh repo view <owner>/<repo> --json name,visibility,defaultBranchRef,isEmpty
gh search repos <keyword> --limit 20                       # 搜公开仓库
```

## 会改远程状态的（谨慎）

```bash
gh repo create <owner>/<name> --public        # 建远程
gh repo create <owner>/<name> --public \
  --source . --remote origin --push           # 建 + 关联当前目录 + 推（一步到位）
git push / gh pr create ...                   # 之后都在改远程
```

推送类操作前先过 `git status` / `git log` / `git remote -v`。

## 命令族对应关系

```
gh repo …    远程仓库（create / list / view / clone / delete）
gh api …     直接打 REST/GraphQL（--jq 过滤输出，脚本友好）
gh pr …      Pull Request
gh issue …   Issue
gh auth …    登录态与 scope（无 --force 参数时不会是破坏性操作）
```

## 排查表

| 现象 | 多半是 | 怎么办 |
|---|---|---|
| `gh: command not found` | 没装 / 不在 PATH | 装完重开终端 |
| `auth status` 未登录，但 `git push` 能推 | 两者凭据是**分开**的（gh 自己一份，Git 凭据管理器一份） | 各查各的，别假设同步 |
| API 调用报权限不足 | token 缺 scope（如 `repo`、`workflow`、`delete_repo`） | `auth status` 看 scope，按需重新授权 |
| `rate_limit` 剩余很低 | 短时间大量 API 调用 | 等窗口重置；脚本里加缓存 |

## 注意

- `gh` 与 Git 的 **HTTP 凭据互不相干**：吊销其中一个不影响另一个推送，排查时要分开验证。
- 脚本里优先用 `--json` + `--jq`，比解析人类可读输出稳。
