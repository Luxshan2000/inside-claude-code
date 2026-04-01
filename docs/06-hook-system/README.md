# ⚡ Hook System

> **Intercept everything.** Run custom logic before and after every tool execution.

[← Back to Main](../../README.md) | [← MCP Integration](../05-mcp-integration/README.md)

---

## What Are Hooks?

Hooks are shell commands that run at tool execution boundaries. They give you the power to:
- **Block dangerous operations** before they execute
- **Log every tool call** for auditing
- **Transform inputs or outputs** programmatically
- **Enforce custom policies** beyond the built-in permission model

---

## Hook Events

```mermaid
flowchart LR
    REQ["Tool Request"] --> PRE["⚡ PreToolUse"]
    PRE -->|"exit 0: Allow"| EXEC["🏃 Execute Tool"]
    PRE -->|"exit 2: Deny"| BLOCK["❌ Blocked"]
    EXEC --> POST["⚡ PostToolUse"]
    POST --> RESULT["📨 Result"]
```

| Event | When | Can Block? |
|-------|------|-----------|
| `PreToolUse` | Before tool execution | ✅ Yes (exit code 2) |
| `PostToolUse` | After tool completion | ❌ No (informational) |

---

## Hook Execution Flow — Detailed

```mermaid
flowchart TD
    START["Tool execution requested"] --> CONFIG{"Hook configured<br/>for this event?"}

    CONFIG -->|"No"| SKIP["Skip hooks,<br/>proceed to execution"]

    CONFIG -->|"Yes"| SPAWN["Spawn subprocess:<br/>shell -lc 'command'"]
    SPAWN --> STDIN["Pass JSON payload<br/>via stdin"]
    STDIN --> ENV["Set environment<br/>variables"]
    ENV --> WAIT["Wait for process<br/>to complete"]

    WAIT --> EXIT{"Exit code?"}
    EXIT -->|"0"| ALLOW["✅ Allowed<br/>Capture stdout"]
    EXIT -->|"2"| DENY["❌ Denied<br/>Capture stdout as reason"]
    EXIT -->|"Other"| WARN["⚠️ Warning<br/>Continue anyway"]

    ALLOW --> NEXT["Continue to<br/>tool execution"]
    DENY --> ERROR["Return hook-denied<br/>error as tool result"]
    WARN --> NEXT
```

---

## Hook Payload — What Your Script Receives

### stdin (JSON)

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "bash",
  "tool_input": "rm -rf /tmp/test",
  "tool_input_json": "{\"command\": \"rm -rf /tmp/test\"}"
}
```

For `PostToolUse`, additional fields:
```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "bash",
  "tool_input": "cargo test",
  "tool_output": "test result: ok. 5 passed",
  "tool_result_is_error": false
}
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `HOOK_EVENT` | `PreToolUse` or `PostToolUse` |
| `HOOK_TOOL_NAME` | Name of the tool being called |
| `HOOK_TOOL_INPUT` | Raw input string |
| `HOOK_TOOL_IS_ERROR` | `"true"` or `"false"` (post-hook only) |
| `HOOK_TOOL_OUTPUT` | Tool output (post-hook only) |

---

## Sequence Diagram — Hook Blocking a Dangerous Command

```mermaid
sequenceDiagram
    participant RT as 🧠 Runtime
    participant HK as ⚡ Hook Script
    participant T as 🔧 Tool

    RT->>RT: Tool requested: bash("rm -rf /")
    RT->>HK: PreToolUse hook via stdin:<br/>{"tool_name": "bash", "tool_input": "rm -rf /"}

    Note over HK: Script checks for dangerous patterns
    HK->>HK: Detects "rm -rf /"
    HK-->>RT: exit code 2<br/>stdout: "Blocked: destructive command detected"

    Note over RT: Tool execution SKIPPED

    RT-->>RT: tool_result: {<br/>  error: "Hook denied: Blocked: destructive command detected",<br/>  is_error: true<br/>}
```

---

## Exit Code Semantics

```mermaid
graph LR
    E0["Exit 0<br/>✅ ALLOW"] --> D0["Tool execution proceeds.<br/>stdout appended to context."]
    E2["Exit 2<br/>❌ DENY"] --> D2["Tool execution blocked.<br/>stdout becomes error message."]
    EX["Exit 1, 3+<br/>⚠️ WARN"] --> DX["Warning logged.<br/>Tool execution proceeds anyway."]

    style E0 fill:#22c55e,color:#fff
    style E2 fill:#ef4444,color:#fff
    style EX fill:#f59e0b,color:#fff
```

---

## Configuration

Hooks are defined in `.claude.json` or `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "command": "python3 /path/to/safety-check.py"
      }
    ],
    "PostToolUse": [
      {
        "command": "bash /path/to/audit-log.sh"
      }
    ]
  }
}
```

---

## Example Hook Scripts

### Safety Guard (PreToolUse)

```bash
#!/bin/bash
# Block dangerous bash commands
read -r payload
tool=$(echo "$payload" | jq -r '.tool_name')
input=$(echo "$payload" | jq -r '.tool_input')

if [[ "$tool" == "bash" ]]; then
    if echo "$input" | grep -qE "rm -rf|DROP TABLE|format c:"; then
        echo "Blocked: Dangerous command pattern detected"
        exit 2
    fi
fi
exit 0
```

### Audit Logger (PostToolUse)

```bash
#!/bin/bash
# Log all tool executions
read -r payload
echo "$(date -u +%FT%TZ) $payload" >> ~/.claude/audit.log
exit 0
```

---

## What's Next?

- **[Session Management →](../07-session-management/README.md)** — Persisting conversations
- **[Config System →](../09-config-system/README.md)** — Where hooks are configured
- **[Permission Model →](../04-permission-model/README.md)** — Hooks complement permissions

---

[← MCP Integration](../05-mcp-integration/README.md) | [Next: Session Management →](../07-session-management/README.md)
