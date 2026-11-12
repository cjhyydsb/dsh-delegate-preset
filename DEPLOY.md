# 部署到 DSH（给 agent / LLM 执行的指令）

目标：把本目录声明的 `delegate`（委派）预设装进**当前 DSH profile**，让它在新建会话的模式选择器里可选。

执行者假设：你是一个运行在 DSH 里的 agent，手上有 `plugin_manager`（可选：`cordis_inspect_query`）。

## 红线（先读）

- **不要**手工编辑 profile 的 `package.json`、`cordis.patch.yml`，**不要**在 profile 目录里跑 `pnpm` —— 安装/启用/卸载全部通过 `plugin_manager`。
- **不要**把任何 token、密钥写进文件或提交进仓库。
- 预设只对**新建会话**生效；不要承诺已有会话里立刻改变。
- 不要替用户改「新会话默认预设」，除非用户明确要求。

## 0. 前置检查

1. 拿到本目录的**绝对路径**（`git clone` 或解压后）。
2. 确认这套 DSH 支持 agent preset：让工具查一次预设声明行

```
cordis_inspect_query(
  platform: "host", provider: "Config", method: "listConfigs",
  input: { name: "@deepseek-ai/dsh-agent-preset" })
```

返回里应有 `preset-standard`、`preset-ptc`、`preset-minimal`、`preset-cordis` 四行（`status: "schema"`）。若一条都没有：这个 profile 不是 Web/Desktop（headless / SDK / ACP profile 没有预设体系），到此停止并如实告诉用户。

## 1. 安装

```
plugin_manager(action: "install_bundle", target: "<本目录绝对路径>")
```

期望返回：`application: "applied"`、`enabled: true`、`warnings: []`、`bundle: "dsh-delegate-preset"`、`version: "0.1.0"`。

- 返回 `error.code: "already-installed"`：说明已经装过。要重载改动就先 `remove_bundle` 再 `install_bundle`。
- 返回 `application: "failed"`：读 `error` / 诊断，按第 4 节处理；**不要**改动 profile 文件。
- 安装会执行插件代码，可能需要 Full access 或用户批准 —— 走正常批准流程，不要绕过。

## 2. 验证（至少跑 a）

**a) 声明行存在**

```
cordis_inspect_query(
  platform: "host", provider: "Config", method: "listConfigs",
  input: { name: "@deepseek-ai/dsh-agent-preset" })
```

应多出 `patchId: "preset-delegate"`、`status: "schema"`。

**b) 行已激活（可选，需要 `plugin_manager`）**

```
plugin_manager(action: "list_plugins", limit: 100, offset: <翻页>)
```

翻页直到出现 `entryId: "include:preset-delegate"`，要求 `enabled: true` 且 `fiberPhase: "active"`。内置预设行与本次安装的 bundle 行通常排在列表末尾（`total` 在两三百之间，一次 100 条，从 `offset: 200` 起翻最快）。

## 3. 交给用户验收

你能做的到此为止 —— 你开不了「委派」模式的新会话，所以最后一步必须交给用户：

1. **新建会话** → 模式选择器 → 「委派」
2. 第一轮工具表应只有：`subagent`、`ask_user_question`、`send_message`、`list_agents`、`wait_agent`；有的安装还会带 harness 级 `working_directory`、`load_workspace_dependencies`（无法通过预设移除）
3. 让它读某个文件：它应该派子代理去读，而不是自己读
4. 给它一个模糊需求：它应该先提问，且每个问题都带 `(Recommended)`

把这三条验收结果报给用户；不要替用户宣称「已经可用」。

## 4. 常见失败

| 现象 | 原因 | 处理 |
|---|---|---|
| `already-installed` | 已装过 | `remove_bundle(target: "dsh-delegate-preset")` 后重装 |
| `application: "failed"` + 诊断 | patch 插件名/字段与当前 DSH 版本不匹配（例如 `agentPreset`、`maxDepth` 被拒） | 按诊断修 `cordis.patch.yml`，重装；不要改 profile |
| 委派时报 `standard` 预设不存在 | 安装里没有 id 为 `standard` 的预设 | 保留/恢复 standard，或把 `agentPreset` 改成实际存在的预设 id |
| 模式选择器里没有「委派」 | profile 不是 Web/Desktop，或 bundle 只装了没启用 | 换 profile；检查第 2 节 b |
| 页面看不到 | 浏览器端名单需刷新 | 刷新页面后重新看 Plugins / 模式选择器 |

## 5. 更新与卸载

- 更新：`git pull`（或改文件）→ `remove_bundle(target: "dsh-delegate-preset")` → `install_bundle(target: "<目录>")`
- 卸载：`plugin_manager(action: "remove_bundle", target: "dsh-delegate-preset")`

## 6. 改配置时只改这里

`cordis.patch.yml` 是本仓库唯一的行为来源：人格（`persona.config.prefix`）、工具范围（`plugins` 列表）、子代理预设（`tool-subagent.config.agentPreset`）、委派深度（`maxDepth`）。改完按第 5 节重装，然后让用户在新会话里重跑第 3 节验收。
