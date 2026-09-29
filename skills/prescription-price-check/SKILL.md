---
name: prescription-price-check
description: "Check if a US prescription price is fair and what to ask at the counter, via CounterScript: what pharmacies pay, a fair cash range, FDA generic equivalents, Medicare negotiated prices. Triggers: \"the pharmacy wants $300 for Januvia, is that fair\", \"what should atorvastatin cost without insurance\", \"is there a generic for Lipitor\". Not medical, insurance, or legal advice; never about a person."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Prescription price check

Tell someone whether a US prescription price looks fair, and what to ask at the counter, using CounterScript (rx.agentlookups.ai). It republishes federal data: what pharmacies pay to buy each drug (the weekly CMS NADAC survey), a fair cash-price estimate built on it, FDA generic equivalents, and Medicare negotiated prices. Free during beta, no account or auth needed, read-only.

The person is often at the counter or holding a receipt. Lead with the fair cash range for their quantity (when the response gives one) and how their price compares, then what they can say. Put caveats inline.

## When to use

Use it when someone asks about the price of a US prescription drug, for example:

- "is $80 for my prescription a rip-off"
- "the pharmacy wants $300 for Januvia, is that fair"
- "how much should atorvastatin cost without insurance"
- "what do pharmacies pay for sertraline"
- "is there a generic for Lipitor"
- "Medicare negotiated price for Eliquis"
- "what do I say to get a lower price"

US only, and only drugs in the weekly NADAC file. A benchmark is not a price anyone owes; a no_match never means a drug lacks a fair price.

Not for (details in the next section): medical advice, dosing, side effects, or switching, stopping, or substituting a drug (prescriber and pharmacist); safety or quality ratings ("is the generic as good", "is this pharmacy safe"); legal advice (is a price illegal); insurance coverage or a Medicare enrollee's copay; pharmacy, coupon, or discount-card picks; non-US prices; anything about a person.

## Never use this skill for

- **Medical advice.** Not dosing, side effects, interactions, or whether a drug suits someone. Never suggest switching, stopping, splitting, or substituting a medication, including to a listed generic; that is for the prescriber, with the pharmacist. The service's own line (methodology page): "We never suggest switching; we show what the record says exists."
- **Safety or quality ratings.** Never rate a drug, a generic, or a pharmacy as safe, good, or better. A `generic_alternatives` entry is an FDA equivalence record, not a quality score; relay the service's line about it word for word: "Generic alternatives listed are FDA therapeutic-equivalence matches for the same ingredient, strength, and form; only your prescriber can decide what you should take."
- **Pharmacy, coupon, or discount-card picks.** The methodology page lists what the service deliberately does not do: "No coupons, no pharmacy price comparisons (we do not scrape retail prices), no advice." Add none of your own (one attributed exception under "At the counter").
- **Plan-specific insurance questions**: coverage, prior authorization, formulary tiers, the copay on someone's card, what a Medicare Part D enrollee pays. The plan and the pharmacist know that; this data does not.
- **Legal advice**: whether a price is illegal or gouging, or whether to file a complaint or sue. Per the `terms` field on every response: "NOT medical, insurance, or legal advice".
- **Accusing a pharmacy.** The service says "Pharmacies may lawfully charge more or less." Report the gap as a number, never as "rip-off" or "overcharged", even when the user uses those words.
- **Anything about a person.** This looks up drugs, never people. Per the shared terms (https://agentlookups.ai/terms/): "This is not a consumer report under the Fair Credit Reporting Act or any state consumer-reporting law."

## The call

One tool. Pass the drug name and strength: `atorvastatin 20 mg`. Do not copy the whole label line. As observed, each query word must match the start of a word in the NADAC description, and the index spells forms and salts short (TAB, CAP, SOD, SUCC, MAG), so label wording often misses: `amlodipine 5 mg tablet` returned `no_match`. If a form word helps, use its short start (`tab`, `cap`).

| Want | REST | MCP (`https://rx.agentlookups.ai/mcp`) |
|---|---|---|
| Benchmarks for one drug | `GET https://rx.agentlookups.ai/v1/price?q=<name+strength>` | tool `drug_price` (title "Look up a US drug's fair cash price", read-only), one argument `query` (string, required) |
| What the index covers | `GET https://rx.agentlookups.ai/v1/coverage` | none |

MCP is Streamable HTTP, no auth; a plain POST of `tools/list` or `tools/call` worked without an initialize step. The tool returns the same JSON as REST, as text content and as `structuredContent`.

For the current snapshot date and size, check `GET /v1/coverage` rather than trusting a figure here; on 2026-09-28 it reported `as_of` 2026-09-23 and 5,942 concepts. CMS publishes NADAC weekly (Wednesdays); the service checks daily and reindexes when a new file appears.

Pace the calls. The service says "rate limits apply", and one answer can take three or four calls (candidates, a retry, the match). On 2026-09-28 about 30 back-to-back calls drew an HTTP 429 "Too Many Requests" HTML page instead of JSON; the same day, calls two seconds apart did not. Space calls out; on a 429, wait a few seconds and retry once before telling the user anything about the drug.

## Workflow

1. **Get the label facts**: drug name, strength, form (tablet, capsule, solution, pen), and how many units are in the fill.
2. **Call** `drug_price` or `/v1/price` with name and strength. The response is a union on `result_type`: `match`, `candidates`, or `no_match`.
3. **Handle the result.**
   - `match`: first confirm that `concept.description` names the same ingredient or ingredients, strength, and form as the label. The service can match a different drug (see "A match for the wrong drug" below). If it names a combination, another strength, or another form, do not answer from it: retry with name and strength only and pick from the candidates. Once it checks out, read it (next section) and answer.
   - `candidates`: the query fits several drugs, strengths, or forms. The list is for choosing only. Show the `description` lines, ask which one is on the label, then call again with that description as the query (`atorvastatin 20 mg tablet`) and answer from the `match`; a query equal to a candidate's description returns that drug as a match. Quote no price from the list: its entries carry no `unit` and the response carries no date, and one list can mix per-ML pens with per-tablet entries (`q=ozempic` did). A slug does not work as a query: `q=synthroid-50-mcg-tablet` returned `no_match`, while `q=synthroid 50 mcg tablet` matched. The list shows at most 12 entries, generics first, then alphabetical. `candidates_total` says how many drugs matched, and when that is more than 12, `message` says the list was cut, for example: "The list was cut: 12 of 29 matching drugs are shown (generics first, then alphabetical). Adding a strength or form to the query narrows the list." As of 2026-09-28, `q=metformin` gave that message and listed 11 combination products and one extended-release form, no plain metformin tablet, while `q=metformin 500 mg` (12 of 12) listed "METFORMIN HCL 500 MG TABLET". If the user's drug is not in the list, add the strength or form and call again; never conclude it is missing.
   - `no_match`: usually a word the index spells another way, not a missing drug. Drop form and salt words (tablet, capsule, sodium, potassium, succinate, magnesium, calcium) and retry with name and strength before saying anything about coverage. Then check spelling: the index has no spelling tolerance, but a `no_match` can carry `did_you_mean`, a list of close drug names (`lipiter 20 mg` gave `["lipitor"]`, `atorvastatine` gave `["atorvastatin"]`). Many drug names look alike, so never swap one in silently: check it against the label, ask the user, and only then call again with that name and the strength. Try the generic name too. Only after those retries say the index does not carry it, and relay the `message`'s caveat: "This does NOT mean the drug does not exist or has no fair price: NADAC covers drugs with sufficient pharmacy survey data, and some brands, OTC products, and new drugs are absent." Prefer name and strength over the message's advice to try the name alone: a name alone matches every strength and form, but the list shows only 12 (`q=amlodipine` showed 12 of 37, all combination products), so it costs another call.
4. **Answer, then give the counter scripts** (see "At the counter").

## Worked examples

All run live on 2026-09-28 against the NADAC snapshot dated 2026-09-23 and the FDA Orange Book file dated 2026-09-11. Numbers move weekly; the shapes should not.

**A generic, exact match**

```
GET https://rx.agentlookups.ai/v1/price?q=atorvastatin+20+mg
```

`match`: `concept.description` "ATORVASTATIN 20 MG TABLET", `generic: true`; `nadac.per_unit_usd` 0.02792, `unit` "EA"; `fair_cash_floor` `{"qty": 30, "low_usd": 9.84, "high_usd": 13.84}`. No generic or Medicare block.

**Label wording misses; name and strength finds it**

```
GET https://rx.agentlookups.ai/v1/price?q=amlodipine+5+mg+tablet
GET https://rx.agentlookups.ai/v1/price?q=amlodipine+5+mg
```

The first returned `no_match`, because the index spells this form "TAB". The second returned `candidates` with `candidates_total` 23 and the "The list was cut: 12 of 23 ..." message; "AMLODIPINE BESYLATE 5 MG TAB" was among the 12, and `q=amlodipine+besylate+5+mg+tab` then matched it. The same held for other common generics: `metoprolol+succinate+er+50+mg+tablet` missed, while `metoprolol+50+mg` listed "METOPROLOL SUCC ER 50 MG TAB" (4 of 4); `rosuvastatin+10+mg+tablet` missed, while `rosuvastatin+10+mg` matched "ROSUVASTATIN CALCIUM 10 MG TAB".

**A misspelling**

```
GET https://rx.agentlookups.ai/v1/price?q=lipiter+20+mg
```

`no_match` with `did_you_mean` `["lipitor"]`. After the user confirmed the label, `q=lipitor+20+mg` matched "LIPITOR 20 MG TABLET", with "ATORVASTATIN 20 MG TABLET" in `generic_alternatives`.

**A match for the wrong drug**

```
GET https://rx.agentlookups.ai/v1/price?q=hydrochlorothiazide+25+mg+tablet
```

`match`, but for "ENALAPRIL-HYDROCHLOROTHIAZIDE 10-25 MG TABLET", a combination product: "25" matched "10-25", and "tablet" ruled out the plain entry, which the index spells "HYDROCHLOROTHIAZIDE 25 MG TAB". Do not answer from it. `q=hydrochlorothiazide+25+mg` returned 12 of 23 candidates with the plain tablet among them, and `q=hydrochlorothiazide+25+mg+tab` matched it.

**A brand with a generic and a Medicare price**

```
GET https://rx.agentlookups.ai/v1/price?q=januvia+100+mg
```

```json
{"name": "drug_price", "arguments": {"query": "januvia 100 mg"}}
```

`match`, "JANUVIA 100 MG TABLET", `generic: false`: $10.55179 per tablet, fair cash $325.55 to $329.55 for 30. `generic_alternatives` holds one entry, "SITAGLIPTIN PHOSPHATE 100 MG TABLET", $3.77084 per tablet, fair cash $122.13 to $126.13 for 30, with its own `human_page`. `medicare_negotiated_prices` holds `{"drug": "JANUVIA", "price_30d": 113, "year": 2026, "note": "Medicare Part D negotiated price, effective 2026-01-01"}` plus a CMS `source_url`.

**A brand with no linked generic**

```
GET https://rx.agentlookups.ai/v1/price?q=diovan+160+mg+tablet
```

`match`, "DIOVAN 160 MG TABLET", no `generic_alternatives`, and a `generic_equivalence` block: `status` "listed_no_price", `ingredient` "valsartan", `index_generics` `["VALSARTAN 160 MG TABLET"]`, `as_of` "2026-09-11", and a `statement` that opens "The FDA Orange Book file we hold (dated 2026-09-11) lists a therapeutically equivalent generic (valsartan) for this product." `q=VALSARTAN 160 MG TABLET` then matched, $0.10604 per tablet, fair cash $12.18 to $16.18 for 30. `norvasc 5 mg tablet` named "AMLODIPINE BESYLATE 5 MG TAB" the same way; `jardiance 10 mg` gave "not_listed"; `synthroid 50 mcg tablet` gave "no_orange_book_match".

**Name only: candidates**

```
GET https://rx.agentlookups.ai/v1/price?q=atorvastatin
```

```json
{"name": "drug_price", "arguments": {"query": "atorvastatin"}}
```

`candidates`, `candidates_total` 4, no `message`: four entries (10, 20, 40, 80 mg tablets), each with `description`, `generic`, `per_unit_usd`, `fair_cash_floor`, and `human_page`, but no `unit`. The response has no `nadac` or `provenance` block, so no date.

**No fair cash range: liquids, inhalers, patches**

```
GET https://rx.agentlookups.ai/v1/price?q=amoxicillin+400+mg%2F5+ml
GET https://rx.agentlookups.ai/v1/price?q=estradiol+0.05+mg+patch+(1%2Fwk)
```

The first: `match`, "AMOXICILLIN 400 MG/5 ML SUSP", $0.03963 per `unit` "ML", and no `fair_cash_floor`. An inhaler (unit "GM") and an insulin vial (unit "ML") came back the same way. The second: "ESTRADIOL 0.05 MG PATCH (1/WK)", $11.28267 per `unit` "EA", and still no `fair_cash_floor`; a nitroglycerin patch, a fluticasone Diskus, and an Eliquis starter pack (all EA) did too. The drug pages say why: "Package sizes vary, so we do not guess a fill quantity for this form."

**Not in the index**

```
GET https://rx.agentlookups.ai/v1/price?q=zzqxnotadrug
```

`no_match`, with a `message` that opens "No drug matching this query is in our NADAC index." and says "some brands, OTC products, and new drugs are absent". This result carries `terms` and `dispute_url` but no `honesty` array and no `did_you_mean`.

## Reading a match

- **`nadac`**: `per_unit_usd` per `unit` (EA is one of whatever the form is: a tablet, capsule, patch, or pack; also ML, GM), with `as_of`. The national average price pharmacies paid; several NDCs sharing a description are combined by median (`concept.ndc_count`).
- **`fair_cash_floor`**: the fair cash estimate for 30 units. The API always prices 30 (a `qty` parameter is ignored). Only when this field is present, for another fill size use the service's own formula from its methodology and drug-page code: per_unit_usd times quantity, plus $9 for the low end and $13 for the high end, rounded to cents. Atorvastatin 20 mg, 90 tablets: $2.51 acquisition, so $11.51 to $15.51. Say you worked it out from the service's figure.
- **No `fair_cash_floor`**, whatever the unit (ML, GM, or an EA patch, inhaler, or pack): no range, and no repricing formula. Give the per-unit acquisition cost, times the amount on the label if useful, and stop there, as the page does: "Multiply it by the amount in your fill to estimate what the pharmacy paid for the whole fill."
- **Comparing a quoted price**: only against a range. Say whether it is below, within, or above the range, in the site's words for the last case: "about $X more than the top of the fair cash range for this quantity".
- **`generic_alternatives`**: FDA Orange Book therapeutic equivalents, same ingredient, strength, and form, each priced. Say "an FDA-listed generic equivalent exists; ask your prescriber or pharmacist", never "switch to it" and never "it is just as good". A generic drug's own match carries neither this block nor `generic_equivalence`.
- **`generic_equivalence`**: on a brand-name match with no `generic_alternatives`, a dated statement of what the FDA Orange Book file shows. Its `as_of` is the Orange Book file date, not the NADAC date. Relay its `statement` with that source and date, and read `status`:
  - `not_listed`: the Orange Book file lists no therapeutically equivalent generic for this product. For `jardiance 10 mg`: "No FDA therapeutically equivalent generic is listed for this product in the FDA Orange Book file we hold (dated 2026-09-11)."
  - `listed_no_price`: it lists one, named in `ingredient`, but the match linked no price. Each name in `index_generics`, when present, can be searched as is; price it and frame it like a `generic_alternatives` entry. Without `index_generics`, relay the statement alone; for `eliquis 5 mg tablet` it ends "A listing does not show that the generic is sold in pharmacies now."
  - `otc_generic_listed`: an OTC brand with a current OTC generic, named in `ingredient`. The FDA does not rate OTC products for therapeutic equivalence, so never call that generic an equivalent. `index_generics` names can be searched as is.
  - `no_orange_book_match` or `not_checked`: says nothing either way about generics. For `novolog 100 unit/ml vial` the statement adds "The Orange Book does not cover biologics such as insulins." Send the user to the pharmacist or prescriber.
- **`medicare_negotiated_prices`**: the negotiated price for a 30-day supply (`price_30d`), with year and CMS source. The methodology page: "They apply to Medicare Part D coverage of the listed drugs." Relay it to someone on Part D; never present it as a cash price anyone can ask for. It is not what the enrollee pays: their Part D plan sets their copay, so send them to the plan or the pharmacist for that number, and never say "you should pay $X" from this field. Never set it beside `fair_cash_floor` either; one is a 30-day supply, the other 30 units (`q=eliquis+5+mg+tablet` gave $231 and $174.63 to $178.63). It can list related products: `q=eliquis+5+mg+tablet` returned two rows (ELIQUIS and ELIQUIS SPRINKLE, both $231), and `q=novolog+100+unit%2Fml+vial` six (NovoLog and Fiasp names, all $119).
- **`concept.otc: true`** marks an over-the-counter product (`loratadine 10 mg tablet`).

## At the counter

Give the user the drug page link, found at `concept.human_page` in a `match` (not at the top level): for atorvastatin 20 mg, https://rx.agentlookups.ai/drug/atorvastatin-20-mg-tablet. Each `generic_alternatives` entry and each `candidates` entry carries its own `human_page` too. The page prints a pharmacist card with the source and date; when the page shows a fair cash range, it also reprices any quantity from 1 to 10,000 and compares a quoted price.

The drug page's own scripts, verbatim:

- Ask for the cash price: "What would this cost if I paid cash, without my insurance?" The page adds: "A normal question; pharmacists hear it every day."
- If an online pharmacy lists it for less: "Could you match this online price?" The page adds: "Show the listed price on your phone. The pharmacy can say no; many will try to help."

The page also says which path tends to cost less. Generic: "For generics the cash price often beats an insurance copay." Brand: "For a brand-name drug like this one, the insurance copay is often the cheaper path; ask for both numbers and compare."

If the user asks where else to look, the service's explainer (https://rx.agentlookups.ai/why-is-my-prescription-so-expensive/) names one online pharmacy: "Check Cost Plus Drugs (Mark Cuban's online pharmacy). Their published formula is the drug's cost plus a fixed 15% margin, plus flat pharmacy and shipping fees." Relay that attributed to the page, and note the drug pages say "We do not have Cost Plus Drugs' price for this drug in our data." Name no other pharmacy, coupon, or discount card.

## Honesty rules (verbatim; relay them, never override)

From https://rx.agentlookups.ai/llms.txt, fetched 2026-09-28:

- "NADAC is what pharmacies PAY to acquire a drug: a benchmark, not a price anyone owes."
- "Fair-cash figures are estimates (acquisition + $9-13 dispensing fee band)."
- "Insurance copays may be lower; cash usually does not count toward deductibles."
- "Not medical advice; a no_match never means a drug lacks a fair price."

From the `honesty` array on every `match` and `candidates` response, same date:

- "NADAC is the national average price pharmacies PAY to acquire this drug. It is a benchmark, not a price you owe."
- "The fair cash estimate adds a dispensing fee, the pharmacy's professional fee for filling a prescription ($9-13, per state Medicaid surveys), to acquisition cost. Pharmacies may lawfully charge more or less."
- "If you have insurance, your copay may be lower than any cash price, and paying cash usually does not count toward your deductible."
- "Not medical advice. Generic alternatives listed are FDA therapeutic-equivalence matches for the same ingredient, strength, and form; only your prescriber can decide what you should take."

From the `terms` field on every response: "Public federal drug-pricing data (CMS NADAC, FDA Orange Book, CMS negotiated prices), republished with sources and dates. NOT medical, insurance, or legal advice; never change medications without your prescriber. Benchmarks describe what pharmacies pay to acquire drugs, not prices anyone owes."

From `/v1/coverage`: "A no_match is never a statement that a drug does not exist or lacks a fair price."

When a response's wording differs from the above, use the response's.

## Output contract

Every price answer carries all four, taken from the response:

1. **Source**: "CMS NADAC" for acquisition cost and the fair cash range, "FDA Orange Book" for generic equivalents and `generic_equivalence`, "CMS negotiated prices" for Medicare. Not just "CounterScript".
2. **Snapshot date**: `nadac.as_of` (or `provenance.as_of`) from the `match`, stated as "as of <that date>" (2026-09-23 for the snapshot these examples used), never "currently" or "today". A `generic_equivalence` statement carries its own Orange Book `as_of`; give that date with it. A `candidates` or `no_match` response carries no date: quote no price from a candidates list until the follow-up `match`, and date a no_match with `as_of` from `GET https://rx.agentlookups.ai/v1/coverage`.
3. **Coverage caveat**: a benchmark, not a price anyone owes; a copay may be lower; no range when `fair_cash_floor` is absent; the Medicare price is Part D only and is not the enrollee's copay; a no_match, and a `generic_equivalence` of "no_orange_book_match" or "not_checked", each prove nothing.
4. **Link**: the drug page at `concept.human_page` (with the pharmacist card), and `source_url` for a Medicare figure. For a number that looks wrong: `dispute_url` (https://rx.agentlookups.ai/dispute/) or corrections@agentlookups.ai.

## What this cannot answer, and where to send people

- Whether to take, change, or stop a drug, or whether a generic is right: the prescriber, with the pharmacist.
- What a given insurance plan covers or charges, including a Medicare Part D copay: the plan, or the pharmacist, who can price it both ways.
- Whether a price is legal, or grounds for a complaint: not this data; a lawyer or local legal aid.
- A pharmacy's live price: only the pharmacy. The service does not scrape retail prices.
- Drugs the NADAC file lacks (some brands, OTC products, new drugs): the pharmacist, who knows their own cost.
- Prices outside the US: not covered.
