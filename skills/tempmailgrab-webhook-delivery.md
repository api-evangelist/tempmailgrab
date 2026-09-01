---
name: tempmailgrab-webhook-delivery
description: Register an HMAC-signed TempMailGrab webhook so parsed messages are pushed to your endpoint instead of polled, verify each delivery correctly, and de-duplicate replays.
generated: '2026-09-01'
method: generated
source: openapi/tempmailgrab-openapi.json + https://tempmailgrab.com/api-docs
api: TempMailGrab REST API
base_url: https://tempmailgrab.com/api/v1
operations:
- createWebhook
- createInboxWebhook
- listWebhooks
- deleteWebhook
- messageReceivedWebhook
---

# Receive parsed mail by webhook

Push beats polling when you control a reachable HTTPS endpoint.

## Choose the scope

- `createWebhook`, `POST /webhooks` — account-level. Fires for **every** inbox owned by the key.
- `createInboxWebhook`, `POST /inbox/{id}/webhook` — scoped to one inbox.

Both return `{ id, secret }` with a `whsec_`-prefixed signing secret. **The secret is returned once, at
creation.** Store it before you do anything else; there is no retrieval endpoint.

There is no idempotency key on either call. A blind retry after a timeout can register a duplicate that
then fires twice for every message. Call `listWebhooks` (`GET /webhooks`) to reconcile before re-creating.

## The delivery

`messageReceivedWebhook` — a `POST` of `application/json` to your URL, with `X-TMG-Signature` set to a
hex-encoded HMAC-SHA256 of the raw body computed with your secret.

Body:

```json
{ "event": "email.received", "sent_at": 1788157327, "data": { ... } }
```

`data` carries `id`, `inbox_address`, `sender`, `subject`, `text_body`, `html_body`, `extracted_otp`,
`extracted_links` and `timestamp`. Return any `2xx` to acknowledge.

## Verify against the raw bytes

Compute the HMAC over the **exact bytes received**, before any JSON parsing. Re-serialising a parsed body
does not reproduce the original — key order, whitespace and unicode escaping all differ — and every
legitimate delivery will then fail to verify. Compare in constant time. On any verification failure return
a bare `400` and say nothing about why.

## Replay

The signature covers the body only; it does not bind a timestamp the way a `t=`/`v1=` scheme does.
`sent_at` sits inside the signed body so it cannot be altered, but a captured delivery replayed
byte-for-byte stays valid indefinitely. Rejecting deliveries whose `sent_at` is far from now narrows the
window, at the cost of dropping mail when clocks skew. The reliable de-duplication is to record
`data.id` and skip ids you have already processed.

The provider publishes no retry schedule, backoff, or dead-letter behaviour for non-2xx responses, so do
not assume a failed delivery will be re-sent.

## Clean up

`deleteWebhook`, `DELETE /webhooks/{id}` stops deliveries. This is the reversal path for both creation
operations; the docs state no time bound on it.
