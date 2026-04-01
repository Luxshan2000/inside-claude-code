# 🧠 Memory Shrinking & Context Compaction

> **How infinite conversations fit into finite context.** The secret to long-running coding sessions.

[← Back to Main](../../README.md) | [← Conversation Loop](../01-conversation-loop/README.md)

---

## The Problem

Language models have a **fixed context window** (e.g., 200K tokens). But coding sessions can go on for hours — reading files, editing code, running tests, iterating. Without memory management, the conversation would hit the limit and break.

Claude Code solves this with **context compaction** — intelligently shrinking the conversation history while preserving what matters.

---

## How Compaction Works — Full Flow

```mermaid
flowchart TD
    START["Every API response"] --> COUNT["Count cumulative<br/>input tokens"]
    COUNT --> CHECK{"tokens > budget?<br/>(default: 200K)"}

    CHECK -->|"No"| CONTINUE["✅ Continue normally"]

    CHECK -->|"Yes"| IDENTIFY["Identify messages<br/>to compress"]
    IDENTIFY --> PRESERVE["Preserve recent N<br/>messages (configurable)"]
    PRESERVE --> SUMMARIZE["Generate XML summary<br/>of removed messages"]
    SUMMARIZE --> INJECT["Inject summary as<br/>system context message"]
    INJECT --> STRIP["Strip old messages<br/>from history"]
    STRIP --> RESUME["Resume conversation<br/>with compacted context"]

    style START fill:#3b82f6,color:#fff
    style CONTINUE fill:#22c55e,color:#fff
    style RESUME fill:#22c55e,color:#fff
```

---

## What Gets Compacted vs. Preserved

```mermaid
graph TB
    subgraph "BEFORE Compaction"
        direction TB
        M1["🟢 System Prompt"]
        M2["👤 User: Read config.rs"]
        M3["🤖 Assistant: Sure, reading..."]
        M4["🔧 Tool: read_file → 200 lines"]
        M5["🤖 Assistant: I see the issue..."]
        M6["🔧 Tool: edit_file → applied"]
        M7["👤 User: Now fix the tests"]
        M8["🤖 Assistant: Looking at tests..."]
        M9["🔧 Tool: read_file → test.rs"]
        M10["🤖 Assistant: Here's the fix..."]
    end

    subgraph "AFTER Compaction"
        direction TB
        N1["🟢 System Prompt"]
        N2["📋 Summary: Previously read config.rs,<br/>found and fixed a bug in line 42"]
        N7["👤 User: Now fix the tests"]
        N8["🤖 Assistant: Looking at tests..."]
        N9["🔧 Tool: read_file → test.rs"]
        N10["🤖 Assistant: Here's the fix..."]
    end

    M1 -.->|"Kept"| N1
    M2 -.->|"Summarized"| N2
    M3 -.->|"Summarized"| N2
    M4 -.->|"Summarized"| N2
    M5 -.->|"Summarized"| N2
    M6 -.->|"Summarized"| N2
    M7 -.->|"Kept"| N7
    M8 -.->|"Kept"| N8
    M9 -.->|"Kept"| N9
    M10 -.->|"Kept"| N10
```

---

## Compaction Configuration

```
┌──────────────────────────────────────────────┐
│ CompactionConfig                             │
├──────────────────────────────────────────────┤
│ max_estimated_tokens: u32    (default: 200K) │
│ preserve_recent_messages: u32 (configurable) │
├──────────────────────────────────────────────┤
│ Env override:                                │
│ CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS=150000 │
└──────────────────────────────────────────────┘
```

---

## Token Estimation

Before compaction can trigger, the system needs to know how many tokens have been used. Here's how it estimates:

```mermaid
flowchart LR
    A["Each message"] --> B["Count characters<br/>in all content blocks"]
    B --> C["Apply ratio:<br/>~4 chars = 1 token"]
    C --> D["Sum across<br/>all messages"]
    D --> E["Compare to budget"]
```

> **Note:** This is an *estimate*. The actual token count from the API (returned in usage) is used for cost tracking, but the estimate is used for compaction decisions to avoid an extra API call.

---

## Summary Generation

When old messages are removed, a **structured summary** replaces them:

```xml
<conversation_summary>
  The user asked to read config.rs and fix a bug.
  The assistant read the file (200 lines), identified an
  off-by-one error on line 42, and applied an edit to fix it.
  The fix was verified with cargo check (0 errors).
</conversation_summary>
```

This summary is injected as a **system-role message** so the model knows what happened before, even though the detailed messages are gone.

---

## Sequence Diagram — Auto-Compaction Mid-Conversation

```mermaid
sequenceDiagram
    participant RT as 🧠 Runtime
    participant API as 🌐 API
    participant C as 📦 Compactor

    Note over RT: Turn 15 — tokens approaching 200K

    RT->>API: Send messages (198K tokens)
    API-->>RT: Response + usage: 201K input tokens

    RT->>RT: Check: 201K > 200K budget ⚠️
    RT->>C: Trigger compaction

    C->>C: Identify turns 1-10 for removal
    C->>C: Generate summary of turns 1-10
    C->>C: Preserve turns 11-15
    C->>C: Inject summary message
    C-->>RT: Compacted history (~80K tokens)

    Note over RT: TurnSummary.auto_compaction = true

    RT->>API: Next request with compacted context
    API-->>RT: Continues naturally
```

---

## Manual Compaction: `/compact`

Users can also trigger compaction manually via the `/compact` slash command:

```mermaid
flowchart LR
    A["User types /compact"] --> B["Force compaction<br/>regardless of token count"]
    B --> C["Same compaction flow"]
    C --> D["Refreshed context<br/>with summary"]
```

This is useful when:
- The conversation feels "confused" by old context
- You want to start a new subtask within the same session
- You're working on a different part of the codebase

---

## Token Budget Visualization

```
Context Window: 200K tokens
┌─────────────────────────────────────────────────────┐
│ System Prompt          │ ~2K tokens                  │
├────────────────────────┼────────────────────────────│
│ Compacted Summary      │ ~1K tokens (after compact) │
├────────────────────────┼────────────────────────────│
│ Recent Messages        │ ~50K-150K tokens           │
├────────────────────────┼────────────────────────────│
│ Tool Definitions       │ ~5K tokens                  │
├────────────────────────┼────────────────────────────│
│ Available for Response │ Remaining                   │
└────────────────────────┴────────────────────────────┘
```

---

## Key Takeaways

| Aspect | Detail |
|--------|--------|
| **Trigger** | Cumulative input tokens exceed budget (default 200K) |
| **What's removed** | Oldest messages beyond preserve_recent count |
| **What's kept** | System prompt, summary, recent N messages |
| **Summary format** | XML-wrapped natural language summary |
| **Manual trigger** | `/compact` slash command |
| **Config override** | `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` env var |
| **Impact** | Transparent to user — conversation continues seamlessly |

---

## What's Next?

- **[Tool System →](../03-tool-system/README.md)** — The tools that generate all those tokens
- **[Session Management →](../07-session-management/README.md)** — How conversations persist across restarts

---

[← Conversation Loop](../01-conversation-loop/README.md) | [Next: Tool System →](../03-tool-system/README.md)
