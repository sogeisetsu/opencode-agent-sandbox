# BSK 的 daemon：为什么要 `--foreground`，扩展"未连接"怎么修

两个问题共用同一个对象：**daemon**。它是 CLI 与浏览器扩展之间唯一的对端。

## 核对口径

- 本机 skill：v1.0.7（`~/.workbuddy/skills/browser-skill/`）。
- **已验证（本机文档）**：`references/environment.md` 全文（Local daemon startup / Verify across tool calls / Extension connection）、`SKILL.md:36-38`、`references/tabs-and-profiles.md:50,58`。
- **已验证（上游原文）**：`Tencent/BrowserSkill` 仓库 `AGENT_INSTALL.md` 第 3、4 节，2026-11-05 取 `main` 分支原文。本机 skill 不附带该文件，只在 `environment.md` 里给链接。
- **推测**：标注了两处（用户自开终端不会被回收的机制解释、`bsk status --json` 是否直接打印监听端口）。

---

# 一、`bsk daemon start --foreground` 是什么

## 一句话

启动 BSK 的**本地常驻服务进程**（CLI ↔ 扩展的中转站），不是"打开浏览器"。`--foreground` 只改一件事：**不要把自己 fork 到后台**。

| 部分 | 含义 |
|---|---|
| `bsk` | CLI（Windows 为 `bsk.exe`，默认装到 `~/.local/bin`，可用 `BSK_INSTALL_DIR` 改） |
| `daemon` | 常驻进程，负责 session 状态、页面操作、与扩展的长连接 |
| `start` | 显式启动（对应 `bsk status` / `bsk daemon stop`） |
| `--foreground` | 进程留在当前终端不自我后台化 |

## 为什么必须加

因为 **WorkBuddy/CodeBuddy 这类宿主会回收 shell 启动的子进程**。原话：

> In WorkBuddy/CodeBuddy, or a host that reaps shell children, use the following workflow before session commands.

自我后台化的进程会脱离任务，随即被回收；后续每次 `bsk` 调用都表现为"连不上 daemon"。

关键的一句容易被忽略：

> `--foreground` alone is insufficient.

也就是说，光加参数不够，**必须再套一层宿主的托管后台机制**（工具参数 `run_in_background: true`）。文档给的两条等价写法：

```json
{"command": "BSK_AUTO_START=0 bsk daemon start --foreground", "run_in_background": true}
```

```json
{"command": "$env:BSK_AUTO_START = '0'; bsk daemon start --foreground", "run_in_background": true}
```

PowerShell 的 `& 'path/to/bsk.exe'` 是调用符，**不是** Unix 的尾部 `&`，别混用。

## 一个看似矛盾的组合

两层，不冲突：

- `--foreground`：命令自身行为 —— 不 fork。
- `run_in_background: true`：宿主调度方式 —— 把这个前台进程装进受托管的后台任务。

合起来 = 一个跑在前台的进程，被宿主后台任务托管，存活期跟着任务走。

## `BSK_AUTO_START=0` 配套用

普通本地命令会**隐式自动启动** daemon。在受管宿主里先把它关掉，再探测：

```sh
BSK_AUTO_START=0 bsk status --json
```

- 关掉的只是隐式启动；显式 `daemon start` 不受影响。
- 目的：**先探测，能复用就复用**，避免多条命令各自起一个 daemon 互相打架。
- 这个环境变量**必须在每次客户端调用里重复带上**（shell 赋值不跨调用持久）；`BSK_HOME` 同理。

## 启动后的判定与生命周期

- **task ID / PID / `daemon ready` 日志都不等于就绪**：必须在**另一次**调用里 `bsk status` 成功才算能用。启动期最多查 5 次、每次间隔 1s；遇到 permission/protocol 错误立即停，不要继续重试。
- 后台任务"一直不结束"是**预期状态**，不要等它完成、也不要当残留任务取消。
- daemon 是**共享的**：多个 session 复用一个；收尾只 `bsk session stop <id>`，不要重启/停掉共享 daemon。`SKILL.md` 明确：`Do not rely on idle cleanup or stop/restart the shared daemon to finish a task.`
- 任务启动后退出或报锁错误 → 看它的输出 + `bsk logs`，再查一次 status：**可能另一个调用方已经把 daemon 起好了**，能连就复用；不要循环重启、不要删运行时文件。
- 空闲会退出、宿主关闭也会带走它；下次用**重新探测**，不要凭旧 task ID 假设还活着。若 daemon 被替换过，先查 session 状态，别盲目重放动作。
- 不要用 `nohup`、`setsid`、`Start-Process`、尾部 `&` 或长 sleep 冒充托管后台。

## 它不做什么

- 不开浏览器窗口、不建页签 —— 那是 `session start` / `navigate` 的事。
- 不执行任务，只挂着等指令。
- 不开"权限"：`doctor` 里的本地进程身份告警只影响提示，IPC 通就能用浏览器命令。

---

# 二、扩展显示"未连接"，怎么让它连上

## 先定义状态

BSK 三件套：**CLI + 本地 daemon + 浏览器扩展**。扩展是客户端，要连上本机的 daemon 才算连接。所以：

> "未连接" = 本机此刻没有可连的 daemon，或扩展没开连接、端口与实际 daemon 不一致。

daemon 本来就不常驻（空闲退出、宿主关闭带走），**平时不跑 BSK 它就该是断开的**，不影响正常上网。

## 不是只有"让 agent 调"一条路

本质条件只有一个：**有一个 daemon 在跑，且扩展指向它**。

| 方式 | 做法 | 适用 |
|---|---|---|
| A. 自己起 | 在**自己的**终端执行 `bsk daemon start` | 只想让扩展变绿、自己手动测 |
| B. 让 agent 起 | `--foreground` + 宿主后台任务，再用 `bsk status --json` 校验 | 紧接着要跑自动化，省一次手动步骤 |

两条路的硬约束相同：**同一个 bsk 可执行文件 + 同一个 `BSK_HOME`**，否则会出现"扩展显示已连接、agent 那边连不上"的假绿。

方式 A 之所以更省事，是因为你自己的终端不是宿主的子进程、不会被连带回收 —— 属**推测**（机制解释，本地文档只在沙箱环境里提供"独立终端兜底"路径，未直接断言进程不被回收）。

## 排查顺序

1. **先诊断，别重装。** `bsk doctor`：每条 `fail` 都带 `hint`，照改一次再跑。
   `bsk status --json` 能返回 = daemon 活着；其中 `browsers` 为空 = **扩展还没连上，不是要再起一个 daemon**（原文：`Empty browsers means the extension needs connecting, not that another daemon is needed.`）。
2. **daemon 没跑** → 自己终端 `bsk daemon start`；在 agent 里则 `--foreground` + 后台任务（见上文）。
3. **daemon 活着但扩展未连接** → 官方安装指南原文：
   > ask the user to open the popup, enable the connection and confirm the port matches the local daemon
   即：打开扩展弹窗、**开启连接开关**、**端口与本地 daemon 对齐**。（"端口从 `bsk status --json` 读"是合理做法，但文档没写 status 会打印监听端口 —— 标**推测**。）
4. **还是连不上** → 依次确认：扩展已启用且装在**实际在用的那个浏览器 / profile**；CLI 与扩展版本匹配（部分能力要协议 1.2+，`request-help` 要 daemon 协议 1.3）；端口未被占用；没有残留多 daemon / 多 `BSK_HOME`；企业策略或安全软件是否拦了本地连接。
5. 浏览器侧动作做完，回到 agent 机器**再跑一次 `bsk doctor`**，清掉剩余 `fail`。

补充一条容易误判的：**全新安装时只有 `extension connected` 一条 fail 属预期**，那正是要去执行第 3 步的信号，不是安装失败。

## 四个误区

| 误区 | 事实 |
|---|---|
| "扩展变绿就能用" | 绿只说明连上了**某个** daemon，不保证是 agent 用的那个（`BSK_HOME`/端口不同即两套） |
| "连不上就重装扩展" | 先 `bsk doctor`；多数是 daemon 没起或端口不对 |
| "杀掉 daemon 重来" | 它是共享的，重启会打断其他在跑的 session |
| "删运行时文件 / 换目录绕过" | 会制造多实例问题；保留原 `BSK_HOME` 与所属 supervisor |

## 一句话

"未连接"不是扩展坏了，只是这台"总机"没人守着。要么自己开终端起，要么让 agent 用托管后台起；**只是正常浏览网页时，让它保持断开完全没问题。**

---

## 来源

- 本机：`~/.workbuddy/skills/browser-skill/SKILL.md`（36-38、87-88）、`references/environment.md`（Local daemon startup、Verify across tool calls、Extension connection）、`references/tabs-and-profiles.md`（50、58）
- 上游：`Tencent/BrowserSkill` → `AGENT_INSTALL.md`（Step 3 Run `bsk doctor` / Step 4 Connect the browser extension）
