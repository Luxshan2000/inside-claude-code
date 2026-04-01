# ⚙️ Configuration System

> **Five sources, one config.** How Claude Code discovers, loads, and merges settings from everywhere.

[← Back to Main](../../README.md) | [← Streaming & SSE](../08-streaming-and-sse/README.md)

---

## The Challenge

Configuration can come from 5+ different files, CLI flags, and environment variables. The config system resolves them all into a single, merged runtime config.

---

## Config Discovery Chain

```mermaid
flowchart TD
    subgraph "1️⃣ User Level"
        U1["~/.claude.json"]
        U2["~/.claude/settings.json"]
    end

    subgraph "2️⃣ Project Level"
        P1[".claude.json<br/>(project root)"]
        P2[".claude/settings.json"]
    end

    subgraph "3️⃣ Local Level"
        L1[".claude/settings.local.json<br/>(gitignored, machine-specific)"]
    end

    subgraph "4️⃣ CLI Flags"
        CLI["--model, --permission-mode,<br/>--config, --output-format"]
    end

    subgraph "5️⃣ Environment Variables"
        ENV["ANTHROPIC_API_KEY<br/>CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS<br/>CLAUDE_CODE_REMOTE"]
    end

    U1 --> MERGE
    U2 --> MERGE
    P1 --> MERGE
    P2 --> MERGE
    L1 --> MERGE
    CLI --> OVERRIDE
    ENV --> OVERRIDE

    MERGE["Deep Merge<br/>(later sources override)"] --> OVERRIDE["CLI & Env Override"]
    OVERRIDE --> FINAL["RuntimeConfig"]
```

---

## Merge Strategy

```mermaid
graph LR
    subgraph "User Config"
        UC["model: 'sonnet'<br/>permissions: 'readOnly'"]
    end

    subgraph "Project Config"
        PC["model: 'opus'<br/>mcpServers: {github: ...}"]
    end

    subgraph "Local Config"
        LC["permissions: 'dontAsk'"]
    end

    UC --> MERGE
    PC --> MERGE
    LC --> MERGE

    MERGE["Deep Merge"] --> RESULT["model: 'opus'<br/>permissions: 'dontAsk'<br/>mcpServers: {github: ...}"]
```

**Rules:**
- Later sources override earlier ones for scalar values
- Objects (like `mcpServers`) are deep-merged
- Arrays are replaced, not concatenated
- CLI flags always win

---

## ConfigSource Scoping

```mermaid
classDiagram
    class ConfigSource {
        <<enumeration>>
        User
        Project
        Local
    }

    class ConfigEntry {
        +source: ConfigSource
        +path: PathBuf
        +data: BTreeMap
    }

    class ConfigLoader {
        +discover() Vec~ConfigEntry~
        +load() RuntimeConfig
    }

    class RuntimeConfig {
        +merged: BTreeMap
        +entries: Vec~ConfigEntry~
        +hooks() HookConfig
        +mcp_servers() McpConfig
        +oauth() OAuthConfig
        +model() String
        +permissions() PermissionConfig
        +sandbox() SandboxConfig
    }

    ConfigLoader --> ConfigEntry
    ConfigEntry --> ConfigSource
    ConfigLoader --> RuntimeConfig
```

---

## Feature Extraction

The RuntimeConfig provides typed accessors for each feature:

```
┌───────────────────────────────────────────────────┐
│ RuntimeConfig                                     │
├───────────────────────────────────────────────────┤
│ .hooks()         → RuntimeHookConfig              │
│ .mcp_servers()   → HashMap<String, McpServerConfig>│
│ .oauth()         → OAuthConfig                    │
│ .model()         → String (e.g., "claude-opus-4-6")│
│ .permissions()   → PermissionConfig               │
│ .sandbox()       → SandboxConfig                  │
└───────────────────────────────────────────────────┘
```

---

## Model Aliases

```mermaid
graph LR
    A1["'opus'"] --> R1["claude-opus-4-6"]
    A2["'sonnet'"] --> R2["claude-sonnet-4-6"]
    A3["'haiku'"] --> R3["claude-haiku-4-5-20251213"]
    DEFAULT["(default)"] --> R1
```

---

## Config File Examples

### Minimal `.claude.json`
```json
{
  "permissions": {
    "defaultMode": "dontAsk"
  }
}
```

### Full-Featured `.claude.json`
```json
{
  "model": "opus",
  "permissions": {
    "defaultMode": "workspaceWrite"
  },
  "hooks": {
    "PreToolUse": [
      { "command": "python3 safety-check.py" }
    ]
  },
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  },
  "sandbox": {
    "enabled": true,
    "filesystem_mode": "WorkspaceOnly"
  }
}
```

---

## What's Next?

- **[Authentication →](../10-authentication/README.md)** — OAuth and API key configuration
- **[Permission Model →](../04-permission-model/README.md)** — Permission config in detail

---

[← Streaming & SSE](../08-streaming-and-sse/README.md) | [Next: Authentication →](../10-authentication/README.md)
