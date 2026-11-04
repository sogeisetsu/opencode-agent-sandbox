# WorkBuddy：会话数据布局与上下文压缩

> 核对：2026-11-03，本机 WorkBuddy 5.9.3。示例路径全部替换成 `<drive>` / `<name>`。不摘录会话正文。

## 1. 目录按工作目录编码命名（已验证）

会话记录放在：

```
~/.workbuddy/projects/<encoded-cwd>/<session-uuid>.jsonl
```

编码规则（对 51 个目录逐一与其 `cwd` 字段比对得出）：

1. 盘符转小写；
2. 盘符后的 `X:\` **整体折叠成一个 `-`**（不是冒号、反斜杠各换一个）；
3. 其余 `\` 各换成一个 `-`；
4. 空格保留，中文保留；
5. 去掉末尾分隔符。

例：

| 原始 cwd | 目录名 |
|---|---|
| `D:\<dir>\<sub>` | `d-<dir>-<sub>` |
| `D:\<dir with space>` | `d-<dir with space>` |

按此规则比对 51 个目录：51/51 命中；按「`:`、`\` 各自替换」的写法 0/51。

同一个项目目录下，一个会话一个 `<uuid>.jsonl`；旁边常有 `<uuid>.file-rollback.ndjson`（文件回滚记录），以及以 `<uuid>` 命名的子目录（附件/中间产物）。

`~/.workbuddy/sessions/*.json` 是另一层索引，不是正文；正文在 `projects/` 下的 JSONL。

## 2. 每条记录的结构（已验证）

逐行 JSON。常见 `type`：

| type | 含义 |
|---|---|
| `session-meta` | 会话元信息，通常在文件开头 |
| `message` | 对话消息，带 `role`（`user` / `assistant`） |
| `function_call` | 工具调用 |
| `function_call_result` | 工具结果 |
| `reasoning` | 推理内容 |
| `file-history-snapshot` | 文件历史快照 |
| `ai-title` | 自动生成的标题 |

`message` 的正文在 `content[]` 里，每项有 `type`（如 `input_text`）；另有 `providerData.agent`（本机见到 `cli`，说明桌面会话也是交给 CLI agent 跑的）。

### 注入块

用户消息里夹着宿主生成的 `<system-reminder data-role="user-context">`，内容包括：

- `user_info`（OS、shell、主题、工作区路径）
- `identity_context`（SOUL / IDENTITY / USER 等身份文件）
- `project_context` / `project_layout`
- `connector-status`（各连接器的 connected / disconnected）
- `memory_and_skills_reminder`（记忆与技能写入规则）
- `manually_attached_skills`

批量分析前先剥掉这一段，再按 `role` 取正文，否则统计会被系统提示词污染。

## 3. 怎么识别「未指定工作空间」

两个特征同时成立（**已验证**）：

1. `cwd` 是程序自动创建的临时目录，形如 `<drive>:\AppData\workbuddy\<timestamp>`；
2. 会话在 `workbuddy.db` 的 `sessions` 表里 `is_playground = 1`。

只查文件系统也能看出第一点，但第二点更可靠。本机数据库里 `is_playground` 两种值都存在（有真正的 playground 会话，也有命名工作区）。

只读查询（不写库）：

```powershell
python -c "import sqlite3,os; p=os.path.expanduser('~/.workbuddy/workbuddy.db'); c=sqlite3.connect('file:'+p+'?mode=ro',uri=True); print([r[1] for r in c.execute('pragma table_info(sessions)')]); print(list(c.execute('select is_playground,count(*) from sessions group by is_playground')))"
```

相关列：`sessions.is_playground`、`cwd`、`source_mode`、`is_background_automation`、`expert_id`、`transport`。`workspaces` 表只有 `path` 和 `last_opened_at`，没有 `is_playground`——别在错表上查。

## 4. 上下文压缩：命令、别名、三条触发路径

压缩是控制命令，不是普通技能。打包代码里明确列出：

- 别名：`/compact` 与 `/总结`（同一命令的中英文别名）。
- 同类：`/clear` 与 `/清空`。

三种触发来源（枚举值，**已验证**）：

| 触发 | 枚举 | 说明 |
|---|---|---|
| 手动 | `user-command` | 用户输入 `/compact` 或 `/总结` |
| 发消息前自动 | `pre-message-auto` | 由产品开关 `enablePreMessageCompact` 控制 |
| 超限紧急 | `emergency-auto` | 上下文超限时由 `contextSummaryAgent` 接管 |

压缩结果会替换原有消息，并可能截断工具结果（记录里有 `compactTruncated`、`compactLimitReached`、`compact-cancelled` 等元数据）。

推论（原说法成立）：**长会话里上下文可能被无感知替换**。重要的中间结论、决策理由、验证输出，应该主动写进文件或记忆，而不是只留在会话里。

## 验证记录

```powershell
# 目录编码：对比目录名与首条记录里的 cwd
python -c "import os,json,glob; ..."   # 见正文规则

# 记录类型分布
Get-Content <session>.jsonl | ForEach-Object { ($_ | ConvertFrom-Json).type }

# 注入块
rg -o "system-reminder|__codebuddyLocal" <session>.jsonl

# 压缩枚举与别名
rg -a "PRE_MESSAGE_AUTO|EMERGENCY_AUTO|user-command|\\\\u603B\\\\u7ED3" "<install>/resources/app.asar"
```

未验证：压缩后摘要的具体保留策略（不同模型/窗口大小是否有差异）；`emergency-auto` 的触发阈值。
