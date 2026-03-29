# Codex 定制指南

定制手段按两个维度分类：**影响什么**（模型行为 vs loop 控制流 vs 工具集 vs 执行安全）和**复杂度**。

---

## 一、影响模型行为（告诉模型怎么做事）

### AGENTS.md（项目级，最轻量）

在项目根目录放 `AGENTS.md`，每次 turn 开始时作为 `user` 角色 message 注入。支持目录层级，越深层优先级越高。

```markdown
# 项目规范
- 提交前必须跑测试
- 不要修改 vendor/ 目录
- 用 conventional commits 格式
```

### Skills（按需激活的操作手册）

放在 `.codex/skills/` 或 `~/.codex/skills/` 下的 Markdown 文件。用 `@skill-name` 显式引用，或 Codex 根据命令隐式匹配。

和 AGENTS.md 的区别：AGENTS.md 是全局规则，Skills 是按需激活的专项 SOP。

在 loop 里的位置：`run_turn` 开头，`build_skill_injections()` 把 skill 内容展开成 `developer` message 追加到历史。

### `developer_instructions`（全局补充，config.toml）

```toml
developer_instructions = """
- 回复时使用中文
- 修改代码前先说明思路
"""
```

作为独立的 `developer` 角色 message 追加，不替换 base instructions。优先级高于 AGENTS.md。

### `base_instructions`（完全替换，慎用）

```toml
base_instructions = "你是一个..."
```

完全替换 catalog 里模型专属的 instructions，同时禁用 personality 模板。除非非常清楚自己在做什么，否则不建议使用。

---

## 二、影响 Loop 控制流（Hooks）

Hooks 是在 agent loop 的特定事件点执行 shell 命令，根据退出码和输出决定是否继续、阻断、或注入上下文。配置在 `~/.codex/config.toml`。

### SessionStart — 会话启动时

位置：`run_turn` 最开头，`run_pending_session_start_hooks()`

收到：session_id、cwd、model、permission_mode、source（startup/resume）

能做：
- 输出文本 → 注入为 `developer` message
- `{"continue": false}` → 阻止这次 turn
- `{"additionalContext": "..."}` → 注入上下文

用途：检查环境就绪、注入动态项目信息、根据 git branch 调整行为。

### UserPromptSubmit — 用户消息提交后、模型调用前

位置：`run_user_prompt_submit_hooks()`，用户消息写入历史之前

收到：用户输入的完整文本

能做：注入额外上下文，或 `{"continue": false}` 阻断整个 turn。

用途：敏感词过滤、把 issue 编号展开成完整描述。

### PreToolUse — 工具执行前

位置：`handle_output_item_done()` 里，工具实际执行之前

收到：tool_name、tool_use_id、command

能做：
- 退出码 0，无输出 → 放行
- `{"permissionDecision": "deny", "permissionDecisionReason": "..."}` → 阻断，原因作为 feedback 告诉模型
- 退出码 2，stderr 有内容 → 阻断

用途：自定义权限策略、审计日志、执行前备份。

### PostToolUse — 工具执行后

位置：工具执行完成后，结果写入历史之前

收到：tool_name、command、tool_response（完整输出）

能做：
- `{"additionalContext": "..."}` → 追加上下文给模型
- `{"decision": "block", "reason": "..."}` → 把 reason 作为 feedback 注入
- `{"continue": false}` → 停止整个 turn

用途：自动运行 linter 并把结果告诉模型、检测敏感信息。

### Stop — 模型完成一轮（needs_follow_up=false）时

位置：`run_turn` 主循环里，`needs_follow_up=false` 之后

收到：last_assistant_message、turn_id、model

能做：
- `{"decision": "block", "reason": "..."}` → 把 reason 作为新 user message 注入，loop 继续（`stop_hook_active = true`）
- 退出码 2，stderr 有内容 → 同上
- `{"continue": false}` → 强制停止

**这是实现自动化 workflow 最强的 hook**：模型说"改完了" → hook 跑测试 → 失败 → 把失败信息注入 → 模型继续修。

### Notify — 轻量级 turn 完成通知（不影响控制流）

```toml
notify = ["notify-send", "Codex"]
# 调用: notify-send Codex '{"type":"agent-turn-complete","turn-id":"..."}'
```

---

## 三、影响工具执行安全

### Exec Policy（Starlark 规则引擎）

`~/.codex/policy.star`，用 Starlark（Python 子集）写命令级别的 allow/deny/prompt 规则。比 PreToolUse Hook 更底层，是硬规则。

位置：`ExecPolicyManager::create_exec_approval_requirement_for_command()`，工具执行前、审批检查之前。

### Sandbox Policy（文件系统/网络隔离）

```toml
[permissions.my-profile.filesystem]
"~/projects/myapp" = "read-write"
"/etc" = "read-only"
":root" = "none"

[permissions.my-profile.network]
enabled = true
[permissions.my-profile.network.domains]
"api.github.com" = "allow"
"*.internal.corp" = "deny"
```

预设：`read-only` / `workspace-write` / `danger-full-access`。沙箱层强制执行，模型无法绕过。

### Approval Policy（人工审批级别）

`ask_for_approval` 字段：`never`（全自动）/ `on-failure` / `on-request` / `unless-trusted` / `granular`（精细控制）。

---

## 四、扩展工具集（MCP 服务器）

```toml
[mcp_servers.my-tools]
command = "python3"
args = ["/path/to/my_mcp_server.py"]
enabled_tools = ["search", "read"]   # 白名单
disabled_tools = ["delete"]          # 黑名单

[mcp_servers.my-tools.tools.search]
approval_mode = "approve"            # 这个工具每次都要审批
```

位置：`built_tools()` 时，MCP 工具和内置工具合并进 `ToolRouter`，一起放进 `Prompt.tools`。

---

## 总结

| 机制 | 影响什么 | 复杂度 |
|---|---|---|
| AGENTS.md | 模型行为（指令） | 最低，写文本 |
| Skills | 模型行为（按需激活） | 低，写 Markdown |
| `developer_instructions` | 模型行为（全局补充） | 低，写 config |
| Hooks（5 个事件点） | Loop 控制流 | 中，写脚本 |
| Exec Policy | 命令执行规则 | 中，写 Starlark |
| Sandbox Policy | 文件/网络访问范围 | 低，写 config |
| Approval Policy | 人工审批触发条件 | 低，写 config |
| MCP 服务器 | 工具集 | 高，实现 MCP 协议 |

最有意思的组合：**Stop Hook + MCP**（自动化 workflow）和 **Exec Policy + Sandbox + Stop Hook**（安全且自动化）。
