# 📝 System Prompt Building

> **Context injection.** How Claude Code dynamically constructs the system prompt with project context.

[← Back to Main](../../README.md) | [← Sandbox Execution](../12-sandbox-execution/README.md)

---

## Why Dynamic Prompts?

Unlike a static chatbot, Claude Code needs to know about *your* project — the directory structure, git status, coding standards, and custom instructions. The system prompt builder dynamically assembles all this context before every API call.

---

## Prompt Construction Flow

```mermaid
flowchart TD
    START["Build system prompt"] --> BASE["Base instructions<br/>(agent identity, capabilities)"]

    BASE --> CLAUDE_MD["Discover CLAUDE.md files"]
    CLAUDE_MD --> GIT["Read git status & diff"]
    GIT --> CONFIG_INST["Load config instructions"]
    CONFIG_INST --> OS_INFO["Inject OS/system metadata"]
    OS_INFO --> TOOLS_DEF["Append available tool definitions"]

    TOOLS_DEF --> ASSEMBLE["Assemble final prompt"]

    ASSEMBLE --> SIZE_CHECK{"Within<br/>size limits?"}
    SIZE_CHECK -->|"Yes"| DONE["✅ System prompt ready"]
    SIZE_CHECK -->|"No"| TRUNCATE["Truncate instruction files<br/>(4KB per file, 12KB total)"]
    TRUNCATE --> DONE
```

---

## CLAUDE.md Discovery

The system searches for instruction files in a specific order:

```mermaid
flowchart TD
    CWD["Current working directory"] --> CHECK1{"./CLAUDE.md<br/>exists?"}
    CHECK1 -->|"Yes"| LOAD1["Load (max 4KB)"]
    CHECK1 -->|"No"| SKIP1["Skip"]

    CWD --> CHECK2{"./claude.md<br/>exists?"}
    CWD --> PARENT["Parent directories"]

    PARENT --> CHECK3{"../CLAUDE.md?"}
    PARENT --> CHECK4{"../../CLAUDE.md?"}

    LOAD1 --> COLLECT["Collected instruction files"]
    CHECK3 -->|"Yes"| LOAD3["Load"]
    LOAD3 --> COLLECT

    COLLECT --> LIMIT{"Total > 12KB?"}
    LIMIT -->|"Yes"| TRUNCATE["Truncate oldest files"]
    LIMIT -->|"No"| INJECT["Inject into prompt"]
```

---

## ProjectContext — What Gets Injected

```mermaid
classDiagram
    class ProjectContext {
        +cwd: PathBuf
        +current_date: String
        +git_status: Option~String~
        +git_diff: Option~String~
        +instruction_files: Vec~InstructionFile~
    }

    class InstructionFile {
        +path: PathBuf
        +content: String
        +truncated: bool
    }

    class SystemPromptBuilder {
        +base_prompt: String
        +with_project_context(ctx) Self
        +with_os_info() Self
        +build() String
    }

    ProjectContext *-- InstructionFile
    SystemPromptBuilder --> ProjectContext
```

---

## System Prompt Structure

```
┌─────────────────────────────────────────────────┐
│ SYSTEM PROMPT                                   │
├─────────────────────────────────────────────────┤
│                                                 │
│ 1. Base Identity                                │
│    "You are Claude Code, an AI coding           │
│    assistant powered by Claude Opus 4.6..."     │
│                                                 │
│ 2. Project Context                              │
│    Working directory: /Users/dev/project        │
│    Date: 2026-04-02                             │
│                                                 │
│ 3. Git Status                                   │
│    On branch: main                              │
│    Modified: 3 files                            │
│                                                 │
│ 4. Instruction Files (CLAUDE.md)                │
│    "This project uses Rust. Run tests with      │
│    cargo test --workspace..."                   │
│                                                 │
│ 5. OS/System Info                               │
│    Platform: darwin, macOS 15.2                  │
│                                                 │
│ ─── DYNAMIC BOUNDARY ───                        │
│                                                 │
│ 6. Tool Definitions                             │
│    [18 built-in + MCP tools]                    │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Git Integration in Prompts

```mermaid
sequenceDiagram
    participant PB as 📝 PromptBuilder
    participant GIT as 🔀 Git

    PB->>GIT: git status --short
    GIT-->>PB: "M  src/main.rs\n?? new_file.rs"

    PB->>GIT: git diff HEAD
    GIT-->>PB: "@@ -10,3 +10,5 @@\n+new_line..."

    PB->>PB: Format into prompt context
```

This gives the model awareness of:
- Current branch
- Uncommitted changes
- New untracked files
- Recent modifications

---

## Size Limits

| Component | Limit |
|-----------|-------|
| Single instruction file | 4 KB |
| Total instruction files | 12 KB |
| Git status | Truncated if too long |
| Git diff | Truncated if too long |
| Dynamic boundary | Marks where tools are injected |

---

## What's Next?

- **[Slash Commands →](../14-slash-commands/README.md)** — Commands like `/memory` interact with prompt context
- **[Config System →](../09-config-system/README.md)** — Where CLAUDE.md paths are configured

---

[← Sandbox Execution](../12-sandbox-execution/README.md) | [Next: Slash Commands →](../14-slash-commands/README.md)
