# OpenAI Codex — 项目枝干概览

## 一句话

OpenAI 的 **Codex CLI**：一个本地运行的编程 AI Agent，通过终端（TUI）或 IDE 插件与开发者交互，核心逻辑用 Rust 实现，外围有 TypeScript CLI 包装和 SDK。

---

## 顶层目录结构

```
.
├── codex-rs/          # 🦀 Rust 主体（~60+ crate 的 Cargo workspace）
├── codex-cli/         # 📦 npm 包 @openai/codex，薄壳启动器（调用 Rust 二进制）
├── sdk/               # SDK（TypeScript + Python）
├── docs/              # 用户/开发者文档
├── scripts/           # CI/构建/调试脚本
├── tools/             # 代码质量工具（如 argument-comment-lint）
├── patches/           # 第三方补丁
├── third_party/       # 第三方依赖
└── 构建配置            # Bazel (MODULE.bazel) + pnpm + Nix
```

---

## Rust 核心 (`codex-rs/`) — 按职责分层

### 1. 入口 & 用户界面
| crate | 作用 |
|---|---|
| `tui` | 终端 UI（基于 ratatui），主交互界面 |
| `cli` | 命令行参数解析、入口 |

### 2. 应用服务层（App Server）
| crate | 作用 |
|---|---|
| `app-server` | WebSocket/JSON-RPC 服务，供 IDE 插件等外部客户端连接 |
| `app-server-protocol` | 协议定义（v1/v2），自动生成 TypeScript 类型 |
| `app-server-client` | 客户端库 |
| `app-server-test-client` | 测试用客户端 |

### 3. 核心引擎
| crate | 作用 |
|---|---|
| `core` | Agent 核心逻辑（对话循环、工具调用编排、沙箱管理等） |
| `protocol` | 内部协议/消息类型 |
| `state` | 会话状态管理 |
| `config` | 配置读写（config.toml） |
| `instructions` | 系统指令/提示词管理 |
| `hooks` | Agent Hook 机制 |
| `skills` / `core-skills` | 技能系统 |
| `features` / `rollout` | Feature flag / 灰度发布 |

### 4. 后端通信
| crate | 作用 |
|---|---|
| `backend-client` | 与 OpenAI API 通信 |
| `codex-client` | Codex 专用客户端封装 |
| `codex-api` | API 模型定义 |
| `responses-api-proxy` | Responses API 代理 |
| `connectors` | 连接器抽象（支持多后端） |
| `chatgpt` | ChatGPT 登录/认证流程 |
| `login` | 登录逻辑 |
| `lmstudio` / `ollama` | 本地模型支持（LM Studio、Ollama） |

### 5. 工具执行 & 沙箱
| crate | 作用 |
|---|---|
| `exec` | 命令执行引擎 |
| `exec-server` | 执行服务（长驻进程） |
| `execpolicy` / `execpolicy-legacy` | 执行策略（允许/拒绝规则） |
| `sandboxing` | 沙箱抽象层 |
| `linux-sandbox` | Linux 沙箱（Landlock/seccomp） |
| `process-hardening` | 进程加固 |
| `shell-command` | Shell 命令构建 |
| `shell-escalation` | 权限提升处理 |

### 6. 工具 & 文件操作
| crate | 作用 |
|---|---|
| `tools` | Agent 可用工具集 |
| `apply-patch` | 补丁应用 |
| `file-search` | 文件搜索 |
| `git-utils` | Git 操作工具 |
| `mcp-server` | MCP 服务端实现 |
| `rmcp-client` | MCP 客户端 |
| `code-mode` | 代码模式处理 |
| `plugin` | 插件系统 |

### 7. 基础设施 & 工具库 (`utils/`)
| crate | 作用 |
|---|---|
| `absolute-path` | 路径规范化 |
| `home-dir` | 主目录解析 |
| `pty` | 伪终端 |
| `cache` | 缓存 |
| `image` | 图片处理 |
| `fuzzy-match` | 模糊匹配 |
| `stream-parser` | SSE 流解析 |
| `template` | 模板引擎 |
| `rustls-provider` | TLS |
| `sleep-inhibitor` | 防休眠 |
| ... 等十余个小工具 crate |

### 8. 可观测性
| crate | 作用 |
|---|---|
| `analytics` | 使用分析 |
| `otel` | OpenTelemetry 集成 |
| `feedback` | 用户反馈 |
| `ansi-escape` | ANSI 转义处理 |
| `terminal-detection` | 终端能力检测 |

---

## TypeScript / Node 侧

- **`codex-cli/`** — npm 发布包 `@openai/codex`，本质是一个薄壳，内嵌 Rust 编译的二进制，通过 `bin/codex.js` 启动。
- **`sdk/typescript/`** — TypeScript SDK，供第三方集成。
- **`sdk/python/`** + **`sdk/python-runtime/`** — Python SDK。

---

## 构建系统

- **Cargo** — Rust 主构建
- **Bazel** — CI/跨语言构建（`MODULE.bazel`、`BUILD.bazel`）
- **pnpm** — JS/TS 包管理
- **Nix** — 可选的可复现构建环境
- **just** — 任务运行器（`justfile`），常用命令如 `just fmt`、`just fix`、`just test`

---

## 数据流简图

```
用户输入 (终端/IDE)
    │
    ▼
┌─────────┐     ┌──────────────┐
│  TUI    │ or  │  App Server  │  ← IDE 通过 WebSocket/JSON-RPC 连接
└────┬────┘     └──────┬───────┘
     │                 │
     ▼                 ▼
┌──────────────────────────┐
│         Core             │  ← Agent 循环：发送 prompt → 解析响应 → 调用工具
│  (对话管理/工具编排/状态) │
└────────────┬─────────────┘
             │
     ┌───────┼───────┐
     ▼       ▼       ▼
  Backend  Exec    MCP/Tools
  Client   (沙箱)  (文件/搜索/Git...)
     │
     ▼
  OpenAI API / 本地模型
```

---

## 关键设计特点

1. **Rust 为主体** — 性能、安全、跨平台，60+ crate 精细拆分
2. **沙箱执行** — 多层沙箱（Linux Landlock/seccomp、macOS Seatbelt），命令执行有策略控制
3. **多入口** — TUI 直接用、App Server 给 IDE 插件用、SDK 给第三方集成
4. **多后端** — 支持 OpenAI API、ChatGPT 登录、LM Studio、Ollama 等本地模型
5. **MCP 支持** — 既是 MCP 客户端也是 MCP 服务端
6. **插件/技能系统** — 可扩展的工具和技能
