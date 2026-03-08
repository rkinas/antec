# WhatsApp Channel Integration Reference

## Platform: WhatsApp Cloud API (Meta)
**Connection**: Webhook for receiving, REST for sending
**Auth**: Bearer Token (permanent system user token or short-lived user token)
**Max message**: 4,096 characters
**Rate limit**: 80 messages/second (business tier dependent)

## Dual Mode Architecture

| Mode | When | Connection |
|------|------|-----------|
| **Cloud API** | Production, official | Webhooks + Graph API REST |
| **Web Gateway** | Development, unofficial | REST proxy to WhatsApp Web |

If `gateway_url` env var is set → Web mode. Otherwise → Cloud API mode.

## Cloud API Flow

### Receiving (Webhook)
1. Register webhook URL in Meta Developer Console
2. Handle `GET` verification: respond with `hub.challenge` when `hub.verify_token` matches
3. Handle `POST` events: parse webhook payload → extract messages
4. Respond `200 OK` immediately (within 5 seconds or Meta retries)

### Sending
```
POST https://graph.facebook.com/v21.0/{phone_number_id}/messages
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "messaging_product": "whatsapp",
  "to": "recipient_phone",
  "type": "text",
  "text": { "body": "message content" }
}
```

## Webhook Payload Structure

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "changes": [{
      "value": {
        "messages": [{
          "id": "wamid.xxx",
          "from": "15551234567",
          "timestamp": "1700000000",
          "type": "text",
          "text": { "body": "hello" }
        }],
        "contacts": [{
          "profile": { "name": "John" },
          "wa_id": "15551234567"
        }]
      }
    }]
  }]
}
```

## Message Types (Inbound)

| Type | Content Location |
|------|-----------------|
| `text` | `message.text.body` |
| `image` | `message.image.id` → fetch via media endpoint |
| `document` | `message.document.id` + `filename` |
| `audio` | `message.audio.id` |
| `location` | `message.location.latitude`, `.longitude` |
| `reaction` | `message.reaction.emoji` on `message_id` |

## Sending Message Types

| Type | Body |
|------|------|
| Text | `{ "type": "text", "text": { "body": "..." } }` |
| Image | `{ "type": "image", "image": { "link": "https://..." } }` |
| Document | `{ "type": "document", "document": { "link": "...", "filename": "..." } }` |
| Location | `{ "type": "location", "location": { "latitude": x, "longitude": y } }` |

## Webhook Verification

Meta sends GET request with:
- `hub.mode` = `"subscribe"`
- `hub.verify_token` = your configured verify token
- `hub.challenge` = random string

Respond with `hub.challenge` value and `200 OK` if verify_token matches.

## Filtering Rules
1. Skip if `messages` array is empty
2. Skip if `from` phone not in allowed_users (when configured)
3. Skip `status` updates (delivery receipts) — only process `messages`
4. Skip if message type is unsupported and no text fallback

## Rate Limiting

| Status | Action |
|--------|--------|
| 200 | Success |
| 429 | Rate limited — back off, check `Retry-After` |
| 400 | Bad request — log error, don't retry |
| 500+ | Server error — exponential backoff |

## Media Handling

Download media: `GET https://graph.facebook.com/v21.0/{media_id}`
Returns URL, then `GET` the URL with Bearer token to download binary.

## Key Implementation Notes

- Phone numbers must include country code without `+` prefix
- Webhook must respond within 5 seconds or Meta marks as failed
- Business accounts require verified Meta Business Manager
- Template messages required for initiating conversations (24h window)
- End-to-end encryption is handled by WhatsApp, transparent to API
- Web gateway mode bypasses official API (for development only)
