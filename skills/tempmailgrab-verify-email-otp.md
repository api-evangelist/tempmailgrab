---
name: tempmailgrab-verify-email-otp
description: Create a disposable TempMailGrab inbox, hand its address to a sign-up or password-reset flow, and read back the server-extracted one-time passcode or verification link without parsing an email body.
generated: '2026-09-01'
method: generated
source: openapi/tempmailgrab-openapi.json + https://tempmailgrab.com/api-docs
api: TempMailGrab REST API
base_url: https://tempmailgrab.com/api/v1
operations:
- createInbox
- listInboxMessages
- getInboxMessage
- deleteInbox
---

# Verify an email OTP or confirmation link

Use this when a flow under test sends a code or a confirm link to an email address and you need to read it
back programmatically.

## Before you start

- Authenticate every request with `Authorization: Bearer <key>` **or** `X-API-Key: <key>`. Keys are prefixed
  `tmg_live_` and are account-scoped — you can only read inboxes your own key created.
- There is **no idempotency key**. If `POST /inbox` times out, do not blindly retry: you may mint a second
  inbox. Treat a transport failure on any POST as unknown and reconcile before retrying.

## Steps

1. **Create the inbox** — `createInbox`, `POST /inbox`.
   Optional body: `prefix` (local part), `domain`, `ttl_seconds` (600–259200, default 86400).
   Returns `{ id, address, created_at, expires_at }`. Keep `address` for the form and `id` for polling.
   Rate limit on this operation is stricter than the rest of the API: 2 per second per key.

2. **Trigger the flow** in the system under test using `address`.

3. **Poll for the message** — `listInboxMessages`, `GET /inbox/{id}/messages`.
   Returns every message, newest first, as `MessageSummary` — `{ id, sender, subject, snippet,
   extracted_otp, created_at }`. There is no `limit` or `page` parameter, so read the whole array.
   The passcode is already parsed for you in `extracted_otp`; do **not** regex the body.
   Poll on an interval (1–2s) with a wall-clock deadline (30s is the SDK default). Stop on the deadline,
   not on a fixed loop count.

4. **If you need the confirm URL rather than a code** — `getInboxMessage`,
   `GET /inbox/{id}/messages/{mid}`. `extracted_links` holds the verification/confirmation URLs;
   `html_body` is sanitized HTML, `text_body` the plain part, `attachments[]` carries `{ id, filename,
   content_type, size, url }`.

5. **Tear down** — `deleteInbox`, `DELETE /inbox/{id}`. Optional: every inbox expires on its own at
   `expires_at` and is purged with its messages and attachment binaries. Deletion is permanent and has no
   restore path, so only call it when you are finished.

## Handling failures

- `401` — missing or invalid key. Terminal; fix the credential, do not retry.
- `404` — the inbox id is unknown **or** the inbox expired and was purged. These are indistinguishable, so
  track `expires_at` yourself rather than probing.
- `429` — rate limited. Honour `Retry-After`; `X-RateLimit-Remaining` and `X-RateLimit-Reset` come back on
  every response. Back off exponentially.
- `500` — transient; retry with backoff.
- Error bodies are `{"error": "<string>"}`. There is no stable machine-readable code — branch on the HTTP
  status.
- Timing out with zero messages usually means the system under test never sent the mail. Timing out with
  messages present means mail is flowing but none of it matched your predicate.

## Parallel test runs

Create one inbox per test rather than sharing an address. Prefixes are sanitized to `[a-z0-9]` (max 12
chars) with a random suffix appended, so parallel workers cannot collide.
