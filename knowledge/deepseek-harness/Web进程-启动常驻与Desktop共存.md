# DSH Web 进程：启动、后台常驻与 Desktop 共存

> **依据与证据等级**：前序会话的本机实测摘要（本机路径一律占位）＋本轮只读复核（`dsh --version`、`dsh --help`、`Invoke-WebRequest` 端口探针、`Test-Path`）。本轮**没有实际启动常驻 Web 进程，也没有创建计划任务**；默认端口、token URL、后台方式与共存代价来自前序会话记录，逐条标「前序实测」。DSH 是 nightly，端口与命令形态会漂移。
>
> 「当前是 Desktop 还是浏览器 Web」的判定信号与鉴权根因见 [运行环境-Desktop还是浏览器Web](运行环境-Desktop还是浏览器Web.md)，本文不重复。

## 结论

- 要在普通浏览器里用 DSH，自己起 `dsh web`（等价于 `dsh --profile web`），**用启动时打印的带 token URL 完成认证**；token 每次启动都变，不要把裸地址当固定网址收藏。
- `dsh web` 是前台常驻进程：终端可最小化，关终端通常会停服务；要长期用就交给 Windows 任务计划程序。
- Desktop 和 Web 可以同时跑，但不建议长期同时常驻；必须同时跑时错开会话/工作区，并在其中一个 profile 停用 Schedule。

## 启动与认证

| 项 | 内容 | 证据 |
|---|---|---|
| 版本 | 本机 `0.2.2-rc.5`（本轮 `dsh --version`） | **已验证** |
| 启动命令 | `dsh web` = `dsh --profile web` | **已验证**（本轮 `dsh --help` 原文） |
| 默认监听 | `http://127.0.0.1:3080/`，启动打印 `http://127.0.0.1:3080/?token=...` | **前序实测**，本轮未复跑 |
| 认证方式 | launch token 随机生成、只在进程内存里，随打印的 URL 给出；裸地址 401 | **已验证**（401 本轮复探）＋**源码级**（token 机制见判定文件） |

本轮 `dsh --help` 的原文（证明别名关系）：

```text
Examples:
  dsh web                                   boot the web profile (same as: dsh --profile web)
```

本轮只读端口探针（只读、不启服务）：

```powershell
$u = $env:DSH_WEB_URL
"DSH_WEB_URL=$u"
try { $r = Invoke-WebRequest -Uri $u -UseBasicParsing -TimeoutSec 5; "status=$($r.StatusCode)" }
catch { "status=$([int]$_.Exception.Response.StatusCode) body=$($_.ErrorDetails.Message)" }
```

输出：

```text
DSH_WEB_URL=http://127.0.0.1:5103
status=401 body=dsh web authentication required; reopen the URL printed by dsh web.
```

这里的 `5103` 是当前 **Desktop 实例**的内部端口，不是 Web 的默认端口 `3080`。端口会随实例变化；返回 401 只说明有进程在听，不能说明当前是浏览器 Web。想在浏览器里用，必须自己起 `dsh web` 并拿它打印的带 token 地址。

## 进程生命周期

| 场景 | 结果 | 证据 |
|---|---|---|
| 前台运行 | `dsh web` 是前台常驻进程，终端被占用 | 前序实测 |
| 最小化终端 | 服务继续运行 | 前序实测 |
| 关闭终端 | 通常会停止服务 | 前序实测 |
| `dsh web &` | 只是 PowerShell 后台任务，关终端后仍可能终止 | 前序实测 |
| 电脑重启 | 不会自启，需重新启动或配置登录自启 | 前序实测 |
| 受限沙箱里启动 | 需写 `~/.dsh/profiles/web/cordis.yml`，沙箱拒绝时报 `EPERM`；换普通终端起 | **已验证**（本轮 `dsh web --help` 亦触发写 profile 而被拒，路径已脱敏） |

## 关掉终端还继续跑

### 临时：后台进程

```powershell
Start-Process -FilePath "<install>\resources\runtime\cli\bin\dsh.cmd" `
  -ArgumentList web `
  -WindowStyle Hidden
```

- `<install>` 是 DSH 安装目录占位；该 `dsh.cmd` 存在已由本轮 `Test-Path` 核实。
- **坑**：`-WindowStyle Hidden` 后看不到启动打印的 token URL，前序记录没说怎么取回；需要实测再定（见未验证项）。

### 长期：Windows 任务计划程序

| 配置项 | 值 |
|---|---|
| 触发器 | 登录时 |
| 程序 | `<install>\resources\runtime\cli\bin\dsh.cmd` |
| 参数 | `web` |
| 起始目录 | `<工作区目录>` |

前序会话给出的配置：关终端不影响服务，重启并登录后自动启动。本轮未创建任务，实际效果未验证。

## Desktop 与 Web 同时开

结论：可以同时运行，但不建议长期同时常驻。

| 代价 | 说明 | 证据 |
|---|---|---|
| 资源 | 两个完整 Host 同时驻留，内存与 CPU 占用增加 | 前序实测，未量化 |
| 数据 | 默认共享 `~/.dsh` 下的会话、凭据与存储 | **已验证**（配置分层文档）＋前序记录 |
| 会话 | 同时操作同一会话可能触发写锁 | 前序实测 |
| 文件 | 同时修改同一项目文件可能互相覆盖 | 推测 |
| 定时任务 | Schedule 可能重复投递或互相覆盖状态 | 前序实测 |
| 端口 | 同端口会冲突 | 推测 |

| 目标 | 做法 |
|---|---|
| 只要 DSH 界面 | 只开 Desktop |
| 浏览器使用且关终端后继续 | 只起 Web，并用任务计划维护 |
| 必须同时运行 | 用不同会话与工作区，避免同时改同一批文件；其中一个 profile 停用 Schedule；要彻底隔离就用独立 `DSH_HOME` |

## 未验证项

- 本轮未实际启动 `dsh web` 常驻进程：默认端口 `3080` 与带 token 的 URL 未复跑，为前序会话实测记录。
- 未创建任务计划任务，登录自启的实际行为未验证。
- 资源占用、写锁、Schedule 重复投递未量化，保留前序会话记录。
- 隐藏窗口后如何取得启动打印的 token URL 未验证。
- `dsh web --help` 在 workspace-write 沙箱下因写 profile 被拒（`EPERM`）：这是沙箱边界，普通终端未复测，不代表正常启动会失败。

相关：[运行环境-Desktop还是浏览器Web](运行环境-Desktop还是浏览器Web.md) · [配置分层与目录](配置分层与目录.md)
