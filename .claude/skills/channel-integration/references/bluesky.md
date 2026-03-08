# Bluesky Channel Integration Reference

## Platform: AT Protocol (atproto)
**Connection**: REST polling (AT Protocol HTTP API) + optional Firehose WebSocket
**Auth**: Session-based (createSession → accessJwt + refreshJwt)
**Max message**: 300 graphemes per post (not characters — emoji = 1 grapheme)
**Rate limit**: 5,000 points/hour (create post = 3 points)

## Architecture

Bluesky uses the AT Protocol — a decentralized social networking protocol.
The adapter interacts with a Personal Data Server (PDS), typically `bsky.social`.

## Authentication Flow

1. `POST /xrpc/com.atproto.server.createSession`
   ```json
   { "identifier": "handle.bsky.social", "password": "app-password" }
   ```
2. Response: `{ "accessJwt": "...", "refreshJwt": "...", "did": "did:plc:..." }`
3. Use `accessJwt` as Bearer token for API calls
4. Refresh via `POST /xrpc/com.atproto.server.refreshSession` when expired

## API Base URL

```
https://bsky.social/xrpc/
```

## Key Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `com.atproto.server.createSession` | Login |
| POST | `com.atproto.server.refreshSession` | Refresh token |
| GET | `app.bsky.notification.listNotifications` | Get mentions, replies |
| POST | `com.atproto.repo.createRecord` | Create post/reply |
| POST | `com.atproto.repo.deleteRecord` | Delete post |
| GET | `app.bsky.feed.getPostThread` | Get thread context |
| POST | `app.bsky.notification.updateSeen` | Mark notifications read |

## Receiving Messages (Polling Notifications)

Poll `listNotifications` for mentions and replies:

```json
GET /xrpc/app.bsky.notification.listNotifications?limit=50

Response:
{
  "notifications": [{
    "uri": "at://did:plc:xxx/app.bsky.feed.post/yyy",
    "reason": "mention",  // or "reply"
    "author": {
      "did": "did:plc:xxx",
      "handle": "user.bsky.social",
      "displayName": "User Name"
    },
    "record": {
      "text": "@bot.bsky.social hello",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "reply": {
        "root": { "uri": "at://...", "cid": "..." },
        "parent": { "uri": "at://...", "cid": "..." }
      }
    },
    "indexedAt": "2024-01-01T00:00:00.000Z",
    "isRead": false
  }]
}
```

## Sending Messages (Creating Posts)

### Reply to a post
```json
POST /xrpc/com.atproto.repo.createRecord
{
  "repo": "did:plc:bot-did",
  "collection": "app.bsky.feed.post",
  "record": {
    "$type": "app.bsky.feed.post",
    "text": "response text",
    "createdAt": "2024-01-01T00:00:00.000Z",
    "reply": {
      "root": { "uri": "at://...", "cid": "..." },
      "parent": { "uri": "at://...", "cid": "..." }
    }
  }
}
```

## Facets (Rich Text)

Bluesky uses "facets" for rich text (mentions, links, hashtags):
```json
{
  "text": "Hello @user.bsky.social check https://example.com",
  "facets": [
    {
      "index": { "byteStart": 6, "byteEnd": 25 },
      "features": [{ "$type": "app.bsky.richtext.facet#mention", "did": "did:plc:xxx" }]
    },
    {
      "index": { "byteStart": 32, "byteEnd": 51 },
      "features": [{ "$type": "app.bsky.richtext.facet#link", "uri": "https://example.com" }]
    }
  ]
}
```

**Important**: Facet indices are byte offsets in UTF-8, not character positions.

## Filtering Rules
1. Skip if notification `author.did == bot_did` (own posts)
2. Skip if `reason` is not `mention` or `reply`
3. Skip if `author.handle` not in allowed_users (when configured)
4. Skip if `isRead == true` (already processed)
5. Mark as read after processing via `updateSeen`

## Grapheme Counting

Posts are limited to 300 **graphemes**, not bytes or characters:
- ASCII letter = 1 grapheme
- Emoji (even multi-codepoint) = 1 grapheme
- Use Unicode segmentation (Rust: `unicode-segmentation` crate)

If response exceeds 300 graphemes, split into thread (reply chain).

## Conversation Threading

- `reply.root` → first post in thread (conversation_id)
- `reply.parent` → immediate parent post
- AT URI format: `at://did:plc:xxx/app.bsky.feed.post/rkey`

## Key Implementation Notes

- App passwords (not main password) should be used for bots
- Access tokens expire quickly (hours) — refresh proactively
- DID (Decentralized Identifier) is the stable user ID, not the handle
- Handle resolution: `GET /xrpc/com.atproto.identity.resolveHandle?handle=...`
- Rate limits return 429 with `RateLimit-Reset` header
- Firehose (`com.atproto.sync.subscribeRepos`) available for real-time but heavy
- Poll interval: 10-30 seconds is reasonable for notifications
