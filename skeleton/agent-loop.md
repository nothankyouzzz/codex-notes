# Agent Loop 骨架

## 最小循环（任何 tool-use agent 的本质）

```
history = [system_prompt, user_message]

loop {
    response = model.call(history, tools)
    history.push(response)

    if 没有工具调用: break

    for tool_call in response.tool_calls:
        result = execute(tool_call)
        history.push(tool_result)
}
```

驱动信号：`needs_follow_up`（模型是否返回了工具调用）

## Codex 里的对应关系

```
run_turn()                  外层循环，驱动 needs_follow_up
  └── run_sampling_request()    构建 Prompt + 调模型（带重试）
        └── try_run_sampling_request()  处理流式响应，执行工具
```

所有代码在 `codex-rs/core/src/codex.rs`。

## 扩展点地图（按执行顺序）

```
loop 进入前
  ├── run_pre_sampling_compact    历史太长 → 先压缩
  ├── record_context_updates      注入环境上下文
  ├── Skills / Plugins 注入       @skill 引用展开
  └── SessionStart / UserPromptSubmit Hooks

构建 Prompt 时
  ├── ContextManager.for_prompt() 归一化历史
  └── built_tools() → ToolRouter  合并内置 + MCP + 动态工具

流处理中
  ├── OutputTextDelta → UI        流式打字效果
  └── OutputItemDone → handle_output_item_done()
        ├── 工具调用 → in_flight（并行执行）
        └── 普通消息 → emit TurnItemCompleted

工具执行（ToolOrchestrator）
  ├── ExecPolicy 评估
  ├── 审批检查（可能暂停等用户）
  ├── 沙箱执行（失败可升级）
  └── 结果写入历史

needs_follow_up=false 后
  ├── Stop Hooks（可让 loop 继续）
  └── After-agent Hooks
```

## 外壳

```
submission_loop     事件分发器，接收 Op（UserTurn / ExecApproval / Interrupt...）
  └── RegularTask   任务抽象，处理 pending input，调 run_turn
        └── run_turn
```

→ 深度解析见 `deep-dive/agent-loop.md`
→ 工具执行细节见 `deep-dive/tool-execution.md`
