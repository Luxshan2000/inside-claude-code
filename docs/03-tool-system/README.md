# 🔧 Tool System

> **The hands of the agent.** How Claude Code reads files, writes code, runs commands, and interacts with the world.

[← Back to Main](../../README.md) | [← Memory & Context](../02-memory-and-context/README.md)

---

## Overview

Claude Code isn't just a chatbot — it can *do things*. The tool system is what gives it hands. Every action — reading a file, editing code, running bash — goes through a unified tool execution pipeline.

---

## Tool Architecture

```mermaid
graph TB
    subgraph "Tool Registration"
        SPEC["Tool Specifications<br/>(name, description, input_schema)"]
        REG["StaticToolExecutor<br/>BTreeMap<name, handler>"]
    end

    subgraph "Tool Execution Pipeline"
        REQ["Tool Request<br/>(name + JSON input)"]
        PERM["🔐 Permission Check"]
        HOOK_PRE["⚡ PreToolUse Hook"]
        EXEC["🏃 Handler Execution"]
        HOOK_POST["⚡ PostToolUse Hook"]
        RES["📨 Tool Result"]
    end

    SPEC --> REG
    REQ --> PERM
    PERM -->|"Allowed"| HOOK_PRE
    PERM -->|"Denied"| DENY["❌ Permission Error"]
    HOOK_PRE -->|"exit 0"| EXEC
    HOOK_PRE -->|"exit 2"| HOOK_DENY["❌ Hook Denied"]
    EXEC --> HOOK_POST
    HOOK_POST --> RES
```

---

## Built-in Tools — Complete Catalog

### File Operations

| Tool | Permission | Description |
|------|-----------|-------------|
| `read_file` | 🟢 ReadOnly | Read file contents with optional line range |
| `write_file` | 🟡 WorkspaceWrite | Create or overwrite files, returns git diff |
| `edit_file` | 🟡 WorkspaceWrite | Apply targeted string replacements |
| `glob_search` | 🟢 ReadOnly | Find files by pattern (e.g., `**/*.rs`) |
| `grep_search` | 🟢 ReadOnly | Search file contents with regex |

### Execution

| Tool | Permission | Description |
|------|-----------|-------------|
| `bash` | 🔴 DangerFullAccess | Run shell commands with timeout and background support |
| `powershell` | 🔴 DangerFullAccess | Windows PowerShell execution |
| `repl` | 🔴 DangerFullAccess | Interactive Python/Node.js REPL |

### Web

| Tool | Permission | Description |
|------|-----------|-------------|
| `web_search` | 🔴 DangerFullAccess | Search the web |
| `web_fetch` | 🔴 DangerFullAccess | Fetch and process URL content |

### Orchestration

| Tool | Permission | Description |
|------|-----------|-------------|
| `agent` | 🟢 ReadOnly | Spawn sub-agent for parallel tasks |
| `skill` | 🟢 ReadOnly | Execute SKILL.md file workflows |
| `todo_write` | 🟢 ReadOnly | Task tracking and management |

### Utility

| Tool | Permission | Description |
|------|-----------|-------------|
| `notebook_edit` | 🟡 WorkspaceWrite | Edit Jupyter notebook cells |
| `tool_search` | 🟢 ReadOnly | Search for available tools |
| `config` | 🟢 ReadOnly | Inspect configuration |

---

## Tool Execution — Sequence Diagram

```mermaid
sequenceDiagram
    participant API as 🌐 API Response
    participant RT as 🧠 Runtime
    participant PM as 🔐 PermissionPolicy
    participant HK as ⚡ HookRunner
    participant TE as 🔧 ToolExecutor

    API->>RT: tool_use: { id, name, input }

    RT->>PM: check(tool_name, required_permission)
    alt Permission Denied
        PM-->>RT: Deny(reason)
        RT-->>API: tool_result: { error: "Permission denied" }
    else Permission Granted
        PM-->>RT: Allow

        RT->>HK: run(PreToolUse, tool_name, input)
        alt Hook Denies (exit code 2)
            HK-->>RT: Denied(stdout_message)
            RT-->>API: tool_result: { error: "Hook denied" }
        else Hook Allows (exit code 0)
            HK-->>RT: Allowed(optional_stdout)

            RT->>TE: execute(tool_name, input_json)
            TE-->>RT: Ok(output) or Err(ToolError)

            RT->>HK: run(PostToolUse, tool_name, output)
            HK-->>RT: Done

            RT-->>API: tool_result: { output, is_error }
        end
    end
```

---

## Tool Input Schema Pattern

Every tool defines its input as a JSON Schema:

```
┌─────────────────────────────────────────────────┐
│ Tool: edit_file                                  │
├─────────────────────────────────────────────────┤
│ Input Schema:                                    │
│ {                                                │
│   "file_path": string    (required)              │
│   "old_string": string   (required)              │
│   "new_string": string   (required)              │
│   "replace_all": boolean (optional, default: false)│
│ }                                                │
├─────────────────────────────────────────────────┤
│ Output: "Edit applied" + git diff                │
│ Error:  "old_string not found" / "not unique"    │
└─────────────────────────────────────────────────┘
```

---

## ToolExecutor — Registration Pattern

```mermaid
classDiagram
    class ToolExecutor {
        <<trait>>
        +execute(tool_name, input) Result~String, ToolError~
    }

    class StaticToolExecutor {
        -handlers: BTreeMap~String, Handler~
        +register(name, handler) Self
        +execute(tool_name, input) Result
    }

    class Handler {
        <<type alias>>
        Box~dyn FnMut(input) → Result~
    }

    ToolExecutor <|.. StaticToolExecutor
    StaticToolExecutor *-- Handler
```

### Builder Pattern

```
StaticToolExecutor::new()
    .register("bash", handle_bash)
    .register("read_file", handle_read)
    .register("write_file", handle_write)
    .register("edit_file", handle_edit)
    .register("glob_search", handle_glob)
    .register("grep_search", handle_grep)
    // ... 12 more tools
```

---

## MCP Tools — Dynamic Extension

Beyond built-in tools, Claude Code can load tools from **MCP (Model Context Protocol) servers**:

```mermaid
flowchart LR
    subgraph "Built-in"
        B1["bash"]
        B2["read_file"]
        B3["write_file"]
        BN["...15 more"]
    end

    subgraph "MCP Servers"
        M1["mcp__github__create_pr"]
        M2["mcp__db__query"]
        M3["mcp__custom__tool"]
    end

    ALL["All Available Tools<br/>(sent to API)"]

    B1 --> ALL
    B2 --> ALL
    B3 --> ALL
    BN --> ALL
    M1 --> ALL
    M2 --> ALL
    M3 --> ALL
```

See **[MCP Integration →](../05-mcp-integration/README.md)** for the full deep dive.

---

## Permission Tiers Per Tool

```mermaid
graph TD
    subgraph "🟢 ReadOnly"
        T1["read_file"]
        T2["glob_search"]
        T3["grep_search"]
        T4["agent"]
        T5["todo_write"]
        T6["tool_search"]
    end

    subgraph "🟡 WorkspaceWrite"
        T7["write_file"]
        T8["edit_file"]
        T9["notebook_edit"]
    end

    subgraph "🔴 DangerFullAccess"
        T10["bash"]
        T11["powershell"]
        T12["repl"]
        T13["web_search"]
        T14["web_fetch"]
    end
```

---

## What's Next?

- **[Permission Model →](../04-permission-model/README.md)** — How the permission checks work
- **[MCP Integration →](../05-mcp-integration/README.md)** — Extending with external tools
- **[Hook System →](../06-hook-system/README.md)** — Wrapping tools with custom logic

---

[← Memory & Context](../02-memory-and-context/README.md) | [Next: Permission Model →](../04-permission-model/README.md)
