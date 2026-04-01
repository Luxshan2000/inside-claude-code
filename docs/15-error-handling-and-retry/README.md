# 🔄 Error Handling & Retry Logic

> **Resilience by design.** How Claude Code handles failures, retries intelligently, and recovers gracefully.

[← Back to Main](../../README.md) | [← Slash Commands](../14-slash-commands/README.md)

---

## Error Type Taxonomy

```mermaid
graph TD
    subgraph "API Errors"
        MISSING["MissingApiKey<br/>No auth configured"]
        EXPIRED["ExpiredOAuthToken<br/>Token refresh failed"]
        AUTH["Auth(String)<br/>Auth-specific failure"]
        HTTP["Http(reqwest::Error)<br/>Network failure"]
        API_ERR["Api { status, error_type,<br/>message, retryable }<br/>Anthropic API error"]
        RETRIES["RetriesExhausted<br/>All retry attempts failed"]
        SSE["InvalidSseFrame<br/>Malformed stream data"]
        BACKOFF["BackoffOverflow<br/>Retry delay too large"]
    end

    subgraph "Session Errors"
        IO["Io — File read/write failure"]
        JSON["Json — Parse failure"]
        FORMAT["Format — Invalid structure"]
    end

    subgraph "Tool Errors"
        TOOL["ToolError(String)<br/>Tool execution failure"]
    end

    subgraph "Config Errors"
        CFG_IO["Io — Config file missing"]
        CFG_PARSE["Parse — Invalid JSON"]
    end
```

---

## Retryability Decision Tree

```mermaid
flowchart TD
    ERROR["API Error occurs"] --> CLASSIFY{"Error type?"}

    CLASSIFY -->|"Connect error"| RETRYABLE["✅ Retryable"]
    CLASSIFY -->|"Timeout"| RETRYABLE
    CLASSIFY -->|"Request timeout"| RETRYABLE
    CLASSIFY -->|"Status 429<br/>(Rate limit)"| RATE_LIMIT["⏳ Wait for<br/>retry-after header"]
    CLASSIFY -->|"Status 500-599<br/>(Server error)"| RETRYABLE
    CLASSIFY -->|"Status 401<br/>(Unauthorized)"| AUTH_RETRY["🔑 Refresh OAuth token"]
    CLASSIFY -->|"Status 400<br/>(Bad request)"| NOT_RETRYABLE["❌ Not retryable"]
    CLASSIFY -->|"Status 403<br/>(Forbidden)"| NOT_RETRYABLE

    RETRYABLE --> BACKOFF["Exponential backoff"]
    RATE_LIMIT --> BACKOFF
    AUTH_RETRY --> BACKOFF
    NOT_RETRYABLE --> FAIL["Display error, end turn"]
```

---

## Exponential Backoff Strategy

```mermaid
graph LR
    R1["Attempt 1<br/>Wait: 200ms"] --> R2["Attempt 2<br/>Wait: 400ms"]
    R2 --> R3["Attempt 3<br/>Wait: 800ms"]
    R3 --> R4["Attempt 4<br/>Wait: 1600ms"]
    R4 --> R5["Attempt 5<br/>Wait: 2000ms (cap)"]
    R5 --> FAIL["❌ RetriesExhausted"]
```

```
┌──────────────────────────────────────────────┐
│ Retry Configuration                          │
├──────────────────────────────────────────────┤
│ Initial delay:    200ms                      │
│ Multiplier:       2x                         │
│ Max delay:        2,000ms (2 seconds)        │
│ Max attempts:     5                          │
│ Jitter:           None (deterministic)       │
└──────────────────────────────────────────────┘
```

---

## Retry Sequence Diagram

```mermaid
sequenceDiagram
    participant RT as 🧠 Runtime
    participant API as 🌐 API

    RT->>API: Request (attempt 1)
    API-->>RT: 500 Internal Server Error

    Note over RT: Wait 200ms

    RT->>API: Request (attempt 2)
    API-->>RT: 500 Internal Server Error

    Note over RT: Wait 400ms

    RT->>API: Request (attempt 3)
    API-->>RT: 200 OK (streaming begins)

    Note over RT: Success! Continue normally.
```

---

## Error Flow in the Conversation Loop

```mermaid
flowchart TD
    STREAM["Streaming API response"] --> ERROR{"Error<br/>during stream?"}

    ERROR -->|"No"| CONTINUE["Continue processing"]

    ERROR -->|"Yes"| TYPE{"Retryable?"}
    TYPE -->|"Yes"| RETRY["Retry with backoff"]
    RETRY --> STREAM

    TYPE -->|"No"| DISPLAY["Display error to user"]
    DISPLAY --> END_TURN["End current turn"]
    END_TURN --> PROMPT["Return to REPL prompt<br/>(user can try again)"]
```

---

## Error Display to User

Different error types produce different user-facing messages:

| Error | User Sees |
|-------|-----------|
| MissingApiKey | "No API key configured. Set ANTHROPIC_API_KEY or run /login" |
| ExpiredOAuthToken | "OAuth token expired. Run /login to re-authenticate" |
| Rate limit (429) | "Rate limited. Waiting 30s before retrying..." |
| Server error (500) | "API server error. Retrying..." (automatic) |
| RetriesExhausted | "Failed after 5 attempts. Please try again." |
| InvalidSseFrame | "Received malformed response. Retrying..." |
| Network error | "Connection failed. Check your internet connection." |

---

## What's Next?

- **[Bootstrap Lifecycle →](../16-bootstrap-lifecycle/README.md)** — Error handling during startup
- **[Streaming & SSE →](../08-streaming-and-sse/README.md)** — Where stream errors originate

---

[← Slash Commands](../14-slash-commands/README.md) | [Next: Bootstrap Lifecycle →](../16-bootstrap-lifecycle/README.md)
