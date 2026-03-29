# Codex 项目骨架

## 一句话

本地运行的编程 AI Agent，核心用 Rust 实现，TypeScript CLI 包装，支持 TUI / IDE 插件 / SDK 三种入口。

## 顶层结构

```
codex-rs/     Rust 主体（60+ crate）
codex-cli/    npm 包 @openai/codex（薄壳，调用 Rust 二进制）
sdk/          TypeScript + Python SDK
```

## Rust 核心分层

```
入口层        tui（终端界面）/ cli（命令行）/ app-server（IDE WebSocket）
引擎层        core（agent loop）/ protocol / state / config / hooks / skills
后端层        backend-client / connectors / chatgpt / lmstudio / ollama
执行层        exec / execpolicy / sandboxing / linux-sandbox
工具层        tools / apply-patch / file-search / git-utils / mcp-server
```

## 数据流

```
用户输入（TUI / IDE）
    ↓
Core（agent loop）
    ↓
Backend Client → OpenAI API / 本地模型
    ↓
Exec（沙箱）→ 工具执行结果
```

## 关键设计

- Rust 为主体，60+ crate 精细拆分
- 多层沙箱（Linux Landlock/seccomp、macOS Seatbelt）
- 多后端（OpenAI、ChatGPT、LM Studio、Ollama）
- MCP 既是客户端也是服务端

→ 细节见 `deep-dive/` 各文件
