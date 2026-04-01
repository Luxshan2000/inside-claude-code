# 🛡️ Sandbox Execution

> **OS-level isolation.** How Claude Code contains bash commands to prevent damage.

[← Back to Main](../../README.md) | [← CLI & REPL](../11-cli-and-repl/README.md)

---

## Why Sandbox?

When an AI agent runs `bash` commands, it has the same power as the user. Sandboxing adds OS-level guardrails to prevent accidental (or adversarial) damage — filesystem isolation, network restrictions, and namespace confinement.

---

## Sandbox Architecture

```mermaid
flowchart TD
    BASH["bash('cargo test')"] --> CONFIG{"Sandbox<br/>enabled?"}

    CONFIG -->|"No"| DIRECT["Execute directly<br/>in host shell"]

    CONFIG -->|"Yes"| MODE{"Filesystem<br/>isolation mode?"}

    MODE -->|"Off"| DIRECT
    MODE -->|"WorkspaceOnly"| WS["Restrict to workspace<br/>directory tree"]
    MODE -->|"AllowList"| AL["Restrict to<br/>allowed mount paths"]

    WS --> NS["Apply namespace<br/>restrictions"]
    AL --> NS

    NS --> NET{"Network<br/>isolation?"}
    NET -->|"Yes"| NO_NET["Block network<br/>access"]
    NET -->|"No"| WITH_NET["Allow network"]

    NO_NET --> EXEC["Execute in<br/>sandboxed environment"]
    WITH_NET --> EXEC
```

---

## Filesystem Isolation Modes

```mermaid
graph TD
    subgraph "Off"
        OFF["No filesystem restrictions<br/>Full access to everything"]
    end

    subgraph "WorkspaceOnly (default)"
        WO["Access limited to:<br/>• Project directory<br/>• /tmp<br/>• Tool-specific paths"]
    end

    subgraph "AllowList"
        AL["Access limited to:<br/>• Explicitly listed paths<br/>• Configured mount points"]
    end
```

---

## Sandbox Configuration

```mermaid
classDiagram
    class SandboxConfig {
        +enabled (yes/no)
        +namespace restrictions
        +network isolation
        +filesystem mode
        +allowed mount paths
    }

    class FilesystemMode {
        <<enumeration>>
        Off
        WorkspaceOnly
        AllowList
    }

    class SandboxStatus {
        +is active
        +filesystem mode
        +network isolated
        +reason if inactive
    }

    SandboxConfig --> FilesystemMode
    SandboxConfig ..> SandboxStatus
```

---

## Container Detection

Before applying sandboxing, the system checks if it's already inside a container:

```mermaid
flowchart TD
    CHECK["Detect environment"] --> DOCKER{"/.dockerenv<br/>exists?"}
    DOCKER -->|"Yes"| CONTAINER["Running in Docker"]
    DOCKER -->|"No"| PODMAN{"/run/.containerenv<br/>exists?"}
    PODMAN -->|"Yes"| CONTAINER
    PODMAN -->|"No"| CGROUP{"Check /proc/1/cgroup<br/>for container markers"}
    CGROUP -->|"Found"| CONTAINER
    CGROUP -->|"Not found"| HOST["Running on host"]

    CONTAINER --> REPORT["SandboxStatus:<br/>reason = 'Already containerized'"]
    HOST --> APPLY["Apply sandbox<br/>restrictions"]
```

---

## Background Execution

Some bash commands run in the background (e.g., dev servers):

```mermaid
flowchart LR
    REQ["bash('npm start',<br/>run_in_background: true)"] --> SPAWN["Spawn subprocess"]
    SPAWN --> DETACH["Detach from<br/>conversation loop"]
    DETACH --> RETURN["Return immediately<br/>with process info"]
```

---

## Configuration Example

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem_mode": "WorkspaceOnly",
    "network_isolation": false,
    "allowed_mounts": [
      "/usr/local/bin",
      "/home/user/.cargo"
    ]
  }
}
```

---

## What's Next?

- **[System Prompt Building →](../13-system-prompt-building/README.md)** — What context the sandbox operates within
- **[Permission Model →](../04-permission-model/README.md)** — Application-level access control

---

[← CLI & REPL](../11-cli-and-repl/README.md) | [Next: System Prompt Building →](../13-system-prompt-building/README.md)
