# 🔑 Authentication

> **Two paths to the API.** API keys for simplicity, OAuth PKCE for security.

[← Back to Main](../../README.md) | [← Config System](../09-config-system/README.md)

---

## Authentication Methods

Claude Code supports two authentication methods that can be used independently or together:

```mermaid
graph TD
    subgraph "Method 1: API Key"
        AK["ANTHROPIC_API_KEY<br/>environment variable"]
        AK --> HEADER1["Header: x-api-key"]
    end

    subgraph "Method 2: OAuth Bearer Token"
        OA["ANTHROPIC_AUTH_TOKEN<br/>or OAuth PKCE flow"]
        OA --> HEADER2["Header: Authorization: Bearer"]
    end

    HEADER1 --> REQ["API Request"]
    HEADER2 --> REQ
```

---

## Auth Resolution Chain

```mermaid
flowchart TD
    START["API request needs auth"] --> CHECK_KEY{"ANTHROPIC_API_KEY<br/>set?"}

    CHECK_KEY -->|"Yes"| HAS_KEY["Use API key"]
    CHECK_KEY -->|"No"| CHECK_TOKEN{"ANTHROPIC_AUTH_TOKEN<br/>set?"}

    CHECK_TOKEN -->|"Yes"| HAS_TOKEN["Use bearer token"]
    CHECK_TOKEN -->|"No"| CHECK_OAUTH{"OAuth tokens<br/>stored?"}

    CHECK_OAUTH -->|"Yes"| EXPIRED{"Token<br/>expired?"}
    EXPIRED -->|"No"| USE_OAUTH["Use stored OAuth token"]
    EXPIRED -->|"Yes"| REFRESH["Refresh token"]
    REFRESH -->|"Success"| USE_OAUTH
    REFRESH -->|"Failure"| NO_AUTH["❌ MissingApiKey error"]

    CHECK_OAUTH -->|"No"| NO_AUTH

    HAS_KEY --> SEND["✅ Send request"]
    HAS_TOKEN --> SEND
    USE_OAUTH --> SEND
```

---

## OAuth PKCE Flow

PKCE (Proof Key for Code Exchange) is the secure OAuth flow for CLI applications:

```mermaid
sequenceDiagram
    participant CLI as 🖥️ Claude CLI
    participant B as 🌐 Browser
    participant AS as 🔐 Auth Server

    Note over CLI: Generate PKCE pair

    CLI->>CLI: Generate random verifier (43-128 chars)
    CLI->>CLI: code_challenge = SHA256(verifier)
    CLI->>CLI: Generate random state (UUID)
    CLI->>CLI: Start local HTTP server on callback_port

    CLI->>B: Open browser to auth URL:<br/>/authorize?response_type=code<br/>&client_id=abc<br/>&redirect_uri=http://localhost:8080<br/>&code_challenge=xyz<br/>&code_challenge_method=S256<br/>&state=uuid

    B->>AS: User logs in & approves
    AS->>B: Redirect to localhost:8080?code=AUTH_CODE&state=uuid

    B->>CLI: GET /callback?code=AUTH_CODE&state=uuid
    CLI->>CLI: Verify state matches

    CLI->>AS: POST /token<br/>{grant_type: authorization_code,<br/>code: AUTH_CODE,<br/>code_verifier: verifier}

    AS-->>CLI: {access_token, refresh_token,<br/>expires_at, scopes}

    CLI->>CLI: Store tokens securely<br/>(SHA256-hashed path in ~/.claude/)
```

---

## Token Storage

```
┌──────────────────────────────────────────┐
│ Token Set                                │
├──────────────────────────────────────────┤
│ Access Token                             │
│ Refresh Token (optional)                 │
│ Expiration Time (optional)               │
│ Granted Scopes                           │
├──────────────────────────────────────────┤
│ Storage: ~/.claude/credentials/<hash>    │
│ Hash: SHA256 of client config            │
│ File permissions: owner read/write only  │
└──────────────────────────────────────────┘
```

---

## Token Refresh — Automatic

```mermaid
flowchart TD
    REQ["API request"] --> CHECK{"Token<br/>expired?"}
    CHECK -->|"No"| USE["Use current token"]
    CHECK -->|"Yes"| REFRESH["POST /token<br/>grant_type=refresh_token"]
    REFRESH --> SUCCESS{"Refresh<br/>succeeded?"}
    SUCCESS -->|"Yes"| STORE["Store new tokens"]
    STORE --> USE
    SUCCESS -->|"No"| ERROR["ExpiredOAuthToken<br/>error → re-login needed"]
```

---

## Security Features

| Feature | Implementation |
|---------|---------------|
| **PKCE** | S256 challenge prevents authorization code interception |
| **State parameter** | UUID prevents CSRF attacks |
| **Secure storage** | SHA256-hashed file paths, 0600 permissions |
| **Token masking** | Sensitive tokens redacted in debug/log output |
| **Loopback redirect** | localhost-only callback (no external server needed) |
| **Automatic refresh** | Expired tokens refreshed transparently |

---

## What's Next?

- **[CLI & REPL →](../11-cli-and-repl/README.md)** — Where auth commands live (`/login`, `/logout`)
- **[Config System →](../09-config-system/README.md)** — Where API keys are configured

---

[← Config System](../09-config-system/README.md) | [Next: CLI & REPL →](../11-cli-and-repl/README.md)
