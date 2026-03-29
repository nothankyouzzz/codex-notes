# 定制点骨架

## 定制点地图

| 机制 | 影响什么 | 插入位置 | 复杂度 |
|---|---|---|---|
| `base_instructions` | System prompt | 每次 turn 开始 | 低（替换） |
| `developer_instructions` | 追加 developer message | 每次 turn 开始 | 低（追加） |
| AGENTS.md | 追加 user message | 每次 turn 开始 | 最低 |
| Skills | 按需追加 developer message | run_turn 开头 | 低 |
| SessionStart Hook | 注入上下文 / 阻断 turn | run_turn 最开头 | 中 |
| UserPromptSubmit Hook | 注入上下文 / 阻断 turn | 用户消息写入前 | 中 |
| PreToolUse Hook | 阻断工具执行 | 工具执行前 | 中 |
| PostToolUse Hook | 注入上下文 / 停止 turn | 工具执行后 | 中 |
| Stop Hook | 让 loop 继续 / 停止 | needs_follow_up=false 后 | 中 |
| Exec Policy | allow/deny/prompt 命令 | 工具执行前（硬规则） | 中 |
| Sandbox Policy | 文件/网络访问范围 | 工具执行时（强制） | 低 |
| Approval Policy | 人工审批触发条件 | 工具执行前 | 低 |
| MCP 服务器 | 扩展工具集 | 构建 Prompt 时 | 高 |

## 按目的分类

**告诉模型怎么做事**：`base_instructions` / `developer_instructions` / AGENTS.md / Skills

**控制 loop 流程**：Hooks（5 个事件点）

**工具执行安全**：Exec Policy / Sandbox Policy / Approval Policy

**扩展工具集**：MCP 服务器

## 最有价值的组合

- Stop Hook + MCP → 自动化 workflow（质量门控）
- Exec Policy + Sandbox + Stop Hook → 安全且自动化

→ 详细用法见 `deep-dive/customization.md`
