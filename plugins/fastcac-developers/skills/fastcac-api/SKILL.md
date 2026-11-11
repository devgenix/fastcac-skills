---
name: fastcac-api
description: Build on the FastCAC API (api.fastcac.com/v1) for Nigerian businesses. Use for any FastCAC integration - registering a business name with CAC (name checks, NIN lookups, documents, submit, verification_failed and CAC queries, resubmitting, certificates), KYB and onboarding checks (searching CAC's register by name or RC/BN number, business lookups with officers, verifying an applicant's business, monitoring for status changes, NIMC NIN lookups, and which calls are free, charged or refunded), and the basics every call needs - fc_test_/fc_live_ keys, test mode and simulate, errors, idempotency keys, rate limits, pagination, the prepaid wallet, webhooks and signature verification, going live.
metadata:
  author: FastCAC
  api-version: "2026-11-01"
---

# The FastCAC API

FastCAC is a REST API for Nigerian businesses: register a Business Name with the Corporate Affairs
Commission (CAC), and search, look up, verify and monitor businesses on CAC's register.

This file covers what every call needs. Read the reference for the job before writing its code:

| Job | Calls | Read |
|---|---|---|
| Register a business name: name check, NIN lookup, create, documents, submit, CAC queries, certificate, testing every outcome | `/v1/businesses/names/availability`, `/v1/identities/nin_lookups`, `/v1/businesses/registrations…` | [Registering a business name](references/registrations.md) |
| Search CAC's register, look up a business with its officers, verify an applicant's business, monitor a business, look up a NIN | `/v1/businesses/search`, `/lookups`, `/verifications`, `/monitors`, `/v1/identities/nin_lookups` | [Business data](references/business-data.md) |

- Base URL: `https://api.fastcac.com/v1`. JSON in and out (document uploads are multipart).
- Docs: https://docs.fastcac.com (page index: https://docs.fastcac.com/llms.txt). Dashboard: https://developers.fastcac.com.
- Prices: never hard-code or quote them. They change and the source of truth is
  https://fastcac.com/api/pricing (and the dashboard's **Billing** page). Read what a call actually
  cost from its `amount_charged` field or the wallet transactions.

## Keys and modes

Send the secret key on every request (except `GET /v1/health` and `GET /v1/openapi.json`):

```bash
curl https://api.fastcac.com/v1/ping -H "Authorization: Bearer $FASTCAC_KEY"
```

| Prefix | Mode |
|---|---|
| `fc_test_` | Test: never reaches CAC or NIMC, never moves real money. Developers, admins and owners can create one. |
| `fc_live_` | Live: real CAC filings, charged from the real wallet. Only admins and owners, after the organization is verified. |

- The key alone picks the mode. Same base URL, same requests, same responses. There is no separate sandbox host.
- Objects don't cross modes: a test registration is `404` with a live key.
- Call from a server only. Any request with an `Origin` header (a browser) is refused with `400 browser_requests_not_allowed`. Keep keys in env vars or a secret manager.
- `GET /v1/ping` confirms a key and returns its organization.

## Errors

Every failure has the same shape. Branch on `error.code`, never on `message`:

```json
{"error": {"type": "not_found_error", "code": "resource_missing",
  "message": "No such registration: reg_0000000000000000000000.",
  "param": "id", "request_id": "req_gbQw9iB4MBVzQzZgN79RVjdz"}}
```

- `param` names the field at fault, e.g. `proprietors[0].nin`.
- `request_id` equals the `FastCAC-Request-Id` header; quote it to support.
- Some errors add `details` (e.g. `insufficient_balance` gives `amount_due`, `balance`, `funding_account`).
- Retry only `429`, `500` and `503`, with backoff. Treat an unknown code by its HTTP status.

Common codes: `400 parameter_invalid`, `401 api_key_missing` / `invalid_api_key`,
`402 insufficient_balance`, `403 test_mode_only`, `403 live_mode_unavailable`,
`404 resource_missing`, `409 invalid_state`, `409 registration_locked`,
`409 idempotency_key_reused`, `429 rate_limited`, `429 too_many_failed_attempts` (20+ failed key
attempts from one IP in a minute), `500 internal_error`, `503 …_unavailable`. Full table:
https://docs.fastcac.com/api-basics/errors

## Idempotency

Send `Idempotency-Key` (up to 255 characters, e.g. a UUID or `order-1182-submit`) on every `POST`.
It only applies to `POST`.

| Retry with the same key | Result |
|---|---|
| Same request, first succeeded | The stored response, header `Idempotent-Replayed: true` (kept 24 hours). Not charged again. |
| Different method, path or body | `409 idempotency_key_reused` |
| First still running | `409 idempotency_request_in_progress` |
| First failed | Runs again; failures aren't stored. |

- Keys are scoped to the organization and mode.
- The match ignores the query string: use a new key when you change `?simulate=`.
- Don't send a key on document uploads: a retried multipart body gets a new boundary and comes back
  `409 idempotency_key_reused`. Re-uploading is safe anyway (it replaces the document).
- On `500`, retry with the same key.

## Rate limits

| Mode | Per key | Per organization |
|---|---|---|
| Test | 25/s | 50/s |
| Live | 100/s | 200/s |

Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (Unix seconds).
Over the limit: `429 rate_limited` with `Retry-After`; wait that long. Per-organization minute
limits: live name checks, NIN lookups, business lookups, verifications and new monitors are 60 a
minute each; searching CAC's register is 30 a minute live, 10 test (live lookups, verifications and
new monitors count towards it).

## Pagination

Registration, monitor and balance-transaction lists return
`{"object": "list", "data": [...], "has_more": bool}`, newest first. Use `limit` (1-100, default 20)
and `starting_after` (the last `id` of the previous page). Registrations also filter by `status` and
`external_id`; monitors by `external_id`.

Webhook lists are different: `GET /v1/webhooks/endpoints`, `/webhooks/events` and
`/webhooks/endpoints/{id}/attempts` return a bare JSON array, and the last two take `limit` only
(1-100, default 50), with no `starting_after`.

## Versioning

Current version `2026-11-01` (`api_version` on webhook events). New endpoints, optional parameters,
fields, enum values and event types can appear at any time: ignore what you don't know. Breaking
changes come as a new dated version in https://docs.fastcac.com/changelog.

## The wallet

Every charge (registration submit, name check, NIN lookup, business lookup, verification with
officers, monitor) comes from a prepaid wallet. Test and live keys each have their own.

- `GET /v1/balance` → `available` (spendable), `held`, `currency` (`NGN`), `low_balance_threshold`,
  `funding_account` (`account_number`, `bank_name`, `account_name`; `null` until active, fake in test).
- Amounts are naira as decimal strings (`"1245000.00"`). Exception: `payment.succeeded` webhooks use kobo.
- Fund it by bank transfer to the funding account. Deposits are credited in full and fire
  `balance.funded`. A large deposit can be `held` until the transfer clears.
- Recommended: fund one registration at a time, when your customer pays you, then submit.
- A charge the wallet can't cover answers `402 insufficient_balance` and charges nothing; the
  object stays as it was. Fund and retry.
- Set a low-balance threshold on the dashboard's **Billing** page to get `balance.low`.
- `GET /v1/balance/transactions` lists every change (`wallet_transaction`, signed `amount`,
  `balance_after`, `type` such as `deposit`, `registration_charge`, `name_check_charge`,
  `nin_lookup_refund`, `cac_lookup_charge`, `business_monitor_charge`, `adjustment`, `test_credit`).
- Test wallet: starts with ₦1,000,000 of test money; add more with
  `POST /v1/test_helpers/deposits {"amount": "350000.00"}` (above 0, at most 10,000,000; test keys only).

Details: https://docs.fastcac.com/guides/balance

## Webhooks

FastCAC POSTs events to your HTTPS endpoint. Test events go only to test endpoints.

- Add one on the dashboard (**Webhooks**) or `POST /v1/webhooks/endpoints {"url": "https://…", "event_types": [...]}`.
  The URL must be public `https://` (local, private and fastcac.com addresses are refused). Up to 10
  per mode (`409 endpoint_limit`). Omit `event_types` to get every event.
- Secret: `GET /v1/webhooks/endpoints/{id}/secret` (`whsec_…`). Rotate with
  `POST /v1/webhooks/endpoints/{id}/secrets` (both secrets sign for a short while).
- Try it: `POST /v1/webhooks/endpoints/{id}/test-events {"event_type": "registration.approved"}`.
- Event types with examples: `GET /v1/webhooks/event-types`.

Envelope: `{"id": "evt_…", "type": "registration.queried", "api_version": "2026-11-01", "created_at": "…", "data": {...}}`.
`data` holds ids and statuses only, no personal data: fetch the object for details.

### Verify every delivery

Signed with the Standard Webhooks scheme. Headers: `webhook-id`, `webhook-timestamp`, `webhook-signature`.
Verify the **raw** body before parsing it. Easiest: the open-source `svix` library, given the
secret with its `whsec_` prefix:

```javascript
import { Webhook } from "svix";
new Webhook(process.env.FASTCAC_WEBHOOK_SECRET).verify(rawBody, headers); // throws if invalid; returns nothing
const event = JSON.parse(rawBody);
```

Python: `Webhook(secret).verify(raw_body, headers)`; PHP: `(new \Svix\Webhook($secret))->verify($rawBody, $headers)`.
By hand: reject a timestamp more than 5 minutes off; base64-decode the secret after `whsec_`;
HMAC-SHA256 over `{webhook-id}.{webhook-timestamp}.{raw body}`, base64-encode; accept if it equals any
`v1,<sig>` entry in the space-separated `webhook-signature`, compared in constant time.

### Handle it

1. Verify (answer `400` if invalid).
2. Skip a `webhook-id` you've already handled; retries resend the same id.
3. Answer `2xx` within 30 seconds; do slow work in the background. Answer `5xx` to get a retry.
4. Don't rely on order: fetch the object and act on its current `status`.
5. Return `2xx` for event types you don't handle.

Retries: after 5 s, 5 min, 30 min, 2 h, 5 h, 10 h, 10 h. An endpoint failing everything for 5 days
is disabled; re-enable with `PATCH /v1/webhooks/endpoints/{id} {"disabled": false}`. Resend with
`POST /v1/webhooks/endpoints/{id}/redeliveries {"since": …}` or one event to one endpoint with `POST /v1/webhooks/events/{id}/deliveries {"endpoint_id": "…"}`.
Ready-made Express, Laravel and Django handlers: https://docs.fastcac.com/webhooks/handlers

## Going live

1. On developers.fastcac.com open **Compliance**: identity (BVN + face check), then an owner or
   admin who is one of the business's officers on CAC (proprietor, partner, director, trustee or
   PSC) verifies its CAC record (live keys unlock straight away), then the funding account opens
   after a review (owners and admins are emailed).
2. Test every `simulate` outcome end to end.
3. Add a live webhook endpoint and verify signatures.
4. An owner or admin creates an `fc_live_` key. Swap the key; nothing else changes.
5. Fund the wallet and submit.

Live differences: one proprietor per registration for now, CAC's own certificate PDF (arrives a
while after approval), `simulate` is ignored, test helpers answer `403 test_mode_only`.
FastCAC re-checks your own business's CAC status weekly; if it stays not active for 30 days the
organization is suspended. An organization live before the officer check has 14 days for an
officer to join as an owner or admin and verify their BVN, then its live keys are refused until
one does (test keys keep working). Details: https://docs.fastcac.com/going-live

## Checklist

- [ ] Key from env/secret manager, server-side only; `fc_test_` until going live.
- [ ] `Idempotency-Key` on every `POST` except document uploads; same key on retry.
- [ ] Branch on `error.code`; retry only 429/500/503 with backoff and `Retry-After`.
- [ ] Handle `402 insufficient_balance` by funding, not by retrying in a loop.
- [ ] No prices in code or copy; link https://fastcac.com/api/pricing or read `amount_charged`.
- [ ] Webhook handler verifies the raw body, dedupes `webhook-id`, answers fast, fetches the object.
