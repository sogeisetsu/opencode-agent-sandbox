# 凭据只有一份：opencode-go 与 opencode 是同一个认证身份

## 一句话

DB 里查不到"opencode-go 的 key"**不是异常，是设计**——`opencode-go` 和 `opencode` 声明同一个环境变量 `OPENCODE_API_KEY`，是同一个认证身份，所以只有一行凭据。

## 证据链

1. 运行实例 `/api/provider` 返回：

   ```json
   {"id":"opencode-go","integrationID":"opencode",
    "settings":{"baseURL":"https://opencode.ai/zen/go/v1"}}
   ```

   `integrationID` 指向 `opencode`，不是自己。

2. 两个 catalog 条目的 `env` 都是 `["OPENCODE_API_KEY"]` → 同一个认证身份（见 `provider三层来源与内置catalog.md`）。
3. 因此 `auth.json` 里 `opencode.key` 与 `opencode-go.key` **同值**；DB 的 credential 表**只有一行**，`integration=opencode`。

## V2 不读 auth.json

- OpenCode 2.x 自己拥有凭据，`auth.json` 只在**一次性迁移**时被导入。
- 实测迁移记录：`20260805200742_import_legacy_credentials`，一次性执行，同一秒创建 3 行——`auth.json` 里 4 条记录因为两组同值被合并成 3 行。
- 该行后来从静态 key（`active=0`）切到 OAuth：

  | 字段 | 值 |
  |---|---|
  | `label` | `Default` |
  | `type` | `oauth` |
  | `access` / `refresh` | 各 39 字符（不落盘、不打印） |
  | `expires` | 一次性日期戳 |
  | `metadata.server` | `https://opencode.ai/console` |

- `deepseek` / `longcat` 也迁移进来了，但 `active=None`，且配置里有 `disabled_providers: ["deepseek","longcat"]` —— 迁移"带进来"不等于"启用"。

## 推论

- 删/改 `opencode.jsonc` 里的 provider 配置块**碰不到凭据**（凭据在 DB，配置在文件，两者不同层）。
- "某 provider 没有独立 key"这类疑问，先看 `integrationID` 指向谁，别急着当 bug 查。

## 已验证 / 推测

- **已验证（他次会话实测，本文件未复验）**：`/api/provider` 的 `integrationID=opencode`、共用 env、`auth.json` 同值、credential 表单行、迁移名字与一次性语义、静态 key→OAuth 的字段、`disabled_providers` 与 `active=None` 的对应。
- **推测**：无。
- 时效提醒：`integrationID`、迁移名、OAuth 元数据都是具体快照，会随版本/时间变化。
