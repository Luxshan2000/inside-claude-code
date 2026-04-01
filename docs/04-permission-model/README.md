# 🔐 Permission Model

> **Security by design.** How Claude Code controls what the agent can and cannot do.

[← Back to Main](../../README.md) | [← Tool System](../03-tool-system/README.md)

---

## The Problem

An AI agent that can run `bash` commands has the power to `rm -rf /`. The permission model is the guardrail that prevents catastrophic actions while still allowing the agent to be useful.

---

## Three-Tier Permission Hierarchy

```mermaid
graph TD
    subgraph "🟢 Tier 1: ReadOnly"
        RO["Safe, read-only operations"]
        RO1["read_file — View file contents"]
        RO2["glob_search — Find files by pattern"]
        RO3["grep_search — Search file contents"]
        RO4["agent — Spawn sub-agents"]
        RO5["todo_write — Task tracking"]
    end

    subgraph "🟡 Tier 2: WorkspaceWrite"
        WW["File modifications within workspace"]
        WW1["write_file — Create/overwrite files"]
        WW2["edit_file — Apply targeted edits"]
        WW3["notebook_edit — Edit notebook cells"]
    end

    subgraph "🔴 Tier 3: DangerFullAccess"
        DA["Unrestricted system access"]
        DA1["bash — Shell command execution"]
        DA2["powershell — Windows shell"]
        DA3["web_search — Internet search"]
        DA4["web_fetch — HTTP requests"]
        DA5["repl — Interactive shells"]
    end

    RO --> WW --> DA
```

---

## Permission Check Flow

```mermaid
flowchart TD
    START["Tool requested:<br/>bash('cargo test')"] --> LOOKUP["Look up tool's<br/>required permission"]
    LOOKUP --> COMPARE{"current_mode ≥<br/>required_mode?"}

    COMPARE -->|"Yes"| ALLOW["✅ Execute tool"]

    COMPARE -->|"No"| MODE{"Current mode?"}
    MODE -->|"Allow (dontAsk)"| ALLOW
    MODE -->|"Prompt"| PROMPT["🔔 Ask user:<br/>Allow bash execution?"]
    PROMPT -->|"Y"| ALLOW
    PROMPT -->|"N"| DENY["❌ Permission Denied"]
    MODE -->|"ReadOnly/WorkspaceWrite"| DENY

    style ALLOW fill:#22c55e,color:#fff
    style DENY fill:#ef4444,color:#fff
    style PROMPT fill:#f59e0b,color:#fff
```

---

## Permission Mode Hierarchy

```
┌────────────────────────────────────────────────────┐
│ Mode Ordering (lowest → highest)                   │
├────────────────────────────────────────────────────┤
│                                                    │
│   ReadOnly  <  WorkspaceWrite  <  DangerFullAccess │
│      │              │                    │         │
│   read only      + file writes      + everything  │
│                                                    │
│   Special modes:                                   │
│   • Prompt → Interactively ask user on escalation  │
│   • Allow (dontAsk) → Auto-approve everything      │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Interactive Permission Prompting

When mode is `Prompt` and a tool requires escalation:

```mermaid
sequenceDiagram
    participant RT as 🧠 Runtime
    participant PM as 🔐 PermissionPolicy
    participant PR as 🔔 Prompter
    participant U as 👤 User

    RT->>PM: check("bash", DangerFullAccess)
    PM->>PM: current=WorkspaceWrite < DangerFullAccess
    PM->>PR: decide(PermissionRequest)

    PR->>U: "Allow bash execution?<br/>Command: cargo test<br/>[Y/n]"
    U-->>PR: "Y"
    PR-->>PM: Allow

    PM-->>RT: ✅ Proceed
```

---

## Permission Engine — Class Diagram

```mermaid
classDiagram
    class PermissionEngine {
        -active_mode: PermissionMode
        -tool_requirements: Map of tool → required mode
        -prompter: Optional interactive prompter
        +check(tool_name, input) Decision
    }

    class PermissionMode {
        <<enumeration>>
        ReadOnly
        WorkspaceWrite
        DangerFullAccess
        Prompt
        Allow
    }

    class PermissionRequest {
        +tool_name
        +tool_input
        +current_mode
        +required_mode
    }

    class Decision {
        <<enumeration>>
        Allow
        Deny with reason
    }

    PermissionEngine --> PermissionMode
    PermissionEngine ..> PermissionRequest
    PermissionEngine ..> Decision
```

---

## Configuration

Permissions can be set at multiple levels:

```mermaid
flowchart LR
    CLI["CLI flag:<br/>--permission-mode"] --> RESOLVE
    CONFIG[".claude.json:<br/>permissions.defaultMode"] --> RESOLVE
    DEFAULT["Default:<br/>DangerFullAccess"] --> RESOLVE
    RESOLVE["Resolution:<br/>CLI > Config > Default"] --> ACTIVE["Active Mode"]
```

### `.claude.json` Example

```json
{
  "permissions": {
    "defaultMode": "dontAsk"
  }
}
```

### CLI Override

```bash
claw --permission-mode read-only
claw --permission-mode workspace-write
claw --permission-mode danger-full-access
```

### Runtime Override

```
> /permissions danger-full-access
Permission mode changed to DangerFullAccess
```

---

## Decision Matrix

| Current Mode | Tool Requires | Result |
|-------------|---------------|--------|
| ReadOnly | ReadOnly | ✅ Allow |
| ReadOnly | WorkspaceWrite | ❌ Deny |
| ReadOnly | DangerFullAccess | ❌ Deny |
| WorkspaceWrite | ReadOnly | ✅ Allow |
| WorkspaceWrite | WorkspaceWrite | ✅ Allow |
| WorkspaceWrite | DangerFullAccess | ❌ Deny |
| DangerFullAccess | Any | ✅ Allow |
| Prompt | Higher than current | 🔔 Ask user |
| Allow (dontAsk) | Any | ✅ Allow |

---

## What's Next?

- **[MCP Integration →](../05-mcp-integration/README.md)** — External tools also go through permissions
- **[Hook System →](../06-hook-system/README.md)** — Hooks can override permission decisions
- **[Sandbox Execution →](../12-sandbox-execution/README.md)** — OS-level isolation beyond permissions

---

[← Tool System](../03-tool-system/README.md) | [Next: MCP Integration →](../05-mcp-integration/README.md)
