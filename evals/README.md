# evals —— 横向测评产物

这里放的是**模型评测的原始产出**：同一提示词丢给多个模型，各自跑出来的成品，原样存档，用来对比。

和 `knowledge/` 分开的原因：那边是"搞明白了什么"（结论、命令、机制），这边是"某个模型做出来什么"（产物本身）。产物不解释、不修改、不互相修好。

## 结构

```
evals/<任务名>/<model-slug>.html
```

- 一个提示词 = 一个子文件夹，名字用任务主题（`pelican-bike`），小写连字符。
- 一个模型 = 一个文件，文件名就是**实际产出模型**的 slug，通常与 multi-run 分支名 `<任务名>/<model-slug>` 的后半段一致。分支名不一定可信（有分支叫 `fusion-xxx`、产物其实是 DeepSeek 的），对不上就以会话库查到的为准。
- 同一任务下想换提示词（比如"加动画版"），另开子文件夹，别覆盖旧文件。

## 怎么收录

multi-run 在工作树里跑完后，只把最终产物按上面的命名拷进对应子文件夹，**不要合并各模型的答卷**，也不要替它改代码。分支和工作树按 `AGENTS.md` 的流程清掉。

## 登记表

新增文件时在下面补一行，写清提示词要点、跑的模型和产出会话 id —— 文件名里只有模型名，其余只能靠这张表。

| 文件 | 提示词（要点） | 模型 | 时间 | 产出会话 |
|---|---|---|---|---|
| `pelican-bike/deepseek-v4.1-flash-1017.html` | 鹈鹕骑自行车（单文件 HTML 演示） | DeepSeek V4.1 flash 1017 | 2026-10 | — |
| `pelican-bike/minimax-m3.1-flash-free.html` | 鹈鹕骑自行车（单文件 HTML 演示，带播放/速度控制） | MiniMax M3.1 Flash Free | 2026-10 | — |
| `html-svg-2d-javascript/deepseek-v4.1-flash.html` | 章鱼骑自行车（单文件 HTML SVG 2D 动画，带播放/速度控制；后续在同一会话里修了触手路径、围巾遮挡、曲柄转向） | DeepSeek V4.1 Flash（`opencode-go`） | 2026-10-15 | `ses_ec06d2736ffe5Semt6V4cHioyb` |
| `pelican-bike-basic-prompt/iceman-contributor-free.html` | 鹈鹕骑自行车，一句话提示词；首版是静态 SVG，用户要求后才追加 CSS 动画 | Iceman Contributor Free（`opencode-go`） | 2026-10-28 | `ses_e7d52818bffeZZeOZOBqe6ZWDU` |

提示词原文未存档（上表写的是任务要点），只有 `pelican-bike-basic-prompt/` 这一条存了原文：

```
Generate an HTML file that renders an SVG of a pelican riding a bicycle
```

（同一会话里追加的两句：「你那个文件并没有看到呀」——只贴代码未落盘；「这应该是一个动画呀」——补动画。）

`pelican-bike/` 与 `pelican-bike-basic-prompt/` 是两套不同提示词的产物，别混着比：前者产出的两版都自带控制面板与丰富场景，后者是"一句话要求"的下限样本。

同一次章鱼任务里其他模型的产出（`octopus-bike.html` 等）没有入库。

产物之间的横向对比结论不放在这里，见：

- [`../knowledge/eval-comparison/鹈鹕骑自行车-两版实现与性能对比.md`](../knowledge/eval-comparison/鹈鹕骑自行车-两版实现与性能对比.md)
- [`../knowledge/eval-comparison/鹈鹕骑自行车-Iceman版动画失效与验证方法.md`](../knowledge/eval-comparison/鹈鹕骑自行车-Iceman版动画失效与验证方法.md)
