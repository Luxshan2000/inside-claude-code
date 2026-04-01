# 🔄 The Agentic Conversation Loop

> **The beating heart.** The core loop that makes Claude Code an *agent*, not just a chatbot.

[← Back to Main](../../README.md) | [← Architecture Overview](../00-architecture-overview/README.md)

---

## What Makes It "Agentic"?

A regular chatbot sends one message and gets one response. An **agentic loop** keeps going — it can call tools, get results, reason about them, call more tools, and iterate until the task is complete. Claude Code's conversation loop is what turns a language model into an autonomous coding assistant.

---

## The Core Loop — Flow Diagram

```mermaid
flowchart TD
    START(["🟢 User sends message"]) --> BUILD["Build message payload<br/>(system prompt + history + tools)"]
    BUILD --> STREAM["📡 Stream request to Anthropic API"]
    STREAM --> PARSE["Parse SSE events"]
    PARSE --> CHECK{"What type of event?"}

    CHECK -->|"text_delta"| DISPLAY["💬 Display text to terminal"]
    DISPLAY --> PARSE

    CHECK -->|"tool_use"| COLLECT["📦 Collect tool call<br/>(id, name, input)"]
    COLLECT --> PARSE

    CHECK -->|"message_stop"| ASSEMBLE["Assemble AssistantMessage"]
    ASSEMBLE --> TOOLS_CHECK{"Any tool calls<br/>in message?"}

    TOOLS_CHECK -->|"No"| DONE(["✅ Turn complete"])

    TOOLS_CHECK -->|"Yes"| EXEC_LOOP["For each tool call:"]
    EXEC_LOOP --> PERM{"🔐 Permission<br/>check"}
    PERM -->|"Denied"| DENY["Return error result"]
    PERM -->|"Allowed"| PRE_HOOK["⚡ Run PreToolUse hook"]
    PRE_HOOK -->|"exit 2"| HOOK_DENY["Return hook-denied result"]
    PRE_HOOK -->|"exit 0"| EXECUTE["🏃 Execute tool"]
    EXECUTE --> POST_HOOK["⚡ Run PostToolUse hook"]
    POST_HOOK --> RESULT["📨 Collect tool result"]
    DENY --> RESULT
    HOOK_DENY --> RESULT

    RESULT --> MORE_TOOLS{"More tools<br/>to execute?"}
    MORE_TOOLS -->|"Yes"| EXEC_LOOP
    MORE_TOOLS -->|"No"| SEND_RESULTS["Send all tool_results to API"]
    SEND_RESULTS --> STREAM

    style START fill:#22c55e,color:#fff
    style DONE fill:#22c55e,color:#fff
    style DENY fill:#ef4444,color:#fff
    style HOOK_DENY fill:#ef4444,color:#fff
```

---

## Sequence Diagram — Multi-Tool Turn

A real-world example: user asks "Read the config file and fix the bug"

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant RT as 🧠 Runtime
    participant API as 🌐 API
    participant T as 🔧 Tools

    U->>RT: "Read config.rs and fix the bug on line 42"
    RT->>API: messages + tool definitions

    Note over API: Decides to read file first
    API-->>RT: tool_use: read_file("config.rs")
    RT->>T: Execute read_file
    T-->>RT: File contents (200 lines)
    RT->>API: tool_result: [file contents]

    Note over API: Analyzes code, finds bug
    API-->>RT: text: "I see the issue..."
    API-->>RT: tool_use: edit_file("config.rs", fix)
    RT->>T: Execute edit_file
    T-->>RT: Edit applied successfully
    RT->>API: tool_result: "success"

    Note over API: Verifies the fix
    API-->>RT: tool_use: bash("cargo check")
    RT->>T: Execute bash
    T-->>RT: "Compiling... 0 errors"
    RT->>API: tool_result: "0 errors"

    API-->>RT: text: "Fixed! The bug was..."
    API-->>RT: message_stop
    RT-->>U: Display complete response
```

---

## Key Data Structures

### Assistant Event — What the API sends

```
┌─────────────────────────────────────────┐
│ Assistant Event Types                   │
├─────────────────────────────────────────┤
│ Text Delta      → Incremental text      │
│ Tool Use        → Tool invocation       │
│                   (id, name, input)      │
│ Usage           → Token count update    │
│ Message Stop    → End of response       │
└─────────────────────────────────────────┘
```

### Turn Summary — What each iteration produces

```
┌─────────────────────────────────────────┐
│ Turn Summary                            │
├─────────────────────────────────────────┤
│ Assistant messages sent                 │
│ Tool results collected                  │
│ Number of iterations                    │
│ Token usage stats                       │
│ Whether auto-compaction triggered       │
└─────────────────────────────────────────┘
```

---

## Stop Conditions

The loop terminates when:

| Condition | What Happens |
|-----------|--------------|
| `stop_reason: "end_turn"` | Model is done — natural completion |
| Max iterations reached | Safety limit to prevent infinite loops |
| Token budget exceeded | Triggers compaction, then continues |
| User cancellation | Ctrl+C interrupts the stream |
| API error (non-retryable) | Error displayed, turn ends |

---

## Auto-Compaction During Loop

When the cumulative input tokens exceed the budget (default 200K), the loop triggers **auto-compaction** mid-conversation:

```mermaid
flowchart LR
    A["Token count check<br/>after each API call"] --> B{"Over budget?"}
    B -->|"No"| C["Continue loop"]
    B -->|"Yes"| D["Compact history"]
    D --> E["Summarize old turns"]
    E --> F["Inject summary"]
    F --> G["Continue with<br/>smaller context"]
```

See **[Memory & Compaction →](../02-memory-and-context/README.md)** for the full deep dive.

---

## Error Recovery in the Loop

```mermaid
flowchart TD
    ERR["API returns error"] --> TYPE{"Error type?"}
    TYPE -->|"Retryable<br/>(timeout, 500)"| RETRY["Wait (exponential backoff)<br/>200ms → 400ms → 800ms → 2s"]
    RETRY --> RESEND["Retry request"]
    TYPE -->|"Rate limit<br/>(429)"| WAIT["Wait for retry-after header"]
    WAIT --> RESEND
    TYPE -->|"Auth error<br/>(401)"| REFRESH["Refresh OAuth token"]
    REFRESH --> RESEND
    TYPE -->|"Non-retryable"| FAIL["Display error,<br/>end turn"]
    RESEND --> CHECK{"Max retries?"}
    CHECK -->|"No"| STREAM["Continue streaming"]
    CHECK -->|"Yes"| EXHAUST["RetriesExhausted error"]
```

---

## What's Next?

- **[Memory & Compaction →](../02-memory-and-context/README.md)** — How the loop handles token limits
- **[Tool System →](../03-tool-system/README.md)** — What happens inside "Execute tool"
- **[Streaming & SSE →](../08-streaming-and-sse/README.md)** — How the SSE parser works

---

[← Architecture Overview](../00-architecture-overview/README.md) | [Next: Memory & Compaction →](../02-memory-and-context/README.md)
