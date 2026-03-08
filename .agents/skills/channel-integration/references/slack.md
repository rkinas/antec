# Slack Channel Integration Reference

## Platform: Slack Socket Mode + Web API
**Connection**: WebSocket (Socket Mode) for events, REST for sending
**Auth**: App Token (`xapp-`) for Socket Mode, Bot Token (`xoxb-`) for Web API
**Max message**: 3,000 recommended (40,000 hard limit for `chat.postMessage`)
**Rate limit**: 1 message/second per channel (Tier 3), varies by method

## Two-Token Architecture

| Token | Prefix | Purpose |
|-------|--------|---------|
| App Token | `xapp-` | Socket Mode WebSocket connection |
| Bot Token | `xoxb-` | Web API calls (send messages, etc.) |

Both are required. App Token connects the WebSocket; Bot Token authenticates API calls.

## Connection Flow (Socket Mode)

1. `POST /apps.connections.open` (App Token) → get WebSocket URL
2. Open WSS connection
3. Receive `{"type": "hello"}` → connection established
4. Validate bot token: `POST /auth.test` (Bot Token) → get bot user ID
5. Process incoming events (type `events_api`)
6. Acknowledge each envelope with `{"envelope_id": "..."}` within 3 seconds

## Event Envelope Structure

```json
{
  "type": "events_api",
  "envelope_id": "abc123",
  "payload": {
    "event": {
      "type": "message",
      "subtype": null,
      "user": "U123",
      "text": "hello",
      "channel": "C456",
      "ts": "1700000000.000100",
      "edited": { "user": "U123", "ts": "..." }
    }
  }
}
```

## Event Types

| Type | Subtype | What |
|------|---------|------|
| `events_api` | — | Wrapper for all events |
| `hello` | — | Connection handshake |
| `disconnect` | — | Server requests reconnect |

### Message Subtypes to Handle
- `null` (no subtype) → normal user message
- `message_changed` → edited message (content in `event.message.text`)

### Message Subtypes to Ignore
- `bot_message` → from a bot
- `channel_join`, `channel_leave` → membership changes
- `channel_topic`, `channel_purpose` → metadata changes

## Filtering Rules
1. Skip if `event.bot_id` is present (bot message)
2. Skip if `event.user == bot_user_id` (own messages)
3. Skip if `event.channel` not in allowed_channels (when configured)
4. Skip if subtype is join/leave/topic/purpose
5. Skip empty text

## Web API Methods

| Method | Purpose | Key Params |
|--------|---------|------------|
| `chat.postMessage` | Send message | `channel`, `text`, `thread_ts` |
| `chat.update` | Edit message | `channel`, `ts`, `text` |
| `reactions.add` | Add reaction | `channel`, `timestamp`, `name` |
| `auth.test` | Validate token | — (returns bot user ID) |
| `apps.connections.open` | Get WS URL | — (uses App Token) |

All Web API calls use `Authorization: Bearer {bot_token}` header and `application/json` body.

## Reconnection Strategy

1. On `disconnect` event: immediately reconnect
2. On WebSocket close: exponential backoff 1s → 60s cap
3. On connection error: new `apps.connections.open` call for fresh URL
4. Socket Mode URLs are single-use — always get a new one on reconnect

## Required Bot Scopes

```
channels:history    -- Read messages in public channels
groups:history      -- Read messages in private channels
im:history          -- Read direct messages
chat:write          -- Send messages
reactions:write     -- Add reactions
```

## Required Event Subscriptions

```
message.channels    -- Messages in public channels
message.groups      -- Messages in private channels
message.im          -- Direct messages
```

## Key Implementation Notes

- Envelope acknowledgement is required within 3 seconds or Slack resends
- Socket Mode requires app-level token, not bot token
- Thread replies: include `thread_ts` in `chat.postMessage`
- Slack uses `ts` (timestamp) as message ID — format: `"1700000000.000100"`
- Rich formatting uses Slack's `mrkdwn` (not standard Markdown)
- App must be installed to workspace (OAuth) before tokens work
