# DSH 当前是 Desktop 还是浏览器 Web：判定信号与行为差异

> 核对：2026-11-16。依据：本机 desktop 会话实测（环境变量、运行时路径）＋打包产物源码核对（`dsh-desktop-host`、`dsh-shell-env`、`dsh-web-app`、`dsh` CLI README）。纯浏览器 Web（`dsh --profile web`）未在本机实跑，相关取值按源码与命名约定给出，标**源码级/推测**。DSH 为 nightly，信号名会漂移。

## 结论

**`DSH_WEB_URL` 存在不能说明是浏览器 Web**——Desktop 也是内嵌 Web GUI 在跑同一个 Web 应用。要组合看：

| 信号 | Desktop | 纯浏览器 Web | 证据 |
|---|---|---|---|
| `DSH_PROFILE` | `desktop` | `web`（`dsh --profile web` 就是浏览器 UI） | **已验证**（desktop）/ **源码级**（web） |
| `DSH_PROFILE_DIR` | `<DSH_HOME>\profiles\desktop` | `<DSH_HOME>\profiles\web` | **已验证**（desktop）/ **源码级**（web，README 写明保存到 `profiles/web/cordis.patch.yml`） |
| `ELECTRON_RUN_AS_NODE` | `1`（Electron 主进程以 Node 模式拉起宿主，工具 shell 继承） | 无（推测：普通终端起的 `dsh` 不带） | **已验证**（desktop 环境变量）/ **源码级**（`desktopNodeEnvironment()`）；Web 侧 **推测** |
| 进程祖先链 | 含 `DeepSeek Harness.exe`（Electron 主进程） | agent 宿主是 `node`/`dsh` 进程，祖先链里没有 Electron 主进程 | **已验证**（desktop）/ **推测**（web）；workspace-write 下 CIM 被拒，本机复核不了 |
| 运行时/checkout 路径 | 在 `resources\app.asar` 内（如 `<install>\resources\app.asar\dsh`） | 普通源码目录或其他部署目录 | **已验证**（本机）/ **推测**（web） |
| `DSH_WEB_URL` | **也有**，指向本机回环端口 | 有 | **已验证** → 不具区分性 |

判定顺序：**`DSH_PROFILE` 为主 → `ELECTRON_RUN_AS_NODE` / checkout 路径佐证 → 进程祖先链只作不受限环境的补充**（workspace-write 下 `Get-CimInstance Win32_Process` 实测被拒）。checkout 路径可再区分「安装版桌面应用」和「源码/其他部署」。

`DSH_PROFILE` 不止两种：内置 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`，`desktop` 保留给 Electron（CLI 拒绝以该名启动）；也可用 `--from-default-profile` 建自定义 profile。只有 `desktop`、`web` 对应有 GUI 的端，其余按「无 GUI 的 CLI / 其他部署」处理，别套浏览器或桌面应用的话术。

## 本机实测输出

```powershell
Get-ChildItem Env: | Where-Object Name -in 'DSH_HOME','DSH_PROFILE','DSH_PROFILE_DIR','DSH_SESSION_ID','DSH_SHELL','DSH_WEB_URL','ELECTRON_RUN_AS_NODE' | Select-Object Name,Value
# DSH_HOME=<DSH_HOME>
# DSH_PROFILE=desktop
# DSH_PROFILE_DIR=<DSH_HOME>\profiles\desktop
# DSH_SESSION_ID=session-...
# DSH_SHELL=1
# DSH_WEB_URL=http://127.0.0.1:<port>
# ELECTRON_RUN_AS_NODE=1

# 进程祖先链（Windows；Get-Process 不带 ParentProcessId，用 CIM；受限沙箱下会被拒）
$p = Get-CimInstance Win32_Process -Filter "ProcessId=$PID"
while ($p) { "{0} {1}" -f $p.ProcessId, $p.Name; $p = Get-CimInstance Win32_Process -Filter "ProcessId=$($p.ParentProcessId)" }
```

`DSH_*` 由 `dsh-shell-env` 在每次 shell 调用时重建注入（不是进程启动快照）；`ELECTRON_RUN_AS_NODE` 是 Electron 主进程传给 Node 模式子进程的环境变量，工具 shell 从宿主继承。

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

- `dsh-desktop-host`：桌面主机 `runProfile({ profile: "desktop", ... })`，注释原文 *“Launch the Desktop profile through the Web application and report its URL to Electron.”* → 这是「Desktop 也是内嵌 Web GUI」的根因；Electron 主进程用 `desktopNodeEnvironment()` 设置 `ELECTRON_RUN_AS_NODE=1` 后以 Node 模式拉起宿主（`lib/main.js`）。
- `dsh-shell-env`：每次 shell 调用重建 `DSH_*` 快照，`DSH_PROFILE` / `DSH_PROFILE_DIR` 取自 profileContext，ambient `DSH_*` 会被丢弃。
- `dsh-web-app`：`dsh --profile web` 的命令描述是 *“Serve the DeepSeek Harness browser UI.”*；`DSH_WEB_URL` 由这个 Web 应用插件提供，描述为 *“Advertised URL of the DeepSeek Harness Web GUI serving this session.”* → 只要 Web GUI 在服务会话就有这个变量，与是否桌面无关。README 写明 Web profile 的配置保存到 `$DSH_HOME/profiles/web/cordis.patch.yml`。
- `dsh-desktop-host` 的 `runtimeArchivePath()` 用 `basename(parent) === "app.asar"` 判断「打包运行时」，对应安装版的路径形态。
- `dsh` CLI README：内置 profile 为 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`，`desktop` 名称保留给 Electron，CLI 拒绝以该名启动。

## 行为规则（拿到判定之后）

- **Desktop**：让用户「重载」时，说**在插件页禁用再启用插件**或**重启应用**；不要说「刷新浏览器」——桌面壳里的浏览器刷新对进程内插件/HMR 不一定生效。
- **纯浏览器 Web**：才可以提「刷新浏览器页面」；改的是服务端插件时，刷新页面不够，要重启启动它的 `dsh` 进程。
- 另外，改客户端插件是否自动生效还取决于是否有前端构建在跑（如开发态 `dev:web`）；**Desktop/Web 判定 ≠ 热更新是否生效**。
- headless / sdk / 自定义 profile：没有 GUI，不提刷新页面或重开应用，按「重跑 CLI 进程」说。
- 配套 skill `dsh-surface-check`（用户级 `~/.dsh/skills/`）把这套判定固化在给建议之前。触发只靠 catalog 里的 name + description：`whenToUse` 等其它 frontmatter 字段当前不渲染给模型，别指望它们触发。

## 坑

- 只看 `DSH_WEB_URL` 会把 Desktop 误判成 Web。
- 只看「浏览器里能打开 127.0.0.1 页面」也不行——那是 Desktop 的 GUI，用户也可能用外部浏览器打开同一个地址。
- `DSH_*` 不是进程启动快照：由 `dsh-shell-env` 在每次 shell 调用时重建注入；但 profile 与 URL 在本会话内不变，中途切权限模式（如工作区内修改 → 完全权限）不改变判定。
- 从 Desktop 的 shell 里再启 `dsh --profile web` 会同时带 `ELECTRON_RUN_AS_NODE`，此时以 `DSH_PROFILE` 为准。
- 受限沙箱下 `Get-CimInstance Win32_Process` 会被拒（本机 workspace-write 实测「拒绝访问」）；判定不必依赖它，用 `DSH_PROFILE*` 与 checkout 字段即可，别为判定去提权。
- `DSH_SHELL=1` 是「由 DSH 托管 shell」的标记，与 Desktop/Web 判定无关。

## 未核实项

- 纯浏览器 Web（`dsh --profile web`）未在本机实跑；`DSH_PROFILE=web` 由 profileContext 注入，目录 `profiles/web` 按 web README 为源码级。
- `headless` / `sdk` / `sdk-minimal` / `acp` / 自定义 profile 未实跑，按 CLI README 的 profile 清单推断为无 GUI。
- Web 侧没有 `ELECTRON_RUN_AS_NODE` 属推测（普通终端起的 `dsh` 不会带）。
- macOS/Linux 桌面版主进程名与路径布局不同（Electron 主进程通常不叫 `DeepSeek Harness.exe`）。
- 其他部署形态（容器、远程托管）下的信号未核对。

相关：[配置分层与目录](配置分层与目录.md) · [权限、沙箱与提权](权限-沙箱与提权.md) · [插件体系与安装](插件体系与安装.md)
