# 鹈鹕骑自行车：Iceman 版动画为什么一眼就坏（旋转原点 + 循环接缝）

## 对比对象

| 产物 | 模型 | 相对路径 |
|---|---|---|
| Iceman 版 | Iceman Contributor Free（`opencode-go`） | [`../../evals/pelican-bike-basic-prompt/iceman-contributor-free.html`](../../evals/pelican-bike-basic-prompt/iceman-contributor-free.html) |

提示词原文（本次有存档，可直接复现）：`Generate an HTML file that renders an SVG of a pelican riding a bicycle`。
后续追加要求：先是「文件没生成」（模型只在回复里贴了代码，没落盘），再是「这应该是一个动画呀」。
两版之间同一会话、同一份文件迭代，会话 `ses_e7d52818bffeZZeOZOBqe6ZWDU`（标题 `Pelican bicycle SVG`）。

另外两版（DeepSeek / MiniMax）的运动学与性能对比见
[`./鹈鹕骑自行车-两版实现与性能对比.md`](./鹈鹕骑自行车-两版实现与性能对比.md)，本文件只讲这一版。

## 一句话结论

**静止那一帧看着像回事，一加上动画轮子就整个飞出车架**——`transform-origin: center` 作用在 SVG 子元素上解析成的是 viewBox 中心，不是轮子自己的圆心。这是本节最值钱的一条：写的时候完全合法，跑起来完全错。

## 一、三处硬伤（已验证）

### 1. 轮子绕画布中心公转，脱离车架

```html
<g class="wheel" transform-origin="100 200"> … </g>
```

```css
.wheel { transform-origin: center; animation: spin .8s linear infinite; }
```

CSS 声明优先级高于 SVG 表现属性，于是 `transform-origin="100 200"` 被整条覆盖。而 `center` 在没有 `transform-box: fill-box` 时按 **view-box** 解析：

```
wheel[0] computed transform-origin = 200px 150px
wheel[1] computed transform-origin = 200px 150px
```

`200/150` 正是 `viewBox="0 0 400 300"` 的中点，不是两个轮心 $(100,200)$、$(250,200)$。后果是两个轮子以画布中心为圆心画圆：实测轮心屏幕坐标在一个旋转周期内 X 走 **872.6 → 1072.7 px**、Y 走 **343.7 → 545 px**（约 200px 幅度，页面缩放约 2×）。

修法任选其一：

```css
.wheel { transform-box: fill-box; transform-origin: 50% 50%; }
```

```css
/* 或显式给绝对原点，且不要再写 center */
.wheel.front { transform-origin: 100px 200px; }
.wheel.rear  { transform-origin: 250px 200px; }
```

要点：**SVG 里出现 `transform-origin` 就顺手写 `transform-box`**，别用 `center` 赌默认值。

### 2. 骑行循环每 4 秒瞬移 100px

```css
@keyframes ride { from { transform: translateX(-50px); } to { transform: translateX(50px); } }
.bike { animation: ride 4s linear infinite; }
```

`from`/`to` 是两端的平行位移，`linear` + `infinite` 在接缝处从 +50 直接跳回 −50，没有 `alternate` 也没有回绕背景来遮。实测（后台标签页，rAF 被节流到约 1.4fps）最大单帧位移 **75px**，而正常前进一帧只该约 14px；理论跳变 100px。

修法：`animation-direction: alternate` 或 `animation-timing-function: ease-in-out` 配 `alternate`（来回晃），要么让地面/背景反向滚动、车身原地不动（真横向位移时必须有可参照的地面）。

### 3. 传动比：轮子转速是车身速度的 12.6 倍

| 量 | 值 |
|---|---|
| 轮半径 | 40（viewBox 单位） |
| 轮周长 | 251.3 |
| 辐条周期 | 0.8 s → 轮面线速度 **314 px/s** |
| 车身位移 | 100 px / 4 s = **25 px/s** |
| 比值 | **0.08**（应为 1.0） |

和第 2 条一样属于"自洽性"问题：轮子像空转的砂轮，车却几乎不动。

## 二、画面上缺的东西（已验证，看截图即可）

- **没有腿、没有曲柄、没有踏板**：鹈鹕只有身体椭圆 + 脖子 + 喙 + 一小片翅膀，两只"脚"是固定在 `(165,170)`、`(185,170)` 的两个椭圆加圆点，从头到尾不动，也没连到任何踏板——静态那一帧就是一团浮在中轴上方的小白块。
- **没有地面、没有背景**：整页只有 `#f0f8ff` 底色，车在空白里横向滑动，没有任何参照物，位移读不出来。
- **辐条画成了两条直径**（4 段线 = 2 条过圆心的直线），具有 90° 四重对称，0.8s 转一圈在视觉上退化成 90° 一跳一跳，看不出转向。
- **`<g class="bike">` 没闭合**：结尾只有 `</svg>`。浏览器容错自动补闭合，所以照样渲染（实测 `bike.contains(pelicanBody) === true`），但文件不是良构 XML —— 走 XML/SVG DOM 解析、做校验时会直接报错。

## 三、验证方法（可直接复用）

必须**跑起来看**，不能只读代码也不能只看第一帧。用 Chrome DevTools MCP：

```js
// 1) 旋转原点到底解析成什么
getComputedStyle(el).transformOrigin          // 期望是轮心，不是 50% 50% 解析出的 viewBox 中心
getComputedStyle(el).transformBox             // 别忘了看这个，默认 view-box

// 2) 元素在屏幕上真的在哪（逐周期采样）
const c = g => { const r = g.getBoundingClientRect(); return [r.x + r.width/2, r.y + r.height/2]; };
// 轮心 X/Y 幅度不应超过 0；超过就是绕错原点

// 3) 循环接缝：比较相邻帧位移，和理论每帧位移对比
//    异常帧 ≫ 理论值 → from/to 不闭合

// 4) 传动比：2πr ÷ 辐条周期 ÷ 车身速度，应 ≈ 1
```

两个前提：截图要**在动画跑起来之后**再截（`t=0` 那一帧恰好是唯一正确的一帧）；后台标签页 rAF 会被节流到 1Hz 左右，量帧间位移够用，量性能不够，见
[`../chrome-devtools-mcp/抢前台-后台开页面与rAF节流.md`](../chrome-devtools-mcp/抢前台-后台开页面与rAF节流.md)。

## 四、和另外两版的对照

| 维度 | Iceman 版 | DeepSeek 版 | MiniMax 版 |
|---|---|---|---|
| 腿 / 曲柄 / 踏板 | **都没有** | 有，脚踝–曲柄偏差 0px | 有，脚掌错位 46px |
| 轮子转轴 | **画布中心（错）** | 轮心 | 轮心 |
| 循环接缝 | **每 4s 跳 100px** | 无缝 | 灌木层每 2.17s 跳 300px |
| 地面 / 纵深 | 无 | 有路面，无纵深 | 六层视差 |
| 文件良构 | 否（`<g>` 未闭合） | — | — |
| 提示词强度 | **最低（一句话，未要求动画）** | 未存档 | 未存档 |

注意最后一行：这一版的提示词确实更弱，先要 SVG、动画是后来追加的。但结论不受影响 —— 动画是用户明确要求后加的，加之后**一次都没验证肉眼效果**（否则轮子飞出车架这种问题不可能留下），属于流程问题而非提示词问题。

## 五、给后续评测的提醒

1. 动画类产物必须至少截两张：`t=0` 和任意 `t>0`；两帧不一致的地方才是考点。
2. SVG 元素上加 CSS `transform`/`transform-origin`，先确认 `transform-box`，再量元素屏幕坐标是否漂移。
3. 任何 `from`/`to` + `infinite` 的位移，都要在接缝处采样一帧。
4. "写了动画" ≠ "动了"：本例里 `.wheel` / `.bike` 两条规则都生效，但效果是坏的，静态检查（属性存在、类名对上）全部通过。
