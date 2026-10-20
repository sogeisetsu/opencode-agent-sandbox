# SVG 动画：把局部坐标和全局坐标互量，验证"元素真的在它该在的位置"

## 结论（已验证）

动画对不对，**看截图会被骗**（截图只是某一帧，而且小图看不清）。要判断"脚有没有踩在踏板上""轮子转得对不对"，用 `getScreenCTM()` 把 SVG 局部坐标换算到全局/viewBox 坐标，直接量距离。

## 三个 API

- `el.getScreenCTM()` —— 元素局部坐标 → 屏幕坐标的变换矩阵。
- `svg.getScreenCTM().inverse()` —— 反过来把屏幕坐标换回 viewBox 坐标。
- `svg.createSVGPoint()` + `p.matrixTransform(m)` —— 用矩阵变换一个点。

于是"某个元素在 viewBox 里的哪一点"可以这样求：

```js
function toVB(el, x, y) {                    // el 自己的局部坐标 -> viewBox 坐标
  const p = svg.createSVGPoint(); p.x = x; p.y = y;
  const s = p.matrixTransform(el.getScreenCTM());
  const v = s.matrixTransform(svg.getScreenCTM().inverse());
  return [v.x, v.y];
}
```

## 实测抓到的真 bug

某 SVG 鹈鹕动画里，IK 解算本身是对的（小腿末端精确落在踏板曲柄端点，实测偏差 0px），但**可见的脚掌图形画错了位置**：

```html
<g id="shinF">
  <path d="M0,0 L0,46" .../>                    <!-- 小腿：从 0 走到 46 -->
  <path d="M-8,-4 L8,-4 L15,10 L-15,10 Z" .../> <!-- 脚掌：没有任何位移 -->
</g>
```

脚掌路径没有偏移，落在 shin 组原点 = **膝盖**。实测：

- 小腿末端（应为踏板）与脚掌图形锚点距离 **46px**（等于小腿长度 L2）。
- 表现：每帧都有两个黄色楔形块卡在膝盖上，小腿光秃秃地戳在踏板上。

另一份对照实现是对的：脚掌所在的 `.foot` 组额外做了一次 `rotate(-d2)` 反向旋转，世界变换叠加后净旋转为 0 —— 脚掌始终保持水平且精确压在踏板上。

## 顺带能一起验的东西

用同样的方法量：

- 曲柄端点 vs 脚踝：偏差 0px = 真的踩住了。
- 两条腿膝盖相对"髋→脚"连线的朝向：叉积 `dx1*dy2 - dy1*dx2`（注意 SVG 的 y 向下，符号与直觉相反），能发现两条腿反向弯。
- 传动系是否自洽：曲柄周期 vs 车轮旋转周期 vs 地面滚动速度（应由 `v = 2πr/T` 推出），三者对不上就是"轮子在原地空转"。

这些用纯源码推演也能做（把两骨 IK 的公式复刻到 node 里跑 360 个采样点），但**活体 DOM 量出来的是最终真相**，两者交叉验证更稳。
