# BrowserSkill 扩展的五个快捷功能

## 一句话

这五项是浏览器自动化的**可观测性/固化工具**：留证据、抓故障、整页存档、把人工流程变成可重放动作、事后查账。日常浏览不需要它们，只在「要证据 / 要固化 / 要留痕」时才值得开。

## 核对口径

- 版本：BrowserSkill v1.0.7（装在 `~/.workbuddy/skills/browser-skill/`，腾讯开源，来自 marketplace）。
- **已验证**：五个功能的参数、上限、行为，逐条对照本机 skill 文档 `SKILL.md` 与 `references/*.md`（`video.md`、`debugging.md`、`screenshots-and-canvas.md`、`interaction-details.md`）。
- **已验证（否定项）**：在 v1.0.7 的 `SKILL.md` 与全部 `references/` 里 `grep -rn -i "audit"` 无任何命中 —— 「操作审计」确实没有文档章节、没有 CLI 命令。
- **推测**：扩展弹窗里那一栏的**中文标签**（「快捷功能」及五项中文名）来自扩展 UI 观察，本地文档里只有英文功能名（Video recording / Website debugging / Full-page screenshot / `bsk record` / 无对应项），两者对应关系按语义判定。

## 入口在哪

浏览器扩展弹窗 →「快捷功能」面板，不在 WorkBuddy 主界面。四项有同名 CLI，一项只有面板入口：

| 扩展入口 | CLI 对应 | 文档章节 |
|---|---|---|
| 视频录制 | `bsk video *` | `references/video.md` |
| 网站调试 | `bsk debug *` | `references/debugging.md` |
| 长截图 | `bsk screenshot --full-page` | `references/screenshots-and-canvas.md` |
| 操作录制 | `bsk record *` | `references/interaction-details.md` |
| 操作审计 | 无 | 无 |

## 1. 视频录制 —— 录像存证

把受控标签页录成 MP4（H.264，**无音轨**）。用户不用在扩展里点开始，CLI 起录即生效。

```sh
bsk video start --session <id> --tab-id <tab> --duration 60s --quality standard --json
bsk video status --recording <rid> --json
bsk video stop --recording <rid> --json
bsk video save --recording <rid> --out <path.mp4> --json
```

- 时长默认 60s，取值 1s ~ 10min；`--max-duration` 是 `--duration` 的别名。
- 画质两档：`standard` ≤1280px/15fps，`clear` ≤1920px/30fps。
- **一个浏览器同时只能录一路**；H.264 不可用时 `start` 直接失败。
- 录制产物留在浏览器本地 **24 小时过期**，没有隐式输出路径；CLI 的 `--out` 写在本机，扩展里的 Save As 写在浏览器那台机器上（可能是两台机器）。
- 只有第一个编码帧落盘后 `start` 才返回；页面异步更新时"动作返回"≠"结果已出现"，应在**目标终态出现后再 stop**，否则得到 partial。
- 中断（关标签页、断连、capture 失败）会尽量保留「可播放但不完整」的结果：`stop/save` 退出码 1，文件存在**不代表成功**，必须看 `state` / `completeness` / `stop_reason`。
- 与 `bsk record` 互斥，同一 session 不能同开。

## 2. 网站调试 —— 抓包看后台

抓请求的 URL/方法/请求体/响应体/响应头/耗时，加控制台日志、页面变更与耗时指标，定位"哪个接口坏了 / 为什么慢"。

```sh
bsk debug start --session <id> --name "<问题>"
bsk debug stop  --session <id>
bsk debug export --session <id> --output <new-file.json>
```

- **只记录开抓之后**的流量，早前的流量不可追回 —— 事后说"刚才那个错误查一下"必然查不到。启动总限时 10s。
- 证据读取按 `debug requests` / `debug request <id> --part response|request|headers|timing` / `--pointer /a/b`（精确取 JSON 字段）/ `debug console` / `debug pages`；body 的 `pending|unavailable|omitted|truncated|evicted` 表示**缺证据**，不是空响应。
- 容量：单次抓取 2000 请求 / 8MiB、20 个 pin；整体 30 天 / 50 条 / 50MiB 过期。`pin` 只免单次抓取的淘汰，**不免整体过期**。
- 规则可 `block` / `modify` / `mock` 请求（mock 不发真实请求）；`replay` 是**真的再发一次请求**，可能改服务端数据，且不重放页面事件处理器、不保证 UI 更新。带改动/屏蔽的流量要与真实流量区分，mock 成功不能证明问题修好了。
- 隐私：常见凭证字段入库前脱敏，自由文本仍可能是敏感数据。

## 3. 长截图 —— 整页一张图

```sh
bsk screenshot --session <id> --full-page --out page.png
bsk screenshot --session <id> --full-page --scope current --out loaded.png
```

- 滚到页底拼图，然后**恢复滚动位置与样式**。
- `--scope follow`（默认）跟着追加内容继续截；`--scope current` 只截当前已加载范围。
- 不支持：浏览器内部页、Web Store、**嵌套滚动面板、虚拟化列表**；吸顶/懒加载会错位。
- 超时默认 2min，`--timeout 5m` 仅在 full-page 模式可延长；取消或失败**不留半张图**。
- `--ref` 与 `--full-page` 不能同时用；CLI 与扩展版本要对齐，旧扩展会拒绝 full-page（可退回当作视口图）。

## 4. 操作录制 —— 演示一遍，记住流程

记录人在页面上的**语义动作**（点了哪个元素、填了什么值、跳到哪个 URL），产出动作序列供 agent 重放。

- 与视频录制的区别：视频是给人看的画面，`bsk record` 是给 agent 看的动作序列；**同一 session 不能同开**。
- 适用：路径很深的后台流程（"数据 → 自定义报表 → 第三个标签 → 导出"）每次都重新摸索。
- **红线**：文档明确禁止录制网银、SSO、密码管理器页面 —— 记的是真实输入值。先读 `bsk record` 的 help 再开。

## 5. 操作审计 —— 事后查账

回溯 agent 在受控标签页上做过的操作。**只有扩展面板入口，v1.0.7 文档无章节、无 CLI**；记录粒度、保留时长以扩展内说明为准（未核对，属推测范围之外的空缺项）。

需要等价能力时，可用 `bsk debug` 的操作窗口（`debug operations` / `debug operation <id>`）替代一部分：它能列出「一次操作 → 关联请求、控制台、页面变更、字段取值链」，但那是抓包视角，不是审计流水。

## 决策速查

| 需求 | 选它 | 关键前置 |
|---|---|---|
| 留"它真做了"的过程影像 | 视频录制 | 开录在动作之前；24h 内导出 |
| 搞清接口 / 报错 / 为什么慢 | 网站调试 | 开抓在复现之前 |
| 一张能发人的完整页面图 | 长截图 | 避开嵌套滚动与虚拟列表 |
| 固化复杂流程，以后照着做 | 操作录制 | 不与视频同开；避开登录页 |
| 事后回查历史操作 | 操作审计 | 仅扩展面板 |

## 通用约束

1. 入口在扩展弹窗，不在 WorkBuddy 内；四项有 CLI、一项没有。
2. 四个功能里三项的产物**会过期或需主动导出**：视频 24h、调试 30 天 / 50 条 / 50MiB、长截图落本地文件；只有操作录制产出的是可重放序列而非证据文件。
3. 抓包与录屏都必须**在动作发生前**启动 —— 没有事前开启就没有记录，这是最容易踩的一条。
4. 全部运行在真实登录态的浏览器里，涉账号与写操作先确认再动；录制类功能一律避开登录/支付页面。
