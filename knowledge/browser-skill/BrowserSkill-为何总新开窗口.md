# 为什么 BSK 总是新开窗口，而不是在当前窗口里操作

## 现象

让 agent 用 `bsk` 干活，先冒出一个**新的浏览器窗口**，任务页开在里面；即使用户已经开好目标页面，agent 也不直接用，而是重新开一页。

## 结论

不是 bug，也不是随机行为。是两件事叠加：

1. **运行时架构**：BSK 在浏览器里划出专属受控窗口（Agent Window），agent 创建的所有页面都落在这里。
2. **文档写死的默认约定**：默认新建 session（后台启动、不抢焦点）、用用户页签必须先显式借用并等确认。

agent 没走「列出 → 借用」这两步，就必然落在新建的 Agent 窗口里。

## 核对口径

- 版本 v1.0.7（`~/.workbuddy/skills/browser-skill/`）。
- **已验证**：以下每条结论均对照本机 `SKILL.md` 与 `references/tabs-and-profiles.md` 原文，引号内为原文。
- 文档行号会随版本漂移，下面按**章节/命令**定位，不写行号。

## 原因一：工作模型是 Agent Window

`SKILL.md` 开篇即定义：

> Use `bsk` in an **Agent Window** with the user's existing logins. User tabs require explicit borrowing.

配套证据（`tabs-and-profiles.md`）：

- 会话建立时默认创建一个 `about:blank` 标签页 —— 所以观感是"先开窗，再开页面"，不是"开页面时顺便开窗"：`A default created tab starts at about:blank.`
- `tab list --scope all` 只返回**本会话**的 Agent 页签 + 用户页签，会隐藏其他会话的 Agent 窗口 —— 这也说明"一个 session 一个 Agent 窗口"是架构事实，不是数量巧合。
- Agent 窗口内**未归属**的页签（页面自己弹的窗不会自动受控）需要用户先移到用户窗口，才能被借用。

## 原因二：默认新建 + 后台启动 + 不抢焦点

`SKILL.md` 主流程第 1 步：

> `bsk session start --json` … `Sessions start in the background; use --focus only on explicit user request.`

即窗口出现但不抢占前台。想让它到前台必须用户明确说"打开到前台"，agent 才会加 `--focus`。

## 原因三：用现有页签必须显式借用

唯一受支持的路径是三连（`tabs-and-profiles.md`）：

```sh
bsk tab list --scope user --session <id>   # 列出用户页签
bsk tab borrow <tab-id> --session <id>     # 借用
bsk tab return <tab-id> --session <id>     # 用完归还
```

- 借用会触发确认，**确认框不可绕过**：`Deprecated --unattended, --no-confirm, and BSK_REQUEST_HELP=off cannot override these settings.` 确认与 human help 由扩展保存的 Automation 设置控制，**默认开启**，且对已存在的 session 同样生效；改浏览器设置/存储去绕过被文档明确禁止。
- `tab borrow --timeout 120s` 只改确认等待时长（默认 60s），不免除确认。
- `borrow_outcome_unknown` 时先查 tab/session 状态（页签可能已经移动过），**不要换浏览器后端绕过去**，也不要重复发起 pending/denied/timed-out 的借用。

### 一处容易误解的细节

借用**不等于**"在你的窗口里原地操作"。文档原话是借用后 `selects the borrowed tab within the Agent Window` —— 页签被纳入受控窗口上下文，只是**不额外抢焦点**。任务结束 `session stop` 时归还，`Returned tabs stay open in the user's window.` 所以"借来的页签不会被关掉"成立，但"它一直在你原来那个窗口里安静地被动"不成立。

## 为什么这么设计

| 目的 | 机制 |
|---|---|
| 不打扰 | 后台启动，`--focus` 需明确要求；不在用户正在浏览的窗口里乱开页 |
| 可隔离 | 一个 session 一个 Agent 窗口；`--scope all` 有意隐藏别的会话，避免误操作 |
| 可追溯 | 只有受控窗口/页签上的操作才能被录制、调试、审计 |

共享代价：同一 browser profile 下所有 session **共用一套 cookie 与站点状态**，两个任务操作同一登录站点或切租户会互相覆盖登录态，且无报错（见 `tabs-and-profiles.md`「Parallel tasks and shared login state」）。隔离是窗口级的，不是登录态级的。

## 怎么让它操作当前页面

1. 指令里点名页签：「用我现在开着的 XX 页面操作，不要新开窗口」→ agent 走 `tab list --scope user` → `tab borrow`，浏览器弹确认框，点了才继续。
2. 要前台：「打开到前台」→ 才会加 `--focus`。
3. 任务结束：`bsk session stop <id>` 会归还借来的页签，页签留在用户窗口不被关闭（`SKILL.md` 主流程第 4 步）。

## 边界

- 文档里**没有**"在用户原窗口里新建标签页"的开关。受支持路径只有两条：新建（落 Agent 窗口）或借用已有页签。
- 确认流程不能靠改设置、换后端、deprecated 参数绕过。
- 借来的页签用完必须归还，不要跨任务长期占用；也不要跨不相关任务复用同一页签。
- 不要把"会话数量"当"窗口隔离"的证据 —— 隔离看的是 `browser_instance_id` 与 Agent 窗口归属。

## 来源

- `~/.workbuddy/skills/browser-skill/SKILL.md`：开篇定义、主流程第 1/2/4 步
- `~/.workbuddy/skills/browser-skill/references/tabs-and-profiles.md`：Borrowing and browser settings、Parallel tasks and shared login state
