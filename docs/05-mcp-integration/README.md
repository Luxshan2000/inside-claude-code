# 🔌 MCP Integration

> **Model Context Protocol.** How Claude Code extends its tool arsenal through external servers.

[← Back to Main](../../README.md) | [← Permission Model](../04-permission-model/README.md)

---

## What Is MCP?

**Model Context Protocol (MCP)** is a standard for connecting AI models to external tools and data sources. Instead of hardcoding every tool, Claude Code can dynamically discover and use tools from MCP servers — turning it into an infinitely extensible agent.

---

## MCP Architecture

```mermaid
graph TB
    subgraph "Claude Code Runtime"
        CONV["Conversation Loop"]
        MCP_MGR["MCP Manager"]
        TOOLS["Tool Registry"]
    end

    subgraph "MCP Servers"
        S1["📁 Filesystem Server<br/>(stdio)"]
        S2["🐙 GitHub Server<br/>(stdio)"]
        S3["🗄️ Database Server<br/>(SSE)"]
        S4["🌐 Custom Server<br/>(WebSocket)"]
    end

    CONV --> TOOLS
    TOOLS --> MCP_MGR
    MCP_MGR <-->|"JSON-RPC"| S1
    MCP_MGR <-->|"JSON-RPC"| S2
    MCP_MGR <-->|"HTTP/SSE"| S3
    MCP_MGR <-->|"WebSocket"| S4
```

---

## Six Transport Types

```mermaid
graph LR
    subgraph "Local"
        STDIO["📟 Stdio<br/>command + args + env"]
    end

    subgraph "Remote"
        SSE["📡 SSE<br/>url + headers"]
        HTTP["🌐 HTTP<br/>url + headers"]
        WS["🔌 WebSocket<br/>url + headers"]
    end

    subgraph "Managed"
        SDK["📦 SDK<br/>named server ref"]
        PROXY["☁️ Claude.ai Proxy<br/>url + server_id"]
    end
```

| Transport | Use Case | Config |
|-----------|----------|--------|
| **Stdio** | Local CLI tools | `command`, `args`, `env` |
| **SSE** | Remote servers (streaming) | `url`, `headers` |
| **HTTP** | Remote servers (request/response) | `url`, `headers` |
| **WebSocket** | Bidirectional real-time | `url`, `headers` |
| **SDK** | Pre-built integrations | `name` reference |
| **Claude.ai Proxy** | Authenticated tunneling | `url`, `server_id` |

---

## Server Lifecycle

```mermaid
sequenceDiagram
    participant RT as 🧠 Runtime
    participant MCP as 📡 MCP Client
    participant SRV as 🔧 Server

    Note over RT: Startup — config has MCP servers

    RT->>MCP: Initialize server "github"
    MCP->>SRV: Spawn process (stdio)<br/>or connect (SSE/WS)

    MCP->>SRV: JSON-RPC: initialize<br/>{protocolVersion, clientInfo}
    SRV-->>MCP: {serverInfo, capabilities}

    MCP->>SRV: JSON-RPC: tools/list
    SRV-->>MCP: [{name, description, inputSchema}, ...]

    MCP->>MCP: Register tools as<br/>mcp__github__create_pr<br/>mcp__github__list_issues

    MCP-->>RT: Tools available

    Note over RT: During conversation...

    RT->>MCP: Execute mcp__github__create_pr
    MCP->>SRV: JSON-RPC: tools/call<br/>{name, arguments}
    SRV-->>MCP: {content: [{type: "text", text: "..."}]}
    MCP-->>RT: Tool result
```

---

## Tool Naming Convention

MCP tools follow a strict naming pattern:

```
mcp__{server_name}__{tool_name}

Examples:
  mcp__github__create_pr
  mcp__database__query
  mcp__filesystem__read_dir
```

**Name normalization:** Non-alphanumeric characters → underscores

```
Server: "my-github-server" → "my_github_server"
Tool:   "create.pull-request" → "create_pull_request"
Result: "mcp__my_github_server__create_pull_request"
```

---

## Configuration

MCP servers are configured in `.claude.json` or `.claude/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_..."
      }
    },
    "database": {
      "type": "sse",
      "url": "https://my-server.com/mcp",
      "headers": {
        "Authorization": "Bearer token123"
      }
    }
  }
}
```

---

## Config Scoping

```mermaid
flowchart TD
    USER["~/.claude.json<br/>Scope: User"] --> MERGE
    PROJECT[".claude.json<br/>Scope: Project"] --> MERGE
    LOCAL[".claude/settings.local.json<br/>Scope: Local"] --> MERGE
    MERGE["Merged Config"] --> SERVERS["MCP Servers<br/>(each tagged with source scope)"]
```

Each server inherits the **scope** of its config source. This determines:
- Which permission policies apply
- Whether the server is shared across projects
- Whether it appears in project exports

---

## OAuth for MCP Servers

Some MCP servers require OAuth authentication:

```mermaid
flowchart LR
    A["Server requires OAuth"] --> B["Check stored tokens"]
    B -->|"Valid token"| C["Attach to requests"]
    B -->|"Expired"| D["Refresh token"]
    B -->|"No token"| E["Initiate OAuth flow"]
    D --> C
    E --> F["PKCE challenge"]
    F --> G["Browser redirect"]
    G --> H["Exchange code for token"]
    H --> C
```

Config:
```json
{
  "mcpServers": {
    "secure-server": {
      "type": "sse",
      "url": "https://server.com/mcp",
      "oauth": {
        "client_id": "abc123",
        "callback_port": 8080,
        "auth_server_metadata_url": "https://auth.server.com/.well-known/oauth"
      }
    }
  }
}
```

---

## MCP Bootstrap Data

```
┌────────────────────────────────────────────────┐
│ McpClientBootstrap                             │
├────────────────────────────────────────────────┤
│ normalized_name: String  ("my_github_server")  │
│ tool_prefix: String      ("mcp__my_github_..") │
│ signature: String        (hash for dedup)      │
│ transport: TransportType (Stdio/SSE/WS/...)    │
│ scope: ConfigSource      (User/Project/Local)  │
└────────────────────────────────────────────────┘
```

---

## What's Next?

- **[Hook System →](../06-hook-system/README.md)** — Hooks work with MCP tools too
- **[Configuration →](../09-config-system/README.md)** — How MCP config is loaded and merged
- **[Authentication →](../10-authentication/README.md)** — OAuth details for MCP servers

---

[← Permission Model](../04-permission-model/README.md) | [Next: Hook System →](../06-hook-system/README.md)
