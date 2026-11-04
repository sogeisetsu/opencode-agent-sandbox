# PowerShell 补全与提示符：PSReadLine 能做什么、做不到什么

> 核对时间：2026-11-05，本机 Windows + PowerShell 7.6.9 + PSReadLine 2.4.5。
> 文中 **已验证** 指本机实跑或读公开文档确认；**推测** 指只有推理，或依赖特定版本行为。

## 一句话

PowerShell 想获得接近 zsh 的交互体验，靠的是 **PSReadLine**（编辑层）+ **Oh My Posh**（渲染层）两件独立的东西；zsh 本身**不能**"适配"进 PowerShell，它们是两个互不兼容的 shell。

## 1. 三个概念先分清

| 层 | 负责什么 | 代表 |
|---|---|---|
| Shell / 解释器 | 解析语法、执行命令、脚本语义 | bash、zsh、PowerShell |
| 行编辑层 | 键位、历史、补全菜单、预测文本、语法高亮 | PSReadLine、zsh 的 ZLE |
| 提示符渲染 | 把当前目录、Git 状态、耗时画成提示符 | Oh My Posh、Starship、Powerlevel10k |

关键点：**换渲染器不会让你得到 zsh 的语法**。Oh My Posh 只画提示符，不执行命令，也不是 WebUI。它官方的 Studio / Visual Configurator 是主题编辑器，同样不执行命令。

## 2. 本机实测的默认状态（已验证）

```
PSVersion                       7.6.9
PSReadLine                      2.4.5
PredictionSource                HistoryAndPlugin
PredictionViewStyle             InlineView
EditMode                        Windows
MaximumHistoryCount             0            # 0 = 不限制
Tab                             TabCompleteNext
Ctrl+Spacebar                   MenuComplete
AppsUseLightTheme               1
```

由此可以推出两条与常见建议冲突的结论：

- **`Tab` 默认是"逐项循环"（TabCompleteNext），不是菜单补全**。改成 `MenuComplete` 是有效改动，但会放弃"按一次 Tab 补一段"的手感。
- **`Ctrl+Spacebar` 已经是 `MenuComplete`**（PSReadLine 自带默认键位），再绑一次是冗余；真正需要绑的是 Tab。

## 3. 一份可用的配置（建议，未逐条写入验证）

```powershell
# 历史预测 + 列视图
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView

# Tab 改成菜单补全
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete

# Emacs 键位（Ctrl+A/E 行首行尾、Ctrl+W 删词等）
Set-PSReadLineOption -EditMode Emacs

# 历史条数
Set-PSReadLineOption -MaximumHistoryCount 10000
```

注意几处容易踩的：

- `-PredictionSource History` 只吃历史；本机默认是 `HistoryAndPlugin`（含插件提供的预测源），改窄会丢插件建议。
- `ListView` 的预测文本颜色和 `InlineView` 不是同一个键，**不要以为设一个就两处都变**。
- `-MaximumHistoryCount 0` 表示不限制；设成 10000 是主动限量。
- 配置文件是 `$PROFILE`，不是 `$PROFILE.CurrentUserAllHosts` 的替代品；改错文件会出现"VS Code 终端生效、控制台不生效"。

## 4. 幂等写入：用标记块

直接 `Add-Content` 会在每次执行时重复追加。稳定写法是把配置夹在标记之间，先删旧块再写：

```powershell
$marker = '# >>> psreadline >>>'
if (-not (Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force | Out-Null }
Copy-Item $PROFILE "$PROFILE.bak-$(Get-Date -Format yyyyMMddHHmmss)" -Force

$content = Get-Content $PROFILE -Raw -ErrorAction SilentlyContinue
if ($content) {
  $content = [regex]::Replace($content, "(?ms)^$([regex]::Escape($marker)).*?^# <<< psreadline <<<\r?\n?", '')
}
$block = @"
$marker
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
# <<< psreadline <<<
"@
Set-Content -Path $PROFILE -Value ($content + $block) -Encoding utf8
```

要点：**先备份**、**标记块前后成对**、**写 UTF-8**（中文注释在 GBK 控制台下会乱码）。

## 5. 按主题自适应预测文本颜色

思路：读注册表 `HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize` 的 `AppsUseLightTheme`（1 = 浅色，0 = 深色），再决定颜色。

```powershell
$light = (Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize' -Name AppsUseLightTheme).AppsUseLightTheme -eq 1
$color = if ($light) { '#666666' } else { '#888888' }   # 颜色值为建议，非官方推荐值
Set-PSReadLineOption -Colors @{ InlinePrediction = $color }
```

- `AppsUseLightTheme = 1` 是浅色主题（已验证，本机返回 1）。
- **推测**：`InlinePrediction` 是 InlineView 预测文本的颜色键；ListView 是否共用同一键未在本机逐项验证，切换视图后应实测确认。
- 颜色只是可读性调整，不影响功能；对比度以实际终端背景为准，别照抄十六进制值。

## 6. 验证方式（只读）

```powershell
$PSVersionTable.PSVersion
Get-Module PSReadLine -ListAvailable | Sort-Object Version -Descending | Select-Object -First 1
Get-PSReadLineOption | Select-Object PredictionSource, PredictionViewStyle, EditMode, MaximumHistoryCount
Get-PSReadLineKeyHandler -Key Tab
Get-PSReadLineKeyHandler -Chord 'Ctrl+Spacebar'
```

改完配置后重开一个终端再跑一遍，确认没有被其他 profile 脚本覆盖。

## 7. 想更接近 zsh

- **要真 zsh**：装 WSL，发行版里 `apt install zsh` + `chsh -s $(which zsh)`，配 Oh My Zsh。Windows 侧的 PowerShell 不受影响。
- **只要模糊历史**：PSFzf（`Install-Module PSFzf`）把 `Ctrl+R` 换成 fzf 的模糊搜索，比 PSReadLine 自带的逐条反向搜索好用。
- **只要好看**：Oh My Posh 装完 `oh-my-posh init pwsh | Invoke-Expression`，主题用于渲染，不改变 shell 语义。
