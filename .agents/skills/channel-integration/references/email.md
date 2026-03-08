# Email Channel Integration Reference

## Platform: IMAP + SMTP
**Connection**: IMAP IDLE for receiving, SMTP for sending
**Auth**: Username/password or OAuth2 (provider-dependent)
**Max message**: Unlimited (practical limit: keep under 25MB with attachments)
**Rate limit**: Provider-dependent (Gmail: 500/day for free, 2000/day for Workspace)

## Architecture

Email uses two separate protocols:
- **IMAP** for receiving (persistent connection with IDLE push)
- **SMTP** for sending

Both support TLS (required).

## Receiving Flow (IMAP)

1. Connect to IMAP server (TLS, port 993)
2. Authenticate (LOGIN or OAuth2)
3. `SELECT INBOX`
4. `IDLE` → wait for new messages (server pushes notification)
5. On new message: `FETCH` → parse MIME → normalize
6. Return to IDLE

### Fallback Polling
If IMAP IDLE not supported: poll with `SEARCH UNSEEN SINCE {date}` every 30 seconds.

## Sending Flow (SMTP)

1. Connect to SMTP server (STARTTLS port 587, or TLS port 465)
2. Authenticate
3. Compose MIME message (headers + body + optional attachments)
4. `MAIL FROM`, `RCPT TO`, `DATA` → send
5. Disconnect (or keep alive for batch)

## MIME Parsing

Inbound emails are MIME multipart. Extract in order:
1. `text/plain` → primary content
2. `text/html` → fallback, strip HTML tags to plain text
3. `multipart/alternative` → prefer `text/plain`, fall back to `text/html`
4. Attachments → `Content-Disposition: attachment` parts

### Key Headers
```
From: sender@example.com
To: bot@example.com
Subject: Re: conversation subject
Date: Mon, 1 Jan 2024 00:00:00 +0000
Message-ID: <unique@example.com>
In-Reply-To: <parent@example.com>
References: <thread@example.com>
```

## Conversation Threading

Email threads via `In-Reply-To` and `References` headers:
- `In-Reply-To` → immediate parent message ID
- `References` → full thread chain (space-separated message IDs)
- conversation_id = first `References` entry (thread root) or `Message-ID` for new threads
- Subject line `Re:` prefix is unreliable — use headers

## MIME Composition (Outbound)

```
From: bot@example.com
To: recipient@example.com
Subject: Re: original subject
In-Reply-To: <original-message-id>
References: <thread-root-id> <original-message-id>
MIME-Version: 1.0
Content-Type: text/plain; charset=utf-8
Content-Transfer-Encoding: quoted-printable

Response body here
```

## Filtering Rules
1. Skip if `From` matches bot's own email address
2. Skip if `From` not in allowed_senders (when configured)
3. Skip automated messages: `Auto-Submitted: auto-*`, `X-Auto-Response-Suppress`
4. Skip mailing list digests: `List-Unsubscribe` header present (optional)
5. Skip if no text content extractable

## Command Parsing
- Subject line starting with `/` → parse as command
- Or first line of body starting with `/` → parse as command
- Otherwise → full body as text content

## OAuth2 (Gmail, Outlook)

For Gmail/Outlook, use OAuth2 instead of password:
1. Register OAuth app with provider
2. Get refresh token via OAuth flow
3. Exchange refresh token → access token before each connection
4. Use `XOAUTH2` SASL mechanism for IMAP/SMTP auth

## Key Implementation Notes

- IMAP IDLE has a 29-minute timeout — must re-issue IDLE before timeout
- Gmail requires "Less secure apps" or OAuth2 (app passwords deprecated)
- Subject line is preserved for conversation context
- HTML emails should be converted to plain text for the agent
- Attachments: store temporarily, pass URL to agent
- Email is inherently asynchronous — no typing indicators, no read receipts
- Consider spam: validate SPF/DKIM on inbound (usually server-side)
- Outbound: set proper `Reply-To`, `Message-ID` for threading
