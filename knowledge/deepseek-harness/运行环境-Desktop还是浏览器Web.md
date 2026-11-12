# DSH 当前是 Desktop 还是浏览器 Web：判定信号与行为差异

> 核对：2026-11-13。依据：本机 desktop 会话实测（环境变量、进程祖先链、运行时路径）＋打包产物源码核对（`dsh-desktop-host`、`dsh-web-app`）。纯浏览器 Web（`dsh --profile web`）未在本机实跑，相关取值按源码与命名约定给出，标**源码级/推测**。DSH 为 nightly，信号名会漂移。

## 结论

**`DSH_WEB_URL` 存在不能说明是浏览器 Web**——Desktop 也是内嵌 Web GUI 在跑同一个 Web 应用。要组合看：

| 信号 | Desktop | 纯浏览器 Web | 证据 |
|---|---|---|---|
| `DSH_PROFILE` | `desktop` | `web`（`dsh --profile web` 就是浏览器 UI） | **已验证**（desktop）/ **源码级**（web） |
| `DSH_PROFILE_DIR` | `<DSH_HOME>\profiles\desktop` | `<DSH_HOME>\profiles\web` | **已验证**（desktop）/ **推测**（web） |
| 进程祖先链 | 含 `DeepSeek Harness.exe`（Electron 主进程） | agent 宿主是 `node`/`dsh` 进程，祖先链里没有 Electron 主进程 | **已验证**（desktop）/ **推测**（web） |
| 运行时/checkout 路径 | 在 `resources\app.asar` 内（如 `<install>\resources\app.asar\dsh`） | 普通源码目录或其他部署目录 | **已验证**（本机）/ **推测**（web） |
| `DSH_WEB_URL` | **也有**，指向本机回环端口 | 有 | **已验证** → 不具区分性 |

最稳的组合是 **`DSH_PROFILE` + 进程祖先链**；再拿 checkout 路径区分「安装版桌面应用」和「源码/其他部署」。

## 本机实测输出

```powershell
Get-ChildItem Env: | Where-Object Name -like 'DSH_*' | Select-Object Name,Value
# DSH_HOME=<DSH_HOME>
# DSH_PROFILE=desktop
# DSH_PROFILE_DIR=<DSH_HOME>\profiles\desktop
# DSH_SESSION_ID=session-...
# DSH_SHELL=1
# DSH_WEB_URL=http://127.0.0.1:<port>

# 进程祖先链（Windows；Get-Process 不带 ParentProcessId，用 CIM）
$p = Get-CimInstance Win32_Process -Filter "ProcessId=$PID"
while ($p) { "{0} {1}" -f $p.ProcessId, $p.Name; $p = Get-CimInstance Win32_Process -Filter "ProcessId=$($p.ParentProcessId)" }
```

祖先链要点（本机）：

```text
pwsh.exe（工具进程）
  ← DeepSeek Harness.exe（agent 宿主；命令行含 ...\resources\app.asar\dsh\node_modules\@deepseek-ai\...）
  ← DeepSeek Harness.exe（--expose-internals，服务进程）
  ← DeepSeek Harness.exe（Electron 主进程）
  ← explorer.exe
```

另外，系统提示里给的「implementation checkout」若形如 `<install>\resources\app.asar\dsh`，即可直接判为安装版桌面。

## 源码依据（源码级）

- `dsh-desktop-host`：桌面主机 `runProfile({ profile: "desktop", ... })`，注释原文 *“Launch the Desktop profile through the Web application and report its URL to Electron.”* → 这是「Desktop 也是内嵌 Web GUI」的根因。
- `dsh-web-app`：`dsh --profile web` 的命令描述是 *“Serve the DeepSeek Harness browser UI.”*；`DSH_WEB_URL` 由这个 Web 应用插件提供，描述为 *“Advertised URL of the DeepSeek Harness Web GUI serving this session.”* → 只要 Web GUI 在服务会话就有这个变量，与是否桌面无关。
- `dsh-desktop-host` 的 `runtimeArchivePath()` 用 `basename(parent) === "app.asar"` 判断「打包运行时」，对应安装版的路径形态。

## 行为规则（拿到判定之后）

- **Desktop**：让用户「重载」时，说**在插件页禁用再启用插件**或**重启应用**；不要说「刷新浏览器」——桌面壳里的浏览器刷新对进程内插件/HMR 不一定生效。
- **纯浏览器 Web**：才可以提「刷新浏览器页面」。
- 另外，改客户端插件是否自动生效还取决于是否有前端构建在跑（如开发态 `dev:web`）；**Desktop/Web 判定 ≠ 热更新是否生效**。

## 坑

- 只看 `DSH_WEB_URL` 会把 Desktop 误判成 Web。
- 只看「浏览器里能打开 127.0.0.1 页面」也不行——那是 Desktop 的 GUI，用户也可能用外部浏览器打开同一个地址。
- 环境变量是**会话启动时的快照**，中途切模式（如工作区内修改 → 完全权限）不改变 profile/URL 判定。
- 受限沙箱下 `Get-CimInstance Win32_Process` 可能被拒（本机在 workspace-write 下遇到过 CIM 访问问题），此时改用系统提示里的 checkout 字段与 `DSH_PROFILE*` 变量。
- `DSH_SHELL=1` 是「由 DSH 托管 shell」的标记，与 Desktop/Web 判定无关。

## 未核实项

- 纯浏览器 Web 侧 `DSH_PROFILE`/`DSH_PROFILE_DIR` 的取值来自 `dsh --profile web` 的命名与 profile 目录约定，未在本机实跑核对。
- macOS/Linux 桌面版主进程名与路径布局不同（Electron 主进程通常不叫 `DeepSeek Harness.exe`）。
- 其他部署形态（容器、远程托管）下的信号未核对。

相关：[配置分层与目录](配置分层与目录.md) · [权限、沙箱与提权](权限-沙箱与提权.md) · [插件体系与安装](插件体系与安装.md)
