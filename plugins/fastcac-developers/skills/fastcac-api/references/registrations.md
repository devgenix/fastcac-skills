# Registering a business name

Business Name registration with Nigeria's Corporate Affairs Commission (CAC):
check the name → (optional) look up each proprietor's NIN → create a `draft` → fill it → upload
documents → submit (charged from the wallet) → CAC decides → download the certificate.

Keys, errors, `Idempotency-Key`, the wallet and webhook verification are in the skill's main
file. Base URL `https://api.fastcac.com/v1`; every call needs `Authorization: Bearer $FASTCAC_KEY`.
Never write prices into code or copy: they're at https://fastcac.com/api/pricing, and every
charged response says what it cost in `amount_charged`.

## 1. Check the name (charged)

```bash
curl https://api.fastcac.com/v1/businesses/names/availability \
  -H "Authorization: Bearer $FASTCAC_KEY" -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ada Foods Ventures", "category": "Educational Services And Consultancy"}'
```

- It's a `POST` with a JSON body (the old `GET` answers `405`). Nothing is reserved.
- `name` 3-100 characters. `category` must exactly match one of CAC's lines of business from
  `GET /v1/businesses/categories` (fetch once, for a picker), or `400 parameter_invalid`.
- Response: `name` (as CAC records it, in capitals), `available`, `requires_certificate`, `reason` (`code`, `message`, and per code
  `accepted_qualifiers`, `words` or `conflicting_name`), `suggested_names`, `similar_names`,
  `similarity_score`, `most_similar_name`, `amount_charged`, `transaction_id`.
- When `available` is `false`, branch on `reason.code`, show `reason.message`. Treat an unknown code as `not_accepted`.
- `requires_certificate: true` means CAC wants a professional certificate, uploaded as a `supporting_document`.
- Not charged: a name failing the rules checked before the registry (`single_word`,
  `restricted_word`, `qualifier_missing`), and `402 insufficient_balance`. `503 name_check_unavailable` is refunded; retry in a minute.

Reason codes: `single_word`, `restricted_word`, `qualifier_missing`, `qualifier_not_last`,
`multiple_qualifiers`, `name_exists`, `name_too_similar` (90%+ similar), `restricted_category`,
`offensive_language`, `religious_reference`, `special_characters`, `low_compliance`, `not_accepted`.

CAC naming rules to enforce in your form: at least two words; ends in exactly one qualifier such as
VENTURES, ENTERPRISES, HUB, CONCEPTS, FASHION, SERVICES, BEAUTY or BIZ (as the last word); no
restricted word anywhere (whole words, e.g. BANK, COMPANY, LIMITED, LTD, GROUP, SCHOOL, CHURCH,
FOUNDATION, INVESTMENT, MONEY; full list at https://docs.fastcac.com/guides/cac-rules).

## 2. Look up the NIN (optional, charged)

```bash
curl https://api.fastcac.com/v1/identities/nin_lookups \
  -H "Authorization: Bearer $FASTCAC_KEY" -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" -d '{"nin": "12345678901", "consent": true}'
```

- `consent: true` is your confirmation the person agreed; without it `400 consent_required`.
- Returns `id` (`ninl_…`), `first_name`, `middle_name`, `last_name`, `gender`, `date_of_birth`,
  `phone`, `address` (one line), `photo` (JPEG `data:` URI), `nimc_data`.
- Not charged / refunded: `422 nin_not_found`, `503 nin_lookup_unavailable`, `402`, `400`.
- `GET /v1/identities/nin_lookups/{id}` reads it again for free.
- Set a proprietor's `nin_lookup` to the id: blank `first_name`, `last_name`, `other_names`
  (from `middle_name`), `gender`, `date_of_birth`, `nin` and `phone` are filled from NIMC. Submit then
  refuses a mismatched name, gender or date of birth with `400 nin_mismatch` (before charging), so
  CAC doesn't query it later. A `nin` that isn't the lookup's is `400 parameter_invalid`.
- The lookup's photo can be the passport: upload `kind=passport` with `nin_lookup=ninl_…` and no `file`.
  Set the proprietor's `nin` (or `nin_lookup`) first, and it must be their own NIN, or it's `400 parameter_invalid`.

## 3. Create and fill the registration

`POST /v1/businesses/registrations` (with `Idempotency-Key`). Nothing is required on create; submit
needs everything below. `PATCH /v1/businesses/registrations/{id}` sets only the fields you send, but
`business` and `proprietors` are replaced whole.

```json
{
  "external_id": "order-1182",
  "proposed_names": ["Ada Foods Ventures"],
  "category": "Educational Services And Consultancy",
  "business": {"email": "hello@adafoods.ng", "phone": "08031234567",
    "address": {"street_number": "12", "street": "Admiralty Way", "city": "Lekki",
                "lga": "Eti-Osa", "state": "Lagos", "postcode": "105102"}},
  "proprietors": [{
    "first_name": "Ada", "last_name": "Obi", "gender": "female", "date_of_birth": "1990-04-12",
    "nationality": "Nigerian", "nin": "12345678901", "email": "ada@example.com",
    "phone": "08031234567",
    "address": {"street_number": "12", "street": "Admiralty Way", "city": "Lekki",
                "lga": "Eti-Osa", "state": "Lagos", "postcode": "105102"}}]
}
```

| Field | Rule |
|---|---|
| `proposed_names` | 1 or 2, ≤100 chars each, stored in capitals. Only the first is filed for now. |
| `category` | A value from `GET /v1/businesses/categories`. |
| `business.email` | Required. `business.phone` optional. |
| `business.address` | `street_number`, `street`, `city`, `state` required; `lga`, `postcode` optional. |
| `proprietors` | 1 to 20. **Live registrations take one proprietor for now** (more is `400 parameter_invalid` on submit). |
| `proprietors[].first_name`, `last_name`, `email` | Required. `other_names` optional. |
| `proprietors[].gender` | `male` or `female`. |
| `proprietors[].date_of_birth` | `YYYY-MM-DD`, 18 or older. |
| `proprietors[].nationality` | `Nigerian` (the default and only value). |
| `proprietors[].nin` | 11 digits. CAC checks it against the name and date of birth. |
| `proprietors[].phone` | Required. 10-15 digits, optional leading `+`. |
| `proprietors[].address` | `street_number`, `street`, `city`, `lga`, `state` and a 6-digit `postcode`, **all required**. |
| `external_id` | Optional, ≤255 chars; filter lists by it. |

`missing_fields` on the registration lists empty required fields as paths (e.g.
`proprietors[0].address.postcode`); documents aren't in it. Create and PATCH reject bad names,
gender or proprietor counts at once; everything else is checked at submit.

## 4. Upload documents

`POST /v1/businesses/registrations/{id}/documents`, multipart: `kind`, `proprietor` (index, default
`0`), `file` (or `nin_lookup` for `passport`). **Don't send an `Idempotency-Key` on uploads.**

```bash
curl https://api.fastcac.com/v1/businesses/registrations/$REG_ID/documents \
  -H "Authorization: Bearer $FASTCAC_KEY" -F kind=passport -F proprietor=0 -F file=@passport.jpg
```

| `kind` | What | Types |
|---|---|---|
| `means_of_id` | The paper NIN slip. No other ID, not the digital slip. | JPEG, PNG, WebP |
| `passport` | Clear, recent passport photo (or NIMC photo via `nin_lookup`). | JPEG, PNG, WebP |
| `signature` | Signature on plain paper. | JPEG, PNG, WebP |
| `supporting_document` | Optional: professional certificate when the name check says `requires_certificate`. | JPEG, PNG, WebP, PDF |

- Each proprietor needs the first three before submit. Over 5 MB is `400 file_too_large`; a PDF over
  2 MB comes back as a rejected document (below), not an error.
- FastCAC only checks the file type suits the kind and the file opens. A bad file is a `201` with
  `status: "rejected"` and a `reason` (and `document.rejected`), not an error. FastCAC does **not**
  check what the document shows; CAC queries what it can't use.
- Uploading the same `kind` again replaces it. List with `GET …/documents`.

## 5. Submit (charged once)

```bash
curl -X POST https://api.fastcac.com/v1/businesses/registrations/$REG_ID/submissions \
  -H "Authorization: Bearer $FASTCAC_KEY" -H "Idempotency-Key: order-1182-submit"
```

- Charges the registration price from the wallet and moves `draft` → `paid`, then FastCAC files it
  with CAC → `submitted`. Webhooks: `registration.paid`, `payment.succeeded`, `registration.submitted`.
- A live response can still say `paid` (test says `submitted`): rely on webhooks.
  While `paid`, `failure.code` may be `filing_unconfirmed`: nothing to do.
- A live resubmit after `verification_failed` goes back through `paid` and sends `registration.paid`
  again, without `payment.succeeded`. Count charges from `payment.succeeded`, not `registration.paid`.
- `400 registration_incomplete` (missing fields or documents; `param` names the first),
  `400 parameter_invalid` (a wrong value), `400 nin_mismatch`, `402 insufficient_balance`
  (stays `draft`, nothing charged; fund and submit again), `409 invalid_state`.

## 6. Statuses and what to do

| Status | Webhook | Do |
|---|---|---|
| `draft` | `registration.created` | Fill until `missing_fields` is empty and documents are `received`, then submit. |
| `paid` | `registration.paid`, `payment.succeeded` | Nothing; FastCAC files it. |
| `verification_failed` | `registration.verification_failed` | CAC's system refused the values. Read `failure` (`code`: `cac_validation_failed`, `cac_refused`, `nin_mismatch`; `fields` can be empty, so show `message`). PATCH and resubmit free. |
| `submitted` | `registration.submitted` | Wait; CAC usually decides within 24 hours. |
| `final_review` | `registration.final_review` | Wait; an officer confirms within about 48 working hours (no weekends). Can still be queried or rejected. |
| `queried` | `registration.queried` | Answer the query (below), resubmit free. |
| `approved` | `registration.approved`, then `registration.certificate_ready` | Read `registered_name` and the seven-digit `registration_number`; download PDFs after `certificate_ready`. |
| `rejected` | `registration.rejected` | Final, not refunded. Start a new registration, usually with another name (charged again). |

Editing: `draft` and `verification_failed` allow anything; `queried` allows only the queried fields
and documents (`400 field_not_queried`); everything else is `409 registration_locked`.
Each registration's `history` lists every status change. Fetch the registration when a webhook
arrives; poll only as a fallback.

## 7. Answer a CAC query

Read `query` on the registration (or `GET …/queries`). Each item has `field` (path to fix, or `null`),
`code` (`name_exists`, `qualifier_missing`, `single_word`, `offensive_language`,
`fraudulent_association`, `document_unclear`, `name_mismatch`, `other`), `message` (CAC's words; show
them) and `resolved`.

1. PATCH only the queried fields; re-upload queried documents. Items flip to `resolved: true`.
2. Submit again (free). It returns to `submitted` with `query.answered_at` set.

Errors: `400 field_not_queried` (changed something not asked; only allowed when an item's `field` is
`null`), `400 query_unresolved`, `422 correction_refused` (CAC refused the answer; recheck),
`503 cac_unavailable` (retry shortly). Once CAC has queried, a resubmit isn't checked against the NIN lookup.

## 8. Certificate and status report

- `GET /v1/businesses/registrations/{id}/certificate` and `…/status-report` return `application/pdf`.
- Before they exist: `409 certificate_not_ready` / `409 status_report_not_ready`. Live PDFs are
  CAC's own and arrive a while after approval, so wait for `registration.certificate_ready`.
- Download as often as you like.

## Test every outcome

Test keys never reach CAC. The test wallet is spent by every submit except `insufficient_balance`.

`POST …/submissions?simulate=`:

| Value | Result |
|---|---|
| `approved` (default) | `submitted`, then `approved` about a minute later. |
| `final_review` | `submitted` → `final_review` → `approved`, about a minute apart. |
| `queried` | `submitted`, then `queried`. Pick items with `simulate_query=` (comma-separated): `name_exists`, `qualifier_missing`, `single_word`, `offensive_language`, `fraudulent_association`, `passport_unclear` (default), `means_of_id_unclear`, `first_name_mismatch`, `last_name_mismatch`, `category_unclear`. A resubmit after a query is approved. `simulate_query` alone implies `queried`; with any other outcome it's `400 parameter_invalid`. |
| `rejected` | `submitted`, then `rejected`. |
| `verification_failed` | At once, on `proprietors[0].nin`. Resubmit doesn't charge again. |
| `insufficient_balance` | `402`, nothing charged. |

- Use a new `Idempotency-Key` whenever you change `simulate` (the query string isn't part of the match).
- Move a registration now: `PATCH /v1/test_helpers/businesses/registrations/{id} {"status": "approved"}`
  (from `submitted`: `approved`, `final_review`, `queried`, `rejected`; from `final_review`: `approved`,
  `queried`, `rejected`; anything else `409 invalid_state`).
- Name checks in test mode apply only the first three naming rules (no registry search); force answers with
  `?simulate=available|requires_certificate|<reason code>|insufficient_balance`.
- NIN lookups in test mode always return the test person ADA CHIOMA OBI (female, 1990-04-12);
  `?simulate=nin_not_found|insufficient_balance`.
- Documents: `?simulate=rejected` forces `document.rejected`.
- Test certificates are sample PDFs marked "SANDBOX - NOT A CAC DOCUMENT", ready at approval.
- Data is validated as in live (11-digit NIN, 6-digit postcode, adults). Use made-up people.

## Gotchas

- `paid` is not editable; only `draft`, `verification_failed`, `queried`.
- Only the first submit charges; resubmits after `verification_failed` or a query are free. A rejection is not refunded.
- `postcode` is optional on the business address but required (6 digits) on each proprietor's address, as is `lga`.
- Wrong-looking documents aren't caught by FastCAC; CAC queries them (`document_unclear`).
- `payment.failed` exists in the event list but is never sent; a short wallet answers `402` instead.

Docs: https://docs.fastcac.com/guides/lifecycle · /guides/registrations · /guides/documents ·
/guides/queries · /guides/certificates · /guides/name-checks · /guides/nin-lookups · /test-mode ·
/webhooks/events
