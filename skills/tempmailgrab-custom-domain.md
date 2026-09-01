---
name: tempmailgrab-custom-domain
description: Register a bring-your-own domain with TempMailGrab, apply the DNS records it returns, and verify TXT ownership plus MX routing so inboxes are issued on your own domain.
generated: '2026-09-01'
method: generated
source: openapi/tempmailgrab-openapi.json + https://tempmailgrab.com/premium
api: TempMailGrab REST API
base_url: https://tempmailgrab.com/api/v1
operations:
- listCustomDomains
- createCustomDomain
- verifyCustomDomain
- createInbox
---

# Bring your own domain (BYOD)

Issue disposable inboxes on a domain you control instead of the shared TempMailGrab domain. BYOD is a
developer-tier feature.

## Steps

1. **Check what is already registered** — `listCustomDomains`, `GET /byod`. Returns the domains attached to
   this API key. Do this first: `createCustomDomain` returns `409` when the domain is already registered,
   and there is no published delete/deregister operation to undo a mistaken registration.

2. **Register** — `createCustomDomain`, `POST /byod`. The response carries the DNS requirements for the
   domain: an ownership TXT record and MX routing.

3. **Apply the records** at your DNS provider. Allow for propagation before verifying.

4. **Verify** — `verifyCustomDomain`, `POST /byod/{domain}/verify`. Checks TXT ownership and MX routing.
   `404` means the domain is not registered under this key. Re-run after DNS propagates; this operation is
   safe to repeat.

5. **Use it** — pass `domain` in the body of `createInbox` to have addresses issued on the verified domain.

## Notes

- Addresses are **receive-only** on every domain, custom included. Nothing can be sent from a TempMailGrab
  address, which is why the service cannot become a spam relay and why your domain's sending reputation is
  not exposed.
- There is no published operation to remove a registered custom domain. Treat registration as one-way and
  confirm the domain before calling `POST /byod`.
- The pricing page lists "multiple domains" as planned rather than shipped; assume one primary domain per
  account until the provider says otherwise.
- `POST /byod` has no idempotency key. On a timeout, call `listCustomDomains` before retrying.
