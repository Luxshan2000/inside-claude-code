# 🏗️ Architecture Overview

> **The big picture.** How all the pieces of Claude Code fit together.

[← Back to Main](../../README.md)

---

## High-Level Architecture

Claude Code is built as a **modular Rust workspace** with 6 crates, each owning a distinct responsibility. The system follows a layered architecture where the CLI sits on top, the runtime orchestrates everything, and specialized crates handle API communication, tool execution, and command parsing.

```mermaid
graph TB
    subgraph "Layer 4: User Interface"
        CLI["rusty-claude-cli<br/>REPL, Rendering, Input"]
    end

    subgraph "Layer 3: Orchestration"
        RT["runtime<br/>Conversation Loop, Config,<br/>Permissions, MCP, Sessions"]
    end

    subgraph "Layer 2: Capabilities"
        TOOLS["tools<br/>18 Built-in Tools"]
        CMDS["commands<br/>15+ Slash Commands"]
        COMPAT["compat-harness<br/>TS Parity Analysis"]
    end

    subgraph "Layer 1: Communication"
        API["api<br/>HTTP Client, SSE Streaming,<br/>Auth, Retry Logic"]
    end

    subgraph "External"
        ANTHROPIC["Anthropic API"]
        MCPEXT["MCP Servers"]
        FS["File System"]
        SHELL["Shell / Bash"]
    end

    CLI --> RT
    CLI --> CMDS
    RT --> API
    RT --> TOOLS
    RT --> MCPEXT
    API --> ANTHROPIC
    TOOLS --> FS
    TOOLS --> SHELL
```

---

## Crate Dependency Graph

```mermaid
graph LR
    CLI["rusty-claude-cli"] --> RT["runtime"]
    CLI --> CMDS["commands"]
    CLI --> TOOLS["tools"]
    RT --> API["api"]
    RT --> TOOLS
    COMPAT["compat-harness"] -.-> RT
```

| Crate | Lines | Purpose |
|-------|-------|---------|
| `api` | ~1,500 | HTTP client, SSE parser, auth, retry |
| `commands` | ~470 | Slash command registry & metadata |
| `compat-harness` | ~200 | TypeScript manifest extraction for parity |
| `runtime` | ~5,300 | **Core** — conversation loop, config, permissions, MCP, sessions |
| `rusty-claude-cli` | ~3,900 | CLI binary, REPL, markdown rendering |
| `tools` | ~4,240 | Built-in tool specifications & execution |

---

## Data Flow: A Single User Turn

This is what happens from the moment you type a message to when you see a response:

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant CLI as 🖥️ CLI/REPL
    participant RT as 🧠 Runtime
    participant PM as 🔐 Permissions
    participant API as 🌐 Anthropic API
    participant T as 🔧 Tools
    participant H as ⚡ Hooks

    U->>CLI: Type message
    CLI->>RT: Submit user message
    RT->>RT: Build system prompt<br/>(CLAUDE.md + git + config)
    RT->>RT: Check token budget
    RT->>API: Stream request<br/>(messages + tools + system)

    loop Until end_turn
        API-->>RT: SSE: text_delta
        RT-->>CLI: Display text chunk
        API-->>RT: SSE: tool_use
        RT->>PM: Check permission for tool
        PM-->>RT: Allowed / Denied / Prompt
        RT->>H: PreToolUse hook
        H-->>RT: Allow (exit 0) / Deny (exit 2)
        RT->>T: Execute tool
        T-->>RT: Tool result
        RT->>H: PostToolUse hook
        RT->>API: Send tool_result
    end

    API-->>RT: SSE: message_stop
    RT->>RT: Track usage & cost
    RT->>RT: Auto-compact if needed
    RT-->>CLI: Turn complete
    CLI-->>U: Display final response
```

---

## Core Design Principles

### 1. Streaming-First
Everything is built around **Server-Sent Events (SSE)**. The API streams tokens incrementally, and the CLI renders them in real-time. No waiting for full responses.

### 2. Permission-by-Default
Every tool declares its required permission level. The runtime enforces this before any execution. Nothing runs without explicit authorization.

### 3. Context-Aware
The system automatically discovers project context — `CLAUDE.md` files, git status, config hierarchies — and injects it into every conversation.

### 4. Composable Tools
Tools are pluggable. Built-in tools handle common operations, MCP servers extend capabilities, and hooks wrap everything with custom logic.

### 5. Memory-Safe
The Rust implementation uses `#![forbid(unsafe_code)]` — zero unsafe blocks. Memory management is handled entirely by Rust's ownership system.

---

## Runtime Component Map

```mermaid
graph TB
    subgraph "runtime crate"
        CONV["conversation.rs<br/>Agentic Loop"]
        CONFIG["config.rs<br/>Config Loading"]
        PERM["permissions.rs<br/>Authorization"]
        HOOKS["hooks.rs<br/>Hook Runner"]
        PROMPT["prompt.rs<br/>System Prompt"]
        SESSION["session.rs<br/>Persistence"]
        MCP["mcp.rs<br/>MCP Protocol"]
        COMPACT["compact.rs<br/>Compaction"]
        OAUTH["oauth.rs<br/>OAuth PKCE"]
        USAGE["usage.rs<br/>Cost Tracking"]
        SANDBOX["sandbox.rs<br/>Isolation"]
        BASH["bash.rs<br/>Shell Exec"]
        BOOT["bootstrap.rs<br/>Startup"]
        REMOTE["remote.rs<br/>Proxy Support"]
    end

    CONV --> CONFIG
    CONV --> PERM
    CONV --> HOOKS
    CONV --> PROMPT
    CONV --> SESSION
    CONV --> MCP
    CONV --> COMPACT
    CONV --> USAGE
    CONFIG --> OAUTH
    PERM --> SANDBOX
    BASH --> SANDBOX
```

---

## What's Next?

Now that you see the big picture, dive into any specific subsystem:

- **[The Conversation Loop →](../01-conversation-loop/README.md)** — The beating heart of the system
- **[Memory & Compaction →](../02-memory-and-context/README.md)** — How infinite conversations work
- **[Tool System →](../03-tool-system/README.md)** — 18 built-in tools and how they execute

---

[← Back to Main](../../README.md) | [Next: Conversation Loop →](../01-conversation-loop/README.md)
