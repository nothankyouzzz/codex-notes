# 配置骨架

## Instructions 注入顺序

```
[system]   base_instructions（catalog 模型专属 > config 覆盖 > prompt.md fallback）
[developer] sandbox/approval 说明 + developer_instructions + memory + skills + ...
[user]     AGENTS.md + 环境上下文（cwd / shell / git info）
```

增量修改推荐：`developer_instructions`（全局）+ AGENTS.md（项目级），不动 `base_instructions`。

## 配置分类速查

**模型**：`model` / `model_provider_id` / `model_reasoning_effort` / `model_context_window` / `model_auto_compact_token_limit`

**Instructions**：`base_instructions`（慎用）/ `developer_instructions`（推荐）/ `compact_prompt`

**执行安全**：`ask_for_approval` / `sandbox` / `[permissions]` / `policy.star`

**MCP**：`[mcp_servers.*]` / `enabled_tools` / `disabled_tools` / `tools.*.approval_mode`

**TUI**：`tui_alternate_screen` / `tui_status_line` / `tui_theme` / `animations`

**会话**：`ephemeral` / `history` / `codex_home` / `sqlite_home`

**记忆**：`[memories]` / `generate_memories` / `use_memories` / `extract_model`

**网络**：`enforce_residency` / `chatgpt_base_url` / `CODEX_CA_CERTIFICATE`

**多 Agent**：`agent_roles` / `agent_max_threads` / `agent_max_depth`

**可观测性**：`otel` / `analytics_enabled` / `hide_agent_reasoning`

→ 完整配置项见 `deep-dive/configuration.md`
→ Instructions 机制详解见 `deep-dive/configuration.md#instructions`
