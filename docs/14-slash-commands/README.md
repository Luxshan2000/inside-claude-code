# ⌨️ Slash Commands

> **Quick controls.** The command registry that lets you switch models, compact memory, and manage sessions without leaving the REPL.

[← Back to Main](../../README.md) | [← System Prompt Building](../13-system-prompt-building/README.md)

---

## Command Overview

Slash commands are prefixed with `/` and executed in the REPL. They provide quick access to runtime controls without needing to ask the AI.

---

## Full Command Registry

```mermaid
graph TD
    subgraph "📊 Information"
        HELP["/help — List all commands"]
        STATUS["/status — Session stats"]
        COST["/cost — Token usage & pricing"]
        VERSION["/version — CLI version"]
    end

    subgraph "🧠 Context"
        COMPACT["/compact — Shrink context"]
        MEMORY["/memory — Show instruction files"]
        CLEAR["/clear — Start fresh conversation"]
    end

    subgraph "⚙️ Configuration"
        MODEL["/model — Switch AI model"]
        PERMISSIONS["/permissions — Change permission level"]
        CONFIG["/config — Inspect config"]
    end

    subgraph "💾 Sessions"
        RESUME["/resume — Load saved session"]
        SESSION["/session — Session management"]
        EXPORT["/export — Export conversation"]
    end

    subgraph "🔀 Git"
        DIFF["/diff — Show git changes"]
        INIT["/init — Setup project config"]
    end
```

---

## Command Parsing Flow

```mermaid
flowchart TD
    INPUT["User types: /model sonnet"] --> PARSE["Split: command='model'<br/>args='sonnet'"]
    PARSE --> LOOKUP{"Command<br/>exists?"}

    LOOKUP -->|"No"| ERROR["❌ Unknown command.<br/>Type /help for list."]
    LOOKUP -->|"Yes"| EXECUTE["Execute handler"]

    EXECUTE --> OUTPUT["Display result"]
```

---

## Command Details

### `/compact` — Memory Shrinking
```
> /compact
Compacting conversation... ✓
Removed 12 messages, preserved 5 recent.
Context reduced from 180K → 45K tokens.
```

### `/model` — Switch Models
```
> /model sonnet
Switched to claude-sonnet-4-6

> /model haiku
Switched to claude-haiku-4-5-20251213

> /model opus
Switched to claude-opus-4-6
```

### `/cost` — Usage Tracking
```
> /cost
Session Usage:
  Input tokens:  45,230 ($0.68)
  Output tokens: 12,450 ($0.93)
  Cache reads:    8,200 ($0.01)
  Total cost:    $1.62
```

### `/permissions` — Permission Control
```
> /permissions read-only
Permission mode changed to ReadOnly

> /permissions danger-full-access
Permission mode changed to DangerFullAccess
```

### `/status` — Session Info
```
> /status
Model: claude-opus-4-6
Permission: DangerFullAccess
Messages: 24
Tokens: 89,450 input / 15,230 output
Session: abc123-def456
```

---

## Command Registration — Class Diagram

```mermaid
classDiagram
    class SlashCommand {
        +name: String
        +description: String
        +usage: String
        +requires_session: bool
    }

    class CommandRegistry {
        +commands: Vec~SlashCommand~
        +get(name) Option~SlashCommand~
        +list() Vec~SlashCommand~
        +list_filtered(has_session) Vec~SlashCommand~
    }

    CommandRegistry *-- SlashCommand
```

---

## Context-Aware Filtering

Some commands only appear when relevant:

```mermaid
flowchart LR
    ALL["All 15+ commands"] --> FILTER{"Session<br/>active?"}
    FILTER -->|"Yes"| FULL["Show all commands<br/>including /resume, /export"]
    FILTER -->|"No"| LIMITED["Hide session-specific<br/>commands"]
```

---

## What's Next?

- **[Error Handling →](../15-error-handling-and-retry/README.md)** — What happens when things go wrong
- **[Memory & Context →](../02-memory-and-context/README.md)** — How `/compact` works internally

---

[← System Prompt Building](../13-system-prompt-building/README.md) | [Next: Error Handling →](../15-error-handling-and-retry/README.md)
