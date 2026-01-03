# Conduit Protocol

Open protocol enabling AI agents to intelligently manage all your communications in one place.

## Overview

Conduit is a unified protocol for personal communication arbitrage. It provides a standardized way to:

- **Receive signals** from diverse sources (email, SMS, Slack, voice, etc.)
- **Route intelligently** through a pub/sub hub architecture
- **Enable AI agents** to prioritize, filter, summarize, and respond to communications
- **Interoperate** with the broader AI ecosystem via A2A protocol support

Think of it as a **universal inbox protocol** - your AI agent subscribes to all your communication channels through one interface.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       CONDUIT ECOSYSTEM                         │
│                                                                 │
│  [Email] → [Adapter] ──┐                                       │
│  [SMS]   → [Adapter] ──┼──→ [Hub] ←──→ [Your AI Agent]        │
│  [Slack] → [Adapter] ──┘      ↑                                │
│                               └────── [External A2A Agents]    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Components:**

| Component | Role |
|-----------|------|
| **Adapter** | Connects to a communication source, publishes signals to Hub |
| **Hub** | Pub/sub message broker that routes signals and actions |
| **Agent** | Subscribes to signals, decides what to do (your implementation) |

## Key Features

- **Pub/Sub Architecture**: Everything is topics and subscriptions
- **Transport Agnostic**: WebSocket, SSE, polling, long-polling, webhooks
- **Privacy First**: TLS mandatory, end-to-end encryption optional
- **A2A Compatible**: External agents can participate as first-class citizens
- **Agent Agnostic**: Protocol defines communication, not AI logic

## Quick Example

**Inbound signal (email received):**
```json
{
  "id": "sig_1704288600_abc123",
  "topic": "signal.email.received",
  "source": { "type": "email", "adapter_id": "gmail_01" },
  "payload": {
    "raw": {
      "from": "alice@example.com",
      "subject": "Meeting tomorrow",
      "body_text": "Hi, can we meet at 3pm?"
    }
  }
}
```

**Outbound action (send reply):**
```json
{
  "id": "act_1704289000_xyz789",
  "topic": "action.email.send",
  "action": {
    "type": "send",
    "target": "email",
    "payload": {
      "to": ["alice@example.com"],
      "subject": "Re: Meeting tomorrow",
      "body_text": "3pm works for me!"
    }
  }
}
```

## Documentation

- [Full Protocol Specification](./conduit-protocol-spec.md)

## Roadmap

| Phase | Deliverable | Status |
|-------|-------------|--------|
| 1 | Protocol Specification | Draft |
| 2 | JSON Schemas | Planned |
| 3 | Rust SDK | Planned |
| 4 | Python/TypeScript Bindings | Planned |
| 5 | Reference Implementations | Planned |

## Design Principles

1. **Hub is thin** - Pure message routing, no business logic
2. **Agents own intelligence** - Extraction, identity resolution, arbitrage logic
3. **Adapters are simple** - Just translate source ↔ Conduit format
4. **Free-form topics** - With recommended conventions for interoperability
5. **Subscriptions can require approval** - User controls who sees their data

## Inspired By

- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) - AI ↔ Tools
- [Agent2Agent Protocol (A2A)](https://github.com/a2aproject/A2A) - Agent ↔ Agent

Conduit fills the gap: **Humans ↔ AI Agents** for personal communications.

## License

[MIT](./LICENSE)

## Contributing

This project is in early development. Contributions, feedback, and discussions are welcome.
