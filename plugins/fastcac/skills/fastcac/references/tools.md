# FastCAC's MCP tools

The server is `https://api.fastcac.com/mcp`. If the server's tool list
differs from this page, the server is right about which tools exist and what they take. Its
descriptions still mention buttons and system events from FastCAC's own chat: for you, those are
`action_required` or `view_on_fastcac` links (see registration.md).

**Free** tools work without an account, within per-visitor limits. **Account** tools make the app
ask the person to sign in to FastCAC the first time one is called.

## Names and CAC's register (free)

| Tool | What it does | Notes |
|---|---|---|
| `check_name_availability` | Searches CAC's register for the name and close matches | Lead with `exact_match`; null means nothing under that exact name |
| `check_business_name_compliance` | CAC's verdict on a name | Needs `proposed_name` and `line_of_business` (an exact CAC entry). Returns `tone`, `message`, `alternatives` |
| `generate_business_names` | Name ideas for a business idea | Each comes with `availability` |
| `search_line_of_business` | Finds CAC's exact line-of-business entries | `exact` false means ask which fits |
| `list_nature_of_business` | Browses CAC's categories and their entries | For "what are my options?" |
| `search_cac_register` | Looks up a registered business by name or RC/BN number | Lists what exists; not an availability check |
| `get_cac_tax_id` | The Tax ID (TIN) CAC shows for a registered business | Takes `company_id`, `rc_number`, `classification_id` from `search_cac_register` |

## Fees, prices and knowledge (free)

| Tool | What it does |
|---|---|
| `get_prices` | FastCAC's own prices, live: registration, the Agent plan, annual returns, credit packs, NIN tools, the developer API. Use it every time |
| `calculate_cac_fees` | CAC's government fees for a business name, a company (by share capital) or an NGO |
| `check_annual_returns` | Which annual returns a business name owes, from its registration date |
| `search_knowledge_base` | FastCAC's articles and policies: refunds, timelines, documents, CAC procedures, CAMA 2020 |

## NIN tools (free to start, paid on fastcac.com)

| Tool | What it does |
|---|---|
| `start_nin_request` | `kind: "phone"` finds a NIN from the phone number given to NIMC; `kind: "slip"` gets the NIN slip. Returns a link for the person and a `request_handle` for you |
| `get_nin_request_status` | Where it stands, `next_step`, and `products` with naira prices. Never the NIN itself |

## The person's businesses (account)

| Tool | What it does |
|---|---|
| `list_my_businesses` | Their businesses, each with its stage and next action. Each business's `id` is the `business_id` other tools take |
| `get_registration_state` | One registration in detail: status, missing fields, documents, payment, NIN confirmed, CAC's query, `fix_before_filing`. Call it before saying anything about a registration |
| `create_business` | Starts a draft registration with the name |
| `update_registration_fields` | Saves form fields (name, line of business, phone, email, addresses) |
| `rename_business` | Renames the business in their FastCAC dashboard only (not once CAC approved it). To change the name filed with CAC, save `proposedOption1` with `update_registration_fields` |
| `request_nin` | The person enters their NIN on FastCAC |
| `request_documents` | The person uploads documents on FastCAC (their own business: after paying) |
| `use_nin_photo_for_passport` | Uses the NIN record's photo as the passport photo (after they agree) |
| `fetch_nin_slip` | Fetches their NIN slip from NIMC's record, free (after they agree) |
| `recheck_documents` | Re-runs checks on stuck documents |
| `refresh_cac_status` | Asks CAC for news on a submitted or queried registration |
| `get_cac_documents` | The certificate and status report, once approved |
| `delete_business` | Deletes an unpaid business (they confirm on FastCAC) |
| `list_share_links`, `show_share_link`, `create_share_link`, `revoke_share_link`, `set_share_review` | Status links anyone can open, and a details form link for a client |

## Payments and the account (account)

| Tool | What it does |
|---|---|
| `request_payment` | The person pays for a registration on FastCAC |
| `apply_discount_code`, `remove_discount_code` | A discount code on an unpaid registration |
| `get_capabilities` | What they can do now and what's blocking it (workspace, subscription, credits, WhatsApp, drafts) |
| `get_credits`, `request_credit_purchase` | Credit balance and packs (Agent subscribers) |
| `get_subscription`, `request_subscription`, `manage_subscription` | The Agent subscription for people registering for clients |
| `request_whatsapp_verification`, `remove_whatsapp` | WhatsApp, needed to start a personal registration and before any payment |
| `switch_workspace` | Personal (their own businesses) or Agent (clients' businesses) |
| `update_profile` | Their account name (not a registration's proprietor name) |
| `get_notification_preferences`, `update_notification_preferences`, `list_notifications`, `mark_notifications_read` | Notifications |
| `get_action_status` | Whether the person finished a step you gave them a FastCAC link for |

## Not available here

Annual return filing isn't on the MCP server yet; FastCAC's team files annual returns (see
SKILL.md). `check_annual_returns` still works out what's owed.

## Common orders

- **"Is this name OK?"** `check_name_availability` → `search_line_of_business` → (ask) →
  `check_business_name_compliance`.
- **"Register my business"** the above → `create_business` → `update_registration_fields` (line
  of business) → `request_nin` → `update_registration_fields` (contact and addresses) →
  `get_registration_state` → `request_payment` →
  `get_action_status` → `request_documents` (or the NIN photo and slip tools) →
  `get_registration_state`.
- **"Where's my registration?"** `list_my_businesses` → `get_registration_state` →
  `refresh_cac_status` if it's with CAC.
- **"Look up this business and its TIN"** `search_cac_register` → `get_cac_tax_id`.
