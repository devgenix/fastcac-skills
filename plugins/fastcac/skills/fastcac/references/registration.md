# Registering a Business Name, step by step

Every tool here needs the person's FastCAC account, except `start_nin_request` and
`get_nin_request_status`. The app asks them to sign in the first time you call one.

## Steps that happen on FastCAC

Some tools don't do the thing themselves. They return:

```json
{
  "status": "action_required",
  "action_id": "...",
  "action_kind": "pay",
  "title": "...",
  "url": "<a FastCAC link>",
  "next_step": "Nothing is done yet: the person has to pay on FastCAC. ..."
}
```

- Nothing has happened yet. Don't say it's done.
- Give them the `url` as a link with one sentence on what they'll do there ("Tap this to pay on
  FastCAC"). Some apps open it for them; give the link anyway if they need it again.
- Then call `get_action_status` with the `action_id`: `pending` (not done yet; don't make them
  repeat it), `started` (doing it now), `completed`, `failed` (tell them; they can try again from
  the same page), `cancelled` (nothing changed), `expired` (make the call again if they still want
  it).

This covers paying (`request_payment`, `request_credit_purchase`, `request_subscription`),
uploading documents (`request_documents`), entering the NIN (`request_nin`), verifying WhatsApp
(`request_whatsapp_verification`), and changes that need their OK on FastCAC (`delete_business`,
`create_share_link` when it replaces a live link, `remove_whatsapp`, `revoke_share_link`, and
`update_registration_fields` on a registration that's already paid or queried).

Tools that only show something (`get_cac_documents`, `show_share_link`, `manage_subscription`,
and `create_share_link` for a new link) come back with a `view_on_fastcac` link instead: the certificate to download, the link to copy or
the billing page is there. Give them that link.

## 1. Name and line of business

Check the name ([name-rules.md](name-rules.md)), then `create_business` with the name they want.
It starts a draft in the workspace they're in.

`create_business` can return:
- `whatsapp_unverified`: a personal registration needs a verified WhatsApp number (it's where
  updates go), and so does every payment (`request_payment` returns the same error). Say so in one sentence and call `request_whatsapp_verification`. Carry on when
  `get_action_status` says `completed`.
- A blocked workspace or draft limit: `get_capabilities` says what's blocking it and which tool
  unblocks it.

People registering businesses for clients work in the Business workspace, which needs the Business
subscription (`get_subscription`, `request_subscription`). There, the NIN, photo and signature are
the client's: say "your client's NIN", not "your NIN".

## 2. The form

`update_registration_fields` with what they tell you. Only these fields can be saved:
the business name, line of business (an exact CAC entry), phone (digits), email, state, LGA, city,
street number, street address, postcode, and the business's own state, city, street number,
address and email.

Order: business name and line of business, then the NIN (step 3), then phone, email and home
address, then the business address (offer to reuse their home address). Ask a few related things
at a time, in plain language. It returns what saved, what was invalid, and what's still missing.

## 3. The NIN

`request_nin`. They type their NIN on FastCAC, never in the chat. Their name, date of birth and
gender are filled from NIMC's record: don't ask for them. If they paste a NIN or card number into
the chat anyway, tell them not to and use `request_nin` instead.

If they don't know their NIN, or need a NIN slip outside a registration: `start_nin_request`
(`kind: "phone"` to find the NIN with the phone number they gave NIMC, `kind: "slip"` for the
slip). Give them the link: it works once, for 2 hours. They do a face check and pay on
fastcac.com; the result goes only to them, never to you.
Follow it with `get_nin_request_status`. For the price, read `products` from that result or call
`get_prices`. NIMC's *346# code costs money and is often unavailable, so don't send them there
first. FastCAC isn't part of NIMC.

## 4. Payment

Call `get_registration_state` first. Once the form is complete, `request_payment`. If they have a
discount code, `apply_discount_code` before that. The amount comes from FastCAC's pricing; you
never see card details or the checkout. Payments go through Bachs.

Never say it's paid until `get_action_status` says `completed` or `get_registration_state` shows
it paid. If they say they paid and it doesn't show, say kindly that you don't see it yet and send
them to FastCAC's team if they were charged.

## 5. Documents

CAC needs a passport photo, the NIN slip (the only ID CAC's filing system accepts) and a
signature, plus a supporting document (the certificate or approval) when the name check came back
`caveat`. `request_documents` asks for every one still missing or failed; pass `document_types`
(`passport`, `means_of_id` for the NIN slip, `signature`, `supporting_doc`) to ask for specific
ones again. Files never pass through you.

For their own business (the Personal workspace), documents come after paying: before that,
`request_documents` returns `not_paid`, so offer `request_payment`. In the Business workspace they
can upload first.

Once their NIN is confirmed, offer the ways that need nothing from them, and call a tool only after
they agree:
- Passport photo: `use_nin_photo_for_passport` (the photo on their NIN record).
- NIN slip: `fetch_nin_slip`, free. If NIMC has no slip on record, they upload a photo of theirs.
- Signature: drawn on the pad on FastCAC's upload page, or a photo of one signed on white paper.

`get_registration_state` shows each document's status and what's wrong with a failed one. Explain
it kindly and ask for just that one again. `recheck_documents` re-runs checks on stuck documents
(not failed ones).

## 6. Filing and after

Once paid and every document is verified, FastCAC files with CAC automatically; there's no submit
step. Most certificates come within 24 hours. If CAC's automated checks pass with a conditional
approval, a CAC officer confirms the record, usually within 48 working hours (CAC doesn't review on
weekends). It's usually approved from there, but the officer can still query or decline it.

- `refresh_cac_status` asks CAC for news on a submitted or queried registration.
- `get_cac_documents` gives the certificate and status report once approved.
- `create_share_link` / `show_share_link` give a status link anyone can open to follow progress.

## Fix before filing, and CAC queries

`get_registration_state` can show:
- `fix_before_filing`: things FastCAC's staff or CAC need changed before filing.
- `query`: CAC sent it back. `query.cac_reason` is CAC's overall reason and `query.items` the
  flagged items, each with `label`, `cac_comment`, `fix_path` and `resolved`.

For each item, say what and why in plain words (translate CAC's terse wording; don't just repeat
it). A form detail: get the new value and `update_registration_fields` (for a refused name, help
them pick one with `check_business_name_compliance` first). A document: `request_documents` with
just that type. Anything else: send them to FastCAC with the item's `fix_path`.

Once everything is fixed, FastCAC re-files automatically with no extra payment. If `refiling_now`
is true, it's already happening. Never promise CAC will accept the fix.

## Deleting

`delete_business` deletes a business only if it hasn't been paid for; it needs their OK on
FastCAC. Call it only when they ask to delete, not when they ask whether they can. A paid business,
a payment, a CAC filing or their account can't be deleted this way: say so and offer FastCAC's
team. A wrong document is replaced, not deleted (`request_documents` for that one).
