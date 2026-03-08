---
name: channel-integration
description: Implement messaging channel adapters (Discord, Telegram, Slack, WhatsApp, Signal, Email, Bluesky, Teams, Twitch). Use when building, reviewing, or extending channel integrations.
---

# Channel Integration Standard

> Universal pattern for implementing messaging channel adapters. Applies to Rust, Python, TypeScript, and any language with async I/O.

## Architecture Overview

Every channel adapter follows the same pattern regardless of the target platform:

```
Platform API ←→ Adapter ←→ Normalized Message ←→ Agent Core
```

The adapter is a **bridge** that:
1. **Connects** to the platform (WebSocket, polling, webhook)
2. **Receives** platform-specific messages and **normalizes** them
3. **Sends** responses back in the platform's native format
4. **Handles** reconnection, rate limiting, and graceful shutdown

## The Channel Trait (Interface Contract)

Every adapter must implement these capabilities:

| Method | Required | Purpose |
|--------|----------|---------|
| `channel_type()` | Yes | Return identifier: `"discord"`, `"telegram"`, etc. |
| `connect()` | Yes | Establish connection to platform backend |
| `disconnect()` | Yes | Clean shutdown of connections |
| `is_connected()` | Yes | Connection health check |
| `send_message()` | Yes | Send text to a conversation (with chunking) |
| `send_chunk()` | No | Stream partial response (default: falls back to `send_message`) |
| `send_typing_indicator()` | No | Show "typing..." status (default: no-op) |
| `edit_message()` | No | Edit previously sent message (default: unsupported error) |
| `add_reaction()` | No | React to a message (default: unsupported error) |
| `capabilities()` | Yes | Declare what this channel supports |

### Capabilities Declaration

Each adapter declares its platform's constraints:

```
max_message_length    -- Character limit per message (0 = unlimited)
supports_streaming    -- Can receive chunked/partial responses
supports_typing       -- Can show typing indicators
supports_editing      -- Can edit sent messages
supports_reactions    -- Can add emoji reactions
supports_attachments  -- Can handle file uploads
supports_threads      -- Can thread replies
supports_rich_text    -- Can render markdown/HTML/formatting
```

## Normalized Message Format

All inbound messages are converted to this unified format before reaching the agent:

```
NormalizedMessage {
    id              -- Platform message ID (string)
    channel_type    -- "discord", "telegram", "slack", etc.
    conversation_id -- Platform conversation/chat/channel ID
    sender_id       -- Platform user ID
    sender_name     -- Display name
    content         -- Message content (see Content Types below)
    timestamp       -- UTC datetime
    is_edit         -- Whether this is an edit of a prior message
    reply_to        -- Optional ID of message being replied to
    metadata        -- Platform-specific extra data (key-value)
}
```

### Content Types

```
Text(string)           -- Plain text message
Command(name, args)    -- Slash command: /name arg1 arg2
Attachment(url, mime)  -- File, image, document
Location(lat, lon)     -- Geographic coordinates
Reaction(emoji)        -- Emoji reaction on a message
```

## Implementation Checklist

For EVERY channel adapter, implement in this order:

### Phase 1: Foundation
- [ ] Struct with credential storage (zeroize sensitive tokens on drop)
- [ ] Constructor (`new()`) with config validation
- [ ] `channel_type()` returning the identifier string
- [ ] `capabilities()` declaring platform constraints
- [ ] `is_connected()` with atomic/lock-protected state

### Phase 2: Receiving Messages
- [ ] Connection method (WebSocket / long-poll / webhook / REST polling)
- [ ] Message parsing: platform JSON → NormalizedMessage
- [ ] Content type detection: text, commands (`/` prefix), attachments
- [ ] Filtering: ignore bot's own messages, respect allowlist
- [ ] Message stream via async channel (bounded, typically 256 capacity)

### Phase 3: Sending Messages
- [ ] Text sending with platform character limit chunking
- [ ] Typing indicator (if supported)
- [ ] Error handling for API failures
- [ ] Rate limit awareness (respect `Retry-After` headers)

### Phase 4: Resilience
- [ ] Reconnection with exponential backoff (1s → 60s cap)
- [ ] Graceful shutdown via cancellation signal
- [ ] Session resumption (where platform supports it)
- [ ] Stale connection detection via heartbeat/ping

### Phase 5: Platform-Specific Features
- [ ] Rich text / formatting (Markdown, HTML, platform-specific)
- [ ] Attachments (images, files, voice)
- [ ] Threads / reply chains
- [ ] Reactions
- [ ] Message editing
- [ ] Presence / online status

### Phase 6: Testing
- [ ] Unit: message parsing (happy path + malformed input)
- [ ] Unit: bot/self message filtering
- [ ] Unit: allowlist filtering
- [ ] Unit: command parsing with arguments
- [ ] Unit: message chunking at platform limit
- [ ] Unit: edited message handling
- [ ] Integration: mock server → adapter → normalized messages
- [ ] Integration: adapter → mock server → message delivery

## Connection Patterns

### Pattern A: WebSocket Gateway (Discord, Slack, Twitch)
```
connect() → GET gateway URL → open WebSocket → identify/authenticate
           → heartbeat loop + message dispatch loop
           → reconnect on disconnect with session resume
```

### Pattern B: Long Polling (Telegram, Signal)
```
connect() → spawn poll loop → GET /getUpdates?offset=X
           → parse messages → yield to stream
           → increment offset → repeat
           → exponential backoff on errors
```

### Pattern C: Webhook Receiver (WhatsApp, Slack Events API, Teams)
```
connect() → spawn HTTP server on configured port
           → verify webhook signature on each request
           → parse event payload → yield to stream
           → respond 200 OK within platform timeout
```

### Pattern D: SMTP/IMAP (Email)
```
connect() → open IMAP connection → IDLE for new messages
           → parse MIME → extract text/html body
send()    → compose MIME → send via SMTP
```

### Pattern E: REST Polling (Bluesky AT Protocol)
```
connect() → authenticate → get session token
           → poll notifications/mentions at interval
           → parse AT Protocol objects → normalize
send()    → create post/reply via AT Protocol
```

## Security Requirements

1. **Token zeroization** -- All API tokens, secrets, and passwords MUST be zeroized on drop. In Rust: `Zeroizing<String>` from `zeroize` crate. In Python: overwrite with zeros in `__del__`. In TypeScript: use `sodium.memzero()`.

2. **Allowlist filtering** -- Every adapter MUST support an allowlist of authorized users/channels. Empty allowlist = allow all (single-user assumption). Non-empty = strict filter.

3. **Webhook verification** -- Adapters using webhooks MUST verify request signatures (HMAC-SHA256 for Slack, SHA256 for WhatsApp, etc.).

4. **No credential logging** -- Tokens must never appear in log output. Redact before logging.

5. **TLS everywhere** -- All platform API connections use HTTPS/WSS. No plaintext.

## Rate Limiting Strategy

Every adapter must handle rate limits gracefully:

| Strategy | When |
|----------|------|
| Respect `Retry-After` header | Platform returns 429 |
| Exponential backoff | Connection failures |
| Message queue with drain rate | Outbound burst protection |
| Platform-specific limits | Discord: 50msg/sec, Telegram: 30msg/sec, Slack: 1msg/sec per channel |

## Message Chunking Rules

When sending exceeds the platform limit, split intelligently:

1. Split at paragraph boundaries (`\n\n`) first
2. Then at sentence boundaries (`. `)
3. Then at word boundaries (` `)
4. Never split mid-word or mid-codeblock
5. Preserve markdown formatting across chunks when possible

| Platform | Max chars | Notes |
|----------|-----------|-------|
| Discord | 2,000 | Per message |
| Telegram | 4,096 | Per message |
| Slack | 3,000 | Recommended (40,000 hard limit) |
| WhatsApp | 4,096 | Per message |
| Signal | 6,000 | Approximate |
| Email | Unlimited | But keep reasonable |
| Bluesky | 300 | Per post (graphemes) |
| Teams | 28,000 | Adaptive card limit |
| Twitch | 500 | Per chat message |

## Per-Channel Quick Reference

Detailed implementation guides for each channel are in the `references/` directory:

- [references/discord.md](references/discord.md) -- Discord Bot Gateway + REST API
- [references/telegram.md](references/telegram.md) -- Telegram Bot API (long polling)
- [references/slack.md](references/slack.md) -- Slack Socket Mode + Web API
- [references/whatsapp.md](references/whatsapp.md) -- WhatsApp Cloud API + Web gateway
- [references/signal.md](references/signal.md) -- Signal via signal-cli REST
- [references/email.md](references/email.md) -- IMAP/SMTP with MIME parsing
- [references/bluesky.md](references/bluesky.md) -- AT Protocol (atproto)
- [references/teams.md](references/teams.md) -- Microsoft Bot Framework
- [references/twitch.md](references/twitch.md) -- Twitch IRC + EventSub

## Testing Pattern (All Channels)

```
#[cfg(test)]
mod tests {
    // 1. Parse a well-formed platform message → verify all NormalizedMessage fields
    // 2. Parse a message from the bot itself → verify it's filtered out
    // 3. Parse a message from disallowed user → verify it's filtered out
    // 4. Parse a command message (/foo bar) → verify Command content type
    // 5. Parse an edited message → verify is_edit flag
    // 6. Send a message exceeding platform limit → verify correct chunking
    // 7. Handle platform API error → verify graceful degradation
    // 8. Verify capabilities() returns correct flags
}
```
