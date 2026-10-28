# SenseNova 图像 skill：五个 skill、三个后端、Windows 能不能跑

上游：`OpenSenseNova/SenseNova-Skills`（MIT，2026-10-26 快照）。

**核对范围**：能力/参数/端点一节来自对上游源码的静态阅读，**没有在本机实跑生图**（无 `SN_API_KEY`）；因此逐条标 `[源码]`（读代码得出，未运行）或 `[本机]`（本机只读命令实测）。下游别把 `[源码]` 当"跑通过"。

## 五个 skill 的分层

一个底层 + 三个玩法 + 一个体检，除 `sn-image-doctor` 外都依赖 `sn-image-base`。

| Skill | 层 | 做什么 | 关键参数 | 依赖 |
|---|---|---|---|---|
| `sn-image-base` | Tier 0 | 唯一底层出口，4 个子命令：文生图 / 改图 / VLM 识图 / LLM 改写 | `--image-size 2k\|4k`、`--aspect-ratio`、`--seed` | httpx、pillow、python-dotenv |
| `sn-infographic` | Tier 1 | 信息图：87 layout × 66 style，打分→扩写→多轮生图→VLM 评审排序 | `max_rounds 1-8`、`output_mode`、`prompts_expand_mode` | `sn-image-base` |
| `sn-image-imitate` | Tier 1 | 参考图仿风格重画：抽长描述 + 布局蓝图，布局锁定 | `max_attempts 3`、`layout_threshold .75` | `sn-image-base` |
| `sn-image-resume` | Tier 1 | 固定三段式竖版作品集简历图（封面/信息/导航） | `aspect_ratio 9:16`、`image_size 2k` | `sn-image-base` |
| `sn-image-doctor` | Tier 0 | 环境体检：装没装、依赖齐不齐、变量缺不缺，可交互补写 `.env` | `--verbose` | — |

`[源码]`

注意 `sn-image-base` 内部还有 `sn-image-edit` 这个改图入口（见坑清单第 3 条），它不在上表的独立 skill 里。

## 三个可切换的生图后端

由 `SN_IMAGE_GEN_MODEL_TYPE` 选，默认 `sensenova`。

| 值 | 端点 | 模型示例 | 鉴权 / 响应 |
|---|---|---|---|
| `sensenova`（默认） | `POST {base}/images/generations`、`POST {base}/images/edits` | `sensenova-u1.5-lite`（支持 4K）、`sensenova-u1-fast`（仅 1K/2K） | Bearer → `data[].url`，下载后 PIL 校验完整性 |
| `nano-banana` | `POST /v1beta/models/{model}:generateContent` | `gemini-3.1-flash-image-preview` | `x-goog-api-key` → `inlineData` base64 |
| `openai-image` | `POST {base}/images/generations` | `gpt-image-2` | Bearer → `b64_json` |

`[源码]` 三者响应形态不同（URL 拉取 / inline base64 / b64_json），换后端时下游取图逻辑要跟着换，不能只改模型名。

## 配置解析优先级

`CLI 参数 > 命令专用变量 > 域共享变量 > 全局变量 > 内置默认` `[源码]`

- `SN_API_KEY` / `SN_BASE_URL`：全局兜底，一个网关两处够用。
- `SN_IMAGE_GEN_*`：生图专用覆盖（key / model / model_type / base_url）。
- `SN_CHAT_*`：文本 + 视觉共享；`SN_TEXT_*` / `SN_VISION_*` 分别覆盖。
- `SN_CHAT_TYPE`：`openai-completions`（默认）或 `anthropic-messages`。
- `.env` 读取顺序：`~/.openclaw/.env`（或 `~/.hermes/.env`）→ 当前目录 `.env` → 进程环境。

## 坑清单

1. **21:9 不可用**：CLI 的 choices 里列了，但 SenseNova 后端比例超过 16:9 直接抛 `ValueError`。`[源码]`
2. **4K 限模型**：只有 `sensenova-u1.5-lite` 支持 4K，`sensenova-u1-fast` 报错。`[源码]`
3. **改图锁模型**：`sn-image-edit` 要求 model 恰好是 `sensenova-u1.5-lite`。`[源码]`
4. **`--poll-interval` 是空转**：参数保留了，后端实为同步单次 POST，调它没用。`[源码]`
5. **配额**：免费档 1500 次 / 5 小时滑动窗口，超限报 `FREE_QUOTA_EXHAUSTED`。`[源码]`
6. **去水印**：默认 `watermark:false`，当前公测免费，文档说将来可能收费。`[源码]`

## Windows 上能不能跑（本机实测，纠正上游速查里的"需 WSL2"）

流程脚本是 bash，用到 `jq`、`mapfile` 和 `/tmp/openclaw/...`。常被说成"Windows 必须 WSL2"，本机实测在 **Git Bash** 下这三个前提都满足：

```powershell
& '<git>\bin\bash.exe' -lc 'bash --version | head -1; mapfile -t a < <(printf "1\n2\n"); echo ${#a[@]}; command -v jq'
```

实测输出：`GNU bash, version 5.3.9(1)-release (x86_64-pc-cygwin)`、`mapfile_n=2`、`jq` 解析到 WinGet 装的那份。`/tmp` 在 Git Bash 里存在（映射到用户临时目录），`/tmp/openclaw` 只是没建，不是不可写。

`[本机]` 结论：**Windows 上不必为 bash/jq/mapfile 上 WSL2**，真正没验证的只剩两点——Python 依赖（httpx / pillow / python-dotenv）是否装好，以及 `sn-infographic` 的 Main/Worker 子代理架构需要宿主提供 subagent 能力。

本机当前状态（可复跑核对）：

```powershell
[bool]$env:SN_API_KEY; [bool]$env:SN_BASE_URL; Test-Path ~/.openclaw/.env; Test-Path ~/.hermes/.env
```

`[本机]` 全部为 `False`：没有 key、没有两个 `.env`、skills 目录里没有 `sn-*`。也就是说这套东西在本机**从来没配起来过**，上面能力全是读代码得出的。

## 什么时候值得用它

要批量出信息图 / 简历图 / 仿风格图，且能接受"默认走 SenseNova 网关 + 免费额度 1500/5h"时再装；只要单张文生图，直接调 provider 的 images 端点更省。
