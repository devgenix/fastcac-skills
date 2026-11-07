# Business data

Same keys, wallet, webhooks and test mode as registrations; those are in the skill's main file.
Base URL `https://api.fastcac.com/v1`; send `Authorization: Bearer $FASTCAC_KEY` and an
`Idempotency-Key` on every `POST` (a retry with the same key gets the first answer and isn't
charged again).

Never write prices into code or copy. They're at https://fastcac.com/api/pricing; each charged
response carries `amount_charged` and `transaction_id`.

## Pick the call

| Need | Call | Charged? |
|---|---|---|
| Find businesses by name or number | `GET /v1/businesses/search?query=` | Free |
| One business's full record with officers and shareholders | `POST /v1/businesses/lookups` | Yes |
| Check what an applicant told you (number, name, active?) | `POST /v1/businesses/verifications` | Free |
| …and whether named people are its officers | same, with `officers` | Yes |
| Get told when a business's status or name changes | `POST /v1/businesses/monitors` | Yes, per 30 days |
| A person's NIMC record from their NIN | `POST /v1/identities/nin_lookups` | Yes |

Reading a lookup, verification or NIN lookup again by id (`GET …/{id}`) is free. Only your
organization, in the same mode, can read them.

## Registration numbers

A number can be digits only (`7654321`) or carry its register prefix (`RC 7654321`, `BN-1234567`).
Registers (`type`): `BN` business name, `RC` company, `IT` incorporated trustees, `LP` limited
partnership, `LLP` limited liability partnership. The same digits can exist in more than one
register: send the prefix or `type`, or you get `400 type_required` with the options in `details.types`.

## Search CAC's register (free)

```bash
curl "https://api.fastcac.com/v1/businesses/search?query=Ada%20Foods" -H "Authorization: Bearer $FASTCAC_KEY"
```

- Up to 50 `cac_record`s, best match first: `number`, `type`, `type_label`, `name`, `status`
  (`ACTIVE`, `INACTIVE`, `STRUCK OFF`…), `active`, `registered_on`.
- Free with test keys too, which search the **real** register.
- 30 a minute per organization live, 10 test (live lookups, verifications and new monitors count too).
- `query` under 2 characters: `400 parameter_invalid`. `503 cac_search_unavailable`: retry in a minute.

## Look up a business (charged)

```bash
curl https://api.fastcac.com/v1/businesses/lookups -H "Authorization: Bearer $FASTCAC_KEY" \
  -H "Idempotency-Key: $(uuidgen)" -H "Content-Type: application/json" -d '{"number": "RC 7654321"}'
```

Returns a `cac_lookup` (`cacl_…`): `name`, `status` (always read fresh), `active`, `registered_on`,
`address`, `email`, and `officers`:

- Everyone who runs or owns it, **once each with every role**: `roles` from `proprietor`, `partner`,
  `trustee`, `director`, `secretary`, `shareholder`, `psc`; `role` is the first of them.
- Per officer: `corporate`, `status`, `appointed_on`, `gender`, `date_of_birth`, `nationality`,
  `occupation`, `phone`, `email`, `address` (null when hidden at CAC), `shares` (`count`, `type`,
  `percent`), `psc` (control details; often null).
- Never returns government ID numbers, or people who only filed paperwork.

| Outcome | Charged |
|---|---|
| The record | Yes (`cac_lookup_charge`) |
| `404 cac_record_missing` | No |
| `503 lookup_unavailable` | Charged then refunded (`cac_lookup_refund`) |
| `402 insufficient_balance` (live: checked before the number is searched, so it comes first) | No |
| `503 cac_search_unavailable`, `400` | No |

Live: 60 a minute per organization.

## Verify a business

```bash
curl https://api.fastcac.com/v1/businesses/verifications -H "Authorization: Bearer $FASTCAC_KEY" \
  -H "Idempotency-Key: $(uuidgen)" -H "Content-Type: application/json" \
  -d '{"number": "RC 7654321", "name": "Ada Foods Ltd", "officers": ["Ada Obi"]}'
```

`officers` is optional, up to 10 names, each at least two words. Response (`bver_…`):

| `result` | When |
|---|---|
| `match` | Name exact, business active, every officer sent matched. |
| `partial` | Name close but not exact, or not active, or an officer didn't match. |
| `no_match` | The name is a different business's. |
| `not_found` | CAC has no business with that number. |

`result` is a summary; apply your own rules to `checks`: `number_found`; `name.result`
(`exact` / `close` / `no_match`), `name.score` (0-1), `name.registered_name`; `active`, `status`;
`officers` (one per name sent: `matched`, and when matched `roles`, `role`, `officer` with the same
details as a lookup; `null` when none sent). Officer names match word for word in any order when
every word of the shorter is in the longer ("Ada Obi" ↔ ADA CHIOMA OBI); initials and common spelling
variants count, misspellings don't. Only the result and checks are stored, not the other officers;
use a lookup for everyone.

| Outcome | Without officers | With officers |
|---|---|---|
| A result | Free (`amount_charged: "0.00"`) | Charged (`business_verification_charge`) |
| `not_found` | Free | Not charged |
| `503 lookup_unavailable` | n/a | Charged then refunded (`business_verification_refund`) |
| `503 cac_search_unavailable`, `402`, `400` | Not charged | Not charged |

Live: 60 a minute per organization. A `not_found` applicant can register a business name through
the API (see [Registering a business name](registrations.md)).

## Monitor a business (charged per 30 days)

```bash
curl https://api.fastcac.com/v1/businesses/monitors -H "Authorization: Bearer $FASTCAC_KEY" \
  -H "Idempotency-Key: $(uuidgen)" -H "Content-Type: application/json" \
  -d '{"number": "RC 7654321", "external_id": "merchant-4471"}'
```

- Returns a `business_monitor` (`mon_…`): `number`, `type`, `type_label`, `name`, `status`, `active`, `external_id`, `state`
  (`active`, `paused`, `deleted`), `paid_through`, `last_checked_at`, `last_changed_at`, `amount_charged`.
- CAC's register is read once a day. Charged on adding and again each time `paid_through` passes
  (`business_monitor_charge`).
- `409 monitor_exists` (the existing id in `details.monitor`), `404 cac_record_missing`,
  `400 type_required`, `402 insufficient_balance`, `503 cac_search_unavailable`: none charged.
- `GET /v1/businesses/monitors` (filter `external_id`, page `starting_after`, `include_deleted=true`),
  `GET …/{id}`, `DELETE …/{id}` stops it at once (the rest of the 30 days isn't refunded).
- A renewal the wallet can't cover pauses it (`monitor.paused`, once; no checks). After funding, the
  next daily check charges, starts a fresh 30 days and sends `monitor.resumed`.

Webhooks (subscribe an endpoint to them), each carrying the monitor with your `external_id`:
`business.status_changed` (`status`, `previous_status`; usually `ACTIVE` → `INACTIVE` for unfiled
annual returns), `business.name_changed` (`name`, `previous_name`), `monitor.paused`, `monitor.resumed`.
If CAC is down or briefly loses a record, nothing changes and no event is sent.

## NIN lookup (charged)

`POST /v1/identities/nin_lookups {"nin": "<11 digits>", "consent": true}` returns NIMC's record:
names, `gender`, `date_of_birth`, `phone`, `address`, `photo` (JPEG `data:` URI), `nimc_data`.
`consent: true` confirms the person agreed (else `400 consent_required`). Not charged or refunded
(`nin_lookup_refund`): `422 nin_not_found`, `503 nin_lookup_unavailable`, `402`, `400`. Live: 60 a
minute. Use it in a registration via the proprietor's `nin_lookup` (see [Registering a business name](registrations.md)).

## Test mode

Test keys never reach CAC or NIMC (search is the exception: it uses the real register, free).

- Lookups, verifications and monitors use one test business: ADA FOODS VENTURES when `type` is `BN`,
  ADA FOODS LIMITED otherwise, active, with ADA CHIOMA OBI as proprietor/director (in the LIMITED she's
  director, shareholder and psc; EMEKA JOHN OBI holds the other 25%). So
  `{"number": "RC 7654321", "name": "Ada Foods Ltd", "officers": ["Ada Obi"]}` is a `match`.
- `simulate` (query parameter):
  - search: `cac_search_unavailable`
  - lookups: `cac_record_missing`, `lookup_unavailable`, `insufficient_balance`
  - verifications: `not_found`, `inactive`, `cac_search_unavailable`; with officers also
    `lookup_unavailable`, `insufficient_balance`
  - monitors: `cac_record_missing`, `insufficient_balance`
  - NIN lookups: `nin_not_found`, `insufficient_balance` (otherwise always the test person ADA CHIOMA OBI)
- Test monitors are never read from CAC or renewed. Trigger a change and its events with
  `POST /v1/test_helpers/businesses/monitors/{id}/changes {"status": "INACTIVE"}` (and/or `name`;
  same value sends nothing; `409 monitor_deleted` on a stopped monitor; `403 test_mode_only` live).
- Charged calls spend the test wallet.

Docs: https://docs.fastcac.com/guides/business-lookups · /guides/business-verifications ·
/guides/business-monitors · /guides/nin-lookups · /test-mode
