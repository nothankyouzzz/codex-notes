# Codex Memory 功能

## 状态

- Stage: `UnderDevelopment`（低于 Experimental，不出现在 `/experimental` 菜单）
- 默认关闭，需手动在 `config.toml` 开启：
  ```toml
  [features]
  memories = true
  ```
- 正在活跃开发中（过去 60 天 93 个相关 commit，最近一次 2 天前）

---

## 一句话

Memory 是 Codex 的跨会话学习系统：在后台把历史对话提炼成结构化记忆，新会话开始时注入给模型，让模型不需要你反复解释偏好、项目背景和踩过的坑。

---

## 整体架构：写路径 + 读路径

```
过去的对话（rollouts）
    │
    ▼ 启动时后台异步运行
┌─────────────────────────────────────────┐
│           写路径（两阶段流水线）          │
│                                         │
│  Phase 1: 逐会话提取                    │
│  gpt-5.1-codex-mini（低推理强度）        │
│  → raw_memory + rollout_summary         │
│  → 存入 SQLite state DB                 │
│                                         │
│  Phase 2: 全局整合                      │
│  gpt-5.3-codex（中推理强度）             │
│  → 启动一个子 agent 来写文件             │
│  → 更新 ~/.codex/memories/ 下的文件     │
└─────────────────────────────────────────┘
    │
    ▼ 新会话开始时
┌─────────────────────────────────────────┐
│           读路径                         │
│  memory_summary.md 注入 system prompt   │
│  模型可以主动读 MEMORY.md / rollout_summaries/ │
└─────────────────────────────────────────┘
```

---

## Phase 1：逐会话提取

触发条件：每次 Codex 启动时，后台异步扫描符合条件的历史会话：
- 在配置的时间窗口内（默认 30 天）
- 距上次活动超过一定时间（默认 6 小时，避免总结还在进行的会话）
- 还没被处理过（SQLite 里有 lease 机制防止重复处理）

产出（发给 `gpt-5.1-codex-mini`，要求结构化输出）：
- `raw_memory`：结构化记忆条目，包含用户偏好、可复用知识、失败教训
- `rollout_summary`：这次会话的详细摘要，供未来 agent 参考
- `rollout_slug`：文件名用的短标识

模型被明确要求：
- 优先提取**用户偏好**（用户反复纠正、中断、要求重做的地方）
- 提取**高价值的程序性知识**（省时的命令、路径、失败模式）
- 如果这次会话没有值得保存的内容，返回空字段（no-op 是被鼓励的）

结果存入 SQLite state DB，并写到 `~/.codex/memories/rollout_summaries/`。

---

## Phase 2：全局整合

Phase 1 产出分散的 per-rollout 记忆，Phase 2 把它们整合成统一文件。

做法：启动一个**内部子 agent**（无审批、无网络、只有本地写权限），让它：
1. 读取所有 Phase 1 的输出（`raw_memories.md`）
2. 对比上次整合结果，计算 added / retained / removed 的 diff
3. 更新 `~/.codex/memories/` 下的文件

产出的文件结构：
```
~/.codex/memories/
├── memory_summary.md      ← 注入 system prompt 的摘要（≤500字）
├── MEMORY.md              ← 可检索的详细记忆手册
├── raw_memories.md        ← Phase 1 输出的合并（临时中间文件）
├── rollout_summaries/     ← 每个会话的详细摘要
│   └── 2026-03-29T...-fix-login-bug.md
└── skills/                ← 可复用的操作手册（可选）
    └── my-skill/
        └── SKILL.md
```

有遗忘机制：超过 `max_unused_days` 没被使用的记忆会被移除，防止无限膨胀。

---

## 读路径：记忆如何进入对话

**自动注入**：每次新会话开始，`memory_summary.md` 的内容被注入到 system prompt（通过 `build_memory_tool_developer_instructions()`），模型每次调用都能看到。

**按需查阅**：`memory_summary.md` 里有导航指引，告诉模型：
- 如果任务相关，去读 `MEMORY.md`（可 grep 的详细手册）
- 如果需要更多证据，去读 `rollout_summaries/` 下的具体文件

模型使用记忆后需附上 `<oai-mem-citation>` 标注，说明引用了哪些文件的哪些行。

---

## 核心设计原则

1. **用户偏好优先于程序性知识**：用户反复纠正、中断、要求重做的地方是最高价值信号。
2. **no-op 是被鼓励的**：没有值得保存的内容时，模型应返回空字段，不写任何东西。
3. **证据驱动，不发明事实**：记忆必须基于对话中实际发生的事。
4. **渐进式披露**：`memory_summary.md` 轻量摘要（每次注入）→ `MEMORY.md` 详细手册（按需查）→ `rollout_summaries/` 原始证据（需要时才读）。

---

## 配置选项

```toml
[memories]
generate_memories = true          # 是否生成记忆（写路径）
use_memories = true               # 是否使用记忆（读路径）
extract_model = "gpt-5.1-codex-mini"    # Phase 1 用的模型
consolidation_model = "gpt-5.3-codex"  # Phase 2 用的模型
max_rollout_age_days = 30         # 只处理多少天内的会话
min_rollout_idle_hours = 6        # 会话结束多久后才处理
max_unused_days = 30              # 记忆多久没用就删除
no_memories_if_mcp_or_web_search = false  # 有 MCP/搜索时不生成记忆
```

---

## 额外限制

- 必须有 SQLite state DB 可用
- 不能是 ephemeral 会话
- 不能是子 agent 会话
- Phase 2 整合时会把自身的 `MemoryTool` feature 关掉（防止递归）
