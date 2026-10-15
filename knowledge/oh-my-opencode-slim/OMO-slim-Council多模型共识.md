# OMO slim 的 Council：多模型共识是怎么跑起来的

> 流程图：[`Council-派发流程.excalidraw`](./Council-派发流程.excalidraw)（OpenChamber 的 Files 视图点开就是画布）

## 一句话

Council 是 OMO slim **内置**的一套"多个异构模型各自独立作答、再由一个合成器收敛"的机制；它由 **orchestrator 派发**，不是自己去跑。

## 归属与启用条件

- 归属：OMO slim 内置，非 OpenCode 本体（见 `OMO-slim-定位与角色名册.md` 的验证方式）。
- **启用条件：必须存在 `council` 配置块。** 源码 `createAgents`：

  ```js
  if (!runtime.council) { disabled.add("council"); disabled.add("councillor"); }
  ```

  所以"没有 `council` 配置" = Council 未启用（`councillor` 也一并停）。
- `council` / `councillor` 的停用关系：若 `council` 被停用，则 `councillor` 强制一起停。
- 不可停用性的边界：`councillor` 属于 `PROTECTED_AGENTS`，`disabled_agents` 里写它会**被过滤掉**；但 `council` 本身**不在**保护名单，可以用 `disabled_agents: ["council"]` 停掉。

## 配置形态

```jsonc
{
  "council": {
    "default_preset": "default",            // 省略时默认 "default"
    "presets": {
      "default": {
        "alpha": { "model": "provider/model-a" },
        "beta":  { "model": "provider/model-b", "variant": "high" },
        "gamma": { "model": ["provider/model-c", "provider/fallback"] },  // 数组 = 该席位兜底链
        "delta": { "model": "provider/model-d", "prompt": "自定义席位提示词" }
      }
    }
  }
}
```

- 席位字段：`model`（单字符串或数组兜底链，数组元素可为 `string` 或 `{ id, variant }`）、`variant?`、`prompt?`。
- **旧式嵌套 `councillors` 键仍支持**（会被拍平）；`master` 键被**静默忽略**（历史遗留）。
- 取哪个预设：`council.default_preset`（缺省 `"default"`）。**该预设不存在时 `buildCouncillorAgents` 返回 `[]`——一个席位都生不出来。**
- Council 合成器本身的模型走常规 `council` 角色配置（即 `agents.council.model` 一类），未配时回落到 `primaryModel`（会话模型）。

## 席位（councillor）行为

- 运行时按席位展开成 `councillor-<席位名>`（源码里 `COUNCILLOR_AGENT_PREFIX = "councillor-"`）。
- 权限：`createReadOnlyAgentPermission()` —— 允许 `read` / `glob` / `grep` / `lsp` / `list` / `codesearch` / `ast_grep_search`；其余一律 deny（含 `bash` / `edit` / `write` / `apply_patch` / `task` / `question`）。
- 结果：**只读访问代码库、不能跑 shell、不能委派子 agent**。它们是顾问，不是实现者。
- 合成器 `council` 是 **synthesis-only**：`createSynthesisOnlyPermission()` 全 deny（连 read 都没有），因为它只负责把各席位的回答收敛成报告。
- **席位互相看不到对方的答案**——这份独立性正是 Council 的价值来源。

## 触发

- Hook：`experimental.chat.messages.transform`（`createCouncilInjectHook`，hook 名 `council-inject`）。
- 命中才注入一段 "## Council Mode" 流程纸。生效条件：
  1. 当前 agent **必须是 `orchestrator`**；
  2. 消息**不以斜杠命令开头**；
  3. 匹配**先剥掉代码块与行内代码**（写在 ``` 或 `` ` `` 里不算）；
  4. **每个会话只注入一次**。
- 触发词正则 `COUNCIL_TRIGGER_PATTERN` 覆盖中英日韩阿多语言，含：`council(s)` / `councillor(s)` / `consensus` / `second opinion(s)` / `roundtable` / `multiple models` / `multi-model` / `multi-agent` / `panel(s)` / `diverse perspectives` / `sounding board`，以及 `议会` `圆桌` `共识` `第二意见` `多模型` `多代理` `多智能体` …等。

## 谁真正派发：是 orchestrator，不是 council

- 触发的本质：给你的**编排器**递一张流程纸（"把问题并行发给每个席位、再调 `council` 合成"），由 orchestrator 用它的 subagent 工具逐个派发。Council 自己不派发席位。
- 其他子 agent 调不到 Council：非 orchestrator 代理的 `task` 权限被设成 `{"*":"deny"}`；顾问型子 agent 更进一步是 `TASK_OBSERVER_DISPATCH_ONLY = {"*":"deny", observer:"allow"}`（即只能调度 observer）。
- **这不是硬锁**：orchestrator 本来就有 subagent 工具、council 也在其名册内；"用户明确要求共识"是提示词层面的约束，不是权限层面的。

## 产出报告结构

合成器被要求产出：

- `## Council Response` —— 收敛后的最终答案；
- `## Per-Councillor Details` —— 按**席位名**（如 `alpha`，不是模型名）逐条列，失败/超时的席位要标注而不是省略；
- `## Council Summary` —— `Consensus Level: unanimous | majority | split`、Agreed Points、Disagreements + resolution、Remaining Uncertainty、Recommended Action。

## 成本

- 路由块自标：**约 3× 慢于 orchestrator、≥3× orchestrator 成本**。
- 但**不触发时零成本**：完整流程不会静态挂在提示词里，只多一行路由说明；只有命中触发词才注入那张纸。
- 席位模型未配 variant 时，数组兜底链里没有自带 variant 的项会落到共享的 `fallbackVariant`。

## 关掉自动触发

`disabled_hooks: ["council-inject"]` → 关键字触发不再注入，之后只能**手动点名** `@council`。注意 `council` / `councillor` **不能**用 `disabled_agents` 成对停掉（`councillor` 受保护，只有 `council` 可停）。

## 已验证 / 推测

- **已验证**：启用条件、配置结构与字段、`master` 忽略、取预设逻辑与"缺预设返回 `[]`"、席位只读权限集、合成器全 deny、触发 hook 的全部条件与正则、`TASK_OBSERVER_DISPATCH_ONLY`、报告结构、3× 成本标注、`council-inject` 关闭方式——均读自 `dist/server/index.js`。
- **推测/未验证**：合法 JSONC 配置块落盘后是否一定生成 `councillor-*`，以及 `agents.council.model` 与预设内 `council.model` 同时存在时的最终胜出者；建议落地后重载插件、看是否出现 `councillor-*`、再发一条带触发词的消息做冒烟确认。
