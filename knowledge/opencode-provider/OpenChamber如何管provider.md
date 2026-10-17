# OpenChamber 怎么管 provider：Disconnect 只删配置，不动凭据

## 一句话

OpenChamber 的 "Disconnect" **只删配置文件里的块，不吊销 key**；而且凭据归 V2 所有，它想删也删不了（`scope=auth` 直接 409）。

## 配置文件的两种拼写

- v1 写 `provider`，v2 写 `providers` —— 写入时**会迁移到 v2**。
- 实测：本机文件已被 OpenChamber 改写并"膨胀"（663 行，带 29 个模型的目录结构）。

## 配置来源（`configuredIn`）三处

| 来源 | 位置 |
|---|---|
| user | `~/.config/opencode/opencode.json[c]` |
| project | `<工作目录>/opencode.json[c]` 或 `.opencode/` |
| custom | `OPENCODE_CONFIG` 指向的文件（没设就永不显示） |

## 什么时候显示 "Disconnect"

- 判定为**可编辑 provider** 的条件：有 `baseURL` 且命中 `/^https?:\/\//`。
- 命中后详情页出现 **Disconnect**。

## 删除接口与 scope

```
DELETE /api/provider/:id/auth?scope=auth|user|project|custom|all
```

- 默认 `scope=auth`。
- `scope=auth` → 返回 `409 PROVIDER_CREDENTIAL_OWNED_BY_OPENCODE`。
- 理由（接口注释原文大意）：OpenCode 2.x **拥有**凭据、一次性导入 `auth.json`、存在 `/api/credential`；OpenChamber 只能删**配置**（"those files are ours"）。

## 结论

- **Disconnect = 只删配置文件里的 provider 块，不吊销 key。**
- 想真正处理凭据（吊销/换），得走 V2 自己的凭据接口，不是 OpenChamber 的 Disconnect。
- 与 `配置块是覆盖层-删了会怎样.md` 配合看：Disconnect 之后 provider 仍在（内置 catalog），只是回落默认。

## 已验证 / 推测

- **已验证（他次会话实测，本文件未复验）**：v1/v2 拼写与迁移、`configuredIn` 三来源、可编辑判定规则、DELETE 的 scope 取值、`409 PROVIDER_CREDENTIAL_OWNED_BY_OPENCODE` 与注释要点、Disconnect 不吊销 key。
- **推测**：无。
- 时效提醒：接口路径、scope 取值、409 错误码属产品面，会随版本变化。
