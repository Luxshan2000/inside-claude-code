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

## Module Dependency Graph

```mermaid
graph LR
    CLI["CLI Interface"] --> RT["Runtime Core"]
    CLI --> CMDS["Commands"]
    CLI --> TOOLS["Tools"]
    RT --> API["API Client"]
    RT --> TOOLS
```

| Module | Purpose |
|--------|---------|
| **API Client** | HTTP communication, SSE streaming, auth, retry |
| **Commands** | Slash command registry & metadata |
| **Runtime Core** | Conversation loop, config, permissions, MCP, sessions |
| **CLI Interface** | Terminal REPL, markdown rendering, input handling |
| **Tools** | Built-in tool specifications & execution |

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
The implementation enforces strict memory safety — no unsafe operations allowed. Memory management is handled by the language's ownership system.

---

## Runtime Component Map

```mermaid
graph TB
    subgraph "Runtime Core"
        CONV["Conversation Engine<br/>Agentic Loop"]
        CONFIG["Configuration Loader"]
        PERM["Permission Engine"]
        HOOKS["Hook Runner"]
        PROMPT["System Prompt Builder"]
        SESSION["Session Manager"]
        MCP["MCP Protocol Client"]
        COMPACT["Context Compactor"]
        OAUTH["OAuth Handler"]
        USAGE["Usage & Cost Tracker"]
        SANDBOX["Sandbox Manager"]
        BASH["Shell Executor"]
        BOOT["Bootstrap Orchestrator"]
        REMOTE["Remote Proxy"]
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
