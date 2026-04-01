# 🖥️ CLI & REPL

> **The terminal experience.** How Claude Code renders markdown, handles input, and creates a polished developer UX.

[← Back to Main](../../README.md) | [← Authentication](../10-authentication/README.md)

---

## CLI Architecture

```mermaid
graph TB
    subgraph "CLI Core"
        INPUT["Input Handler<br/>(line reader + completion)"]
        COMMANDS["Command Dispatcher<br/>(slash commands)"]
        STREAM["Stream Display<br/>(real-time token rendering)"]
        TOOLS_UI["Tool Call UI<br/>(box borders, spinner)"]
        PERM_UI["Permission Prompter<br/>(Y/N for escalation)"]
    end

    subgraph "Terminal Renderer"
        MD["Markdown Parser"]
        SYNTAX["Syntax Highlighter"]
        SPINNER["Spinner Widget"]
        TABLE["Table Formatter"]
    end

    subgraph "External"
        RT["Runtime Core"]
        TERM["Terminal Output"]
    end

    INPUT --> COMMANDS
    INPUT --> RT
    RT --> STREAM
    STREAM --> MD
    MD --> SYNTAX
    MD --> TABLE
    TOOLS_UI --> SPINNER
    MD --> TERM
    SYNTAX --> TERM
    SPINNER --> TERM
```

---

## REPL Loop

```mermaid
flowchart TD
    START(["🟢 Start REPL"]) --> INIT["Initialize:<br/>- Load config<br/>- Build system prompt<br/>- Connect MCP servers<br/>- Show welcome message"]

    INIT --> PROMPT["Show prompt: >"]
    PROMPT --> READ["Read input<br/>(rustyline with history)"]

    READ --> CHECK{"Input type?"}
    CHECK -->|"Empty"| PROMPT
    CHECK -->|"Starts with /"| SLASH["Execute slash command"]
    CHECK -->|"Regular text"| SEND["Send to conversation loop"]

    SLASH --> PROMPT
    SEND --> STREAM_DISPLAY["Stream & display response"]
    STREAM_DISPLAY --> PROMPT

    CHECK -->|"Ctrl+C"| EXIT(["🔴 Exit"])
    CHECK -->|"Ctrl+D (EOF)"| EXIT
```

---

## Input Features

```mermaid
graph LR
    subgraph "Input Handling"
        RL["Rustyline Library"]
        HIST["Command History<br/>(~/.claude/history)"]
        COMP["Tab Completion<br/>(slash commands)"]
        MULTI["Multi-line Input<br/>(Shift+Enter)"]
    end

    RL --> HIST
    RL --> COMP
    RL --> MULTI
```

| Feature | How |
|---------|-----|
| **History** | Up/Down arrows cycle through previous inputs |
| **Completion** | Tab completes slash commands (`/com` → `/compact`) |
| **Multi-line** | Shift+Enter adds a newline without submitting |
| **Editing** | Emacs-style keybindings (Ctrl+A, Ctrl+E, etc.) |

---

## Markdown Rendering Pipeline

```mermaid
flowchart TD
    INPUT["Streaming text<br/>from assistant"] --> STATE["MarkdownStreamState"]

    STATE --> DETECT{"Detect block type"}

    DETECT -->|"```language"| CODE["Code Block"]
    CODE --> SYNTAX["Apply syntax highlighting<br/>(language-aware)"]
    SYNTAX --> BOX["Draw box borders<br/>┌──────────┐<br/>│ code     │<br/>└──────────┘"]

    DETECT -->|"# Heading"| HEADING["Bold + Color"]

    DETECT -->|"- item"| LIST["Bullet formatting"]

    DETECT -->|"|col|col|"| TABLE["Table alignment<br/>& borders"]

    DETECT -->|"Regular text"| WRAP["Word wrap to<br/>terminal width"]

    BOX --> OUTPUT["Terminal Output"]
    HEADING --> OUTPUT
    LIST --> OUTPUT
    TABLE --> OUTPUT
    WRAP --> OUTPUT
```

---

## Tool Call Rendering

When the assistant calls a tool, it's displayed with clear visual boundaries:

```
╭─────────────────────────────────────────╮
│ 🔧 read_file                            │
│ file_path: "src/main.rs"                │
╰─────────────────────────────────────────╯
  ⠋ Reading file...

╭─ Result ────────────────────────────────╮
│ fn main() {                              │
│     println!("Hello, world!");           │
│ }                                        │
╰─────────────────────────────────────────╯
```

---

## Output Formats

```mermaid
graph TD
    OUTPUT{"--output-format?"}
    OUTPUT -->|"text (default)"| TEXT["Rich terminal output<br/>Markdown, colors, borders"]
    OUTPUT -->|"json"| JSON["Structured JSON<br/>One object per message"]
    OUTPUT -->|"ndjson"| NDJSON["Newline-delimited JSON<br/>Stream-friendly"]
```

---

## CLI Arguments

```
claw [OPTIONS] [PROMPT]

Options:
  --model <MODEL>              Model to use (opus/sonnet/haiku)
  --permission-mode <MODE>     ReadOnly/WorkspaceWrite/DangerFullAccess
  --config <PATH>              Custom config file
  --output-format <FORMAT>     text/json/ndjson
  --resume <SESSION_ID>        Resume a saved session

Subcommands:
  init          Initialize project configuration
  login         Authenticate via OAuth
  logout        Clear stored credentials
```

---

## What's Next?

- **[Sandbox Execution →](../12-sandbox-execution/README.md)** — How bash commands are isolated
- **[Slash Commands →](../14-slash-commands/README.md)** — The full command registry

---

[← Authentication](../10-authentication/README.md) | [Next: Sandbox Execution →](../12-sandbox-execution/README.md)
