# 用 DevTools trace 量化前端动画的每帧开销

## 结论（已验证）

要比较两个 HTML 动画谁更省资源，别只看 FPS。用 CDP trace 拆到每帧各环节，才能定位"到底哪一块贵"。

## 流程

1. **页面必须在前台**（后台标签 rAF 被节流到 1Hz，见同目录 `抢前台-后台开页面与rAF节流.md`）。
2. `performance_start_trace({ pageId, reload: false, autoStop: false })`
3. `evaluate_script` 在页面里 sleep 3~4 秒，让动画跑起来。
4. `performance_stop_trace({ pageId, filePath: "<tmp>/x-trace.json.gz" })` —— 原始 trace 落盘。
5. 用 node 解压解析 `traceEvents`，按事件名聚合。

### 窗口范围怎么取（关键坑）

**不要**用整份 trace 的 `min/max ts` 当测量窗口 —— 文件里混着别的进程和导航事件，跨度会离谱（实测同一份文件按全量 min/max 算出 57 小时）。正确做法是用 `FireAnimationFrame`（rAF 回调）事件的首尾定窗口：

```
t0 = 第一个 FireAnimationFrame.ts
t1 = 最后一个 FireAnimationFrame.ts + dur
spanS = (t1 - t0) / 1e6
帧数 = FireAnimationFrame 计数
```

## 指标含义

| trace 事件名 | 含义 | 备注 |
|---|---|---|
| `FireAnimationFrame` | rAF 回调（自己写的 JS） | 主要看这个 |
| `FunctionCall` | 脚本执行 | 包含回调外的脚本 |
| `Paint` | 绘制 | SVG 改动通常每帧 2 次 |
| `UpdateLayoutTree` | 样式重算 | 次数 ≈ 每帧被改动的元素数 |
| `Layout` | 布局 | SVG 改 transform 也会走这里 |
| `PrePaint` | 绘制前准备 | |
| `RasterTask` | 光栅（合成线程） | 不占主线程 |
| `Commit` | 提交帧 | |

每帧平均 = 总量 ÷ 帧数。
主线程每帧总耗时 ≈ `FireAnimationFrame + Paint + UpdateLayoutTree + Layout + PrePaint + Commit`。

## 实测例子（两个单文件 SVG 鹈鹕动画）

| 指标 | DeepSeek V4.1 flash | MiniMax M3.1 Flash |
|---|---|---|
| 主线程每帧总耗时 | 1.83 ms | 1.72 ms |
| └ rAF 回调 | 0.51 ms | 0.42 ms |
| └ Paint | 0.40 ms（2 次/帧） | 0.46 ms（2 次/帧） |
| └ UpdateLayoutTree | 0.39 ms / 4.3 次 | 0.36 ms / 2.6 次 |
| └ Layout | 0.19 ms | 0.16 ms |
| └ RasterTask | 0.20 ms | 0.21 ms |
| 每帧 DOM 属性写 | 10 次 | 15 次 |
| 长任务 | 0 | 0 |

结论：两者都远低于 16.7ms 预算（约 10 倍余量），性能上分不出高下；差别只在写法 —— 一个把背景滚动交给 SMIL 声明式动画（每帧少写属性），一个每帧手写 15 次 transform。

## 更轻量的替代：页面内自测

只要"每帧写几次 DOM"，不必开 trace，页面里挂个 `MutationObserver` 数属性变更即可：

```js
const mo = new MutationObserver(l => { muts += l.length; });
mo.observe(document.body, { attributes: true, subtree: true });
```

3 秒后 `muts / 3` 就是每秒属性写次数，除以帧数即每帧次数。实测数值与源码里 `setAttribute` 的条数完全对得上（10 和 15），可以用来交叉验证 trace 的结论。

## 注意

headless Chrome 的 rAF **不锁 vsync**，能跑到 100+ fps。这个数字只适合横向对比，不能当成"用户实际看到的帧率"。

`performance_stop_trace` 的返回值里有一份自然语言的 trace 摘要，但里面没有逐项聚合的耗时表；真正的数字得自己解析落盘的 `trace.json.gz`。
