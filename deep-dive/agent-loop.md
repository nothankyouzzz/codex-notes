# 深度解析 Codex Agent Loop

Codex 是 OpenAI 开源的本地编程 Agent，核心逻辑全部用 Rust 实现，代码在 `codex-rs/core/` 下。这篇文章从最小的 agent loop 骨架出发，逐层展开，结合源码讲清楚每一个环节实际发生了什么。

## 第一层：最小骨架

任何 tool-use agent 的本质都是同一个循环：

```
history = [system_prompt]
history.push(user_message)

loop {
    response = model.call(history, tools)
    history.push(response)

    if response 里没有工具调用:
        break

    for each tool_call in response.tool_calls:
        result = execute(tool_call)
        history.push(tool_result)
}
```

三个核心概念：**对话历史**（累积的上下文）、**模型调用**（发请求收响应）、**工具执行**（跑命令改文件）。驱动循环的唯一信号是：模型有没有返回工具调用。

Codex 的实现把这个骨架分布在三个函数里，都在 `codex-rs/core/src/codex.rs`。

---

## 第二层：三个核心函数

### `run_turn`（L5548）— 外层循环

这是 agent loop 的主体。简化后的骨架：

```rust
pub(crate) async fn run_turn(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    input: Vec<UserInput>,
    prewarmed_client_session: Option<ModelClientSession>,
    cancellation_token: CancellationToken,
) -> Option<String> {
    // ...（loop 进入前的准备工作）

    loop {
        // 从 Session 拿到完整对话历史，组装成模型输入
        let sampling_request_input: Vec<ResponseItem> =
            sess.clone_history().await.for_prompt(&turn_context.model_info.input_modalities);

        // 调模型
        match run_sampling_request(sess, turn_context, input, ...).await {
            Ok(SamplingRequestResult { needs_follow_up, .. }) => {
                if !needs_follow_up {
                    break;  // 模型没调工具，回合结束
                }
                continue;   // 模型调了工具，结果已写入 history，继续
            }
            Err(CodexErr::TurnAborted) => break,
            Err(e) => { /* 发送错误事件，break */ }
        }
    }
}
```

`needs_follow_up` 是驱动循环的唯一信号。

注意 `sess.clone_history().await.for_prompt(...)` 这一行。`clone_history()` 返回一个 `ContextManager` 的克隆，`for_prompt()` 在发给模型之前做三件事（`normalize_history`）：

1. **确保每个工具调用都有对应的输出**（`ensure_call_outputs_present`）——如果历史里有孤立的 function_call 没有对应的 function_call_output，会自动补一个空输出，防止 API 报错
2. **删除孤立的工具输出**（`remove_orphan_outputs`）——有输出但没有对应调用的，删掉
3. **按模型能力过滤模态**（`strip_images_when_unsupported`）——如果模型不支持图片，把历史里的图片内容剥掉

这个归一化步骤保证了发给模型的历史永远是合法的。

### `run_sampling_request`（L6327）— 构建 Prompt + 调模型

```rust
async fn run_sampling_request(..., input: Vec<ResponseItem>, ...) {
    // ① 构建工具路由器
    let router = built_tools(sess, turn_context, &input, ...).await?;

    // ② 组装 Prompt
    let prompt = build_prompt(input, router.as_ref(), turn_context, base_instructions);

    // ③ 发请求（带重试）
    let mut retries = 0;
    loop {
        match try_run_sampling_request(..., &prompt, ...).await {
            Ok(output) => return Ok(output),
            Err(err) if err.is_retryable() => {
                // 重试逻辑：指数退避，超过上限后尝试切换 WebSocket → HTTP fallback
                retries += 1;
                tokio::time::sleep(backoff(retries)).await;
            }
            Err(err) => return Err(err),
        }
    }
}
```

`built_tools()` 是工具列表的组装过程，它把以下来源合并成一个 `ToolRouter`：
- **内置工具**：`shell`、`apply_patch`、`js_repl`、`list_dir` 等（由 `ToolsConfig` 决定哪些开启）
- **MCP 工具**：从所有已连接的 MCP 服务器列出的工具
- **App 工具**（Connectors）：ChatGPT 插件/连接器暴露的工具
- **动态工具**：运行时注入的工具

`build_prompt()` 把这些组装成发给模型的 `Prompt` 结构体：

```rust
Prompt {
    input,                    // 对话历史（Vec<ResponseItem>）
    tools,                    // 工具列表（Vec<ToolSpec>）
    parallel_tool_calls,      // 是否允许并行工具调用
    base_instructions,        // system prompt
    personality,              // 人格设定（可选）
    output_schema,            // 结构化输出 schema（可选）
}
```

### `try_run_sampling_request`（L7140）— 处理流式响应

这是单次模型 API 调用的完整处理。Codex 使用 OpenAI Responses API，支持 SSE 和 WebSocket 两种传输方式。

```rust
async fn try_run_sampling_request(..., prompt: &Prompt, ...) {
    // 发起流式请求
    let mut stream = client_session.stream(prompt, ...).await?;
    let mut in_flight: FuturesOrdered<BoxFuture<'static, _>> = FuturesOrdered::new();
    let mut needs_follow_up = false;

    loop {
        match stream.next().await {
            ResponseEvent::OutputItemAdded(item) => {
                // 新的输出项开始（消息或工具调用）
                // → 发送 TurnItemStarted 事件给 UI（实时显示）
            }
            ResponseEvent::OutputTextDelta(delta) => {
                // 流式文本片段 → 转发给 UI（打字效果）
            }
            ResponseEvent::OutputItemDone(item) => {
                // 一个输出项完成，这是关键分支点
                let result = handle_output_item_done(&mut ctx, item, ...).await?;
                if let Some(tool_future) = result.tool_future {
                    in_flight.push_back(tool_future);  // 工具异步执行
                }
                needs_follow_up |= result.needs_follow_up;
            }
            ResponseEvent::Completed { token_usage } => {
                sess.update_token_usage_info(&turn_context, token_usage.as_ref()).await;
                break Ok(SamplingRequestResult { needs_follow_up, .. });
            }
            ResponseEvent::RateLimits(snapshot) => { /* 更新限流状态 */ }
            ResponseEvent::ReasoningSummaryDelta { .. } => { /* 推理过程流式输出 */ }
            // ...
        }
    }

    // 等所有并行工具执行完
    drain_in_flight(&mut in_flight, sess, turn_context).await?;
}
```

`ResponseEvent::OutputItemDone` 是整个流处理的核心分支点，由 `handle_output_item_done`（在 `stream_events_utils.rs`）处理：

```rust
pub(crate) async fn handle_output_item_done(ctx, item, ...) {
    match ToolRouter::build_tool_call(ctx.sess.as_ref(), item.clone()).await {
        // 模型返回了工具调用
        Ok(Some(call)) => {
            record_completed_response_item(...).await;  // 持久化到历史
            let tool_future = ctx.tool_runtime.clone().handle_tool_call(call, ...);
            output.needs_follow_up = true;
            output.tool_future = Some(Box::pin(tool_future));
        }
        // 模型返回了普通消息
        Ok(None) => {
            ctx.sess.emit_turn_item_completed(&ctx.turn_context, turn_item).await;
            record_completed_response_item(...).await;
            output.last_agent_message = last_assistant_message_from_item(&item, ...);
        }
        // 工具调用格式错误，直接回复错误给模型
        Err(FunctionCallError::RespondToModel(message)) => {
            // 把错误作为 function_call_output 写入历史
            output.needs_follow_up = true;
        }
    }
}
```

`ToolRouter::build_tool_call` 识别 `ResponseItem` 的类型：`FunctionCall`、`LocalShellCall`、`CustomToolCall`、`ToolSearchCall`，把它们统一转换成 `ToolCall` 结构体，携带 `tool_name`、`call_id` 和 `payload`。

工具调用被放进 `in_flight`（一个 `FuturesOrdered`），**并行执行**。`drain_in_flight` 在流结束后等待所有工具完成，把结果写入对话历史。

---

## 第三层：工具执行的完整流程

工具执行不是简单地"跑命令"，它经过了一个完整的审批 + 沙箱流水线，由 `ToolOrchestrator` 管理（`tools/orchestrator.rs`）。

```
ToolCallRuntime.handle_tool_call(call)
    │
    ▼
ToolRouter.dispatch_tool_call_with_code_mode_result(call)
    │
    ▼
ToolRegistry.dispatch_any(invocation)
    │
    ▼
具体 Handler（ShellHandler / ApplyPatchHandler / McpToolHandler / ...）
    │
    ▼
ToolOrchestrator.run(tool, req, ...)
    │
    ├── 1. 审批检查
    │   ├── ExecPolicy 评估（Starlark 规则）
    │   ├── 根据 approval_policy 决定是否需要用户确认
    │   └── 如果需要：暂停，发送 ExecApprovalRequest 事件，等待用户响应
    │
    ├── 2. 第一次执行（带沙箱）
    │   ├── 选择沙箱类型（Linux Landlock/seccomp、macOS Seatbelt、无沙箱）
    │   └── 在沙箱内执行命令
    │
    └── 3. 沙箱拒绝时的升级流程
        ├── 如果沙箱拒绝了（SandboxErr::Denied）
        ├── 根据 approval_policy 决定是否可以升级
        ├── 如果可以：再次请求用户审批
        └── 用户批准后：不带沙箱重试
```

审批流程的核心逻辑（`ToolOrchestrator.run`）：

```rust
// 1. 确定审批需求
let requirement = tool.exec_approval_requirement(req).unwrap_or_else(|| {
    default_exec_approval_requirement(approval_policy, &file_system_sandbox_policy)
});

match requirement {
    ExecApprovalRequirement::Skip { .. } => { /* 直接执行 */ }
    ExecApprovalRequirement::Forbidden { reason } => {
        return Err(ToolError::Rejected(reason));  // exec policy 拒绝
    }
    ExecApprovalRequirement::NeedsApproval { reason, .. } => {
        // 暂停，等用户决定
        let decision = tool.start_approval_async(req, approval_ctx).await;
        match decision {
            ReviewDecision::Denied | ReviewDecision::Abort => {
                return Err(ToolError::Rejected("rejected by user".to_string()));
            }
            ReviewDecision::Approved | ReviewDecision::ApprovedForSession => {
                already_approved = true;
            }
        }
    }
}

// 2. 选择沙箱，执行
let initial_sandbox = sandbox_manager.select_initial(
    &file_system_sandbox_policy,
    network_sandbox_policy,
    tool.sandbox_preference(),
    ...
);
let (first_result, ..) = Self::run_attempt(tool, req, &initial_attempt, ...).await;

// 3. 沙箱拒绝 → 升级
if let Err(SandboxErr::Denied { output, .. }) = first_result {
    if tool.wants_no_sandbox_approval(approval_policy) {
        // 再次请求审批，然后不带沙箱重试
        let decision = tool.start_approval_async(req, retry_ctx).await;
        // ...
        let (retry_result, ..) = Self::run_attempt(tool, req, &escalated_attempt, ...).await;
    }
}
```

这个"先沙箱执行，失败了再升级"的设计很有意思：它让大多数命令在沙箱里安全运行，只有真正需要更多权限的命令才会触发用户审批。

---

## 第四层：loop 进入前发生了什么

`run_turn` 在进入主循环之前，做了大量准备工作。按执行顺序：

**1. 上下文压缩检查（`run_pre_sampling_compact`）**

```rust
async fn run_pre_sampling_compact(sess, turn_context) {
    // 检查是否需要先压缩（上一个模型的历史太长）
    maybe_run_previous_model_inline_compact(sess, turn_context, ...).await?;

    // 检查当前 token 使用量是否超过自动压缩阈值
    let total_usage_tokens = sess.get_total_token_usage().await;
    if total_usage_tokens >= auto_compact_limit {
        run_auto_compact(sess, turn_context, InitialContextInjection::DoNotInject).await?;
    }
}
```

`auto_compact_limit` 来自模型 catalog 的配置，超过就调用模型对历史做摘要压缩，然后用压缩后的历史替换原来的。

**2. 环境上下文更新（`record_context_updates_and_set_reference_context_item`）**

注入文件变化、工作目录变化等环境信息。

**3. Skills / Plugins 解析**

从用户输入里提取 `@skill-name` 引用，把对应的 Markdown 文件内容展开成 `developer` 角色的 message，追加到对话历史。

**4. Hooks 执行**

- `run_pending_session_start_hooks()`：会话首次启动时的 hook
- `run_user_prompt_submit_hooks()`：用户消息提交后的 hook，可以注入上下文或阻断 turn

**5. 用户消息写入历史**

```rust
sess.record_user_prompt_and_emit_turn_item(turn_context, &input, response_item).await;
```

同时发送 `TurnItemStarted` / `TurnItemCompleted` 事件给 UI。

---

## 第五层：loop 结束后发生了什么

当 `needs_follow_up = false`（模型没有更多工具调用），主循环 break 之前，还有一个重要步骤：**Stop Hooks**。

```rust
if !needs_follow_up {
    let stop_outcome = sess.hooks().run_stop(stop_request).await;

    if stop_outcome.should_block {
        // Stop hook 返回了 continuation prompt
        // 把它作为新的 user message 注入历史
        sess.record_conversation_items(&turn_context, &[hook_prompt_message]).await;
        stop_hook_active = true;
        continue;  // 循环继续！
    }

    if stop_outcome.should_stop {
        break;
    }

    // 运行 after_agent hooks
    let hook_outcomes = sess.hooks().dispatch(HookPayload {
        hook_event: HookEvent::AfterAgent { ... },
    }).await;
}
```

Stop Hook 可以让已经"完成"的 loop 重新继续——这是实现自动化质量门控的关键机制。

---

## 第六层：围绕 loop 的外壳

`run_turn` 本身不是直接被调用的，它被包在两层外壳里。

### `RegularTask`（`tasks/regular.rs`）

```rust
impl SessionTask for RegularTask {
    async fn run(self, session, ctx, input, cancellation_token) -> Option<String> {
        let sess = session.clone_session();

        // 发送 TurnStarted 事件
        sess.send_event(ctx.as_ref(), EventMsg::TurnStarted(...)).await;

        // 尝试使用预热的 WebSocket 连接
        let prewarmed_client_session = sess.consume_startup_prewarm_for_regular_turn(...).await;

        // 处理 pending input：用户在模型运行时插入的消息
        let mut next_input = input;
        loop {
            let last_agent_message = run_turn(sess, ctx, next_input, ...).await;
            if !sess.has_pending_input().await {
                return last_agent_message;
            }
            next_input = Vec::new();  // 有 pending input，继续跑
        }
    }
}
```

`RegularTask` 处理了一个细节：用户在模型运行时发来的消息（`pending_input`）。这些消息不会打断当前的工具执行，而是在当前 turn 结束后，作为下一轮的输入继续处理。

### `submission_loop`（L4245）— 事件驱动的命令分发器

```rust
async fn submission_loop(sess, config, rx_sub) {
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op {
            Op::UserTurn { .. } => {
                // Op::UserTurn → Session::spawn_task(RegularTask) → RegularTask.run() → run_turn()
                handlers::user_input_or_turn(&sess, sub.id, sub.op).await;
            }
            Op::ExecApproval { id, decision, .. } => {
                // 用户批准/拒绝了工具执行
                handlers::exec_approval(&sess, id, decision).await;
            }
            Op::Interrupt => {
                handlers::interrupt(&sess).await;
            }
            Op::Compact => { handlers::compact(&sess, sub.id).await; }
            Op::Undo => { handlers::undo(&sess, sub.id).await; }
            Op::Shutdown => {
                if handlers::shutdown(&sess, sub.id).await { break; }
            }
            // ... 几十种 Op
        }
    }
}
```

这是整个系统的消息总线。外部（TUI、IDE 插件、SDK）通过 `Codex::submit(Op)` 往 channel 发消息，`submission_loop` 在另一端接收并分发。

注意 `Op::ExecApproval`：当工具执行需要用户审批时，`ToolOrchestrator` 会暂停等待，同时发送 `ExecApprovalRequest` 事件给 UI。用户点击批准/拒绝后，UI 发送 `Op::ExecApproval`，`submission_loop` 接收后调用 `handlers::exec_approval`，唤醒等待中的工具执行。这是一个完整的异步审批流程。

---

## 完整数据流

把所有层次串起来：

```
用户输入
    │
    ▼ Codex::submit(Op::UserTurn)
    │
    ▼ submission_loop 接收
    │
    ▼ Session::spawn_task(RegularTask)
    │
    ▼ RegularTask.run()
    │   ├── 发送 TurnStarted 事件
    │   └── loop（处理 pending input）
    │
    ▼ run_turn()
    │   ├── run_pre_sampling_compact（必要时压缩历史）
    │   ├── record_context_updates（注入环境上下文）
    │   ├── Skills / Plugins 解析注入
    │   ├── run_user_prompt_submit_hooks（Hook 拦截）
    │   ├── 用户消息写入历史
    │   │
    │   └── loop {
    │       ├── clone_history().for_prompt()（归一化历史）
    │       ├── run_sampling_request()
    │       │   ├── built_tools()（组装工具列表）
    │       │   ├── build_prompt()（组装请求包）
    │       │   └── try_run_sampling_request()
    │       │       ├── client_session.stream()（发起流式请求）
    │       │       ├── loop { match stream.next() }
    │       │       │   ├── OutputItemAdded → 发 TurnItemStarted 给 UI
    │       │       │   ├── OutputTextDelta → 发流式文本给 UI
    │       │       │   ├── OutputItemDone → handle_output_item_done()
    │       │       │   │   ├── 工具调用 → in_flight.push(tool_future)
    │       │       │   │   │              needs_follow_up = true
    │       │       │   │   └── 普通消息 → 发 TurnItemCompleted 给 UI
    │       │       │   └── Completed → break
    │       │       └── drain_in_flight()（等待所有工具执行完）
    │       │           └── 每个工具：ToolOrchestrator.run()
    │       │               ├── ExecPolicy 评估
    │       │               ├── 审批检查（可能暂停等用户）
    │       │               ├── 沙箱执行
    │       │               └── 结果写入历史
    │       │
    │       ├── needs_follow_up=true → continue
    │       └── needs_follow_up=false
    │           ├── run_stop_hooks()（可能让 loop 继续）
    │           └── break
    │       }
    │
    └── 返回 last_agent_message
```

---

## 关键设计决策

**`needs_follow_up` 是唯一的循环驱动信号**。整个 loop 的复杂性都收敛到这一个 bool。工具执行的结果、审批的结果、Stop Hook 的结果，最终都通过这个信号决定循环是否继续。

**工具并行执行**。`in_flight` 是一个 `FuturesOrdered`，多个工具调用可以同时运行。`ToolCallRuntime` 用一个 `RwLock` 控制并发：支持并行的工具（如文件读取）拿读锁，不支持并行的工具（如 shell 命令）拿写锁，保证互斥。

**审批是异步的**。工具执行暂停等待用户审批时，不阻塞整个 tokio runtime。`submission_loop` 继续运行，可以处理其他 Op（比如用户的 Interrupt）。

**历史归一化在每次循环都执行**。`for_prompt()` 每次都重新归一化，确保历史始终合法，即使中间有工具执行失败或被中断。

**沙箱失败可以升级**。第一次在沙箱里执行失败，不是直接报错，而是可以请求用户审批后不带沙箱重试。这让"默认安全，需要时升级"成为可能。

---

## 一句话

Codex 的 agent loop 是 `run_turn` 里的那个 `loop`：拿历史 → 调模型 → 有工具调用就经过审批+沙箱执行、写回历史、继续；没有就跑 Stop Hook、可能继续也可能结束。`needs_follow_up` 是驱动循环的唯一信号。其他一切——压缩、Hooks、Skills、流式 UI、并行工具执行、审批升级——都是在这个骨架的固定位置插入的扩展点。