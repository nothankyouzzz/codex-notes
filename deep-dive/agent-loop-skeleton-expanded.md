# Codex Core — Agent Loop 骨干（从最小到完整）

## 第零层：任何 Agent 的最小循环

抛开 Codex 的所有工程复杂度，一个 tool-use agent 的本质就是：

```
history = [system_prompt]
history.push(user_message)

loop {
    response = model.call(history, tools)
    history.push(response)

    if response 里没有工具调用:
        break   // 回合结束，模型说完了

    for each tool_call in response.tool_calls:
        result = execute(tool_call)
        history.push(tool_result)

    // 带着工具结果继续问模型 → 回到 loop 顶部
}
```

三个核心概念：
1. **对话历史**（history）— 累积的上下文
2. **模型调用**（model.call）— 发请求、收流式响应
3. **工具执行**（execute）— 跑命令、改文件、调 MCP 等

一个关键信号：**模型是否返回了工具调用**。有 → 继续循环；没有 → 结束。

---

## 第一层：Codex 里这个最小循环在哪

对应到 `codex-rs/core/src/codex.rs`，骨架分布在三个函数里：

### 1. `run_turn`（L5548）— 外层循环

```rust
// 简化后的骨架：
loop {
    // ① 从 Session 拿到完整对话历史
    let input: Vec<ResponseItem> = sess.clone_history().await.for_prompt(...);

    // ② 调模型
    match run_sampling_request(sess, turn_context, input, ...).await {
        Ok(SamplingRequestResult { needs_follow_up, .. }) => {
            if !needs_follow_up {
                break;  // 模型没调工具，回合结束
            }
            continue;   // 模型调了工具，结果已写入 history，继续循环
        }
        Err(e) => { /* 报错，break */ }
    }
}
```

`needs_follow_up` 就是那个"模型是否返回了工具调用"的信号。

### 2. `run_sampling_request`（L6327）— 构建 Prompt + 调模型

```rust
// 简化后的骨架：
async fn run_sampling_request(..., input: Vec<ResponseItem>, ...) {
    // ① 收集可用工具
    let router = built_tools(sess, turn_context, &input, ...).await?;

    // ② 组装 Prompt = 历史 + 工具列表 + 系统指令
    let prompt = build_prompt(input, router, turn_context, base_instructions);

    // ③ 发请求（带重试）
    loop {
        match try_run_sampling_request(..., &prompt, ...).await {
            Ok(output) => return Ok(output),
            Err(err) if err.is_retryable() => { /* 重试 */ }
            Err(err) => return Err(err),
        }
    }
}
```

### 3. `try_run_sampling_request`（L7140）— 处理流式响应

```rust
// 简化后的骨架：
async fn try_run_sampling_request(..., prompt: &Prompt, ...) {
    // ① 发起流式请求
    let mut stream = client_session.stream(prompt, ...).await?;
    let mut in_flight = FuturesOrdered::new();  // 并行工具执行队列
    let mut needs_follow_up = false;

    // ② 逐事件处理流
    loop {
        match stream.next().await {
            ResponseEvent::OutputItemDone(item) => {
                // 核心分支：这个 item 是消息还是工具调用？
                let result = handle_output_item_done(item).await?;
                if let Some(tool_future) = result.tool_future {
                    in_flight.push_back(tool_future);  // 工具异步执行
                }
                needs_follow_up |= result.needs_follow_up;  // 有工具调用 → true
            }
            ResponseEvent::OutputTextDelta(delta) => {
                // 流式文本 → 转发给 UI
            }
            ResponseEvent::Completed { token_usage } => {
                break Ok(SamplingRequestResult { needs_follow_up, .. });
            }
            // ... 其他事件（reasoning delta、rate limits 等）
        }
    }

    // ③ 等所有并行工具执行完
    drain_in_flight(&mut in_flight).await?;
}
```

**关键路径**：`handle_output_item_done` 是分水岭。如果模型输出的是 `FunctionCall`，它会启动工具执行（返回一个 future 放进 `in_flight`），并把 `needs_follow_up` 设为 true。工具执行完后结果被写入 `Session` 的对话历史。回到 `run_turn` 的 loop 顶部，历史里已经有了工具结果，再次发给模型。

---

## 第二层：骨架上的扩展点

在上面那个最小循环的各个位置，Codex 插入了额外逻辑。按执行顺序：

| 位置 | 扩展 | 解决什么问题 |
|---|---|---|
| loop 进入前 | `run_pre_sampling_compact` | 历史太长 → 先压缩再调模型 |
| loop 进入前 | `record_context_updates` | 注入文件变化等环境上下文 |
| loop 进入前 | Skills / Plugins 解析 + 注入 | 把 @skill 引用展开为 system message |
| loop 进入前 | `run_user_prompt_submit_hooks` | Hook：用户提交前拦截 |
| 构建 history 时 | `ContextManager.for_prompt()` | 截断、归一化、去除不支持的模态 |
| 构建 tools 时 | `built_tools` → `ToolRouter` | 合并内置工具 + MCP 工具 + 动态工具 |
| 流处理中 | `OutputTextDelta` → Event | 实时推送打字效果给 UI |
| 工具执行前 | 审批检查（approval policy） | 危险命令需要用户确认 |
| 工具执行时 | 沙箱（sandbox） | 隔离执行环境 |
| 工具执行后 | 结果写入 history | `record_response_item_and_emit_turn_item` |
| needs_follow_up=false 后 | Stop hooks | Hook：模型停下来前拦截，可能让它继续 |
| needs_follow_up=true 且 token 超限 | `run_auto_compact` | 自动压缩后继续 |
| 整个 turn 结束后 | After-agent hooks | Hook：回合结束后触发 |

---

## 第三层：围绕循环的外壳

最小循环之外还有两层包装：

### `submission_loop`（L4245）— 消息分发器

```
while let Ok(sub) = rx_sub.recv().await {
    match sub.op {
        Op::UserTurn { .. } => handlers::user_input_or_turn(...)  // → 最终调 run_turn
        Op::ExecApproval { .. } => ...   // 用户批准了工具执行
        Op::Interrupt => ...              // 用户按了 Ctrl+C
        Op::Shutdown => break             // 退出
        // ... 几十种 Op
    }
}
```

这不是 agent loop 本身，而是一个**事件驱动的命令分发器**。用户的每次操作（发消息、批准、中断...）都是一个 `Op`，通过 channel 发给这个循环。

### `SessionTask`（tasks/mod.rs）— 任务抽象

`Op::UserTurn` 不直接调 `run_turn`，中间还有一层 `RegularTask`：

```
Op::UserTurn → Session::spawn_task(RegularTask) → RegularTask.run() → run_turn()
```

`SessionTask` 是对"一次用户交互"的抽象，除了 `RegularTask`（普通对话），还有 `CompactTask`、`UndoTask`、`ReviewTask` 等。它们共享取消、中断、状态管理等基础设施。

---

## 关键数据结构（最小集）

| 结构 | 在哪 | 是什么 |
|---|---|---|
| `Session` | codex.rs:797 | 一切状态的容器。持有对话历史、配置、模型客户端、MCP 连接等 |
| `ContextManager` | context_manager/history.rs | 对话历史 = `Vec<ResponseItem>` + token 统计 |
| `Prompt` | client_common.rs | 发给模型的请求包 = history + tools + instructions |
| `ResponseEvent` | 来自 codex-api | 模型流式返回的事件（Created / OutputItemDone / TextDelta / Completed） |
| `TurnContext` | codex.rs:833 | 单回合的不可变快照（模型、策略、cwd 等） |
| `Op` / `Event` | protocol crate | 外部→Agent 的指令 / Agent→外部的事件 |
| `SamplingRequestResult` | codex.rs:6591 | 单次模型调用的结果，核心字段就是 `needs_follow_up: bool` |

---

## 一句话

剥掉所有扩展后，Codex 的 agent loop 就是 `run_turn` 里的那个 `loop`：拿历史 → 调模型 → 有工具调用就执行、写回历史、继续；没有就结束。`needs_follow_up` 是驱动循环的唯一信号。其他一切（审批、沙箱、compact、hooks、skills、streaming UI...）都是在这个骨架的固定位置插入的扩展点。