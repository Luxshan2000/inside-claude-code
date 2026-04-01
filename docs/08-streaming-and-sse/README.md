# 📡 Streaming & SSE Parsing

> **Real-time token delivery.** How Claude Code parses Server-Sent Events for live streaming output.

[← Back to Main](../../README.md) | [← Session Management](../07-session-management/README.md)

---

## Why Streaming?

Without streaming, you'd wait 10-30 seconds staring at a blank screen while the model generates its full response. With **Server-Sent Events (SSE)**, tokens arrive one-by-one as they're generated — giving instant feedback and a responsive terminal experience.

---

## SSE Protocol — How It Works

```mermaid
sequenceDiagram
    participant CLI as 🖥️ CLI
    participant API as 🌐 Anthropic API

    CLI->>API: POST /v1/messages (stream=true)

    Note over API: Model starts generating

    API-->>CLI: event: message_start\ndata: {"type":"message_start"...}
    API-->>CLI: event: content_block_start\ndata: {"type":"content_block_start"...}
    API-->>CLI: event: content_block_delta\ndata: {"type":"content_block_delta","delta":{"text":"I"}}
    API-->>CLI: event: content_block_delta\ndata: {"type":"content_block_delta","delta":{"text":"'ll"}}
    API-->>CLI: event: content_block_delta\ndata: {"type":"content_block_delta","delta":{"text":" read"}}

    Note over CLI: Each delta rendered immediately

    API-->>CLI: event: content_block_stop
    API-->>CLI: event: message_delta\ndata: {"usage":{"output_tokens":15}}
    API-->>CLI: event: message_stop
```

---

## IncrementalSseParser — The Parser

```mermaid
flowchart TD
    CHUNK["Raw HTTP chunk arrives<br/>bytes from network"] --> BUFFER["Append to<br/>line buffer"]
    BUFFER --> SCAN{"Scan for<br/>empty line?"}

    SCAN -->|"No empty line"| WAIT["Wait for<br/>more data"]
    WAIT --> CHUNK

    SCAN -->|"Empty line found"| PARSE["Parse accumulated<br/>field lines"]
    PARSE --> FIELDS["Extract fields:<br/>event, data, id, retry"]
    FIELDS --> EMIT["Emit SseEvent"]
    EMIT --> CLEAR["Clear buffer"]
    CLEAR --> SCAN
```

---

## SSE Event Structure

```
┌──────────────────────────────────────┐
│ SseEvent                             │
├──────────────────────────────────────┤
│ event: Option<String>  (event name)  │
│ data: String           (JSON payload)│
│ id: Option<String>     (event ID)    │
│ retry: Option<u64>     (retry ms)    │
└──────────────────────────────────────┘
```

### Raw SSE Wire Format

```
event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" world"}}

event: message_stop
data: {"type":"message_stop"}
```

Key rules:
- Lines starting with `event:` set the event name
- Lines starting with `data:` append to data (multi-line concatenation)
- An **empty line** signals the end of an event
- Lines starting with `:` are comments (heartbeat keep-alives)

---

## Event Types from Anthropic API

```mermaid
graph TD
    subgraph "Lifecycle Events"
        MS["message_start"]
        MD["message_delta<br/>(usage stats)"]
        MST["message_stop"]
    end

    subgraph "Content Events"
        CBS["content_block_start<br/>(text or tool_use)"]
        CBD["content_block_delta<br/>(incremental content)"]
        CBST["content_block_stop"]
    end

    subgraph "Error Events"
        ERR["error<br/>(overloaded, rate_limit)"]
    end

    MS --> CBS --> CBD --> CBST --> MD --> MST
    CBD -->|"loop"| CBD
```

---

## Stream → AssistantEvent Mapping

```mermaid
flowchart LR
    SSE["SseEvent"] --> PARSE["Parse JSON<br/>data field"]
    PARSE --> TYPE{"Event type?"}

    TYPE -->|"content_block_delta<br/>+ text_delta"| TEXT["AssistantEvent::<br/>TextDelta(text)"]
    TYPE -->|"content_block_start<br/>+ tool_use"| TOOL_START["Buffer tool_use<br/>id + name"]
    TYPE -->|"content_block_delta<br/>+ input_json_delta"| TOOL_INPUT["Accumulate<br/>tool input JSON"]
    TYPE -->|"content_block_stop<br/>(tool_use)"| TOOL["AssistantEvent::<br/>ToolUse{id, name, input}"]
    TYPE -->|"message_delta"| USAGE["AssistantEvent::<br/>Usage(tokens)"]
    TYPE -->|"message_stop"| STOP["AssistantEvent::<br/>MessageStop"]
```

---

## Terminal Rendering Pipeline

```mermaid
flowchart TD
    EVENTS["Stream of<br/>AssistantEvents"] --> RENDER{"Event type?"}

    RENDER -->|"TextDelta"| MD["Markdown<br/>Stream State"]
    MD --> CHECK{"In code block?<br/>In heading?<br/>In list?"}
    CHECK -->|"Code"| SYNTAX["Syntax highlight<br/>+ box borders"]
    CHECK -->|"Heading"| BOLD["Bold +<br/>color formatting"]
    CHECK -->|"Plain"| WRAP["Word wrap<br/>to terminal width"]

    SYNTAX --> PRINT["Print to terminal"]
    BOLD --> PRINT
    WRAP --> PRINT

    RENDER -->|"ToolUse"| TOOL_BOX["Render tool call<br/>with box borders"]
    TOOL_BOX --> SPINNER["Show spinner<br/>while executing"]

    RENDER -->|"MessageStop"| DONE["Print newline<br/>Update cost display"]
```

---

## Retry on Stream Failure

```mermaid
flowchart TD
    STREAM["Streaming..."] --> FAIL{"Connection<br/>dropped?"}
    FAIL -->|"No"| CONTINUE["Continue parsing"]
    FAIL -->|"Yes"| RETRY["Exponential backoff:<br/>200ms → 400ms → 800ms → 2s"]
    RETRY --> RECONNECT["Reconnect and<br/>resend request"]
    RECONNECT --> STREAM
```

---

## What's Next?

- **[Config System →](../09-config-system/README.md)** — API configuration and model selection
- **[Error Handling →](../15-error-handling-and-retry/README.md)** — Retry logic deep dive

---

[← Session Management](../07-session-management/README.md) | [Next: Config System →](../09-config-system/README.md)
