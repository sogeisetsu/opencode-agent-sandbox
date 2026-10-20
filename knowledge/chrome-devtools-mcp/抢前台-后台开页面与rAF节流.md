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
