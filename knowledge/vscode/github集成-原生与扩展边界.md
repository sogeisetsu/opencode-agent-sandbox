# VS Code 里的 GitHub：原生只有登录和 Git 操作

> 来源：本机 VS Code 1.143 实测记录。标 **待确认** 的是推断，未复核。

## 能力边界

原生只有 **登录 + 基本 Git 操作**。PR / Issue 相关功能要靠扩展，**GitHub Repositories**（远程浏览仓库）也是扩展。

## 内置 github 扩展实际可用的命令

- `Publish to GitHub`
- `Open on GitHub`
- `Copy vscode.dev Link`

`Create PR` 在它的 `package.json` 里是 `when=false`，**不会出现在命令面板**——不是坏了，是被关掉了。

## gh 的登录和 VS Code 的内置认证是两套

`gh auth` 和 VS Code 的 GitHub 认证各存各的，互不相认。想让 `git` 复用 `gh` 的凭据：

```powershell
git config --global credential.helper "!gh auth git-credential"
```

改的是全局 Git 配置；反悔就 `git config --global --unset credential.helper`（有多个 helper 时按值精确删）。
