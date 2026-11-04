# Git Bash 是 MINGW64 + MSYS2，不是 Linux

> 核对时间：2026-11-05。结论来自本机实跑 `uname` 与安装目录结构，**已验证**。

## 一句话

MINGW64 是**在 Windows 上编译 Windows 程序的 GCC 工具链**；Git Bash 里能敲 `ls`、`grep`、`sed`，靠的是捆绑进来的 **MSYS2** 兼容层（把这些工具编成 Windows 程序并模拟 POSIX 行为），不是因为跑了个 Linux。

## 1. 实测输出

```console
$ uname -a
MINGW64_NT-10.0-26220 <host> 3.6.9-b4195d69.x86_64 2026-06-06 17:49 UTC x86_64 Msys

$ echo $MSYSTEM
MINGW64
```

注意 `uname` 的机器类型写的是 **Msys**，不是 `Linux`。这一行就把"这是 Linux 环境"的说法否掉了。

再看解释器自身路径（WorkBuddy 桌面内置终端）：

```console
$ command -v bash
/c/Users/<user>/.workbuddy/binaries/PortableGit/versions/<ver>/bin/bash.exe
```

`bash.exe` —— 是个 Windows 可执行文件，来自 PortableGit 包。

## 2. 三个名字的关系

| 名字 | 是什么 | 容易误解的点 |
|---|---|---|
| **MinGW / MINGW64** | Minimalist GNU for Windows，提供 `gcc`、`ld`、`make` 等，产出 **PE/Windows 程序** | 以为它是 Linux 环境；实际编译目标是 Windows |
| **MSYS2** | 在 Windows 上模拟 POSIX 的运行时与包管理（`pacman`），提供 `/c/...` 路径映射、fork/pty 等 | 以为 MSYS2 = MinGW；两者常常一起装，但职责不同 |
| **Git Bash** | 一个 MSYS2 环境 + Git for Windows 的 shell 启动器 | 以为它自带完整 POSIX；很多 Linux 专有命令、`/proc`、systemd 都没有 |

## 3. 会踩的地方

- **跑不了 Linux ELF 二进制**。下载一个 Linux 版 `ripgrep` 放进 PATH，会报"无法执行"或直接被当数据文件；要装 Windows 版。
- **路径是 `/c/Users/...`**，不是 `/home/...`。脚本里写死 `/home` 到 Windows 侧会落到 Git 安装目录下的假 home。
- **换行与编码**：`git config core.autocrlf`、终端代码页（`chcp 65001`）都会影响中文与脚本；中文文件名在 `git status` 里显示成八进制转义是正常的，不是乱码。
- **符号链接需要开发者模式或管理员权限**，否则 `ln -s` 退化成复制。
- **`sudo` 不存在**；提权要靠另开管理员终端（本项目有 `windows-elevated-shell` 技能处理这个）。
- **性能**：`/c/...` 下大量小文件操作比 WSL 内部文件系统慢，跨盘 `git status` 尤其明显。

## 4. 与 WSL 的分界

| 维度 | MINGW64 / Git Bash | WSL 2 |
|---|---|---|
| 执行什么程序 | Windows 程序（PE） | Linux 程序（ELF） |
| 内核 | Windows 内核 | 真实 Linux 内核（轻量 VM） |
| 典型用途 | Git 操作、编译 Windows 目标、跑 shell 脚本 | Linux 工具链、容器、需要 POSIX 语义的服务 |
| 文件系统 | 直接访问 NTFS（`/c/...`） | ext4 虚拟磁盘，`/mnt/c` 跨系统访问较慢 |
| 隔离性 | 无，和普通 Windows 进程同级 | 一层 VM 隔离，可再叠加容器 |

**结论**：Git Bash 适合"在 Windows 上顺手敲几条 Unix 命令"，不适合当 Linux 开发环境。需要后者就上 WSL，并把项目放进 WSL 自己的文件系统（见 `WSL-磁盘迁移与容器工具.md`）。

## 5. 验证方式（只读）

```bash
uname -a
echo "$MSYSTEM"
command -v bash
file "$(command -v bash)"     # 若可用，会显示 PE32+
```

看到 `Msys` / `MINGW64` / `PE32+` 任一条，就说明当前不是 Linux。
