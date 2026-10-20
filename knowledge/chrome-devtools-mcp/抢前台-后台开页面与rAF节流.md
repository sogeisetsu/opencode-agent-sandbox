# Chrome DevTools MCP 会把你的 Chrome 窗口顶到前台

## 结论（已验证）

用 chrome-devtools MCP 开页面/切页面，默认会把 Chrome 窗口拉到前台，抢走你正在用的窗口。

- `new_page` 不带 `background: true` → 新标签页**前台**打开，窗口跳到前台。
- `select_page` 带 `bringToFront: true` → 激活标签页，窗口也跳到前台。
- 实测后果：人在写别的东西时被打断（用户明确反馈过这一点）。

## 不抢前台怎么用（已验证）

- `new_page({ url, background: true })` —— 后台开标签页，窗口不动。
- `select_page({ pageId })` **不传** `bringToFront`（默认 false）。
- 收工 `close_page` 掉所有开过的页；最后一个 `about:blank` 关不掉，会一直留着。

## 坑：后台标签页的 rAF 被节流（已验证）

后台标签页 `document.visibilityState === 'hidden'`，Chrome 把 `requestAnimationFrame` 降到 1Hz：

| 状态 | 3 秒内帧数 | 帧间隔 | 折算 fps |
|---|---|---|---|
| 前台标签页 | ~140 | 13.3 ms | ~47 |
| 后台标签页 | 4 | ~1000 ms | 1.3 |

所以：

- **测帧率必须在页面前台时做**，后台测出来的数字完全失真。
- 后台页**截图仍然能拿到当前画面**（`take_screenshot` 对隐藏标签页有效），所以"只看不测"的视觉检查可以全程后台做，不打扰用户。

## 为什么

Chrome 对不可见标签页做 rAF 节流省电，`visibilitychange → hidden` 时生效。`select_page` 切标签会让新标签变成 visible，旧标签变 hidden —— 顺带会把窗口也激活到前台。

## 建议

在自动化浏览器任务里，除非明确需要前台渲染，默认一律 `background: true`；只有真的要测帧率/动画时序时才把页面顶到前台，并且提前告知用户。

## 收工：怎么把 MCP 的 Chrome 彻底关掉（已验证）

`close_page` 关不掉最后一个页面（工具明说"最后打开的页面不能关闭"），所以哪怕把所有标签页都关了，**MCP 拉起来的那个 Chrome 进程还会一直活着，窗口继续留在你桌面上**。不处理的话，任务做完了你的桌面还杵着一个空白 Chrome 窗口。

### 先分清哪个是你的 Chrome（只读核对）

MCP 的浏览器命令行里带独立的 profile 路径：

```
--user-data-dir=<user>/.cache/chrome-devtools-mcp/chrome-profile --remote-debugging-pipe
```

你日常那个 Chrome 没有这个 profile 路径。核对命令：

```powershell
Get-CimInstance Win32_Process -Filter "Name='chrome.exe'" |
  Select-Object ProcessId, ParentProcessId, CommandLine
```

### 关法

找出**根浏览器进程**（`--type=` 之外的第一个，它的 `ParentProcessId` 是 chrome-devtools-mcp 的 node 服务），`Stop-Process -Id <pid> -Force`，子进程（gpu/utility/renderer/crashpad）会跟着一起退：

```powershell
Stop-Process -Id <browserRootPid> -Force
```

杀完再核对一遍，确认没有残留：

```powershell
Get-CimInstance Win32_Process -Filter "Name='chrome.exe'" |
  Where-Object { $_.CommandLine -like '*chrome-devtools-mcp*' }
```

### 注意

- **下次再调用 chrome-devtools 的任何工具，Chrome 会被重新拉起来**（冷启动几秒，窗口可能又抢前台）。所以这是"任务彻底结束、短期不再用浏览器"时才做的收尾。
- 只杀带 `chrome-devtools-mcp` profile 的那棵进程树，别按 `chrome.exe` 名字批量杀 —— 会连你日常浏览器一起干掉。

