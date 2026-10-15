# 工作树的原理与清理（含"目录被占用"排查）

## 一句话

`git worktree` 让同一个仓库同时检出多份工作目录，共享同一个 `.git`；删的时候**不是 `rm -rf`**，要让 Git 撤销登记。

## 布局长这样

> 草图：[`主工作树与链接工作树.excalidraw`](./主工作树与链接工作树.excalidraw)、[`worktree-删除流程与占用排查.excalidraw`](./worktree-删除流程与占用排查.excalidraw)（OpenChamber 的 Files 视图里点开就是画布）

```
<repo>/                       ← 主工作树：真正拥有 .git 目录
├── .git/
│   ├── worktrees/
│   │   ├── featA/                每个链接工作树一份登记
│   │   │   ├── gitdir      → 指向链接工作树里的 .git 文件（绝对路径）
│   │   │   └── HEAD        → 该工作树当前分支
│   │   └── featB/
│   └── ...
└── ...

<somewhere>/featA/            ← 链接工作树：只有一个 .git 文件
└── .git                      → 内容是 "gitdir: <repo>/.git/worktrees/featA"
```

要点：主工作树的路径**不落盘**，链接工作树的路径**以绝对路径落在 `gitdir` 里**。

## 看现状（只读）

```bash
git worktree list            # 人类可读，一行一个
git worktree list --porcelain  # 机器可读，脚本用这个
```

## 删除

```bash
git worktree remove <path>          # 干净时够用
git worktree remove --force <path>  # 有未提交/未跟踪改动时必须加，否则拒绝
git worktree prune                  # 清掉"登记还在、目录已没了"的僵尸项
git worktree list                  # 自检
```

`--force` 会**连同未提交内容一起删掉**，不可恢复。删前先看清哪些是脏的：

```bash
git worktree list --porcelain | Select-String '^worktree ' |
  ForEach-Object { $p = $_.ToString().Substring(9); git -C $p status --porcelain }
```

## 坑：`failed to delete ... Permission denied` / `being used by another process`

现象：`git worktree remove` 报错，或者登记没了、目录还在；`Remove-Item` 也报 "被另一个进程使用"。

原因：Windows 上只要**任何进程把该目录作为自己的工作目录（cwd）**，整个目录就被句柄锁住，删不掉。常见占用者：还开着的编辑器、终端、AI 会话进程。文件全删光了但目录节点留着，通常就是这种情况。

处理顺序：

1. 先结束占用它的进程（或重启宿主程序）；
2. 再删目录：`Remove-Item -LiteralPath '<path>' -Recurse -Force`；
3. 最后 `git worktree prune` 兜底清理登记。

判断有没有残留：`git worktree list` 只剩主工作树 + `Remove-Item` 成功即可。

## 已验证 / 推测

- 已验证：`--force` 语义、`prune` 作用、占用导致的删不掉、清理后自检方式。
- 推测：不同宿主程序对"占用"的具体持有方式（cwd 锁 vs 打开文件句柄），未逐个复现。
