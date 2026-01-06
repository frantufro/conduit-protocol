# Conduit Hub Reference Implementation (Rust)

**Status**: Planning
**Target**: Full reference implementation for personal use, protocol validation, and developer demos

## Overview

A complete Rust implementation of the Conduit Protocol, including:

- **Hub**: Central pub/sub broker with all protocol features
- **Adapters**: 12 adapters covering email, messaging, notifications, and calendars
- **CLI Agent**: Interactive agent for testing and manual signal management
- **Hub CLI**: Management tool for configuration, keys, and subscription approval

## Architecture

```
conduit/
├── crates/
│   ├── conduit-hub/          # Core Hub server
│   ├── conduit-cli/          # Hub management CLI
│   ├── conduit-agent-cli/    # Interactive CLI agent
│   ├── conduit-common/       # Shared types, schemas, utilities
│   │
│   ├── adapter-echo/         # Echo adapter (testing)
│   ├── adapter-cli/          # CLI adapter (stdin/stdout)
│   ├── adapter-rss/          # RSS feed adapter
│   ├── adapter-github/       # GitHub notifications
│   ├── adapter-telegram/     # Telegram Bot API
│   ├── adapter-gmail/        # Gmail (IMAP + OAuth)
│   ├── adapter-fastmail/     # Fastmail (JMAP)
│   ├── adapter-slack/        # Slack Events API
│   ├── adapter-discord/      # Discord Gateway
│   ├── adapter-gcal/         # Google Calendar
│   └── adapter-caldav/       # CalDAV (generic)
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
└── config/
    └── conduit.example.toml
```

## Hub Features

### Core Protocol

| Feature | Status | Notes |
|---------|--------|-------|
| JSON-RPC 2.0 wire protocol | Required | All methods per spec |
| Pub/Sub routing | Required | Topic matching with wildcards |
| Subscription management | Required | Auto + user-approved |
| Signal acknowledgment | Required | At-least-once delivery |
| Redelivery with backoff | Required | Exponential backoff, dead-letter queue |
| Version negotiation | Required | Semver, handshake validation |

### Transports

| Transport | Status | Notes |
|-----------|--------|-------|
| WebSocket | Required | Full-duplex, real-time |
| SSE | Required | Unidirectional streaming |
| HTTP Polling | Required | Simple request/response |
| Long Polling | Required | Held connection until signals |
| Webhooks | Required | Server-to-server push |

### Integrations

| Integration | Status | Notes |
|-------------|--------|-------|
| A2A Gateway | Required | Agent Card, SendMessage, push notifications |
| MCP Interface | Required | Resources, Tools, Prompts per spec Section 10 |

### Security

| Feature | Status | Notes |
|---------|--------|-------|
| TLS | Required | TLS 1.2+ for all connections |
| API Key auth | Required | Static keys in config |
| JWT auth | Deferred | Future enhancement |
| E2E encryption | Deferred | Future enhancement |

### Storage

| Backend | Status | Notes |
|---------|--------|-------|
| SQLite | Required | Embedded, single-file database |
| PostgreSQL | Deferred | Future enhancement |

## Hub CLI (`conduit-cli`)

Management commands for the Hub:

```bash
# Configuration
conduit init                    # Create config file
conduit config show             # Display current config
conduit config set <key> <val>  # Update config

# API Keys
conduit keys list               # List all API keys
conduit keys create <name>      # Generate new API key
conduit keys revoke <id>        # Revoke an API key

# Subscriptions
conduit subscriptions list      # List all subscriptions
conduit subscriptions pending   # List pending approvals
conduit subscriptions approve <id>  # Approve subscription
conduit subscriptions deny <id>     # Deny subscription
conduit subscriptions revoke <id>   # Revoke active subscription

# Topics
conduit topics list             # List known topics
conduit topics public           # List public topics (A2A accessible)

# Server
conduit serve                   # Start the Hub server
conduit status                  # Check Hub health
```

## CLI Agent (`conduit-agent-cli`)

Interactive agent for testing and manual operation:

```bash
conduit-agent connect wss://localhost:8080/conduit/v1/ws

# In interactive mode:
> subscribe signal.email.*
Subscribed to signal.email.* (sub_abc123)

> signals
[sig_123] signal.email.received from=alice@example.com subject="Meeting"
[sig_456] signal.slack.message channel=#general "Hey team"

> ack sig_123
Acknowledged sig_123

> send action.email.send to=alice@example.com subject="Re: Meeting" body="Works for me!"
Published act_789

> quit
```

## Adapters

### Tier 1: Testing (Ship First)

| Adapter | Protocol | Signals | Actions |
|---------|----------|---------|---------|
| **Echo** | Internal | Mirrors any signal back | Mirrors any action |
| **CLI** | stdin/stdout | Text lines → signals | Actions → stdout |

### Tier 2: Easy (Good APIs)

| Adapter | Protocol | Signals | Actions |
|---------|----------|---------|---------|
| **RSS** | HTTP polling | Feed items → `signal.rss.item` | N/A (read-only) |
| **GitHub** | REST + Webhooks | Notifications → `signal.github.*` | Create issues, comments |
| **Telegram** | Bot API | Messages → `signal.telegram.message` | Send messages |

### Tier 3: Medium (OAuth Required)

| Adapter | Protocol | Signals | Actions |
|---------|----------|---------|---------|
| **Gmail** | IMAP + OAuth | Emails → `signal.email.received` | Send, reply, forward |
| **Fastmail** | JMAP | Emails → `signal.email.received` | Send, reply, forward |
| **Slack** | Events API | Messages → `signal.slack.message` | Send, react |
| **Discord** | Gateway API | Messages → `signal.discord.message` | Send, react |
| **Google Calendar** | REST + OAuth | Events → `signal.calendar.*` | Accept, decline, create |
| **CalDAV** | CalDAV | Events → `signal.calendar.*` | Accept, decline, create |

### Deferred

| Adapter | Protocol | Notes |
|---------|----------|-------|
| **WhatsApp** | Business API | Requires Meta approval. Revisit post-MVP. |

## Configuration

```toml
# conduit.toml

[hub]
address = "0.0.0.0:8080"
database = "conduit.db"

[hub.tls]
cert = "/path/to/cert.pem"
key = "/path/to/key.pem"

[hub.transports]
websocket = true
sse = true
polling = true
long_polling = true
webhooks = true

[hub.a2a]
enabled = true
public_topics = ["signal.blog.*", "signal.status.*"]

[hub.mcp]
enabled = true

# Adapter configs
[adapters.gmail]
enabled = true
client_id = "..."
client_secret = "..."
poll_interval = "60s"

[adapters.slack]
enabled = true
app_token = "xapp-..."
bot_token = "xoxb-..."

# ... more adapters
```

## Distribution

### Binary Releases

```bash
# Install from crates.io
cargo install conduit-hub
cargo install conduit-cli
cargo install conduit-agent-cli

# Or download from GitHub releases
curl -L https://github.com/.../releases/latest/download/conduit-hub-x86_64-linux -o conduit-hub
```

### Docker

```bash
# Run Hub
docker run -p 8080:8080 -v ./config:/config conduit/hub

# Docker Compose (Hub + adapters)
docker-compose up
```

```yaml
# docker-compose.yml
services:
  hub:
    image: conduit/hub
    ports:
      - "8080:8080"
    volumes:
      - ./config:/config
      - ./data:/data

  gmail-adapter:
    image: conduit/adapter-gmail
    environment:
      CONDUIT_HUB_URL: ws://hub:8080
      CONDUIT_API_KEY: ${GMAIL_ADAPTER_KEY}
    depends_on:
      - hub

  slack-adapter:
    image: conduit/adapter-slack
    environment:
      CONDUIT_HUB_URL: ws://hub:8080
      CONDUIT_API_KEY: ${SLACK_ADAPTER_KEY}
    depends_on:
      - hub
```

## Development Phases

### Phase 1: Core Hub

1. Project setup (Cargo workspace, CI)
2. Common types and JSON schema validation
3. SQLite storage layer
4. Pub/sub engine (topics, subscriptions, routing)
5. WebSocket transport
6. JSON-RPC method handlers
7. Acknowledgment and redelivery

### Phase 2: Additional Transports

1. SSE transport
2. HTTP Polling transport
3. Long Polling transport
4. Webhook transport

### Phase 3: Testing Infrastructure

1. Echo adapter
2. CLI adapter
3. CLI agent
4. Hub CLI (basic commands)

### Phase 4: Integrations

1. A2A Gateway (Agent Card, SendMessage, push)
2. MCP Interface (Resources, Tools, Prompts)

### Phase 5: Real Adapters (Tier 2)

1. RSS adapter
2. GitHub adapter
3. Telegram adapter

### Phase 6: Real Adapters (Tier 3)

1. Gmail adapter
2. Fastmail adapter
3. Slack adapter
4. Discord adapter
5. Google Calendar adapter
6. CalDAV adapter

### Phase 7: Polish

1. Hub CLI (full commands)
2. Docker images
3. Documentation

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Rust (stable) |
| Async runtime | Tokio |
| Web framework | Axum |
| WebSocket | tokio-tungstenite |
| Database | SQLite via rusqlite/sqlx |
| Serialization | serde + serde_json |
| Schema validation | jsonschema |
| CLI parsing | clap |
| Logging | tracing |
| Config | toml + config |

## Success Criteria

1. **Protocol Compliance**: All spec methods work correctly
2. **Reliable Delivery**: Signals survive Hub restarts, unacked signals redeliver
3. **Multi-Transport**: Same agent can connect via any transport
4. **A2A Interop**: External A2A agents can discover and interact with Hub
5. **MCP Interop**: MCP clients can subscribe and publish via tools
6. **Personal Use**: Can run on a laptop or VPS for real email/Slack/etc.
7. **Developer Experience**: Easy to build new adapters using examples

## Design Decisions

1. **Adapter process model**: Both separate binaries and Hub plugins are valid.
   The protocol is agnostic to deployment model. Start with separate binaries
   for simplicity; plugins can be added later if beneficial.

2. **OAuth token storage**: Each adapter stores its own tokens in a local file.
   Keeps adapters self-contained and simplifies the Hub.

3. **WhatsApp**: Deferred. Business API requires Meta approval which is complex.
   Can be revisited post-MVP via Matrix bridge or official API.
