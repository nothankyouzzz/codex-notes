# 深入 Codex Agent Loop：你能定制什么，怎么定制

Codex 是 OpenAI 开源的本地编程 Agent，核心逻辑用 Rust 实现。它不只是一个"调用 GPT 的壳"——它有完整的工具执行引擎、沙箱隔离、审批系统和 Hook 机制。理解它的 agent loop，是理解你能定制什么、在哪里定制的前提。

## 先搞清楚 agent loop 是什么

任何 tool-use agent 的本质都是同一个循环：

```
history = [system_prompt]
history.push(user_message)

loop {
    response = model.call(history, tools)
    history.push(response)

    if response 里没有工具调用:
        break   // 模型说完了，回合结束

    for each tool_call in response.tool_calls:
        result = execute(tool_call)
        history.push(tool_result)
    // 带着工具结果继续问模型
}
```

三个核心概念：**对话历史**（累积的上下文）、**模型调用**（发请求收响应）、**工具执行**（跑命令改文件）。驱动循环的唯一信号是：模型有没有返回工具调用。有就继续，没有就结束。

Codex 的实现在 `codex-rs/core/src/codex.rs` 里，骨架是这样的：

```rust
// run_turn 里的主循环（简化）
loop {
    let input = sess.clone_history().await.for_prompt(...);

    match run_sampling_request(sess, turn_context, input, ...).await {
        Ok(SamplingRequestResult { needs_follow_up, .. }) => {
            if !needs_follow_up { break; }  // 没有工具调用，结束
            continue;                        // 有工具调用，继续
        }
        Err(e) => { break; }
    }
}
```

`needs_follow_up` 就是那个信号。`run_sampling_request` 负责构建 Prompt 并调模型，`try_run_sampling_request` 处理流式响应——当模型返回 `FunctionCall` 时，启动工具执行，把结果写回历史，`needs_follow_up` 置为 true，循环继续。

这个骨架很简单。Codex 的复杂性来自于它在这个骨架的**固定位置**插入了大量扩展点。而这些扩展点，正是你可以定制的地方。

## 定制点地图

把 agent loop 展开，定制点分布在这些位置：

```
loop 开始前
  ├── system prompt（base_instructions）
  ├── developer_instructions（config.toml）
  ├── AGENTS.md / Skills 注入
  └── SessionStart Hook

用户消息提交后
  └── UserPromptSubmit Hook

构建 Prompt 时
  ├── 模型参数（model / effort / context_window）
  └── 工具列表（内置工具 + MCP 工具）

工具执行前
  ├── Exec Policy（Starlark 规则）
  ├── Sandbox Policy（文件/网络范围）
  ├── Approval Policy（人工审批）
  └── PreToolUse Hook

工具执行后
  └── PostToolUse Hook

模型完成一轮后（needs_follow_up=false）
  ├── Stop Hook（可让模型继续工作）
  └── Notify（纯通知）
```

下面按"影响什么"来分类讲。

## 一、告诉模型怎么做事

这一层不影响 loop 的控制流，只影响模型的行为。

**AGENTS.md** 是最轻量的方式。在项目根目录放一个文件，每次 turn 开始时作为 `user` 角色 message 注入。支持目录层级，越深层优先级越高：

```markdown
# 项目规范
- 提交前必须跑测试
- 不要修改 vendor/ 目录
- 用 conventional commits 格式
```

**`developer_instructions`** 是全局补充，在 `~/.codex/config.toml` 里配置，作为 `developer` 角色 message 追加（优先级高于 AGENTS.md）：

```toml
developer_instructions = """
- 回复时使用中文
- 修改代码前先说明思路
"""
```

**Skills** 是按需激活的操作手册，放在 `.codex/skills/` 下的 Markdown 文件，用 `@skill-name` 引用。和 AGENTS.md 的区别：AGENTS.md 是全局规则，Skills 是针对特定任务类型的 SOP，只在被引用时注入。

这三者的注入顺序（每次 turn）：

```
base_instructions（模型专属 system prompt）
  ↓
developer message：developer_instructions + skills + 沙箱策略说明 + ...
  ↓
user message：AGENTS.md + 环境上下文（cwd、git info 等）
```

## 二、控制 loop 的执行流程（Hooks）

Hooks 是 Codex 最强大的定制机制。本质是：在 loop 的特定事件点执行你的 shell 脚本，根据退出码和输出决定是否继续、阻断、或向模型注入额外上下文。

Codex 提供五个 Hook 事件点：

**SessionStart**：会话启动时触发。脚本可以输出文本注入为 developer message，或返回 `{"continue": false}` 阻止这次 turn。适合检查环境就绪、注入动态项目信息。

**UserPromptSubmit**：用户消息提交后、模型调用前触发。可以注入额外上下文，或阻断整个 turn。适合敏感词过滤、把 issue 编号展开成完整描述。

**PreToolUse**：工具执行前触发。可以放行、阻断（原因作为 feedback 告诉模型）。适合自定义权限策略、审计日志。

**PostToolUse**：工具执行后触发。可以追加上下文给模型，或把 feedback 注入让模型知道"这次执行有问题"。适合自动运行 linter 并把结果告诉模型。

**Stop**：模型完成一轮（没有更多工具调用）时触发。这是最强的 Hook——你可以把一段文字作为新的 user message 注入，让 loop 继续运行：

```python
# stop_hook.py
import json, subprocess, sys

result = subprocess.run(["pytest", "--tb=short"], capture_output=True, text=True)
if result.returncode != 0:
    # 测试失败，让模型继续修
    print(json.dumps({
        "decision": "block",
        "reason": f"测试失败，请修复：\n{result.stdout}"
    }))
```

这个模式可以实现完全自动化的质量门控：模型说"改完了" → hook 跑测试 → 失败 → 把失败信息注入 → 模型继续修 → 直到测试通过。

## 三、控制工具执行的安全边界

**Exec Policy** 是命令级别的规则引擎，用 Starlark（Python 子集）写：

```python
# ~/.codex/policy.star
def policy(cmd):
    if "rm" in cmd and "-rf" in cmd:
        return deny("禁止 rm -rf")
    if cmd.startswith("git push"):
        return prompt("确认推送？")
    return allow()
```

三种裁决：`allow`（直接执行）、`deny`（拒绝并告诉模型原因）、`prompt`（弹出审批请求）。这比 PreToolUse Hook 更底层，是硬规则，在沙箱层面执行。

**Sandbox Policy** 控制文件系统和网络访问范围：

```toml
[permissions.strict.filesystem]
"~/projects/myapp" = "read-write"
":root" = "none"

[permissions.strict.network]
enabled = true
[permissions.strict.network.domains]
"api.github.com" = "allow"
"*" = "deny"
```

预设有 `read-only`、`workspace-write`、`danger-full-access`。沙箱层强制执行，模型无法绕过。

**Approval Policy** 控制哪些操作需要人工确认：`never`（全自动，适合 CI）、`on-failure`、`on-request`、`unless-trusted`、`granular`（精细控制每类工具）。

## 四、扩展工具集（MCP）

通过配置 MCP 服务器，可以给模型提供任意自定义工具：

```toml
[mcp_servers.my-tools]
command = "python3"
args = ["/path/to/my_mcp_server.py"]
enabled_tools = ["search", "read"]
disabled_tools = ["delete"]

[mcp_servers.my-tools.tools.search]
approval_mode = "approve"   # 这个工具每次都要审批
```

MCP 工具和内置的 shell/apply_patch 工具地位相同，都会出现在发给模型的 `Prompt.tools` 里。

## 组合使用

单个机制的威力有限，组合起来才有意思。

**自动化 CI 流程**：Stop Hook 跑测试 + PostToolUse Hook 跑 linter + Exec Policy 禁止危险命令。模型改代码 → linter 自动检查 → 测试自动验证 → 不通过就继续改，全程无需人工干预。

**安全的自动化**：Sandbox Policy 限制文件访问范围 + Exec Policy 写硬规则 + Approval Policy 设为 `never`。既能全自动运行，又有明确的安全边界，模型无法越界。

**接入内部系统**：MCP 服务器暴露内部 API（查 Jira、读 Confluence、触发 CI）+ AGENTS.md 告诉模型这些工具的使用规范 + Stop Hook 在完成后自动创建 PR。

## 总结

Codex 的 agent loop 本质上是一个带扩展点的 while 循环。理解了这个循环，就理解了所有定制机制的本质：

- **AGENTS.md / Skills / `developer_instructions`**：在 loop 开始前修改模型看到的上下文
- **Hooks**：在 loop 的关键节点插入你的逻辑，可以注入信息、阻断执行、让循环继续
- **Exec Policy / Sandbox / Approval**：在工具执行层面设置安全边界
- **MCP**：扩展模型能调用的工具集

从最简单的 AGENTS.md 到最复杂的 Stop Hook + MCP 组合，这些机制覆盖了从"告诉模型规范"到"构建完全自动化 workflow"的全部需求。选哪个取决于你想控制的是模型的行为，还是 loop 的执行流程，还是工具执行的安全边界。

---