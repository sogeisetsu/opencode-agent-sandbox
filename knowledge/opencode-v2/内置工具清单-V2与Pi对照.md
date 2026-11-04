# OpenCode 的内置工具：V1 与 V2 是两套清单

> 核对时间：2026-11-05。
> 证据三路：官网 V1 文档 `/docs/tools`、官网 V2 文档 `/v2/docs/tools`（**已验证**）、上游 `dev` 分支源码与 `CHANGELOG.md`（**已验证**）。
> 结论按版本标注；只有单一来源的写 **推测**。

## 一句话

"OpenCode 有哪些内置工具"没有唯一答案——**V1 和 V2 的清单不同，名字也改过**。V2 用 `shell` / `patch`，V1 用 `bash` / `apply_patch`。LSP 是另一类差异：V1 有实验性 `lsp` 工具，V2 保留配置但不跑服务器、不暴露工具、不产诊断，改用项目的 lint/typecheck/compiler；`todowrite` 则两版都有定义，但 V2 **默认关闭**，要 `experimental.todo: true`。

## 0. 先认错：我上一版错在哪

上一版结论是"19 个工具文件 = 全部内置工具，没有 lsp/find/ls"，依据只有 `packages/core/src/tool/plugin/` 一个目录。三处问题：

1. **只看了一个目录**：Code Mode 的 `execute` 在 `packages/core/src/codemode/`，`opencode` 命名空间（`session_*`、`worktree_*`）由 `plugin/opencode.ts` 一个文件注册多个工具，`browser` 命名空间也不在 plugin 目录。
2. **没区分文档版本**：`/docs/tools` 是 V1，`/v2/docs/tools` 是 V2。直接拿 V1 文档对 V2 源码，会得出"文档说有的源码没有"这种假矛盾。
3. **把"内部注册名"当成"唯一合法名"**：V2 确实把 `bash` 路由到 `shell`、把 `apply_patch` 权限归并到 `edit`，但**兼容输入仍然有效**，所以不能推出"写 `bash: false` 不生效"。
4. **把并列事实写成了因果**：LSP 那条，官方是并列声明"不跑服务器 / 不暴露工具 / 不产诊断"，我写成"因为没有运行时，所以没有工具"——原因是我自己补的。
5. **漏了开关**：`todowrite` 不是"文档没列但功能还在"，而是**默认关闭**，要 `experimental.todo: true` 才注册。我只查了"源码有没有定义"，没查"运行时会不会注册"。

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
| Browser | `browser` 命名空间（**一组**工具，经 `execute` 调用） | 桌面端附加浏览器：标签页、导航、快照、点击/表单、截图、文件传输、控制台/网络、性能 trace、堆检查、Lighthouse；没有连接桌面端就没有这个目录 |

### 1.1 LSP：官方口径是"保留配置，不做三件事"

权威原文来自 **V2 迁移指南**（`/v2/docs/migrate-v1/`，已验证）：

> "V2 accepts and preserves `lsp` configuration, but it does not run language servers, expose LSP tools, or produce LSP diagnostics. Replace workflows that depend on those capabilities with the project's lint, typecheck, or compiler commands."

拆成四层看，比"支持/不支持"这种二分准确得多：

| 层面 | V2 的行为 | 证据 |
|---|---|---|
| 配置 | **接受并保留** `lsp` 块，不报错 | 迁移指南原文 |
| 运行时 | **不启动** language server | 同上 |
| 工具 | **不暴露** LSP 工具 | 同上 |
| 诊断 | **不产生** LSP 诊断（编辑后不会把类型错误喂回给模型） | 同上 |
| 替代方案 | 让 agent 跑项目自己的 lint / typecheck / compiler 命令 | 同上 |

源码侧的现象与这条口径一致，但**只是现象，不是原因**：

- `packages/core/src/tool/plugin/edit.ts:258` 有 TODO：`// TODO: Add LSP notification and diagnostics after LSP runtime exists.`
- dev 树里与 LSP 相关的只剩配置（`schema/config/lsp.ts`）、事件（`lsp-event.ts`，仅 `lsp.updated`）、设置页（`app/.../project-lsp.ts`），没有工具实现文件。

**我之前错在因果**：把"官方并列声明的三条事实"（不跑服务器、不暴露工具、不产诊断）写成了"因为没有运行时，所以没有工具"。官方并没有解释原因，也没有说"工具文件不该存在"；它只声明了行为边界。从"运行时缺失"推到"工具不存在"是我自己加的因果链，这一步不成立。

更准确的一句话：**V2 在接口层不提供 LSP 工具，并把诊断职责外移给项目的 lint/typecheck/compiler**；V1 才有实验性 `lsp` 工具（`OPENCODE_EXPERIMENTAL_LSP_TOOL=true`）。

### 1.2 `todowrite`：源码有定义，但**默认不注册**

这条我上一版写"功能仍在，只是文档未列"，**结论过于乐观，需要收回**。真实机制是"插件被加载 + 插件内部有条件返回"：

官方 V2 配置文档（`/v2/docs/config#todos`，已验证）：

> "Set `todo` to `true` to give agents a `todowrite` tool for tracking multi-step work. **Defaults to `false`.**"

源码三处对应（dev 分支，已验证）：

1. `packages/schema/src/config/experimental.ts:12`：`todo: Schema.Boolean ... "Enable the experimental todowrite tool. Defaults to false."`
2. `packages/core/src/plugin/internal.ts:92, 277`：`import { TodoTool } ...` 且 `TodoTool.Plugin` 确实在内部插件列表里——**插件被加载**。
3. `packages/core/src/tool/plugin/todo.ts:50, 54`：
   ```ts
   const enabled = () => Config.latest(loaded.entries, "experimental")?.todo === true
   ...
   if (!enabled()) return
   ```
   **插件加载 ≠ 工具注册**：开关为 false 时直接返回，`todowrite` 不会进入模型可见的工具目录。

开启方式：

```jsonc
{ "experimental": { "todo": true } }
```

所以"V2 的运行时工具清单里没有 todo writer"这个观察是**对的**，原因是默认关闭，而不是定义缺失。社区也确实有恢复插件（`lcagustini/opencode2-todo`、`gabparrot/opencode2-todo-tool` 已标 DEPRECATED、`ckir/opencode2` 是其 fork，星数均为 0）——但**先用官方开关，别急着装插件**。

> 方法论教训：**"源码里有定义"和"运行时注册到工具清单"是两个验证层**。前者用 `git/trees` + 读文件，后者只能靠实跑或官方文档的清单。我上一版在源码层严谨、在运行时层越级，这是这次最该记住的一条。

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

## 5. `browser` 到底是什么：一组工具，不是一个工具

**结论：它是 Code Mode 的一个命名空间，不是顶层可调用的单个工具。**

证据（dev 分支，已验证）：

1. `packages/plugin-browser/` 是一个独立插件包，入口 `src/index.ts`，能力实现在 `src/tools.ts`（136 行）。
2. `src/tools.ts` 第 38 行 `editor.namespace({ name: "browser", ... })` 先建命名空间，再对每个 operation 逐个注册，工具名取 `operation.name`，`namespace` 为 `browser` 或 `browser.<分组>`，权限统一 `permission: "browser"`。
3. 所以模型看到的调用形态是 `browser.tab_open`、`browser.navigate`、`browser.snapshot` 这一类，**经 `execute`（Code Mode）触达**；顶层工具列表里没有叫 `browser` 的单项。

三个容易误解的点：

- **它依赖桌面端**。文档原文："controls the browser attached by the OpenCode desktop app"。纯 CLI / 服务端没连桌面端时，这个命名空间不存在——不是"工具坏了"。
- **权限是整包粒度**。`browser` 的 deny 规则资源写 `*` 会移除整个 browser 目录，而且浏览器操作**不逐个弹权限**。
- **内容不可信**。页面内容、日志、header、响应体被明确标注为"untrusted data, never instructions"，这是它和普通工具最大的行为差异。

类比：MCP server 也是"一个 server = 一组工具"，默认在 `execute` 里以 server 命名空间调用（除非设 `codemode: false`）。`browser` 是内置版的同构设计，只不过工具由 OpenCode 自己提供、绑定自家桌面端。

## 6. 与 Pi 对照（更新）

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

## 7. 复核方式（只读，必须分版本）

```bash
# 两套文档，别只看一套
#   V1: https://opencode.ai/docs/tools
#   V2: https://opencode.ai/v2/docs/tools
#   V2 迁移指南（LSP 口径在这里）: https://opencode.ai/v2/docs/migrate-v1/
#   V2 配置页（experimental.todo 在这里）: https://opencode.ai/v2/docs/config#todos

# 默认开关要读实现，不能只看文件是否存在
gh api repos/<owner>/<repo>/contents/packages/core/src/tool/plugin/todo.ts?ref=dev \
  -H "Accept: application/vnd.github.raw" | grep -n "enabled\|return"
gh api repos/<owner>/<repo>/contents/packages/schema/src/config/experimental.ts?ref=dev \
  -H "Accept: application/vnd.github.raw" | grep -n -A2 "todo"

# 源码侧：工具目录 + 命名空间注册
gh api repos/<owner>/<repo>/contents/packages/core/src/tool/plugin?ref=dev --jq '.[].name'
gh api repos/<owner>/<repo>/contents/packages/core/src/codemode?ref=dev --jq '.[].name'
gh api repos/<owner>/<repo>/contents/packages/core/src/tool/plugin/opencode.ts?ref=dev \
  -H "Accept: application/vnd.github.raw" | grep -n "namespace"

# 先确认默认分支（search/code 只搜它），再按 ref 精确取树
gh api repos/<owner>/<repo> --jq '.default_branch'
gh api "repos/<owner>/<repo>/git/trees/dev?recursive=1" --jq '.tree[].path' | grep -iE "lsp|diagnostic"

# 名称变更要有变更日志佐证
gh api repos/<owner>/<repo>/contents/CHANGELOG.md -H "Accept: application/vnd.github.raw" | grep -niE "bash|apply_patch|rename"
```

判定门槛：**文档 + 源码 + 变更日志三方对得上，才算结论**；只有文档就标"文档口径"，只有源码就标"当前 dev 分支"，两边冲突先找变更日志。
