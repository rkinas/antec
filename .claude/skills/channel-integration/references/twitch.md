# Twitch Channel Integration Reference

## Platform: Twitch IRC + EventSub
**Connection**: IRC over WebSocket for chat, EventSub for events
**Auth**: OAuth2 token with chat scopes
**Max message**: 500 characters per chat message
**Rate limit**: 20 messages/30s (non-mod), 100 messages/30s (mod)

## Architecture

Twitch chat uses IRC protocol over WebSocket:
- **IRC (WSS)** — real-time chat messages
- **EventSub** — subscriptions, follows, raids (optional, for extended features)
- **Helix API** — user info, channel info

## Authentication

1. Register app at dev.twitch.tv → get `client_id` + `client_secret`
2. Generate user OAuth token with scopes:
   ```
   chat:read chat:edit user:read:email
   ```
3. Token format: `oauth:{access_token}`

### Token Validation
```
GET https://id.twitch.tv/oauth2/validate
Authorization: OAuth {access_token}
```

## IRC Connection Flow

1. Connect WSS to `wss://irc-ws.chat.twitch.tv:443`
2. Send: `CAP REQ :twitch.tv/membership twitch.tv/tags twitch.tv/commands`
3. Send: `PASS oauth:{token}`
4. Send: `NICK {bot_username}`
5. Receive: `:tmi.twitch.tv 376 {bot} :>`  (welcome)
6. Send: `JOIN #{channel_name}` for each channel
7. Process PRIVMSG, PING, NOTICE, etc.

## IRC Message Format

```
@badge-info=;badges=moderator/1;color=#FF0000;display-name=User;emotes=;
 id=msg-id;mod=0;room-id=12345;subscriber=0;tmi-sent-ts=1700000000000;
 turbo=0;user-id=67890;user-type=
 :user!user@user.tmi.twitch.tv PRIVMSG #channel :message text here
```

### Parsing Steps
1. Parse IRC tags (`;`-separated `key=value` pairs before the message)
2. Extract `display-name` → sender name
3. Extract `user-id` → sender ID
4. Extract `room-id` → channel/conversation ID
5. Extract text after `PRIVMSG #channel :` → message content
6. `tmi-sent-ts` → timestamp (milliseconds since epoch)

## Message Types

| IRC Command | Purpose |
|-------------|---------|
| `PRIVMSG` | Chat message (process these) |
| `PING` | Keep alive (respond with `PONG :tmi.twitch.tv`) |
| `NOTICE` | System notice (login failures, etc.) |
| `RECONNECT` | Server requests reconnect |
| `USERNOTICE` | Subscriptions, raids, etc. |
| `CLEARCHAT` | Timeout/ban events |

## Sending Messages

```
PRIVMSG #channel :response text here
```

- Max 500 characters
- Rate limit: 20 messages per 30 seconds (non-moderator)
- If bot is moderator in channel: 100 messages per 30 seconds

### Helix API Alternative
```
POST https://api.twitch.tv/helix/chat/messages
Authorization: Bearer {token}
Client-Id: {client_id}

{
  "broadcaster_id": "12345",
  "sender_id": "67890",
  "message": "response text"
}
```

## Filtering Rules
1. Skip if sender username matches bot's own username
2. Skip if `user-id` not in allowed_users (when configured)
3. Skip PING, NOTICE, USERNOTICE (unless handling events)
4. Skip messages not starting with trigger prefix (e.g., `!` or bot @mention)

## Command Parsing

Twitch convention uses `!` prefix (not `/`):
```
!ask what is Rust  → Command { name: "ask", args: ["what", "is", "Rust"] }
```

Also support `@botname` mention detection in message text.

## Reconnection Strategy

1. On `RECONNECT` command: immediately reconnect
2. On WebSocket close: exponential backoff 1s → 60s
3. Re-authenticate and re-join channels on reconnect
4. PING/PONG heartbeat: respond to `PING` within 30s or connection drops

## Key Implementation Notes

- IRC tags contain useful metadata (badges, subscriber status, emotes)
- Channel names in IRC are lowercase with `#` prefix
- Twitch IRC is read-only without `chat:edit` scope
- Bot must join each channel explicitly with `JOIN`
- Consider command cooldowns to prevent spam
- Whispers (DMs) use separate endpoint and different rate limits
- Emotes are encoded as ID ranges in the `emotes` tag
- Some channels require verified email to chat
- EventSub (optional) provides richer events but requires webhook/WebSocket setup
