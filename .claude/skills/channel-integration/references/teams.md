# Microsoft Teams Channel Integration Reference

## Platform: Microsoft Bot Framework + Teams API
**Connection**: Webhook (Bot Framework receives activities via HTTPS)
**Auth**: Azure AD OAuth2 (client credentials flow)
**Max message**: 28,000 characters (Adaptive Card), 4,000 for plain text
**Rate limit**: Varies by tenant and API

## Architecture

Teams bots use Microsoft Bot Framework:
1. Register bot in Azure Bot Service
2. Bot Framework routes messages to your webhook endpoint
3. You respond via Bot Framework REST API

## Authentication Flow

1. Register app in Azure AD → get `app_id` + `app_secret`
2. Get token: `POST https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token`
   ```
   grant_type=client_credentials
   client_id={app_id}
   client_secret={app_secret}
   scope=https://api.botframework.com/.default
   ```
3. Use `access_token` as Bearer for Bot Framework API calls
4. Token expires (typically 1 hour) — refresh before expiry

## Receiving Messages (Webhook)

Bot Framework POSTs activities to your registered endpoint:

```json
{
  "type": "message",
  "id": "activity-id",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "serviceUrl": "https://smba.trafficmanager.net/teams/",
  "channelId": "msteams",
  "from": { "id": "29:user-id", "name": "John Doe", "aadObjectId": "..." },
  "conversation": { "id": "19:meeting-id@thread.v2", "tenantId": "..." },
  "recipient": { "id": "28:bot-id", "name": "Antec Bot" },
  "text": "hello",
  "textFormat": "plain"
}
```

### Activity Types
| Type | Purpose |
|------|---------|
| `message` | User sent a message |
| `conversationUpdate` | Member added/removed |
| `messageReaction` | Reaction added/removed |
| `invoke` | Adaptive Card action, task module |

## Sending Messages

### Reply to activity
```
POST {serviceUrl}/v3/conversations/{conversationId}/activities/{activityId}
Authorization: Bearer {token}

{
  "type": "message",
  "text": "response text",
  "textFormat": "markdown"
}
```

### Proactive message (new conversation)
```
POST {serviceUrl}/v3/conversations
{
  "bot": { "id": "28:bot-id" },
  "members": [{ "id": "29:user-id" }],
  "channelData": { "tenant": { "id": "tenant-id" } }
}
```

## Webhook Verification

Bot Framework includes `Authorization` header with JWT:
1. Fetch OpenID metadata: `GET https://login.botframework.com/v1/.well-known/openidconfiguration`
2. Validate JWT signature against published keys
3. Verify `iss`, `aud` (must match your app_id), and `exp` claims

## Filtering Rules
1. Skip if `from.id` matches bot's own ID
2. Skip if activity `type` is not `message`
3. Skip if `from.aadObjectId` not in allowed_users (when configured)
4. Skip empty text

## Rich Text & Adaptive Cards

Teams supports:
- Markdown in `text` field (`textFormat: "markdown"`)
- Adaptive Cards for rich interactive UI
- HTML subset (limited)

### Adaptive Card Message
```json
{
  "type": "message",
  "attachments": [{
    "contentType": "application/vnd.microsoft.card.adaptive",
    "content": {
      "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
      "type": "AdaptiveCard",
      "version": "1.5",
      "body": [
        { "type": "TextBlock", "text": "Response", "wrap": true }
      ]
    }
  }]
}
```

## Message Threading

- `conversation.id` identifies the channel/chat
- Reply with `replyToId` to thread under a specific message
- 1:1 chats vs group chats vs channel conversations have different ID formats

## Typing Indicator

```
POST {serviceUrl}/v3/conversations/{conversationId}/activities
{
  "type": "typing"
}
```

## Key Implementation Notes

- `serviceUrl` varies by region — always use the URL from the incoming activity
- Token must be refreshed before expiry (cache with TTL)
- Teams uses `@mention` format: `<at>Bot Name</at>` — strip from content
- Adaptive Cards have 28KB size limit
- Proactive messaging requires prior conversation or admin consent
- Bot must be installed in the Teams tenant to receive messages
- Azure Bot Service registration required (free tier available)
- `tenantId` is needed for multi-tenant bots
