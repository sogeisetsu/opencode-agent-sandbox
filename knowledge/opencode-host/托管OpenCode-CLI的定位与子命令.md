# 托管 OpenCode CLI：怎么定位、有哪些子命令

适用对象：OpenChamber 托管的那份 `opencode.exe`（不是用户自己装的）。

## 怎么定位它（权威、可脚本）

OpenChamber 每个 host 进程写一份 `%USERPROFILE%\.config\openchamber\managed-opencode\<pid>.json`，里面的 `binary` 字段就是托管 `opencode.exe` 的绝对路径。执行该 exe 拿信息即可，别去猜安装目录。

```powershell
# 只读：列出所有 pid 文件里登记的 binary
Get-ChildItem "$env:USERPROFILE\.config\openchamber\managed-opencode" -Filter *.json |
  ForEach-Object { (Get-Content $_.FullName -Raw | ConvertFrom-Json).binary }
```

（这段机制全局 `AGENTS.md` 的"环境"一节也写了，这里只做知识留档。）

## 子命令现状（实测）

- 版本：`opencode --version` → 当时为 `v2.0.28`（2.x 系列）。
- **没有 `agent` 子命令**。`--help` 里可见的子命令：`acp`、`api`、`mcp`、`models`、`run`、`service`、`serve`；全局选项有 `--standalone`、`--server <url>`、`--print-logs` 等。
- `opencode models` 能列出可用模型；其中 `opencode-go/*` 是 Go 套餐里的一批（含 `…-free` 免费项）。数量会随套餐变动——**别记死数字**（当时实测 20 多个）。
- `opencode api` 是打到运行中的服务器，不是本地直接执行；对需要鉴权的路由会返回 `401`。

## 配置文件的托管关系

`~/.config/opencode/opencode.jsonc` 受 OpenChamber 托管，旁边有 `opencode.jsonc.openchamber.backup`。**手工改这份文件有被 host 覆盖的风险**，改前先确认并留副本。

## 已验证 / 推测

- **已验证**：pid 文件的 `binary` 字段定位法、当前 build 无 `agent` 子命令、`--help` 列出的子命令集、`models` 能列套餐模型、`opencode.jsonc.openchamber.backup` 的存在。
- **推测**：`api` 路由的鉴权细节（只确认了会 401，未逐一验证各路由）。
- 时效提醒：版本号、子命令、模型数量都会随 host 更新变化，引用前重新跑一次 `--version` / `--help` / `models`。
