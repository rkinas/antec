# Discord Channel Integration Reference

## Platform: Discord Bot API
**Connection**: WebSocket Gateway (v10) + REST API (v10)
**Auth**: Bot Token (`Bot {token}` header)
**Max message**: 2,000 characters
**Rate limit**: 50 requests/second global, 5 messages/second per channel

## Connection Flow

1. `GET /gateway/bot` → obtain WebSocket URL + shard info
2. Open WSS connection to gateway URL
3. Receive Opcode 10 (`HELLO`) with `heartbeat_interval`
4. Send Opcode 2 (`IDENTIFY`) with bot token + intents
5. Receive Opcode 0 (`DISPATCH`, event `READY`) with session_id + bot user
6. Start heartbeat loop at prescribed interval
7. Process `MESSAGE_CREATE` events from dispatch

## Gateway Opcodes

| Opcode | Name | Direction | Purpose |
|--------|------|-----------|---------|
| 0 | DISPATCH | Receive | Event payload (MESSAGE_CREATE, etc.) |
| 1 | HEARTBEAT | Both | Keep connection alive |
| 2 | IDENTIFY | Send | Authenticate on connect |
| 6 | RESUME | Send | Resume dropped session |
| 7 | RECONNECT | Receive | Server requests reconnect |
| 9 | INVALID_SESSION | Receive | Session no longer valid |
| 10 | HELLO | Receive | First message, contains heartbeat_interval |
| 11 | HEARTBEAT_ACK | Receive | Server acknowledges heartbeat |

## Required Intents

```
GUILDS                 (1 << 0)
GUILD_MESSAGES         (1 << 9)
DIRECT_MESSAGES        (1 << 12)
MESSAGE_CONTENT        (1 << 15)  -- Privileged, must enable in Developer Portal
```

## Message Parsing

Event `MESSAGE_CREATE`:
```json
{
  "id": "message_id",
  "channel_id": "channel_id",
  "author": { "id": "user_id", "username": "name", "discriminator": "0000", "bot": false },
  "content": "message text",
  "timestamp": "2024-01-01T00:00:00.000000+00:00",
  "edited_timestamp": null,
  "mention_everyone": false,
  "mentions": [{ "id": "bot_user_id" }]
}
```

### Filtering Rules
1. Skip if `author.id == bot_user_id` (own messages)
2. Skip if `author.bot == true` (when `ignore_bots` enabled)
3. Skip if guild_id not in allowed_guilds (when configured)
4. Skip if author.id not in allowed_users (when configured)
5. Skip empty content

### Mention Detection
- Check `mentions` array for bot user ID
- Check content for `<@bot_user_id>` pattern
- Strip mention from content when detected

### Command Parsing
Content starting with `/` → extract command name and arguments

## REST Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/channels/{id}/messages` | Send message (body: `{ "content": "text" }`) |
| POST | `/channels/{id}/typing` | Trigger typing indicator (lasts 10s) |
| PATCH | `/channels/{id}/messages/{id}` | Edit message |
| PUT | `/channels/{id}/messages/{id}/reactions/{emoji}/@me` | Add reaction |
| GET | `/gateway/bot` | Get WebSocket gateway URL |

## Reconnection Strategy

1. On disconnect: attempt RESUME with session_id + last sequence number
2. If INVALID_SESSION (resumable=true): wait 1-5s, send RESUME
3. If INVALID_SESSION (resumable=false): clear state, send fresh IDENTIFY
4. Exponential backoff: 1s → 2s → 4s → 8s → ... → 60s cap

## Key Implementation Notes

- Sequence numbers must be tracked per-connection for resume
- Heartbeat must be sent within `heartbeat_interval` or connection drops
- Message content requires MESSAGE_CONTENT privileged intent
- DM channel IDs differ from guild channel IDs
- `discriminator: "0"` means new username system (no #tag)
