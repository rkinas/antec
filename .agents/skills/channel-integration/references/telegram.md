# Telegram Channel Integration Reference

## Platform: Telegram Bot API
**Connection**: Long polling (`getUpdates`) or Webhook
**Auth**: Bot Token in URL path
**Max message**: 4,096 characters
**Rate limit**: 30 messages/second global, 1 message/second per chat (group: 20/minute)

## Connection Flow (Long Polling)

1. `POST /deleteWebhook` → clear any existing webhook (prevents 409 conflicts)
2. `GET /getUpdates?offset=0&timeout=30` → long poll for events
3. Parse response → extract messages
4. Update offset to `last_update_id + 1`
5. Repeat step 2

## API Base URL

```
https://api.telegram.org/bot{token}/{method}
```

## Key API Methods

| Method | Purpose | Parameters |
|--------|---------|------------|
| `getUpdates` | Long poll for messages | `offset`, `timeout`, `allowed_updates` |
| `sendMessage` | Send text | `chat_id`, `text`, `parse_mode` |
| `sendPhoto` | Send image | `chat_id`, `photo`, `caption` |
| `sendDocument` | Send file | `chat_id`, `document`, `caption` |
| `sendVoice` | Send voice | `chat_id`, `voice` |
| `sendLocation` | Send coordinates | `chat_id`, `latitude`, `longitude` |
| `sendChatAction` | Typing indicator | `chat_id`, `action: "typing"` |
| `editMessageText` | Edit message | `chat_id`, `message_id`, `text` |
| `getFile` | Get file download URL | `file_id` → `file_path` |
| `deleteWebhook` | Remove webhook | — |

## Update Structure

```json
{
  "update_id": 123456,
  "message": {
    "message_id": 789,
    "from": { "id": 111, "first_name": "John", "username": "john" },
    "chat": { "id": -100123, "type": "group", "title": "My Group" },
    "date": 1700000000,
    "text": "hello",
    "entities": [{ "type": "bot_command", "offset": 0, "length": 5 }]
  }
}
```

Also handle `edited_message` (same structure, different field name).

## Filtering Rules
1. Skip if `from.id` is bot's own ID
2. Skip if `from.id` not in allowed_users (when configured)
3. Skip if no `text` and no supported content type
4. For groups: only process if bot is mentioned or command is directed at bot

## Command Detection
- Check `entities` array for `type: "bot_command"`
- Parse text: `/command@botname arg1 arg2` → name: `command`, args: `["arg1", "arg2"]`
- Strip `@botname` suffix from commands

## Content Types
- `text` → Text content
- `photo` → array of PhotoSize, use largest (last element), resolve via `getFile`
- `document` → file_id + file_name + mime_type
- `voice` → file_id + duration
- `location` → latitude + longitude

## HTML Parse Mode

Telegram supports limited HTML. Allowed tags:
```
<b>, <i>, <u>, <s>, <a href="...">, <code>, <pre>, <blockquote>, <tg-spoiler>
```
All other tags must be escaped to `&lt;` / `&gt;`.

## Rate Limit Handling

| Status | Action |
|--------|--------|
| 200 | Success |
| 409 | Another instance polling — back off, retry |
| 429 | Rate limited — parse `retry_after` from response, wait that many seconds |
| 5xx | Server error — exponential backoff |

## Key Implementation Notes

- Always call `deleteWebhook` on startup to prevent 409 conflicts
- `getUpdates` timeout should be 30-60s for efficient long polling
- File URLs: `https://api.telegram.org/file/bot{token}/{file_path}`
- `chat.id` is negative for groups, positive for DMs
- Bot commands only work if registered via @BotFather
