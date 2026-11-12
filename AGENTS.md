# Agent 指引

如果你是一个被要求「把这个仓库装到 DSH」的 agent：**读 `DEPLOY.md` 并照做**，不要凭记忆猜命令。

要点：用 `plugin_manager` 的 `install_bundle`（target 为本目录绝对路径）安装，**不要**手工改 profile 的 `package.json` / `cordis.patch.yml`，**不要**在 profile 目录跑 `pnpm`；装完把「新建会话 → 选委派 → 验证工具表」交给用户验收。

人类用户请读 `README.md`。
