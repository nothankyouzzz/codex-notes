# Codex 配置参考

## Model Instructions 机制

### 来源与优先级

1. `models.json` catalog 里每个模型的 `base_instructions`（模型专属，最高优先级）
2. `config.toml` 的 `base_instructions` 覆盖（完全替换，慎用）
3. `codex-rs/core/prompt.md`（fallback，模型不在 catalog 时使用）

### 完整注入顺序（每次 turn）

```
[system / instructions]
  base_instructions（catalog 或 config 覆盖或 prompt.md fallback）

[developer message]（合并成一条）
  1. sandbox / approval 策略说明（自动生成）
  2. developer_instructions（config.toml）
  3. memory 指令（MemoryTool feature 开启时）
  4. collaboration mode 指令（plan mode 等）
  5. skills / plugins 说明
  6. commit attribution 规则

[user message]（contextual）
  1. AGENTS.md 内容
  2. 环境上下文（cwd、shell、git info 等）
```

### 增量修改建议

- 全局补充：`developer_instructions`（developer 角色，优先级高）
- 项目级补充：`AGENTS.md`（user 角色，支持目录层级）
- 完全替换：`base_instructions`（慎用，会丢失模型专属优化）

---

## 配置项速查

### 认证与账户
- `cli_auth_credentials_store_mode` — 凭证存储（file / keyring / auto）
- `mcp_oauth_credentials_store_mode` — MCP OAuth 凭证存储
- `mcp_oauth_callback_port` / `mcp_oauth_callback_url` — OAuth 回调地址
- `forced_login_method` — 强制指定登录方式
- `forced_chatgpt_workspace_id` — 限定 ChatGPT workspace

### 模型与 Provider
- `model` / `review_model` — 主模型和 review 专用模型
- `model_provider_id` / `model_providers` — 选择 provider（OpenAI / LM Studio / Ollama / 自定义）
- `model_context_window` — 上下文窗口大小
- `model_auto_compact_token_limit` — 超过此 token 数自动压缩历史
- `tool_output_token_limit` — 工具输出的 token 上限
- `model_reasoning_effort` / `plan_mode_reasoning_effort` — 推理强度（low / medium / high）
- `model_reasoning_summary` — 推理摘要模式
- `model_verbosity` — 输出详细程度（GPT-5 系列）
- `model_catalog` — 替换内置模型目录
- `service_tier` — fast / flex

### Instructions
- `base_instructions` — 完全替换 system prompt（慎用）
- `developer_instructions` — 追加为 developer message（推荐）
- `compact_prompt` — 替换上下文压缩时用的 prompt

### TUI 界面
- `tui_alternate_screen` — alternate screen buffer（auto / always / never）
- `tui_status_line` — 状态栏显示项（自定义排列）
- `tui_terminal_title` — 终端标题栏内容
- `tui_theme` — 语法高亮主题
- `tui_notifications` / `tui_notification_method` — 桌面通知（osc9 / bel）
- `animations` — 动画效果开关
- `show_tooltips` — 启动提示开关
- `disable_paste_burst` — 关闭粘贴防抖检测

### 会话与历史持久化
- `ephemeral` — 会话不写磁盘
- `history` — history.jsonl 写入策略（save-all / none）和大小上限
- `codex_home` / `sqlite_home` / `log_dir` — 数据目录
- `ghost_snapshot` — undo 快照配置

### 记忆系统（Memories）
- `memories.generate_memories` / `use_memories` — 写路径/读路径开关
- `memories.extract_model` / `consolidation_model` — Phase 1/2 使用的模型
- `memories.max_rollout_age_days` / `min_rollout_idle_hours` — 触发条件
- `memories.max_unused_days` — 记忆多久没用就删除
- `memories.no_memories_if_mcp_or_web_search` — 有 MCP/搜索时不生成记忆

### 网络与代理
- `enforce_residency` — 地理合规限制
- `chatgpt_base_url` — ChatGPT 请求基础 URL
- `CODEX_CA_CERTIFICATE` 环境变量 — 自定义 CA 证书（企业代理场景）
- 各 provider 的 `base_url` — 指向自托管或代理端点

### 项目与多 Agent
- `project_doc_max_bytes` — AGENTS.md 最大读取字节数
- `project_doc_fallback_filenames` — 除 AGENTS.md 外还查找哪些文件名
- `[project]` — 项目信任级别（trusted / untrusted）
- `agent_roles` — 自定义 agent 角色定义
- `agent_max_threads` / `agent_max_depth` — 子 agent 并发和嵌套深度限制
- `agent_job_max_runtime_seconds` — 任务超时

### 工具
- `include_apply_patch_tool` — 是否暴露 apply_patch 工具给模型
- `web_search_mode` / `web_search_config` — 网络搜索开关和参数
- `background_terminal_max_timeout` — 后台终端轮询超时
- `use_experimental_unified_exec_tool` — 实验性统一执行工具
- `js_repl_node_path` / `js_repl_node_module_dirs` — JS REPL 运行时路径

### 可观测性
- `otel` — OpenTelemetry 导出配置（exporter 类型、endpoint、headers、TLS）
- `analytics_enabled` — 使用分析开关
- `feedback_enabled` — 反馈收集开关
- `hide_agent_reasoning` / `show_raw_agent_reasoning` — 推理过程显示控制

### 杂项
- `notify` — turn 完成后调用的外部通知命令
- `commit_attribution` — git commit 的 co-author 署名
- `file_opener` — 文件引用的 URI scheme（vscode / cursor / windsurf / none）
- `check_for_update_on_startup` — 启动时检查更新
- `suppress_unstable_features_warning` — 屏蔽实验性功能警告
