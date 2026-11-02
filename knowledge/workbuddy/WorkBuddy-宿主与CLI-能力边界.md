# WorkBuddy：宿主与同源 CLI 的能力边界

> 核对：2026-11-03，本机 WorkBuddy Desktop（genie 5.9.3）+ `codebuddy` CLI 2.176.0。安装目录用 `<install>` 代替，用户目录用 `~` 代替。标 **推测** 的没实证。

## 一句话

桌面和 CLI 共用一套账号与产品定义，但**数据目录、技能目录、记忆文件、连接器配置各自独立**。在一边装的东西，另一边默认看不到。

## 实测：两侧目录对比

```powershell
$h="$env:USERPROFILE\.workbuddy"; $c="$env:USERPROFILE\.codebuddy"
foreach($p in @("$h\skills","$c\skills","$h\MEMORY.md","$c\MEMORY.md","$h\connectors","$c\connectors","$h\mcp.json","$c\mcp.json")){
  "$([bool](Test-Path $p))`t$p"
}
```

本机结果（**已验证**）：

| 路径 | 宿主 `~/.workbuddy` | CLI `~/.codebuddy` |
|---|---|---|
| `skills/` | 存在 | 不存在 |
| `MEMORY.md` | 存在 | 不存在 |
| `connectors/` | 存在 | 不存在 |
| `mcp.json` | 存在 | 不存在 |
| `projects/`、`sessions/` | 存在 | 存在（各自一套） |
| `models.json` | 存在 | 存在（各自一套） |

两侧都有 `projects/`、`sessions/`、`logs/`，但内容是分开的。CLI 目录里没有 `skills`、`MEMORY.md`、`connectors`，所以「宿主里装了技能，CLI 就能用」不成立。

**已验证**：`codebuddy --version` 返回 `2.176.0`；usage 第一行是 `codebuddy|cbc`，说明还有 `cbc` 别名。

## 纠正：「CLI 不是另一套二进制」

原说法是「CLI 访问支持是账号权益，不是另一套二进制」。后半句在本机**不成立**：

- `codebuddy` 是独立的 npm 包，命令入口在 `~/AppData/Roaming/npm/codebuddy.ps1`。
- 桌面应用自己也带一份 CLI：`<install>/resources/app.asar.unpacked/cli/`（含 `product.json`、`dist/`、`bin/`）。

所以更准确的表述：**CLI 是一件独立的可执行物，账号权益只决定它能不能用同一个身份跑。** 权益这件事本身没有直接观测到（只看到通用的 entitlement 网关 `[WBIPC][entitlement] ... allowed=`），标 **推测**。

判断能力是否打通的实用方法（原说法里的做法，**已验证**）：比两侧的技能目录、记忆文件、连接器配置是否可见。看不到就是没打通，不用猜账号。

## 内置工具 ≠ 技能

两者在磁盘上的形态不同：

- **技能**：有独立目录 + `SKILL.md` 正文。宿主用户级在 `~/.workbuddy/skills/<name>/SKILL.md`；内置技能在 `~/.workbuddy/plugins/cache/workbuddy-builtin/skill-*/`。
- **工具型插件**：没有 `SKILL.md`，只有 `.codebuddy-plugin/`、`.mcp.json`、`dist/`。本机 `mcp-miora` 就是这种，递归搜索 `SKILL.md` 数量为 0（**已验证**）。

判断某项能力能不能替换/迁移：先看有没有 `SKILL.md`。有就能按技能改写；没有就是宿主/插件暴露的工具，改配置或换插件，不要去找正文。

## 输出形态由场景 + 技能正文决定

在会话记录里能看到用户输入带 `@scene#<id>:"<场景名>"`。宿主的组合层把它展开成：

- `sceneId` / `microScenePluginIds`
- `capsulePluginId`（场景胶囊插件）
- `cliComposeEnabled`（是否走 CLI 组合）

这些字段在打包后的 `app.asar` 里能 grep 到（**已验证**）。场景本身来自产品配置/云端，不在本地某个 `scenes.json` 里。

输出成页面而不是聊天文本，通常不是模型临时发挥，而是：

1. 场景入口把任务路由给某个技能；
2. 技能正文写明「按 workflow 文件执行」；
3. workflow/模板文件规定产物形态。

本机 `ardot-design-to-code/SKILL.md` 就是这种写法：正文只列 workflow 路径（`workflows/design-to-code-workflow.md`），实际步骤和产物格式在 workflow 里（**已验证**）。

排查「为什么总是这个输出形态」的顺序：场景配置 → 技能正文 → 正文引用的 workflow/模板文件。

## 验证记录

```powershell
codebuddy --version
Test-Path ~/.workbuddy/skills, ~/.codebuddy/skills
Get-ChildItem ~/.workbuddy/plugins/cache/workbuddy-builtin/mcp-miora -Recurse -Filter SKILL.md   # 0 个
rg -a "microScenePluginIds|capsulePluginId|scene://" "<install>/resources/app.asar"
Get-Content ~/.workbuddy/projects/<encoded>/<session>.jsonl   # 首条 user 消息里可见 @scene#...
```

没有验证的：账号权益的具体判定规则、场景配置的云端下发格式。
