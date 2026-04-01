# 🚀 Bootstrap Lifecycle

> **From zero to ready.** The ordered startup sequence that initializes everything before the first prompt.

[← Back to Main](../../README.md) | [← Error Handling](../15-error-handling-and-retry/README.md)

---

## What Is Bootstrap?

When you type `claw` and hit enter, a lot happens before you see the `>` prompt. The bootstrap lifecycle defines 12 ordered phases that initialize config, auth, MCP servers, and the conversation runtime.

---

## Bootstrap Phases

```mermaid
flowchart TD
    P0["Phase 0: CliEntry<br/>Parse CLI arguments"] --> P1["Phase 1: FastPathVersion<br/>Handle --version, exit early"]
    P1 --> P2["Phase 2: StartupProfiler<br/>Initialize performance tracking"]
    P2 --> P3["Phase 3: SystemPromptFastPath<br/>Build base system prompt"]
    P3 --> P4["Phase 4: ChromeMcpFastPath<br/>Chrome MCP server init"]
    P4 --> P5["Phase 5: DaemonWorkerFastPath<br/>Background worker setup"]
    P5 --> P6["Phase 6: BridgeFastPath<br/>Bridge connection init"]
    P6 --> P7["Phase 7: DaemonFastPath<br/>Daemon process init"]
    P7 --> P8["Phase 8: BackgroundSessionFastPath<br/>Background session setup"]
    P8 --> P9["Phase 9: TemplateFastPath<br/>Project template init"]
    P9 --> P10["Phase 10: EnvironmentRunnerFastPath<br/>Environment runner setup"]
    P10 --> P11["Phase 11: MainRuntime<br/>🟢 Full runtime ready"]

    style P0 fill:#3b82f6,color:#fff
    style P11 fill:#22c55e,color:#fff
```

---

## Phase Categories

```mermaid
graph LR
    subgraph "🔵 Entry"
        P0["CliEntry"]
        P1["FastPathVersion"]
    end

    subgraph "📊 Setup"
        P2["StartupProfiler"]
        P3["SystemPromptFastPath"]
    end

    subgraph "🔌 Connections"
        P4["ChromeMcpFastPath"]
        P5["DaemonWorkerFastPath"]
        P6["BridgeFastPath"]
        P7["DaemonFastPath"]
    end

    subgraph "💾 Sessions"
        P8["BackgroundSessionFastPath"]
        P9["TemplateFastPath"]
        P10["EnvironmentRunnerFastPath"]
    end

    subgraph "🟢 Ready"
        P11["MainRuntime"]
    end
```

---

## Startup Sequence Diagram

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant CLI as 🖥️ CLI
    participant CFG as ⚙️ Config
    participant AUTH as 🔑 Auth
    participant MCP as 🔌 MCP
    participant RT as 🧠 Runtime

    U->>CLI: $ claw

    CLI->>CLI: Parse CLI args
    CLI->>CLI: Check --version (fast exit)

    CLI->>CFG: Discover & load configs
    CFG-->>CLI: RuntimeConfig

    CLI->>AUTH: Check authentication
    AUTH-->>CLI: Auth ready (API key or OAuth)

    CLI->>MCP: Initialize MCP servers
    MCP->>MCP: Spawn stdio processes
    MCP->>MCP: Connect SSE/WS servers
    MCP->>MCP: Discover tools
    MCP-->>CLI: MCP tools registered

    CLI->>RT: Initialize ConversationRuntime
    RT->>RT: Build system prompt
    RT->>RT: Load session (if --resume)
    RT-->>CLI: Runtime ready

    CLI-->>U: Welcome message + ">" prompt
```

---

## Bootstrap Plan — Deduplication

The bootstrap plan ensures each phase runs exactly once, even if referenced multiple times:

```
┌────────────────────────────────────────────┐
│ BootstrapPlan                              │
├────────────────────────────────────────────┤
│ phases: Vec<Phase>  (ordered, unique)      │
│                                            │
│ Dedup: If same phase added twice,          │
│        second instance is silently dropped │
└────────────────────────────────────────────┘
```

---

## Fast Path Exits

Several phases support "fast path" exits — completing the request without reaching the full runtime:

```mermaid
flowchart TD
    START["CLI Entry"] --> VERSION{"--version?"}
    VERSION -->|"Yes"| PRINT_VERSION["Print version, exit"]
    VERSION -->|"No"| TEMPLATE{"--init?"}
    TEMPLATE -->|"Yes"| INIT["Run project init, exit"]
    TEMPLATE -->|"No"| LOGIN{"login subcommand?"}
    LOGIN -->|"Yes"| AUTH["Run OAuth flow, exit"]
    LOGIN -->|"No"| FULL["Continue to MainRuntime"]

    style PRINT_VERSION fill:#f59e0b,color:#fff
    style INIT fill:#f59e0b,color:#fff
    style AUTH fill:#f59e0b,color:#fff
    style FULL fill:#22c55e,color:#fff
```

---

## What's Next?

You've reached the end of the deep dives! Here's where to go from here:

- **[Back to Architecture Overview →](../00-architecture-overview/README.md)** — See how it all fits together
- **[Back to Main README →](../../README.md)** — Full table of contents

---

[← Error Handling](../15-error-handling-and-retry/README.md) | [Back to Main →](../../README.md)
