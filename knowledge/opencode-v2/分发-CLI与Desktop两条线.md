# OpenCode 的分发：CLI 与 Desktop 是两条独立线

## 一句话

装 CLI 不会带来 Desktop；反过来 Desktop 里**打包了一份完整 CLI**，运行时把它当后台 server 拉起来。

> 草图：[`分发两条线.excalidraw`](./分发两条线.excalidraw)（OpenChamber 的 Files 视图点开就是画布）

## CLI 这条线

安装途径（官方 v2 文档 `docs/` 的 Intro）：

- curl 安装脚本、homebrew（`anomalyco/tap/opencode-v2`）
- npm / bun / pnpm / yarn / vp 上的 `@opencode/cli`、AUR 的 `opencode-beta`、mise（需较新版本）
- 也有各平台独立二进制：macOS（arm64/x64/baseline）、Windows（arm64/x64/baseline）、Linux（glibc 与 musl，各 arm64/x64）

两个坑：

- **Windows 不支持包管理器**（官方原文 "Windows package managers are not supported"），只能下 zip。
- npm 包靠 postinstall 脚本按平台选原生二进制；bun / pnpm 必须显式允许该脚本（`--trust` / `--allow-build`）。

用户侧入口仍是**单个 `opencode` 可执行文件**：`opencode`（TUI）、`opencode run`、`opencode serve`、`opencode pair` 都在它里面。

## Desktop 这条线

- 分发格式：macOS `dmg`、Windows `exe`、Linux `deb` / `rpm` / `AppImage`。
- 包内含一份完整 CLI 二进制：构建脚本把 CLI 产物复制成 `resources/opencode-cli`（Windows 为 `opencode-cli.exe`），打包配置用 `extraResources` 打进安装包。
- 运行时把它 **stage（复制）到用户数据目录** `cli/<version>-<arch>/`，并且**只以 server 方式启动**（sidecar），不做交互式用途。
- 每次更新会 stage 一份新的；现在有清理逻辑（见下）。
- 反向不成立：CLI 安装包里没有 Desktop。

## Windows 只装 Desktop：终端里没有 `opencode`

这是最容易踩的一条：**Windows 单独装了 Desktop，终端里敲不出 `opencode`，也就开不了 TUI。**

机制（源码）：

- Desktop 把内置 CLI **stage 到一个不会进 PATH 的目录**：Windows 上就是 `%APPDATA%\ai.opencode.desktop\cli\<version>-<arch>\opencode-cli.exe`。
  - `userData` 被显式设成 `appData` + `APP_ID`；prod 的 `APP_ID` 就是 `ai.opencode.desktop`。
- 它只用这份二进制**当 server 拉起**，既不写进 PATH、也不建命令别名 → 终端找不到 `opencode`。（"没进 PATH"来自：staging 代码里没有任何 PATH / 别名操作。）
- 唯一会把 CLI 装到 PATH 的动作（`install`）**硬 gate macOS**：`process.platform !== "darwin"` 直接失败。macOS 那条路是跑仓库里的 `install` 脚本（`bash -s -- --binary <bundled>`），装到 `~/.opencode/bin/opencode`，默认还会改 shell 配置（`--no-modify-path` 可关）—— 所以 macOS 装完 Desktop 终端能用，Windows 不行。
- Windows 又**不支持包管理器** → 想在 Windows 终端用 `opencode`，只能**自己下 CLI 的 zip，手动加进 PATH**。

- **macOS 没有这个问题**：那份 CLI 进了 `~/.opencode/bin` 而且在 PATH 里，装完提示"重启终端"之后终端就能用 `opencode`。所以"装了 Desktop 也用不了终端 TUI"是 **Windows 专属**的坑。

结论：Windows 上"装了 Desktop" ≠ "装了 CLI"；这个"安装 CLI"入口只在 macOS 存在（Windows / Linux 根本没有）。

> 这个 Windows 短板**已提交 issue**：`#55772`（"Windows desktop ships the CLI but there is no way to run it"，open）。可能某天会修 —— 一旦修好，本节和两张草图都要更新。

## staged CLI 目录会清理

- 旧行为曾是：每次更新 stage 一份、不清理，长期积累占用可观。
- 现在有 `cleanStages`：删除非当前 stage 目录；且 stage 目录按 `<version>-<arch>` 分（x64 与 ARM64 在同一 userData 下要分开）。
- Windows 上删不掉"尚未退出的 exe"会记日志，下次启动重试（源码注释里明写）。

## 已验证 / 随时间变化

- 已验证（官方文档）：安装途径、Windows 无包管理器、Desktop 分发格式。
- 已验证（源码）：内含 CLI 二进制、stage 到 userData、只当 server、macOS gate、`cleanStages`。
- **随时间变化**：仓库包结构与 stage 具体行为迭代很快，引用前以当前源码为准。
