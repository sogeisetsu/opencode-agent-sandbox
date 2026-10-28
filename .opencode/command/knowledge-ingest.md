---
description: 把粘贴/对话里的一大段知识收进 knowledge/：去重归类、脱敏、登记索引，然后提交并 push
agent: build
---

输入是 `$ARGUMENTS`，或用户这条消息里粘贴的原文（`[pasted-context-1.txt]` 一类）。

1. 先读 `knowledge/README.md` 的目录表，看有没有现成类别能装。能装就不新建；装不下才新建子文件夹，并同步目录表。
2. 逐条跟既有文件比对，重复的并进原文，不新开一份。发现分类不合理（该拆、该并、名字看不懂）就直接调整，顺手把索引改对。
3. 写文件：文件名自解释；每条结论标 **已验证** / **推测**；命令优先只读版，破坏性命令写清风险与前置检查。
4. 脱敏扫描用 grep 模式跑，不靠肉眼：账号名、邮箱、仓库 URL、本机绝对路径、token 及其派生物，一律换成 `<owner>/<repo>`、`<path>`、`~/.config/<app>`。
5. 更新 `knowledge/README.md` 目录表；新增类别必加一行。
6. `git status --short` → `git add` → 提交（`docs(knowledge): …`）→ `git push origin main`，然后贴出 `git status --short` 与 `git log --oneline -1` 的真实输出。

不要：整篇搬教程、写没实测过的结论、往 `knowledge/` 放评测产物（那是 `evals/`）。
