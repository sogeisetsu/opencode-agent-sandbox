# OpenCode 的内置工具：V1 与 V2 是两套清单

> 核对时间：2026-11-05。
> 证据三路：官网 V1 文档 `/docs/tools`、官网 V2 文档 `/v2/docs/tools`（**已验证**）、上游 `dev` 分支源码与 `CHANGELOG.md`（**已验证**）。
> 结论按版本标注；只有单一来源的写 **推测**。

## 一句话

"OpenCode 有哪些内置工具"没有唯一答案——**V1 和 V2 的清单不同，名字也改过**。V2 用 `shell` / `patch`，V1 用 `bash` / `apply_patch`；`lsp` 只存在于 V1 文档，V2 文档与源码都没有。

## 0. 先认错：我上一版错在哪

上一版结论是"19 个工具文件 = 全部内置工具，没有 lsp/find/ls"，依据只有 `packages/core/src/tool/plugin/` 一个目录。三处问题：

1. **只看了一个目录**：Code Mode 的 `execute` 在 `packages/core/src/codemode/`，`opencode` 命名空间（`session_*`、`worktree_*`）由 `plugin/opencode.ts` 一个文件注册多个工具，`browser` 命名空间也不在 plugin 目录。
2. **没区分文档版本**：`/docs/tools` 是 V1，`/v2/docs/tools` 是 V2。直接拿 V1 文档对 V2 源码，会得出"文档说有的源码没有"这种假矛盾。
3. **把"内部注册名"当成"唯一合法名"**：V2 确实把 `bash` 路由到 `shell`、把 `apply_patch` 权限归并到 `edit`，但**兼容输入仍然有效**，所以不能推出"写 `bash: false` 不生效"。

这三个坑正好对应 `源码对照与旧结论纠错.md` 里那两条教训：**在一个包里搜不到 ≠ 不存在**；**否定判断有保质期**。

## 1. V2 官方清单（`/v2/docs/tools`，已验证）

| 分类 | 工具 | 关键说明 |
|---|---|---|
| Files | `read` | 读文件或**非递归目录列表**；`offset`/`limit` 分页；单页上限 2000 行 / 50 KiB；支持 PNG/JPEG/GIF/WebP/PDF |
| Files | `glob` | 按模式找路径；默认上限 100 个文件、超时 30 秒 |
| Files | `grep` | 正则内容搜索；默认 100 行、超时 30 秒；权限资源是 **pattern** 不是路径 |
| Files | `edit` | `oldString` → `newString`，默认要求唯一匹配，`replaceAll` 可放开 |
| Files | `write` | 新建或整体覆盖；自动建父目录，可能触发格式化 |
| Files | `patch` | `patchText` 支持 Add/Update/Move/Delete 多文件；**只对受支持的 GPT 模型暴露**，其他模型拿 `edit` + `write` |
| Commands | `shell` | 在宿主 shell 执行；`workdir`、`timeout`、`background`；拥有宿主的文件/进程/网络权限 |
| Web | `webfetch` | 单个 URL 取 markdown/text/html，最长 120 秒；不处理图片等二进制 |
| Web | `websearch` | 联网搜索；provider 与密钥见第 3 节 |
| Images | `experimental_image_generate` | 实验性；连接的 provider 有图像模型时出现 |
| Interaction | `question` | 暂停执行向用户提问，支持多问题与自由输入 |
| Interaction | `skill` | 按 skill ID 加载 `SKILL.md` 及随包资源 |
| Automation | `subagent` | 起子会话；前台等待，`background: true` 立即返回；默认嵌套深度 1 |
| Automation | `execute` | Code Mode 里跑 JavaScript，组合调用目录中的工具 |
| Automation | `opencode` 命名空间 | `session_rename`、`session_move`、`worktree_create`、`worktree_list`、`worktree_remove` |
| Browser | `browser` 命名空间 | 桌面端浏览器：开标签页、快照、点击、截图、控制台/网络、性能 trace、Lighthouse |

**V2 文档里没有 `lsp`，也没有 `todowrite`。** 前者源码里同样没有实现文件；后者源码有（`todo.ts`，注册名 `todowrite`），但 V2 文档未列——**推测**：功能仍在，只是未被文档承诺，升级后可能变。

## 2. V1 官方清单（`/docs/tools`，已验证）

`bash`、`edit`、`write`、`read`、`grep`、`glob`、`lsp (experimental)`、`apply_patch`、`skill`、`todowrite`、`webfetch`、`websearch`、`question`。

其中 `lsp` 的原文条件是：**仅当 `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` 或 `OPENCODE_EXPERIMENTAL=true` 时可用**，支持 `goToDefinition`、`findReferences`、`hover`、`documentSymbol`、`workspaceSymbol`、`goToImplementation`、`prepareCallHierarchy`、`incomingCalls`、`outgoingCalls`。

## 3. 三处名字/条件差异（最容易踩）

| 项 | V1 | V2 | 怎么办 |
|---|---|---|---|
| shell | `bash` | `shell` | 写 V2 配置用 `shell`；旧配置里的 `bash` 会被路由（见下），不会报错 |
| 补丁 | `apply_patch` | `patch` | `apply_patch` 的权限已迁移到 `edit`；hook 里判断工具名要按目标版本写 |
| 搜索密钥 | 文档称**无需 API key**（走 OpenCode / OpenCode Go 托管服务，或 `OPENCODE_ENABLE_EXA` / `OPENCODE_ENABLE_PARALLEL` 开启） | **五个 provider 各自要 key**：`EXA_API_KEY`、`FIRECRAWL_API_KEY`、`PARALLEL_API_KEY`、`TAVILY_API_KEY`、`TINYFISH_API_KEY`（可选） | 别把 V1 的"免费"套到 V2；V2 要 `/connect` 或设环境变量 |

CHANGELOG 两条直接证据（**已验证**）：

- `#60687`：**"Calls to a tool named `bash` now run as `shell` instead of failing, with the same permissions. A plugin tool actually named `bash` still takes precedence."**
- `#57900`：**"Legacy `apply_patch` permissions now migrate to the `edit` permission."**

所以正确说法是：**V2 的内部规范名是 `shell` / `patch`，但 `bash` 仍是合法调用（会被路由过去，权限相同）**。我上一版"写 `bash: false` 不会关掉它"是错的——它会被规范到 `shell` 再判定。

## 4. 源码侧：19 个文件不等于 19 个工具

`packages/core/src/tool/plugin/` 下 19 个模块，能取到顶层注册名的：

| 文件 | 注册名 |
|---|---|
| `shell.ts` / `patch.ts` | `shell` / `patch` |
| `read.ts` / `write.ts` / `edit.ts` | `read` / `write` / `edit` |
| `grep.ts` / `glob.ts` | `grep` / `glob` |
| `todo.ts` | `todowrite` |
| `question.ts` / `skill.ts` / `subagent.ts` | `question` / `skill` / `subagent` |
| `webfetch.ts` / `websearch.ts` | `webfetch` / `websearch` |
| `image-generate.ts` | `experimental_image_generate` |

没有顶层 `name` 的文件（`artifact`、`file-diff`、`mcp-resource`、`opencode`、`schedule`）不代表"没有注册"——`opencode.ts` 一个文件就注册了 `opencode` 命名空间下的多处工具（第 113–227 行，`draft.namespace({...})` + `options: { namespace: "opencode", codemode: true }`）。**一个模块可以注册多个工具，所以"文件数"和"工具数"不能互相换算。**

`effectiveName()`（`packages/core/src/tool/runtime.ts:290`）只做两件事：把非法字符替换成 `_`、拼上 namespace 前缀。**它不做 `bash → shell` 的映射**——那个映射在 TUI 显示层（`packages/tui/src/util/tool-display.ts` 的 `canonicalToolName`）与兼容调用路径里。

## 5. 与 Pi 对照（更新）

| 维度 | OpenCode V2 | Pi |
|---|---|---|
| 默认工具 | 见第 1 节，开箱即用 | `read` / `bash` / `edit` / `write` |
| 目录读取 | `read` 直接支持非递归目录列表 | 需要 `ls`（默认关闭，`--tools` 开） |
| 额外内置 | 无（都在默认集） | `powershell`、`grep`、`find`、`ls`，默认关闭 |
| 工具白名单 | 配置里按 agent 声明权限 | `--tools a,b,c`（覆盖默认集）、`--exclude-tools`、`--no-tools` |
| 联网搜索 | 内置 `websearch`（V2 需 provider key） | 无内置，靠扩展或外部 CLI |
| 子代理 | 内置 `subagent` | 无内置，靠扩展 |
| Skill | 内置 `skill` 工具加载 `SKILL.md` | 有 skill 机制（描述常驻、正文按需），但不是可调用的工具 |
| 代码执行 | `execute`（Code Mode，跑 JS 组合调用工具） | 无对应内置；靠 `bash` 或扩展 |
| 哲学 | 框架给能力，配置约束 | 内核极简，扩展装出来 |

## 6. 复核方式（只读，必须分版本）

```bash
# 两套文档，别只看一套
#   V1: https://opencode.ai/docs/tools
#   V2: https://opencode.ai/v2/docs/tools

# 源码侧：工具目录 + 命名空间注册
gh api repos/<owner>/<repo>/contents/packages/core/src/tool/plugin?ref=dev --jq '.[].name'
gh api repos/<owner>/<repo>/contents/packages/core/src/codemode?ref=dev --jq '.[].name'
gh api repos/<owner>/<repo>/contents/packages/core/src/tool/plugin/opencode.ts?ref=dev \
  -H "Accept: application/vnd.github.raw" | grep -n "namespace"

# 名称变更要有变更日志佐证
gh api repos/<owner>/<repo>/contents/CHANGELOG.md -H "Accept: application/vnd.github.raw" | grep -niE "bash|apply_patch|rename"
```

判定门槛：**文档 + 源码 + 变更日志三方对得上，才算结论**；只有文档就标"文档口径"，只有源码就标"当前 dev 分支"，两边冲突先找变更日志。
