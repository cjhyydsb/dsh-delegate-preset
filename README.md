# 委派（delegate）— DeepSeek Harness 的「只委派不执行」模式

给 DeepSeek Harness 增加一个 agent preset：**根模型只有委派权和对话权，所有实际工作交给子代理；子代理按标准模式（standard）的全套工具干活。**

适合把「执行」和「编排」分开：根模型不读文件、不跑命令、不搜索，只负责拆任务、写自包含的任务书、并行派活、汇总报告，并在有疑问时向你确认。根上下文不会被工具输出撑爆，也不会边做边猜。

## 根模型能用的工具

| 工具 | 作用 |
|---|---|
| `subagent` | 派活。spawn 一个全新子代理，预设固定为 `standard`，深度上限 1 |
| `ask_user_question` | 向你提问；每个问题都必须带推荐答案 |
| `send_message` | 给某个子代理追加任务 |
| `list_agents` | 查看子代理树与状态 |
| `wait_agent` | 在本轮内等待下一次输入/结果 |

另外，安装环境可能自带几个 **harness 级工具**，预设无法移除：`working_directory`（读/切会话工作目录）、`load_workspace_dependencies`（返回打包的 Python/Node 路径，仅 Desktop 且配置了 primary runtime 时出现）。它们都不能读写项目文件、不能执行命令。

**明确没有**：shell（bash/pwsh）、文件读写、glob/grep、web 搜索与抓取、skill、Office、todo、plan-mode、goal、schedule、jobs、workflow、present、`subagent_fork`、子代理模型选择。

根模型的行为约定（写在 persona 里）：

- 自己不猜、不装懂；凡是能查的都派子代理去查
- 每份任务书自包含：目标、路径、约束、交付物、回报方式
- 独立任务并行派，不轮询、不空等
- 目标、约束、交付物或你的意图不清楚时，**反复确认**（同一轮里可以问多轮，一次一个问题），每个问题都给出推荐答案并放在第一项
- 汇总子代理的结果后再回答你，用你的语言回复

## 子代理

- 预设：`standard` —— 完整标准工具集（shell、文件、搜索、web、skill、Office、job、schedule、todo、plan、goal 等）
- 深度：`maxDepth: 1`，它到第 1 层就到顶，不再往下派，自己把活干完
- 继承：当前会话的 provider/model、reasoning effort、工作目录、权限设置（sandbox + approval）
- 上下文：spawn 是全新会话，看不到你和根模型的对话，所以根模型的任务书必须自包含
- 刻意没有 `subagent_fork`：fork 子代理会继承父预设，生出来还是「只会委派」的壳，没有意义

## 安装

前置条件：DSH 0.2.x（带 agent-preset 预设体系）的 Web 或 Desktop profile —— 需要有预设选择器。

本仓库是私有仓库，推荐先 clone，再按本地目录安装：

```powershell
git clone https://github.com/cjhyydsb/dsh-delegate-preset.git
```

1. 打开 DSH → 左侧 **Plugins** → **Add plugin**（或箭头 → **Install a third-party plugin**）
2. 粘贴 clone 下来的**目录绝对路径** → **Install** → **Enable**
3. **新建会话** → 模式选择器 → 「委派」

也可以让带 `plugin_manager` 工具的 agent 安装：`install_bundle`，`target` 为该目录的绝对路径。

> 不要把私有仓库 URL 直接粘进 Add plugin —— 那需要 GitHub 凭据，而且把 token 拼进 URL 更不可取。

## 第一次用怎么验证

- 新建会话的模式选择器里有「委派」，会话工具表就是上面那几个
- 让它读一个文件：它会派子代理去读，而不是自己读
- 给它一个模糊需求：它应该先问你，且每个问题都带 `(Recommended)`

## 自定义

全部集中在 `cordis.patch.yml`：

| 想改什么 | 改哪里 |
|---|---|
| 人格与纪律 | `plugins` 里 `persona` 行的 `config.prefix` |
| 工具范围 | `plugins` 列表：去掉 `tool-ask-user`，或加回 `tool-todo`、`tool-web` 等 |
| 子代理预设 | `tool-subagent` 行的 `agentPreset`（默认 `standard`） |
| 能否再往下派 | `maxDepth`（`1` = 子代理不能再派） |
| 模式名/说明/排序 | `config.name`、`config.description`、`config.order` |

改完文件后，在 Plugins 里把该 bundle 关掉再打开（最可靠是 **Remove → Install**）；新会话生效，已有会话保持旧版本。

## 更新 / 卸载

- 更新：在 clone 目录 `git pull`，然后 Plugins 里 Remove → Install
- 卸载：Plugins → 该 bundle → Remove（profile 依赖一并移除，clone 的文件不受影响）

## 已知限制

- 依赖安装里存在 id 为 `standard` 的预设作为子代理预设；删掉或改名它，委派会失败
- 只有 Web/Desktop profile 有预设选择器；headless / SDK / ACP profile 没有 agent-preset
- 上述 harness 级工具无法通过预设移除
- 从 git 或本地目录安装的 bundle 没有「Versions」自动更新段，更新只能 Remove → Install

## 安全

- 本仓库不含任何密钥或凭据
- 这个模式只是**不给根模型执行能力**，不是更强的沙箱：子代理沿用会话的 sandbox / approval 设置
- 不要把 PAT、`gh.txt` 之类凭据提交进仓库

## License

MIT

---

**English.** A DeepSeek Harness agent preset (`delegate`) that turns the root model into a delegate-only orchestrator: its whole toolset is `subagent`, `ask_user_question`, `send_message`, `list_agents`, and `wait_agent`. Every child is spawned on the `standard` preset, so the real work happens in full-tooled subagents at depth 1. Install by cloning this repository and adding the directory as a plugin in DSH, then pick 「委派」 for a new session.
