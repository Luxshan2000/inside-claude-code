# 💾 Session Management

> **Pick up where you left off.** How Claude Code persists and resumes conversations.

[← Back to Main](../../README.md) | [← Hook System](../06-hook-system/README.md)

---

## Why Sessions Matter

Coding tasks often span multiple sittings. You might start debugging in the morning, close your terminal, and want to continue in the afternoon. Sessions make this seamless — every message, tool result, and usage stat is persisted.

---

## Session Data Model

```mermaid
classDiagram
    class Session {
        +version: u32
        +messages: Vec~ConversationMessage~
        +save_to_path(path)
        +load_from_path(path) Session
    }

    class ConversationMessage {
        +role: Role
        +blocks: Vec~ContentBlock~
        +usage: Option~TokenUsage~
    }

    class Role {
        <<enumeration>>
        System
        User
        Assistant
        Tool
    }

    class ContentBlock {
        <<enumeration>>
        Text(text: String)
        ToolUse(id, name, input)
        ToolResult(tool_use_id, tool_name, output, is_error)
    }

    class TokenUsage {
        +input: u32
        +output: u32
        +cache_creation: u32
        +cache_read: u32
    }

    Session *-- ConversationMessage
    ConversationMessage --> Role
    ConversationMessage *-- ContentBlock
    ConversationMessage --> TokenUsage
```

---

## Session Lifecycle

```mermaid
flowchart TD
    START(["🟢 Start"]) --> CHECK{"Resume<br/>flag set?"}

    CHECK -->|"No"| NEW["Create new Session<br/>version: 1, messages: []"]
    CHECK -->|"Yes"| LOAD["Load session from<br/>JSON file"]
    LOAD --> VALIDATE{"Valid<br/>format?"}
    VALIDATE -->|"No"| ERROR["❌ Session error"]
    VALIDATE -->|"Yes"| RESUME["Restore messages<br/>and continue"]

    NEW --> LOOP["Conversation Loop"]
    RESUME --> LOOP

    LOOP --> MSG["User/Assistant messages<br/>added to session"]
    MSG --> LOOP

    LOOP --> SAVE_CHECK{"User exits or<br/>/session save?"}
    SAVE_CHECK -->|"Yes"| PERSIST["Serialize to JSON<br/>and save to file"]
    SAVE_CHECK -->|"No"| LOOP
```

---

## JSON Serialization Format

Sessions are stored as JSON files with deterministic key ordering (BTreeMap):

```json
{
  "version": 1,
  "messages": [
    {
      "role": "user",
      "blocks": [
        {
          "type": "text",
          "text": "Read the config file"
        }
      ]
    },
    {
      "role": "assistant",
      "blocks": [
        {
          "type": "text",
          "text": "I'll read config.rs for you."
        },
        {
          "type": "tool_use",
          "id": "tool_abc123",
          "name": "read_file",
          "input": "{\"file_path\": \"config.rs\"}"
        }
      ],
      "usage": {
        "input": 1520,
        "output": 340,
        "cache_creation": 0,
        "cache_read": 150
      }
    },
    {
      "role": "tool",
      "blocks": [
        {
          "type": "tool_result",
          "tool_use_id": "tool_abc123",
          "tool_name": "read_file",
          "output": "fn main() { ... }",
          "is_error": false
        }
      ]
    }
  ]
}
```

---

## Resume Flow — Sequence Diagram

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant CLI as 🖥️ CLI
    participant S as 💾 Session
    participant RT as 🧠 Runtime

    U->>CLI: claw --resume session_id
    CLI->>S: load_from_path("~/.claude/sessions/session_id.json")
    S-->>CLI: Session { version, messages }

    CLI->>RT: Initialize with restored messages
    RT->>RT: Rebuild system prompt
    RT->>RT: Calculate current token usage

    CLI-->>U: "Resumed session (15 messages, 45K tokens)"
    U->>CLI: Continue conversation...
```

---

## Session Commands

```
/session          — Show current session info
/session save     — Save session to file
/resume <id>      — Resume a saved session
/export           — Export session (JSON/text)
```

---

## What's Next?

- **[Streaming & SSE →](../08-streaming-and-sse/README.md)** — How messages stream in real-time
- **[Memory & Context →](../02-memory-and-context/README.md)** — How old sessions get compacted

---

[← Hook System](../06-hook-system/README.md) | [Next: Streaming & SSE →](../08-streaming-and-sse/README.md)
