# OMO slim 新增的三个行为约束型 skill：估工、防照抄、收手

## 一句话

v3.0.5 的 `CUSTOM_SKILLS` 从 8 个涨到 11 个，新增 `ai-estimates`、`deanchor`、`step-back`。这仨**不是工具类 skill**（没有脚本、没有 MCP、不碰外部系统），是纯文本的行为约束：只改模型"怎么干活"的判断，不改它能调用什么。

## 三个各自管什么

| skill | 触发时机 | 约束什么 | 核心做法 |
|---|---|---|---|
| `ai-estimates` | 要估工期时（编码、写作、调研、分析、设计、数据活都算） | 别按纯人工的直觉报"几周" | 把人+AI 协作拆两段：AI 能直接产出的按 AI 速度算（分钟到小时，加复核纠错时间），只有人或外部世界能定的（决策、品味、审批、权限、真机验证、等别人）按人工速度算；给区间、说清哪段占大头 |
| `deanchor` | 请求里带示例、样本、模板、参考输出时 | 别把示例轻度改写就交 | 先提炼示例共有的东西（意图、约束、品味），把偶然的扔掉（措辞、长度、主题、顺序），产出同级但原创的版本；自检标准是"能不能被看成某个示例的轻改"，是就重来。只有用户明确点名要那个格式时才照抄形式 |
| `step-back` | 补丁开始叠加时（反复修、加兼容层、旧路径旁边再开一条新路） | 别顺着惯性闷头修 | 停下看全局：目标还是原来那个吗？用现在知道的信息从零开始会不会这么设计？答案"不会"就几行讲清——正漂去哪、为什么漂、更干净的替代方案、建议；然后交给用户定。答案"会"就继续干，别每步都自我怀疑 |

共同点：三者都是**在动手前或动手中途插一个判断**，输出的都是"该不该这么干/该怎么估"，而不是一份交付物。

> 全部 11 个随包 skill（含类型与授权对象）见 [OMO-slim-随包skill全表.md](OMO-slim-随包skill全表.md)。

## 什么时候会真的生效

- 三个都挂在 `allowedAgents: ["orchestrator"]` 上——只有主 agent 能加载，子 agent（explorer/fixer/designer 等）拿不到。所以实际影响的是编排层：怎么估、怎么派活、什么时候停手。
- 靠 description 匹配触发，用户不必点名。想让它们不生效，就是在提示词里明说，或从 `skills` 授权里摘掉。

## 来源与版本核对（只读）

```powershell
# 实际装的插件在 npm 缓存里，按版本目录看
Get-ChildItem "$env:USERPROFILE\.cache\opencode\npm" -Filter 'oh-my-opencode-slim*' -Directory
```

- 本机版本 `oh-my-opencode-slim@3.0.5`；`dist/server/index.js` 的 `CUSTOM_SKILLS` 数组共 11 项：`clonedeps`、`codemap`、`simplify`、`deepwork`、`verification-planning`、`step-back`、`deanchor`、`ai-estimates`、`reflect`、`oh-my-opencode-slim`、`worktrees`。
- 三个新 skill 的原文在 `node_modules/oh-my-opencode-slim/src/skills/<name>/SKILL.md`（随包源码，可直接读，不用反编译 `dist`）。
- 用户级 `~/.config/opencode/skills/` 里**没有**这三个的副本 → 只来自插件，不存在覆盖歧义。
- 顺手发现的坑：`src/skills/loop-engineering/SKILL.md` 文件**还在磁盘上**，但不在 `CUSTOM_SKILLS` 里 → 判定"这个 skill 还在不在"看数组，别只看文件有没有。
- 另一个坑：`reflect` 同时存在于插件和 `~/.config/opencode/skills/`，同名 id 并存。模型可见列表里显示的是用户本地那份的 description（实测）。

## 已验证 / 推测

- **已验证**：版本号 3.0.5、`CUSTOM_SKILLS` 11 项的完整清单、三个 skill 的 `allowedAgents` 都是 orchestrator、三个 SKILL.md 原文内容、本地 `~/.config/opencode/skills/` 无同名副本、`loop-engineering` 源码残留、`reflect` 重名。
- **推测**：`reflect` 重名时到底哪份被加载（可见列表的 description 指向本地那份，但未见加载日志，未做写入侧验证）。
