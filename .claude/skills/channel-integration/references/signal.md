# Signal Channel Integration Reference

## Platform: Signal via signal-cli REST API
**Connection**: REST polling (signal-cli exposes HTTP API)
**Auth**: Phone number registration (no API token — signal-cli manages auth)
**Max message**: ~6,000 characters (soft limit)
**Rate limit**: Platform-imposed, varies

## Architecture

Signal doesn't provide a public bot API. The standard approach:
1. Run `signal-cli` as a daemon exposing a REST API
2. Register a phone number with Signal
3. Poll the REST API for incoming messages
4. Send via REST API

Alternative: `signal-cli` in JSON-RPC mode over stdio.

## Connection Flow

1. Configure signal-cli REST endpoint URL + registered phone number
2. Start polling loop: `GET /v1/receive/{phone_number}`
3. Parse JSON envelope array → extract messages
4. Sleep 2 seconds → repeat
5. On error: exponential backoff

## API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/receive/{number}` | Fetch pending messages |
| POST | `/v2/send` | Send message |
| PUT | `/v1/reactions/{number}` | Send reaction |
| GET | `/v1/about` | Health check |

## Receive Response Structure

```json
[
  {
    "envelope": {
      "source": "+15551234567",
      "sourceDevice": 1,
      "timestamp": 1700000000000,
      "dataMessage": {
        "timestamp": 1700000000000,
        "message": "hello",
        "groupInfo": null
      }
    }
  }
]
```

## Send Request

```json
POST /v2/send
{
  "message": "response text",
  "number": "+15551234567",
  "recipients": ["+15559876543"]
}
```

## Filtering Rules
1. Skip if `source` is empty or equals own phone number
2. Skip if `source` not in allowed_users (when configured)
3. Skip if no `dataMessage` field (delivery receipts, typing indicators)
4. Skip if `dataMessage.message` is null or empty

## Command Parsing
- Messages starting with `/` → parse as command with arguments
- Otherwise → plain text content

## Group Messages
- `dataMessage.groupInfo` is non-null for group messages
- Group ID serves as conversation_id
- For DMs: source phone number serves as conversation_id

## Error Handling

| Scenario | Action |
|----------|--------|
| signal-cli not running | Log error, retry with backoff |
| 400 Bad Request | Log, skip message |
| Network timeout | Exponential backoff 1s → 60s |
| Empty response | Normal — no new messages, continue polling |

## Key Implementation Notes

- signal-cli must be separately installed and registered with a phone number
- Registration requires SMS/voice verification (one-time)
- Poll interval: 2 seconds is reasonable default
- Signal doesn't support bots natively — this is a bridge pattern
- Messages are end-to-end encrypted (signal-cli handles crypto)
- Phone numbers must include `+` country code prefix
- Group message support requires signal-cli group membership
- No typing indicator API in signal-cli REST
- No message editing API in signal-cli REST
