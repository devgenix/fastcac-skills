# CAC's naming rules

FastCAC's name tools apply these before CAC's own engine sees a name. You don't have to enforce
them yourself (`check_business_name_compliance` does), but knowing them lets you explain a
refusal and avoid suggesting a name that will fail.

## The order to check a name

1. `check_name_availability` with the name. It searches CAC's whole register (business names,
   companies, LLPs, trustees), active and inactive.
   - `exact_match` set: say plainly "X is already registered".
   - `exact_match` null: say nothing is registered under that exact name, then lead with the
     closest `close_matches`. A close match is often the same name with another ending (ADAEZE
     ENTERPRISES for "Adaeze Ventures"), and CAC treats that as a clash.
2. Find their line of business (`search_line_of_business`, below).
3. `check_business_name_compliance` with the name and that line of business. It returns:
   - `tone`: `ok`, `caveat` (CAC wants a certificate or approval for it, uploaded later as a
     supporting document), or `blocked`
   - `message`: the reason in plain words; give it to them
   - `alternatives`: names to offer, each already passing the rules; offer them exactly as written
   - `similar_names`, `most_similar_name`, `similarity_score`: the registered names it's close to

CAC makes the final call. A name that passes FastCAC's checks can still be refused by CAC.

## Business Name rules

- **More than one word.** A single word is refused.
- **It must end with one of these endings:** Ventures, Enterprises, Services, Concepts, Hub,
  Fashion, Beauty or Biz. FastCAC's check refuses any other ending. "Kd Events" is refused; "Kd Events Ventures" isn't.
- **No company words.** Limited, Ltd, Inc, Company, Group, Holding, Incorporated and similar
  belong to companies, not business names.
- **Restricted words, anywhere in the name.** CAC refuses these in any position, including when
  they're the owner's own first name ("Fortune Okafor Ventures", "Trust Ade Enterprises"). There's
  no exemption for given names. Singular and plural both count ("fund"/"funds").

### Restricted words (need a regulator's or the government's consent)

bank, insurance, assurance, trust, finance, fund, investment, federal, national, state,
government, municipal, engineering, architects, surveyors, medical, pharmaceutical, legal, law,
chartered, audit, ledger, tax, fiscal, treasury, pay, crypto, fortune, wealth, money, cash,
cooperative, movement, registry

### Other words CAC refuses in a business name

wealth, company, empire, investment, finance, force, state, national, federal, cooperative,
limited, betting, sport, construction, mining, quarry, exploration, mosque, church, institute,
cash, monie, money, foundation, bank, center, centre, lmt, ltd, inc, holding, group, consortium,
funds, exchange, moni, incorporated, academy, school, apostolic, community

These lists are FastCAC's own checks and may grow. Don't
promise a name is fine because a word isn't listed here; run the check.

## Company names (Ltd)

A company name must end in Limited or Ltd (a private company) or PLC (a public one), and the
restricted (consent) words apply too. The business-name-only list doesn't (a company name can say
Limited, Group or Company). A name ending in Limited/Ltd/PLC is checked as a company. People
can't register a company themselves on FastCAC yet; FastCAC's team does it (see SKILL.md, "What
FastCAC's team does by hand").

## Lines of business

CAC accepts only entries from its fixed list. A registration must use one of them, word for word.

- `search_line_of_business` with what the business does in their words. It returns `matches` and
  `exact`.
- `exact: true`: the first match is theirs. Confirm it with them.
- `exact: false`: say plainly that CAC has no option for exactly that, name the closest entry as
  written (and one or two others if they fit), and ask which is right.
- `list_nature_of_business` browses the list by category when they want to see the options.
- `check_business_name_compliance` returns `line_of_business_invalid` with `choices` when the line
  of business isn't an entry. Show the choices and ask. It isn't a failure, and you never pick for
  them.

## Suggesting names

`generate_business_names` with their idea (and line of business if you have it). Each name comes
back checked against CAC's register with `availability`: `likely_available`,
`similar_names_exist`, or `unchecked` (CAC's search was busy). Still run
`check_business_name_compliance` on the one they choose.
