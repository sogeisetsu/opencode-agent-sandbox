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

- 版本：`opencode --version` → `v2.0.28`（2.x 系列）。
- **没有 `agent` 子命令**。
- `--help` 的 SUBCOMMANDS（v2.0.28 实测，共 17 个）：

  `upgrade` / `update`、`uninstall`、`acp`、`api`、`debug`、`auth`、`mcp`、`plugin`、`models`、`stats`、`mini`、`run`、`session`、`service`、`reload`、`pair`、`serve`。

  早期记录只列了 `acp/api/mcp/models/run/service/serve` 七个——那是漏记，不是版本差异。
- 全局选项：`--standalone`、`--server <url>`、`--auto`、`--continue`、`--session`、`--prompt`、`--wizard`、`--completions`、`--log-level`、`--print-logs`。
- 子命令自带 `--help`（如 `service --help` 列出 start/restart/status/stop/get/set/unset），**端口默认值不在 `--help` 里**，别指望从帮助文本读到。
- `opencode models` 能列出可用模型；其中 `opencode-go/*` 是 Go 套餐里的一批（含 `…-free` 免费项）。数量会随套餐变动——**别记死数字**（当时实测 20 多个）。
- `opencode api` 是打到运行中的服务器，不是本地直接执行；对需要鉴权的路由会返回 `401`。

## 机器上可能同时有两份 `opencode`（实测，容易测错对象）

| 入口 | 来源 | 实测版本 |
|---|---|---|
| 终端里裸敲 `opencode` | npm 全局 `opencode-ai` 包（`<npm-prefix>/node_modules/opencode-ai/bin/opencode.exe`） | **1.18.32（V1）** |
| 托管 binary | `managed-opencode/<pid>.json` 的 `binary` 字段 | **v2.0.28（V2）** |

- 两者是**完全独立的安装**，连大版本都不同。
- 所以「跑个命令看看」时必须先确认跑的是哪一份：拿 `--version` 对一下，或直接用 pid 文件里的绝对路径调用。
- 反面教训：在本机用裸 `opencode` 验证 V2 的端口/子命令/行为，得到的其实是 **V1** 的答案——正好落进「V1/V2 混淆」这个坑（见 `opencode-v2/源码对照与旧结论纠错.md`）。

## 配置文件的托管关系

`~/.config/opencode/opencode.jsonc` 受 OpenChamber 托管，旁边有 `opencode.jsonc.openchamber.backup`。**手工改这份文件有被 host 覆盖的风险**，改前先确认并留副本。

## 已验证 / 推测

- **已验证**：pid 文件的 `binary` 字段定位法、无 `agent` 子命令、`--help` 的 17 个子命令与全局选项、`models` 能列套餐模型、`opencode.jsonc.openchamber.backup` 的存在、PATH 里另一份是 V1 的 `opencode-ai` npm 包（2026-10-17 复核）。
- **推测**：`api` 路由的鉴权细节（只确认了会 401，未逐一验证各路由）。
- 时效提醒：版本号、子命令、模型数量都会随 host 更新变化，引用前重新跑一次 `--version` / `--help` / `models`。
