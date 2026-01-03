# Conduit Protocol Specification

**Version:** 1.0.0-draft
**Status:** Draft
**Date:** 2026-01-03

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture](#2-architecture)
3. [Components](#3-components)
4. [Message Format](#4-message-format)
5. [Signal Schema](#5-signal-schema)
6. [Pub/Sub System](#6-pubsub-system)
7. [Transport Bindings](#7-transport-bindings)
8. [Security](#8-security)
9. [A2A Integration](#9-a2a-integration)
10. [Error Handling](#10-error-handling)
11. [Versioning](#11-versioning)
12. [Appendices](#12-appendices)

---

## 1. Introduction

### 1.1 Purpose

The Conduit Protocol is an open standard for personal communication arbitrage. It provides a unified way to receive signals from diverse communication sources (email, SMS, messaging apps, voice, etc.) and enables AI agents to intelligently prioritize, filter, route, summarize, and respond to these signals based on individual preferences.

### 1.2 Design Principles

1. **Transport Agnostic**: Support multiple transport mechanisms (WebSocket, SSE, polling, webhooks, long-polling)
2. **Agent Agnostic**: The protocol defines communication patterns; agent implementation is out of scope
3. **Privacy First**: TLS mandatory, end-to-end encryption as optional extension
4. **Extensible**: Free-form topics with recommended conventions
5. **Interoperable**: Built on JSON-RPC 2.0, compatible with A2A protocol

### 1.3 Terminology

| Term | Definition |
|------|------------|
| **Signal** | A unit of communication from any source (email, SMS, Slack message, etc.) |
| **Adapter** | Component that connects to a communication source and publishes signals |
| **Hub** | Central pub/sub message broker that routes signals and actions |
| **Agent** | AI system that subscribes to signals and publishes actions (out of protocol scope) |
| **Topic** | Named channel for pub/sub messaging |
| **Subscription** | Registration to receive messages on specific topics |

### 1.4 Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 2. Architecture

### 2.1 Overview

Conduit follows a hybrid hub-and-spoke architecture with adapters translating source-specific protocols into a unified signal format.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            CONDUIT ECOSYSTEM                            │
│                                                                         │
│  ┌─────────┐     ┌─────────┐                                           │
│  │  Email  │     │   SMS   │                                           │
│  │ Source  │     │ Source  │    ... other sources                      │
│  └────┬────┘     └────┬────┘                                           │
│       │               │                                                 │
│       ▼               ▼                                                 │
│  ┌─────────┐     ┌─────────┐                                           │
│  │  Email  │     │   SMS   │    ... other adapters                     │
│  │ Adapter │     │ Adapter │                                           │
│  └────┬────┘     └────┬────┘                                           │
│       │               │                                                 │
│       │   PUBLISH     │   PUBLISH                                       │
│       ▼               ▼                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         CONDUIT HUB                               │  │
│  │                      (Pub/Sub Broker)                             │  │
│  │                                                                   │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │  │
│  │  │ Topic       │  │Subscription │  │   Queue     │               │  │
│  │  │ Registry    │  │  Manager    │  │  Manager    │               │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘               │  │
│  │                                                                   │  │
│  │  ┌─────────────┐  ┌─────────────┐                                │  │
│  │  │ A2A Gateway │  │  Security   │                                │  │
│  │  │             │  │   Layer     │                                │  │
│  │  └─────────────┘  └─────────────┘                                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│       │               │               │                                 │
│       │  SUBSCRIBE    │  SUBSCRIBE    │  A2A                           │
│       ▼               ▼               ▼                                 │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                          │
│  │ Conduit │     │ Conduit │     │External │                          │
│  │ Agent 1 │     │ Agent 2 │     │A2A Agent│                          │
│  └─────────┘     └─────────┘     └─────────┘                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow

#### 2.2.1 Inbound Signal Flow

1. Source generates communication (e.g., email received)
2. Adapter receives from source using source-native protocol
3. Adapter wraps in Conduit signal format
4. Adapter publishes to Hub on appropriate topic
5. Hub routes to all subscribers of that topic
6. Agents receive and process signal

#### 2.2.2 Outbound Action Flow

1. Agent decides to send communication (e.g., reply to email)
2. Agent publishes action to Hub on appropriate topic
3. Adapter (subscribed to action topics) receives action
4. Adapter translates to source-native format
5. Adapter sends via source

#### 2.2.3 Agent-to-Agent Flow (via A2A)

1. External A2A agent connects to Hub's A2A endpoint
2. External agent subscribes to signals or sends tasks
3. Hub treats A2A agent as first-class subscriber
4. Responses flow back via A2A protocol (SSE default)

---

## 3. Components

### 3.1 Adapter

An Adapter connects a communication source to the Conduit Hub.

#### 3.1.1 Responsibilities

| Responsibility | Description |
|---------------|-------------|
| Source Authentication | Authenticate with the source (e.g., OAuth for Gmail) - out of protocol scope |
| Signal Translation | Convert source-native format to Conduit signal format |
| Publishing | Publish signals to Hub |
| Subscribing | Subscribe to action topics for outbound messages |
| Queuing | Queue signals locally if Hub is unavailable |
| Acknowledgment | Acknowledge successful signal delivery |

#### 3.1.2 Requirements

- Adapters MUST implement local queuing for resilience
- Adapters MUST generate unique signal IDs
- Adapters MUST support at least one transport binding
- Adapters SHOULD support TLS
- Adapters MAY support E2E encryption

#### 3.1.3 Queuing Behavior

When the Hub is unavailable:

1. Adapter MUST queue signals locally
2. Adapter MUST persist queue across restarts
3. Adapter MUST retry delivery with exponential backoff
4. Adapter SHOULD implement queue size limits
5. Adapter SHOULD implement oldest-first eviction when queue is full

```json
{
  "queue_config": {
    "max_size": 10000,
    "retry_interval_ms": 30000,
    "max_retry_interval_ms": 300000,
    "backoff_multiplier": 2.0,
    "eviction_policy": "oldest_first"
  }
}
```

### 3.2 Hub

The Hub is the central pub/sub message broker.

#### 3.2.1 Responsibilities

| Responsibility | Description |
|---------------|-------------|
| Pub/Sub Routing | Route messages between publishers and subscribers |
| Subscription Management | Manage subscriptions (auto and user-approved) |
| Queue Management | Buffer messages for offline subscribers |
| Delivery Tracking | Track acknowledgments and retry failed deliveries |
| A2A Gateway | Expose A2A endpoint for external agents |
| Security | Enforce TLS, manage sessions, optional E2E |

#### 3.2.2 Requirements

- Hub MUST implement pub/sub message routing
- Hub MUST support all five transport bindings
- Hub MUST support subscription management
- Hub MUST enforce TLS for all connections
- Hub MUST expose A2A-compatible endpoint
- Hub MUST NOT inspect E2E encrypted payloads
- Hub SHOULD implement message persistence
- Hub SHOULD support wildcard subscriptions

### 3.3 Agent

Agents subscribe to signals and publish actions. Agent implementation is **out of protocol scope**.

#### 3.3.1 Typical Agent Responsibilities (Informative)

These are not protocol requirements, but typical agent functions:

- Content extraction (PDF → text, audio → transcript)
- Identity resolution (contacts database)
- Arbitrage logic (prioritize, filter, summarize)
- Action decisions (archive, snooze, respond, delegate)
- Response generation

#### 3.3.2 Protocol Requirements for Agents

- Agents MUST acknowledge received signals
- Agents MUST use valid topic names when publishing
- Agents MUST support at least one transport binding

---

## 4. Message Format

### 4.1 Wire Protocol

Conduit uses JSON-RPC 2.0 as the wire protocol.

#### 4.1.1 Request

```json
{
  "jsonrpc": "2.0",
  "id": "req_abc123",
  "method": "conduit.subscribe",
  "params": {
    "topics": ["signal.email.*"],
    "transport": "sse"
  }
}
```

#### 4.1.2 Response

```json
{
  "jsonrpc": "2.0",
  "id": "req_abc123",
  "result": {
    "subscription_id": "sub_xyz789",
    "status": "active"
  }
}
```

#### 4.1.3 Notification (No Response Expected)

```json
{
  "jsonrpc": "2.0",
  "method": "conduit.signal",
  "params": {
    "topic": "signal.email.received",
    "signal": { ... }
  }
}
```

#### 4.1.4 Error Response

```json
{
  "jsonrpc": "2.0",
  "id": "req_abc123",
  "error": {
    "code": -32600,
    "message": "Invalid Request",
    "data": {
      "details": "Missing required field: topics"
    }
  }
}
```

### 4.2 Method Catalog

#### 4.2.1 Connection Methods

| Method | Direction | Description |
|--------|-----------|-------------|
| `conduit.hello` | Client → Hub | Initial handshake, capability negotiation |
| `conduit.goodbye` | Client → Hub | Graceful disconnect |
| `conduit.ping` | Bidirectional | Keepalive |
| `conduit.pong` | Bidirectional | Keepalive response |

#### 4.2.2 Pub/Sub Methods

| Method | Direction | Description |
|--------|-----------|-------------|
| `conduit.publish` | Client → Hub | Publish message to topic |
| `conduit.subscribe` | Client → Hub | Subscribe to topic(s) |
| `conduit.unsubscribe` | Client → Hub | Unsubscribe from topic(s) |
| `conduit.signal` | Hub → Client | Deliver signal to subscriber |
| `conduit.ack` | Client → Hub | Acknowledge signal receipt |

#### 4.2.3 Subscription Management Methods

| Method | Direction | Description |
|--------|-----------|-------------|
| `conduit.subscription.request` | Client → Hub | Request subscription (for user-approved) |
| `conduit.subscription.approve` | User → Hub | Approve pending subscription |
| `conduit.subscription.deny` | User → Hub | Deny pending subscription |
| `conduit.subscription.list` | Client → Hub | List active subscriptions |
| `conduit.subscription.revoke` | User → Hub | Revoke existing subscription |

---

## 5. Signal Schema

### 5.1 Core Signal Structure

Every signal MUST contain these fields:

```json
{
  "id": "sig_1704288600_abc123def456",
  "version": "1.0",
  "timestamp": "2026-01-03T10:30:00.000Z",
  "source": {
    "type": "email",
    "adapter_id": "adapter_gmail_01",
    "native_id": "18d4a2b3c4d5e6f7"
  },
  "topic": "signal.email.received",
  "payload": { ... },
  "metadata": { ... }
}
```

### 5.2 Field Definitions

#### 5.2.1 Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Globally unique signal identifier. Format: `sig_<timestamp>_<random>` |
| `version` | string | Protocol version (semver) |
| `timestamp` | string | ISO 8601 timestamp when signal was created at source |
| `source` | object | Source information |
| `source.type` | string | Source type identifier (e.g., "email", "sms", "slack") |
| `source.adapter_id` | string | Identifier of the adapter that produced this signal |
| `source.native_id` | string | Original ID from the source system |
| `topic` | string | Topic this signal is published to |
| `payload` | object | Signal content (format depends on source type) |

#### 5.2.2 Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `metadata` | object | Additional metadata |
| `metadata.thread_id` | string | Conversation/thread identifier |
| `metadata.in_reply_to` | string | Signal ID this is replying to |
| `metadata.references` | array | Related signal IDs |
| `metadata.priority` | string | Source-indicated priority ("low", "normal", "high", "urgent") |
| `metadata.tags` | array | Source-provided tags/labels |
| `encrypted` | object | E2E encryption information (see Section 8.3) |

### 5.3 Payload Structure

The payload structure is source-type specific but SHOULD follow these conventions:

#### 5.3.1 Common Payload Fields

```json
{
  "payload": {
    "raw": { ... },
    "content_type": "text/html",
    "size_bytes": 4523
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `raw` | any | Raw content in source-native format |
| `content_type` | string | MIME type of raw content |
| `size_bytes` | integer | Size of raw content in bytes |

#### 5.3.2 Example: Email Signal

```json
{
  "id": "sig_1704288600_abc123",
  "version": "1.0",
  "timestamp": "2026-01-03T10:30:00.000Z",
  "source": {
    "type": "email",
    "adapter_id": "adapter_gmail_01",
    "native_id": "18d4a2b3c4d5e6f7"
  },
  "topic": "signal.email.received",
  "payload": {
    "raw": {
      "from": "alice@example.com",
      "to": ["bob@example.com"],
      "cc": [],
      "bcc": [],
      "subject": "Meeting tomorrow",
      "body_text": "Hi Bob, can we meet tomorrow at 3pm?",
      "body_html": "<p>Hi Bob, can we meet tomorrow at 3pm?</p>",
      "attachments": []
    },
    "content_type": "message/rfc822",
    "size_bytes": 1234
  },
  "metadata": {
    "thread_id": "thread_abc123",
    "priority": "normal",
    "tags": ["inbox"]
  }
}
```

#### 5.3.3 Example: SMS Signal

```json
{
  "id": "sig_1704288700_def456",
  "version": "1.0",
  "timestamp": "2026-01-03T10:31:40.000Z",
  "source": {
    "type": "sms",
    "adapter_id": "adapter_twilio_01",
    "native_id": "SM1234567890abcdef"
  },
  "topic": "signal.sms.received",
  "payload": {
    "raw": {
      "from": "+15551234567",
      "to": "+15559876543",
      "body": "Running 10 mins late"
    },
    "content_type": "text/plain",
    "size_bytes": 21
  },
  "metadata": {
    "thread_id": "thread_sms_15551234567"
  }
}
```

#### 5.3.4 Example: Voice/Audio Signal

```json
{
  "id": "sig_1704288800_ghi789",
  "version": "1.0",
  "timestamp": "2026-01-03T10:33:20.000Z",
  "source": {
    "type": "voicemail",
    "adapter_id": "adapter_phone_01",
    "native_id": "vm_123456"
  },
  "topic": "signal.voicemail.received",
  "payload": {
    "raw": {
      "from": "+15551234567",
      "duration_seconds": 45,
      "audio_url": "https://storage.example.com/vm/123456.mp3",
      "audio_format": "audio/mpeg"
    },
    "content_type": "audio/mpeg",
    "size_bytes": 720000
  },
  "metadata": {}
}
```

### 5.4 Action Schema

Actions are signals published by agents to trigger outbound communications.

#### 5.4.1 Action Structure

```json
{
  "id": "act_1704289000_xyz789",
  "version": "1.0",
  "timestamp": "2026-01-03T10:36:40.000Z",
  "topic": "action.email.send",
  "action": {
    "type": "send",
    "target": "email",
    "payload": { ... }
  },
  "context": {
    "in_reply_to": "sig_1704288600_abc123",
    "agent_id": "agent_primary_01"
  }
}
```

#### 5.4.2 Example: Send Email Action

```json
{
  "id": "act_1704289000_xyz789",
  "version": "1.0",
  "timestamp": "2026-01-03T10:36:40.000Z",
  "topic": "action.email.send",
  "action": {
    "type": "send",
    "target": "email",
    "payload": {
      "to": ["alice@example.com"],
      "cc": [],
      "subject": "Re: Meeting tomorrow",
      "body_text": "Hi Alice, 3pm works for me. See you then!",
      "body_html": "<p>Hi Alice, 3pm works for me. See you then!</p>"
    }
  },
  "context": {
    "in_reply_to": "sig_1704288600_abc123",
    "agent_id": "agent_primary_01"
  }
}
```

---

## 6. Pub/Sub System

### 6.1 Topics

#### 6.1.1 Topic Format

Topics are UTF-8 strings with the following constraints:

- MUST be between 1 and 255 characters
- MUST contain only alphanumeric characters, dots (`.`), hyphens (`-`), and underscores (`_`)
- MUST NOT start or end with a dot
- MUST NOT contain consecutive dots

#### 6.1.2 Recommended Topic Convention

While topics are free-form, the following convention is RECOMMENDED for interoperability:

```
<category>.<source/target>.<event/command>
```

**Categories:**

| Category | Usage |
|----------|-------|
| `signal` | Inbound signals from sources |
| `action` | Outbound actions to sources |
| `system` | System events (connections, errors) |

**Examples:**

```
signal.email.received
signal.email.sent
signal.sms.received
signal.slack.message
signal.slack.reaction
action.email.send
action.sms.send
action.slack.reply
system.adapter.connected
system.adapter.disconnected
system.subscription.approved
```

#### 6.1.3 Wildcards

Subscribers MAY use wildcards in topic subscriptions:

| Wildcard | Matches | Example |
|----------|---------|---------|
| `*` | Single segment | `signal.*.received` matches `signal.email.received`, `signal.sms.received` |
| `**` | Multiple segments | `signal.**` matches all signal topics |

Publishers MUST NOT use wildcards in topic names.

### 6.2 Subscriptions

#### 6.2.1 Subscription Types

| Type | Description | Approval |
|------|-------------|----------|
| **Automatic** | Subscription activates immediately | Not required |
| **User-Approved** | Subscription requires user consent | Required |

#### 6.2.2 Subscription Request

```json
{
  "jsonrpc": "2.0",
  "id": "req_sub_001",
  "method": "conduit.subscribe",
  "params": {
    "topics": ["signal.email.*", "signal.sms.*"],
    "approval_type": "automatic"
  }
}
```

#### 6.2.3 Subscription Response

```json
{
  "jsonrpc": "2.0",
  "id": "req_sub_001",
  "result": {
    "subscription_id": "sub_abc123",
    "status": "active",
    "topics": ["signal.email.*", "signal.sms.*"],
    "created_at": "2026-01-03T10:30:00.000Z"
  }
}
```

#### 6.2.4 User-Approved Subscription Flow

1. Client sends subscription request with `approval_type: "user_approved"`
2. Hub creates pending subscription
3. Hub notifies user (via system topic or out-of-band)
4. User approves or denies
5. Hub activates or removes subscription
6. Hub notifies client of result

```json
// Step 1: Request
{
  "method": "conduit.subscribe",
  "params": {
    "topics": ["signal.**"],
    "approval_type": "user_approved",
    "reason": "Third-party agent requesting access to all signals"
  }
}

// Step 2: Pending response
{
  "result": {
    "subscription_id": "sub_pending_001",
    "status": "pending_approval",
    "topics": ["signal.**"]
  }
}

// Step 3: User approval (separate request from user/admin)
{
  "method": "conduit.subscription.approve",
  "params": {
    "subscription_id": "sub_pending_001",
    "restrictions": {
      "topics": ["signal.email.*"],  // Optionally restrict
      "expires_at": "2026-02-03T00:00:00Z"
    }
  }
}

// Step 4: Client notification
{
  "method": "conduit.subscription.status",
  "params": {
    "subscription_id": "sub_pending_001",
    "status": "active",
    "topics": ["signal.email.*"],
    "expires_at": "2026-02-03T00:00:00Z"
  }
}
```

### 6.3 Publishing

#### 6.3.1 Publish Request

```json
{
  "jsonrpc": "2.0",
  "id": "req_pub_001",
  "method": "conduit.publish",
  "params": {
    "topic": "signal.email.received",
    "message": {
      "id": "sig_1704288600_abc123",
      "version": "1.0",
      "timestamp": "2026-01-03T10:30:00.000Z",
      "source": { ... },
      "topic": "signal.email.received",
      "payload": { ... }
    }
  }
}
```

#### 6.3.2 Publish Response

```json
{
  "jsonrpc": "2.0",
  "id": "req_pub_001",
  "result": {
    "message_id": "msg_abc123",
    "delivered_to": 3,
    "queued_for": 1
  }
}
```

### 6.4 Acknowledgments

Subscribers MUST acknowledge received signals to enable reliable delivery.

#### 6.4.1 Acknowledgment Request

```json
{
  "jsonrpc": "2.0",
  "id": "req_ack_001",
  "method": "conduit.ack",
  "params": {
    "signal_ids": ["sig_1704288600_abc123", "sig_1704288700_def456"],
    "subscription_id": "sub_xyz789"
  }
}
```

#### 6.4.2 Acknowledgment Semantics

- Hub MUST track unacknowledged signals per subscription
- Hub MUST redeliver unacknowledged signals after timeout
- Hub SHOULD implement exponential backoff for redelivery
- Hub MAY move signals to dead-letter queue after max retries

---

## 7. Transport Bindings

Conduit supports five transport mechanisms. Implementations MUST support at least one; Hub implementations MUST support all five.

### 7.1 WebSocket

Full-duplex, persistent connection suitable for real-time bidirectional communication.

#### 7.1.1 Connection

```
wss://hub.example.com/conduit/v1/ws
```

#### 7.1.2 Handshake

After WebSocket connection is established:

```json
// Client → Hub
{
  "jsonrpc": "2.0",
  "id": "handshake_001",
  "method": "conduit.hello",
  "params": {
    "protocol_version": "1.0",
    "client_id": "agent_001",
    "capabilities": ["subscribe", "publish", "ack"],
    "auth": {
      "type": "bearer",
      "token": "eyJhbG..."
    }
  }
}

// Hub → Client
{
  "jsonrpc": "2.0",
  "id": "handshake_001",
  "result": {
    "session_id": "sess_abc123",
    "server_version": "1.0",
    "capabilities": ["subscribe", "publish", "ack", "e2e_encryption"]
  }
}
```

#### 7.1.3 Message Exchange

All JSON-RPC messages are sent as WebSocket text frames.

#### 7.1.4 Keepalive

```json
// Client → Hub
{"jsonrpc": "2.0", "method": "conduit.ping", "params": {"timestamp": 1704288600000}}

// Hub → Client
{"jsonrpc": "2.0", "method": "conduit.pong", "params": {"timestamp": 1704288600000}}
```

### 7.2 Server-Sent Events (SSE)

Unidirectional server-to-client streaming, suitable for signal delivery.

#### 7.2.1 Connection

```
GET /conduit/v1/sse?subscription_id=sub_abc123
Authorization: Bearer eyJhbG...
Accept: text/event-stream
```

#### 7.2.2 Event Format

```
event: signal
id: sig_1704288600_abc123
data: {"jsonrpc":"2.0","method":"conduit.signal","params":{...}}

event: signal
id: sig_1704288700_def456
data: {"jsonrpc":"2.0","method":"conduit.signal","params":{...}}

event: keepalive
data: {"timestamp":1704288800000}
```

#### 7.2.3 Acknowledgment

Since SSE is unidirectional, acknowledgments are sent via separate HTTP POST:

```
POST /conduit/v1/ack
Authorization: Bearer eyJhbG...
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": "req_ack_001",
  "method": "conduit.ack",
  "params": {
    "signal_ids": ["sig_1704288600_abc123"],
    "subscription_id": "sub_abc123"
  }
}
```

### 7.3 HTTP Polling

Client periodically requests new signals. Suitable for simple clients or serverless environments.

#### 7.3.1 Poll Request

```
GET /conduit/v1/poll?subscription_id=sub_abc123&last_id=sig_1704288500_xyz
Authorization: Bearer eyJhbG...
```

#### 7.3.2 Poll Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "signals": [
      {"id": "sig_1704288600_abc123", ...},
      {"id": "sig_1704288700_def456", ...}
    ],
    "has_more": false,
    "next_poll_after_ms": 5000
  }
}
```

### 7.4 Long Polling

Client holds connection open until signals are available or timeout.

#### 7.4.1 Long Poll Request

```
GET /conduit/v1/poll?subscription_id=sub_abc123&wait=true&timeout=30000
Authorization: Bearer eyJhbG...
```

#### 7.4.2 Long Poll Response

Same as regular polling, but connection is held until:
- Signals are available
- Timeout expires (returns empty signals array)
- Server decides to close (client should reconnect)

### 7.5 Webhooks

Hub pushes signals to client's HTTP endpoint.

#### 7.5.1 Webhook Registration

```json
{
  "jsonrpc": "2.0",
  "id": "webhook_001",
  "method": "conduit.subscribe",
  "params": {
    "topics": ["signal.email.*"],
    "transport": "webhook",
    "webhook": {
      "url": "https://agent.example.com/conduit/webhook",
      "secret": "whsec_abc123",
      "headers": {
        "X-Custom-Header": "value"
      }
    }
  }
}
```

#### 7.5.2 Webhook Delivery

Hub sends POST request to registered URL:

```
POST /conduit/webhook
Content-Type: application/json
X-Conduit-Signature: sha256=abc123...
X-Conduit-Timestamp: 1704288600
X-Conduit-Delivery-Id: del_xyz789

{
  "jsonrpc": "2.0",
  "method": "conduit.signal",
  "params": {
    "topic": "signal.email.received",
    "signal": {...}
  }
}
```

#### 7.5.3 Webhook Signature Verification

```
signature = HMAC-SHA256(secret, timestamp + "." + body)
```

#### 7.5.4 Webhook Response

Client MUST respond with 2xx status to acknowledge receipt. Any other status triggers redelivery.

```json
{
  "ack": ["sig_1704288600_abc123"]
}
```

---

## 8. Security

### 8.1 Transport Security (TLS)

- All connections MUST use TLS 1.2 or higher
- Implementations SHOULD use TLS 1.3
- Self-signed certificates MAY be used for development
- Production deployments MUST use valid certificates

### 8.2 Authentication

The protocol does not mandate a specific authentication mechanism. Implementations MUST support at least one of:

| Method | Description |
|--------|-------------|
| **API Key** | Static key passed in header |
| **Bearer Token** | JWT or opaque token in Authorization header |
| **mTLS** | Mutual TLS with client certificates |
| **OAuth 2.0** | Standard OAuth 2.0 flows |

#### 8.2.1 Authentication Header

```
Authorization: Bearer <token>
```

or

```
X-Conduit-API-Key: <api_key>
```

#### 8.2.2 Session Management

After authentication, Hub SHOULD issue a session ID:

```
X-Conduit-Session-Id: sess_abc123
```

Clients MUST include session ID in subsequent requests.

### 8.3 End-to-End Encryption (Optional Extension)

E2E encryption ensures the Hub cannot read signal content.

#### 8.3.1 Encryption Envelope

When E2E is enabled, the signal structure changes:

```json
{
  "id": "sig_1704288600_abc123",
  "version": "1.0",
  "timestamp": "2026-01-03T10:30:00.000Z",
  "source": {
    "type": "email",
    "adapter_id": "adapter_gmail_01"
  },
  "topic": "signal.email.received",
  "encrypted": {
    "algorithm": "x25519-xsalsa20-poly1305",
    "recipient_public_key": "base64...",
    "nonce": "base64...",
    "ciphertext": "base64..."
  }
}
```

The Hub sees:
- Signal ID, timestamp, source type, topic (for routing)
- Encrypted blob (cannot read content)

#### 8.3.2 Supported Algorithms

Implementations supporting E2E MUST implement:

| Algorithm ID | Description |
|-------------|-------------|
| `x25519-xsalsa20-poly1305` | NaCl/libsodium box (REQUIRED if E2E supported) |

Implementations MAY support:

| Algorithm ID | Description |
|-------------|-------------|
| `A256GCM` | JWE with AES-256-GCM |
| `xchacha20-poly1305` | XChaCha20-Poly1305 |

#### 8.3.3 Key Exchange

Key exchange is performed during subscription:

```json
{
  "method": "conduit.subscribe",
  "params": {
    "topics": ["signal.email.*"],
    "e2e": {
      "enabled": true,
      "public_key": "base64...",
      "supported_algorithms": ["x25519-xsalsa20-poly1305"]
    }
  }
}
```

Adapters MUST encrypt signals destined for E2E-enabled subscriptions.

#### 8.3.4 Algorithm Negotiation

1. Subscriber advertises supported algorithms in subscription request
2. Hub stores subscriber's public key and algorithm preferences
3. Publishers query Hub for subscriber encryption requirements
4. Publishers encrypt using subscriber's public key and preferred algorithm

---

## 9. A2A Integration

Conduit Hub exposes an A2A-compatible endpoint allowing external A2A agents to participate as first-class subscribers.

### 9.1 A2A Endpoint

```
https://hub.example.com/conduit/v1/a2a
```

### 9.2 Agent Card

The Hub exposes an A2A Agent Card at:

```
https://hub.example.com/.well-known/agent.json
```

```json
{
  "name": "Conduit Hub",
  "description": "Personal communication arbitrage hub",
  "version": "1.0",
  "capabilities": [
    "signal_subscription",
    "action_publishing",
    "streaming"
  ],
  "endpoints": {
    "a2a": "https://hub.example.com/conduit/v1/a2a",
    "sse": "https://hub.example.com/conduit/v1/sse"
  },
  "input_schema": {
    "type": "object",
    "properties": {
      "action": {
        "type": "string",
        "enum": ["subscribe", "publish", "list_topics"]
      },
      "topics": {
        "type": "array",
        "items": {"type": "string"}
      }
    }
  }
}
```

### 9.3 A2A Task Mapping

Conduit operations map to A2A tasks:

| Conduit Operation | A2A Task |
|------------------|----------|
| Subscribe to signals | Long-running task with streaming artifacts |
| Publish action | One-shot task |
| List topics | One-shot task |

### 9.4 A2A Streaming

Signals are delivered to A2A agents as streaming artifacts:

```json
{
  "task_id": "task_abc123",
  "status": "streaming",
  "artifact": {
    "type": "signal",
    "data": {
      "id": "sig_1704288600_abc123",
      "topic": "signal.email.received",
      "payload": {...}
    }
  }
}
```

### 9.5 A2A Agent Subscriptions

A2A agents can subscribe like any other agent:

1. A2A agent sends task to Conduit Hub endpoint
2. Hub creates subscription for A2A agent
3. Hub streams signals as A2A artifacts (via SSE)
4. A2A agent acknowledges via task updates

---

## 10. Error Handling

### 10.1 Error Response Format

```json
{
  "jsonrpc": "2.0",
  "id": "req_001",
  "error": {
    "code": -32001,
    "message": "Subscription not found",
    "data": {
      "subscription_id": "sub_invalid",
      "suggestion": "Create a new subscription"
    }
  }
}
```

### 10.2 Error Codes

#### 10.2.1 JSON-RPC Standard Errors

| Code | Message | Description |
|------|---------|-------------|
| -32700 | Parse error | Invalid JSON |
| -32600 | Invalid Request | Not a valid JSON-RPC request |
| -32601 | Method not found | Unknown method |
| -32602 | Invalid params | Invalid method parameters |
| -32603 | Internal error | Server error |

#### 10.2.2 Conduit Protocol Errors

| Code | Message | Description |
|------|---------|-------------|
| -32001 | Subscription not found | Subscription ID does not exist |
| -32002 | Topic not found | Topic does not exist |
| -32003 | Not authorized | Insufficient permissions |
| -32004 | Subscription pending | Awaiting user approval |
| -32005 | Subscription denied | User denied subscription |
| -32006 | Rate limited | Too many requests |
| -32007 | Signal too large | Payload exceeds size limit |
| -32008 | Encryption required | E2E encryption required but not provided |
| -32009 | Invalid encryption | E2E decryption failed |
| -32010 | Adapter unavailable | Target adapter is offline |
| -32011 | Delivery failed | Could not deliver to subscriber |
| -32012 | Queue full | Publisher's queue is full |
| -32013 | Session expired | Session has expired, re-authenticate |
| -32014 | Unsupported transport | Requested transport not supported |
| -32015 | Invalid topic | Topic name is malformed |

### 10.3 Retry Semantics

#### 10.3.1 Publisher Retry

When publishing fails:

| Error Code | Retry? | Strategy |
|------------|--------|----------|
| -32010 | Yes | Exponential backoff, queue locally |
| -32012 | Yes | Wait for queue space |
| -32006 | Yes | Wait for rate limit window |
| -32007 | No | Signal too large, cannot retry |
| -32003 | No | Fix authorization |

#### 10.3.2 Hub Redelivery

For unacknowledged signals:

```json
{
  "redelivery_config": {
    "initial_delay_ms": 1000,
    "max_delay_ms": 60000,
    "backoff_multiplier": 2.0,
    "max_attempts": 10,
    "dead_letter_topic": "system.dead_letter"
  }
}
```

---

## 11. Versioning

### 11.1 Protocol Version

The protocol version follows semantic versioning (MAJOR.MINOR.PATCH):

- **MAJOR**: Breaking changes to message format or semantics
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, clarifications

### 11.2 Version Negotiation

During handshake:

```json
{
  "method": "conduit.hello",
  "params": {
    "protocol_version": "1.0",
    "min_protocol_version": "1.0",
    "max_protocol_version": "1.2"
  }
}
```

Hub responds with negotiated version:

```json
{
  "result": {
    "protocol_version": "1.0"
  }
}
```

### 11.3 Backward Compatibility

- Hub SHOULD support previous MINOR versions
- Hub MAY support previous MAJOR versions
- Clients SHOULD include version in all requests
- Hub MUST reject unsupported versions with clear error

---

## 12. Appendices

### Appendix A: Signal ID Format

Signal IDs MUST be globally unique. Recommended format:

```
sig_<unix_timestamp_seconds>_<random_12_chars>
```

Example: `sig_1704288600_a1b2c3d4e5f6`

### Appendix B: Topic Naming Examples

#### Inbound Signals

```
signal.email.received
signal.email.sent
signal.email.bounced
signal.sms.received
signal.sms.delivered
signal.slack.message
signal.slack.mention
signal.slack.reaction
signal.telegram.message
signal.whatsapp.message
signal.discord.message
signal.calendar.invite
signal.calendar.reminder
signal.github.notification
signal.github.mention
signal.voicemail.received
signal.call.missed
```

#### Outbound Actions

```
action.email.send
action.email.reply
action.email.forward
action.sms.send
action.slack.send
action.slack.react
action.telegram.send
action.calendar.accept
action.calendar.decline
action.calendar.create
```

#### System Events

```
system.adapter.connected
system.adapter.disconnected
system.adapter.error
system.subscription.created
system.subscription.approved
system.subscription.denied
system.subscription.revoked
system.agent.connected
system.agent.disconnected
```

### Appendix C: Size Limits

| Item | Limit |
|------|-------|
| Topic name | 255 characters |
| Signal payload | 10 MB |
| Attachments (per signal) | 25 MB total |
| Topics per subscription | 100 |
| Subscriptions per client | 1000 |
| Signals per publish batch | 100 |

### Appendix D: Content Type Registry

Common content types for signal payloads:

| Content Type | Source Type |
|-------------|-------------|
| `message/rfc822` | Email |
| `text/plain` | SMS, plain text messages |
| `application/json` | Slack, Discord, API notifications |
| `audio/mpeg` | Voicemail, audio messages |
| `audio/ogg` | Voice messages (Telegram, WhatsApp) |
| `image/jpeg` | Photo attachments |
| `image/png` | Screenshot, image attachments |
| `application/pdf` | Document attachments |
| `video/mp4` | Video messages |

### Appendix E: Future Extensions (v1.1+)

The following features are planned for future versions:

1. **Multi-Hub Broadcasting**: Adapters publishing to multiple hubs for resilience
2. **Hub Federation**: Hubs forwarding signals to each other
3. **Signal Transformation**: Hub-side content transformation
4. **Priority Queues**: Quality-of-service tiers for signal delivery
5. **Batch Operations**: Efficient batch subscribe/unsubscribe/ack
6. **Compression**: Signal payload compression for large messages

---

## References

- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
- [Agent2Agent Protocol (A2A)](https://github.com/a2aproject/A2A)
- [RFC 2119 - Key Words](https://www.rfc-editor.org/rfc/rfc2119)
- [RFC 8259 - JSON](https://www.rfc-editor.org/rfc/rfc8259)
- [RFC 6455 - WebSocket](https://www.rfc-editor.org/rfc/rfc6455)
- [Server-Sent Events - W3C](https://html.spec.whatwg.org/multipage/server-sent-events.html)

---

## Changelog

### v1.0.0-draft (2026-01-03)

- Initial draft specification
- Core architecture: Adapters, Hub, Agents
- Pub/Sub system with topics and subscriptions
- Five transport bindings: WebSocket, SSE, Polling, Long Polling, Webhooks
- Security: TLS mandatory, E2E optional
- A2A integration for external agents
- Error handling and retry semantics
