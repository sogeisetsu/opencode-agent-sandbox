# 为什么装 OMO slim 后 `explore` / `general` 被停用

## 一句话

这不是 OpenChamber 干的，是 **OMO slim 的安装脚本**主动往宿主 `opencode.jsonc` 里写 `disable: true`。

## 机制（源码 `dist/cli/index.js`）

- 常量：`DEFAULT_OPENCODE_AGENTS_TO_DISABLE = ["explore", "general"]`。
- 函数 `disableDefaultAgents()`：对 `opencode.jsonc` 做 `mutateJsonFile`，给每个名字写

  ```jsonc
  "agent": { "<name>": { "disable": true } }
  ```

  已存在的同名对象会被保留其余字段、只补 `disable: true`。写失败会返回 `Failed to disable default agents: ...`。
- 安装流程里对应一步的标签是 `Disabling OpenCode default agents...`。**重跑安装/更新会再次写回停用**，想长期改开就得知道这点。

## 为什么是这两个

- **分工重叠**：OMO 自己的编排分工已覆盖通用委派，宿主的 `general` 是个"野路子"通道——带写权限、跑宿主默认模型、不受 OMO 预设模型链与按代理权限收紧的约束。
- **撞名**：OMO 里有别名归一表

  ```js
  AGENT_ALIASES = { "explore": "explorer", "frontend-ui-ux-engineer": "designer" }
  ```

  合并宿主配置时会把 `agent.explore` 的设置折进 OMO 的 `explorer`，语义容易打架。
- **两者都白占委派预算**（占据编排器名册名额却与自有角色功能重叠）。

## 启用回去的后果（了解即可）

- `general`：开放一条带写权限、不受预设约束的通用通道。
- `explore`：与 `explorer` 别名冲突。
- 而且下次安装/更新，脚本会把它们再停用一遍。

## 另：OMO 可能整份重写它自己的配置文件

`dist/cli/index.js` 的 `writeLiteConfig()` → `generateLiteConfig()` 会**从零生成**一份 `~/.config/opencode/oh-my-opencode-slim.json` 再整体写入（不合并你已有的内容）。也就是说自定义的 `presets` 有被安装/再生成流程覆盖的风险——动手改这份配置前先留一份副本。

## 已验证 / 推测

- **已验证**：`DEFAULT_OPENCODE_AGENTS_TO_DISABLE`、`disableDefaultAgents()` 的写入内容、安装步骤标签、`AGENT_ALIASES` 映射、`writeLiteConfig` 整份重写行为——均读自 `dist/cli/index.js`。
- **推测**：被停用后宿主 UI 是否还有残留入口，未逐个复现。
