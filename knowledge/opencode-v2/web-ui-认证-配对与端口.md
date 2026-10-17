# Web UI 与认证：一次配对链接，30 天会话

## 一句话

OpenCode 自带的 web ui 由"驱动 TUI 的同一个 server"提供，默认开启、密码保护；用 `opencode pair` 拿一个**一次性、5 分钟**的链接换 cookie，会话**保持 30 天且不可配**。

## 怎么进（pair）

```bash
$ opencode pair

  Open a link to connect. Links work once and expire in 5 minutes.

  http://127.0.0.1:<port>/auth/connect/...
```

- 同时打印用户名 / 密码；**用户名默认 `opencode`**。
- 在浏览器打开链接 → 写入会话 cookie 并进入 web ui。
- 扫码（App 里）或把链接 / 服务器地址填进 App，一样等效。
- **也能不用链接**：连接界面有 `link` 和 `password` 两种方式，手输服务器地址 + 密码即可连（App 的 "add server" 对话框里有密码字段）。
- 一次性语义来自源码：配对码 `TTL = 5 分钟`，用 `Cache` 的 `consume` 实现"用过即失效"。

## cookie 与会话

- 会话**保持 30 天**。源码里是硬编码常量 `SESSION_TTL_SECONDS = 30 * 24 * 60 * 60`，**没有 env / 配置开关** → 不可配。
- 改 server 密码 → **所有会话立刻登出**：会话 token 用密码派生的密钥签名（HMAC(password)），密码一变签名全废。
- cookie 名**带端口**（`opencode_session_<port>`）：浏览器在同 host 上不区分端口地共享 cookie，带端口才能把本机多个 server 分开。

## 端口与配置

- 默认只听 localhost。
- **实测（Windows + 托管 v2.0.28，2026-10-17）**：共享 service 地址是 `http://127.0.0.1:49374`，且 `opencode service get` 返回 `{}`（没有任何 port / hostname 持久化覆盖）→ **49374 就是默认值**，不是谁设出来的。
- 早先按官方文档记的是「Windows **45278** / macOS·Linux·WSL **49374**」，理由是「Windows 从 49152 起保留端口，要避开」。**这条与实测冲突，标为存疑**：
  - 45278 落在 49152 **以下**，和「避开保留段」这个理由本身就对不上；
  - 但也只有一个 Windows 样本，排除不了构建差异或环境变量影响。
  - 复核方式：`opencode service status` 看地址 → `opencode service get` 确认没有显式配置 → 换一台干净的 Windows 再取一次。
- `opencode service status` 只打印一行 URL（如 `http://127.0.0.1:49374`），不带额外状态字段。
- 用 `opencode service set` 改：

```bash
opencode service set hostname 0.0.0.0
opencode service set port 49374
opencode service set password "a-long-secret"
opencode service set cors "https://a.example.com,https://b.example.com"
opencode service set env OPENCODE_LOG_LEVEL DEBUG
```

- **改任一设置会停掉后台 server**，要 `opencode service start` 重新拉起才生效。
- `service get cors` 返回 JSON 数组；`service unset cors` 移除。

## serve / pair 的其它形态

- `opencode serve --hostname 0.0.0.0 --port 4096`：前台跑**同一个** server，打印 `server password <pw>`；client 用 `opencode --server <url>` 连。
- `opencode serve --service`：用持久化的 cors 列表；带 `--cors` 则覆盖该进程。
- `opencode pair --remote`：通过 OpenTunnel 把 server 暴露到公网。首次要交互确认；非交互要 `--yes`。官方说明：接受过一次后，后续运行不再提示，即使之后用 `service set remote false` 关掉远程也一样跳过。
- Web 界面（web shell + 静态资源）**打进 CLI 二进制**：构建时把 app 打成归档（`bun-app-archive.bin`）嵌进可执行文件，运行时从内嵌资源里取。`/api`、`/auth/`、`/openapi.json` 不走 UI 分支 —— 所以**未登录的浏览器会先拿到 app 自带的登录界面**。
- SSH 场景：server 只听 localhost 时可转发端口，再在本地打开打印的链接：

```bash
ssh -L 49374:127.0.0.1:49374 <server>
```

## 已验证 / 随时间变化

- 已验证（官方文档）：pair 链接 5 分钟一次性、30 天会话、改密码登出、`service set` 各项、`serve`、`pair --remote` 与 `--yes`、CORS 语义。
- 已验证（本机）：Windows 上的实际监听端口是 49374，且无持久化覆盖（与官方文档的 45278 不符，见上节存疑说明）。
- 已验证（源码）：`SESSION_TTL_SECONDS` 硬编码、token = HMAC(password)、cookie 名带端口、配对码 5 分钟一次性、web shell 由内嵌 app 归档提供。
- **随时间变化**：这是产品面，数值（TTL / 端口）可能被改。
