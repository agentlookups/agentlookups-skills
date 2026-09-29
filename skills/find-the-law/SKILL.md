---
name: find-the-law
description: "What does the law say where I live? Quotes US statutes, federal and Virginia rules, and NYC code from GroundRules, with citation, link, and date. For \"what does Texas law say about security deposits\", \"which laws apply at this address\", \"pull up 29 USC 2612\". Not legal advice or whether someone has a case; no court opinions or city codes outside NYC; never a background check on a person."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Find the law

Answer "what does the law say about X where I live?" with the words of the law itself, from GroundRules (https://law.agentlookups.ai). It resolves a US address to every level that governs it (federal, state, county, city), names each level's code sources and whether it holds their text, and searches and returns that text verbatim with citation, source link, and date. In the service's words: "Currently free during beta, with no account required. We do not sell queries." No auth, read-only.

Your job is to find the right section, quote it exactly, label its date, and say plainly what was not searched. You do not tell the user what the law means for their situation.

## When to use

Use it for questions like:

- "What does Texas law say about security deposits?"
- "How long does my landlord have to return my deposit in Maryland?"
- "Which laws apply at this address?"
- "Is there an NYC rule on after-hours construction?"
- "Pull up 29 USC 2612."

Renters, landlords, buyers, and agents all ask these. GroundRules holds the US Code, the CFR, the statutes of all 50 states plus DC and Guam, the Virginia Administrative Code, and the New York City Charter and Administrative Code (check `law_coverage` for today's list).

Not for:

- Legal advice, small-claims strategy, or whether someone has a case or broke the law.
- Court opinions, bills not yet passed, or laws outside the US.
- City or county codes outside NYC, such as zoning or noise rules, and state regulations other than Virginia's. GroundRules names where those live; send the user there.
- Anything about a person.
- Questions a sibling skill owns: permits or doing the work yourself on a home project (`home-project-permits`), a contractor's license (`hire-a-contractor`), and a Maryland property tax assessment or its appeal (`maryland-assessment-appeal`).

An empty search never means no law applies.

## Never use this skill for

- **Legal advice.** The service: "Not legal advice; no individualized guidance." You may restate a quoted sentence in plain words beside the quote. Never say whether the user has a claim, is in the right, broke the law, or what a court would do. For a dispute, send them to a licensed attorney or a legal aid office.
- **Anything about a person.** GroundRules holds codes and regulations, not court files or records about people. The agentlookups.ai terms (https://agentlookups.ai/terms/), which cover GroundRules and its four sibling services, add: "Not a background check. This is not a consumer report under the Fair Credit Reporting Act or any state consumer-reporting law."
- **Ranking, scoring, or summing up the law.** "we never summarize, characterize, or rank law by importance; the calling agent interprets." Quote; do not grade.
- **Court opinions, pending bills, laws outside the US.** Coverage lists codes and regulations only.

## Calls

Use the GroundRules MCP tools first. This plugin's `.mcp.json` connects them (`https://law.agentlookups.ai/mcp`, Streamable HTTP, no auth). They return the law text as structured data, so the words reach you exactly as published. REST is the fallback when the MCP tools are not available. A web-fetch tool may summarize or reword the page it reads, and law text must be quoted verbatim. If you must use REST through such a tool, ask it for the `body` field exactly as written, and tell the user to check the quote at the official link.

| Want | MCP tool (primary) | REST (fallback) |
|---|---|---|
| Every level that governs a place | `law_for_location` `{"address": ...}` or `{"lat": ..., "lon": ...}` | `GET https://law.agentlookups.ai/v1/resolve?address=<street, city, state>` (or `?lat=&lon=`) |
| Search the text | `law_search` `{"q": ..., "jurisdictions": "us,us-tx"}` or `{"q": ..., "address": ...}` | `GET https://law.agentlookups.ai/v1/search?q=<words>&jurisdictions=us,us-tx` |
| One section by id | `law_get_section` `{"id": ...}` | `GET https://law.agentlookups.ai/v1/section/<id>` |
| One section by citation | `law_get_section` `{"citation": ...}` | `GET https://law.agentlookups.ai/v1/lookup?citation=<citation>` |
| Up to 10 sections | `law_get_section` `{"references": [...]}` | `POST https://law.agentlookups.ai/v1/sections` with `{"references": [...]}` |
| What is hosted, its dates, what is stale | `law_coverage` `{}` | `GET https://law.agentlookups.ai/v1/coverage` |

MCP puts the payload under `result.structuredContent.result`; REST puts it under `data`. The fields inside match. A batch returns one item per reference, each with `section` or its own `error`. An MCP error comes back with `isError: true` and the message as text; REST returns `{"error": {"status": ..., "message": ...}}` with the same message. A plain POST of `tools/call` (headers `content-type: application/json`, `accept: application/json, text/event-stream`) works with no `initialize`.

Search options an agent needs: `jurisdictions` takes jurisdiction or source ids from coverage (`us` is the US Code plus the CFR; `us-cfr` alone is the CFR); `sources` `us` is the US Code only; `address` resolves the place and searches its hosted sources; `mode` `keyword` matches words only (use it for exact terms); `include_text` `1` to `3` adds the full text in a `text` field on the top hits (sections use `body`); `limit` defaults to 20. Pass one of `jurisdictions`, `sources`, or `address`: mixing them fails with "pass sources, jurisdictions, or address, not several", or for `jurisdictions` plus `address`, "pass jurisdictions or address, not both" (REST HTTP 400). `search_mode` in the response says how it ran: `citation` (an exact citation matched), `hybrid` (words plus meaning), or `keyword`. If it reports `hybrid-partial`, `keyword-fallback`, or `legacy`, the index is mid-update; retry with other words or `mode` `keyword` before you say nothing matched.

Pace your calls. One answer takes several (resolve, search, one or more section reads), so make them one after another, not in parallel, and read up to 10 sections in one batch rather than one call each. The llms.txt: "No auth is currently required for reasonable rates; automated clients are supported." If a burst gets HTTP 429, wait a few seconds and retry.

For what is hosted, its dates, and what is stale, check `law_coverage`; do not trust a list in this file. As of 2026-09-28 it listed 56 sources: the US Code, the CFR, the statutes of all 50 states plus DC and Guam, the Virginia Administrative Code, and the New York City Charter and Administrative Code; no other state's regulations and no other city or county code. Ten carried a `vintage` label (AR, AZ, GA, HI, IN, KY, MI, MS, OK, TN). The `stale` count moved within that day, from five sources to none.

## Workflow

1. **Get the place.** For state or federal law, the state is enough. County and city law need a point: a street address or `lat`/`lon`. The Census geocoder misses some landmark addresses (`1 Centre St, New York, NY 10007` failed with "the Census geocoder did not match this location; check the address, or pass lat= and lon= instead"); retry with coordinates. Check `matched_address` against what the user gave: on 2026-09-28, `250 Broadway, New York, NY 10007` matched "250 E BROADWAY, NEW YORK, NY, 10002", a different street. When it differs, confirm the address or use coordinates.
2. **Resolve it** (`law_for_location`) and read the stack: the state's `id` (`us-md`; DC is one level, `district`, id `us-dc`), the county and place names, each source's `hosted` flag, and the `honesty` lines. An unincorporated address (a census-designated place, or no place at all) has a county code and no city code. County and place levels have no hosted text except New York City (`id` `us-nyc`).
3. **Check each `honesty` line against the `hosted` flag of the source it names.** As of 2026-09-28 the stacks for Maryland, New York, New Jersey, Massachusetts, Georgia, Pennsylvania, Illinois (Chicago), and Virginia marked the state statutes `hosted: true`, matching `law_coverage`. One line still disagreed: Georgia's stack marks its code `hosted: true` with a `vintage` label, and its `honesty` line says, "The official Official Code of Georgia Annotated is published through a commercial vendor. We do not host or fetch its text; the stack links the official landing page." When a line says a source is not hosted but the stack or coverage marks it hosted, do not relay that line as is. Say GroundRules holds a copy, give its `current_through` (and `vintage` when present), and send the user to the stack's `official_url` for the current official text. Lines about sources marked `hosted: false` (such as New Jersey's Administrative Code) are accurate; relay them.
4. **Search.** Pass the address, or the ids from the stack: `jurisdictions` `us,us-<st>`, plus `us-nyc` in New York City. An address search covers the stack's hosted sources, and its `note` says so ("auto-scoped to the hosted sources of the resolved jurisdiction stack: us, us-md"). On later searches, reuse the ids rather than the address (llms.txt: "Reuse jurisdictions from location resolution on later searches."). If the user names one state in the question and you pass no scope, the service picks that state plus federal law on its own. Always read `searched_sources`: if the state id is not in it, the state's law was not searched, and you must say so or search again.
5. **Pick the section that answers the question.** Read `heading` and `path`, not only the snippet; a top hit can be a nearby section about something else. The service: "Results are starting points for reading. A section can match your question without applying to your situation." Keep the hit's `html_url` and `publisher_url`, then fetch the full section with `law_get_section` and quote the operative words exactly.
   - **Follow the exceptions.** When the quoted words say "Except as provided by", "Subject to", or "Notwithstanding", fetch the section they name (`law_get_section` `references` takes up to 10) and quote its operative words too, or name it as an exception you did not read. Read the rest of a long section for other rules on the same point. Never state a deadline or duty without its stated exceptions.
6. **Label the date on every quote**: the hit's own `current_through`, plus `vintage` when it is not empty and `stale` when it is true (next section).
7. **Name the gaps**: levels with no hosted text, `missing_sources`, and anything not searched. For a local question (zoning, noise, animals, parking, fences, short-term rentals) outside New York City, the answer is usually in a city or county code GroundRules does not hold; a state or federal hit is not that answer.

## Reading the labels

- **`current_through`** is per hit, not per state, and its format varies by publisher (`2026-01-01`, `89th 2nd Called Legislative Session, 2025`, `Public Law 119-111 (09/18/2026)`). Relay it as written. Laws passed after that point are not in the text. The service: "A section's date reflects its source's publication date, edition, or archive date. It is not necessarily the date the law took effect." Never write "in effect since <current_through>".
- **`vintage`** non-empty means a dated snapshot. The llms.txt rule: "vintage-labeled text is NEVER current law, and the label travels adjacent to the text in every response." Put the `vintage` string next to the quote exactly as written, whatever its form, and send the user to the official source for current text. Most are named snapshots (Georgia: "Public.Resource.Org bulk O.C.G.A., Release 86 (2022-11), retrieved 2026-09-17"), but as of 2026-09-28 Arizona, Indiana, and Kentucky carried a bare date or year ("2026-09-15", "2026", "09/16/2026"). Relay those as written too; do not invent a snapshot year. Arkansas, Mississippi, and Tennessee mix two copies, so hits from one state can carry different labels; use each hit's own.
- **`stale: true`** means, per llms.txt, "a source that has fallen behind is marked stale in /v1/coverage and every hit and section from it carries stale:true, adjacent to the text." Say the copy may lag the official text.
- **`hits: []`** means no match in `searched_sources`. **`missing_sources`** names requested ids GroundRules does not host. The llms.txt: "Missing coverage remains distinct from no matching sections." Neither one means no law applies.
- **Links.** A search hit carries `html_url` (GroundRules' readable page, with a "Read at publisher" link), `publisher_url` (the official page when the publisher has one), and `source_url` (what GroundRules read). A section, lookup, or batch response (`law_get_section`, `/v1/section`, `/v1/lookup`, `POST /v1/sections`) carries only `source_url`, often a bulk file. For a bare citation, also run it through `law_search` with the citation as `q`: it returns `search_mode` `citation` and one hit with both links. Or build the page link as `https://law.agentlookups.ai/law/<id>`. When the official link is a whole-title or bulk file rather than a page for the section (`.zip`, `.odt`, `.parquet`, `.xml`, a whole-title `.pdf`), give the stack's `official_url` for that source instead (resolve any public address in the state, such as its capitol, to get it).

## Worked examples

Each ran live on 2026-09-28 through the MCP tools. Texts and dates will move; the shapes should not.

### 1. Texas: resolve, search, quote

```
law_for_location {"address": "301 W 2nd St, Austin, TX 78701"}
```

Austin City Hall. `stack`: federal `us` (US Code and eCFR, both `hosted: true`); state `us-tx` with "Texas Statutes" `hosted: true`, `current_through` "89th 2nd Called Legislative Session, 2025", and "Texas Administrative Code" `hosted: false`; county "Travis County" and place "Austin city", both `hosted: false` with no sources. `honesty` included: "Austin city is an incorporated municipality, but we do not host municipal code text (New York City is the only hosted exception in v1)." and a line naming Municode, American Legal, and eCode360 as where a local code most likely lives.

```
law_search {"q": "security deposit refund", "jurisdictions": "us,us-tx"}
law_get_section {"citation": "Tex. Property Code 92.103"}
```

`searched_sources` `["us","us-cfr","us-tx"]`; the search led with `Tex. Property Code § 92.103`, `html_url` https://law.agentlookups.ai/law/us-tx/tex.-property-code-92.103, `publisher_url` https://statutes.capitol.texas.gov/Docs/PR/htm/PR.92.htm#92.103. The section read returned `Tex. Property Code § 92.103`, heading "OBLIGATION TO REFUND.", `stale: false`. Its body opens: "(a) Except as provided by Section 92.107, the landlord shall refund a security deposit to the tenant on or before the 30th day after the date the tenant surrenders the premises." That names an exception, so fetch it:

```
law_get_section {"references": ["Tex. Property Code 92.107"]}
```

`Tex. Property Code § 92.107`, "TENANT'S FORWARDING ADDRESS.": "(a) The landlord is not obligated to return a tenant's security deposit or give the tenant a written description of damages and charges until the tenant gives the landlord a written statement of the tenant's forwarding address for the purpose of refunding the security deposit." Quote both, with their citations, "current through the 89th 2nd Called Legislative Session, 2025", and the links. Never give the user "30 days" without the forwarding-address condition.

### 2. Maryland: one section, several clocks

`law_search` `{"q": "security deposit returned to tenant", "address": "100 State Circle, Annapolis, MD 21401"}` (the State House) searched `["us","us-cfr","us-md"]`, with the `note` "auto-scoped to the hosted sources of the resolved jurisdiction stack: us, us-md". It led with `Md. Code, Real Property § 8–203` (`current_through` "2026-01-01", empty `heading`). Its full text, from `law_get_section` `{"id": "us-md/md.-code-real-property-8-203"}`, includes: "(e) (1) Within 45 days after the end of the tenancy, the landlord shall return the security deposit to the tenant together with simple interest which has accrued at the daily U.S. Treasury yield curve rate for 1 year, as of the first business day of each year, or 1.5% a year, whichever is greater, less any damages rightfully withheld." The same section has other clocks. Subsection (h) says (e)(1) does not apply to a tenant who was evicted or abandoned the premises, and gives that tenant a 45-day window to demand the deposit by first-class mail. Subsection (k)(3) gives the landlord 30 days after finishing repairs to return any amount withheld on an estimate beyond the actual cost. Quote the rule that fits the user, and name the others.

### 3. Unincorporated area: county code only

`law_for_location` `{"address": "12000 Government Center Pkwy, Fairfax, VA 22035"}` (Fairfax County Government Center) resolved to Fairfax County and "Fair Oaks CDP", with no place level. `honesty`: "This location is not inside an incorporated municipality (Fair Oaks CDP is a census-designated place, a statistical area with no municipal government). The county's code is the local code that applies; there is no city or town code for this address." Tell the user the county code governs, that GroundRules does not hold it, and where it most likely lives.

### 4. New York City: the one hosted local code

`law_for_location` `{"lat": 40.7127, "lon": -74.0059}` gave place `us-nyc` "New York City": the Charter and Administrative Code `hosted: true`, `current_through` "Local Law 2026/147 (enacted September 12, 2026)", `official_url` https://codelibrary.amlegal.com/codes/newyorkcity/latest/overview; the Rules of the City of New York `hosted: false`. `law_search` `{"q": "construction noise permitted hours", "jurisdictions": "us-nyc"}` led with `N.Y.C. Admin. Code § 24-222`, heading "§ 24-222 After hours and weekend limits on construction work." Its `publisher_url` is a bulk `XML.zip`, so give the stack's `official_url` instead. For state and city together, `jurisdictions` `us,us-ny,us-nyc` searched `["us","us-cfr","us-ny","us-nyc"]`.

### 5. A local question with no local text

`law_search` `{"q": "can I keep chickens in my backyard", "jurisdictions": "us,us-tx"}` returned `9 CFR 145.82` and other federal poultry-program rules. Austin's code is not hosted, so these hits do not answer the question. Say that, and point to the city's own code or the city clerk.

### 6. Dated snapshot and stale copy

`law_search` `{"q": "security deposit return", "jurisdictions": "us-ga"}` led with `O.C.G.A. § 44-7-34`, `current_through` "Release 86 (2022-11)", `vintage` "Public.Resource.Org bulk O.C.G.A., Release 86 (2022-11), retrieved 2026-09-17", and a `publisher_url` pointing at an archive.org `.odt` file. Quote it as a 2022 snapshot, not current law. `law_get_section` `{"citation": "California Civil Code 1950.5"}` returned `CIV § 1950.5` and a `source_url` that is the bulk `pubinfo_2025.zip`. Early on 2026-09-28 it carried `current_through` "2026-09-20" and `stale: true`; later that day it gave "2026-09-27" and `stale: false`. Read `stale` on every response; never carry it over from an earlier call or from this file. `law_search` `{"q": "California Civil Code 1950.5"}` returned `search_mode` `citation` and one hit with `html_url` https://law.agentlookups.ai/law/us-ca/civ-1950.5 and a `publisher_url` on leginfo.legislature.ca.gov for the section; give those two links.

### 7. Not hosted, not found, not a citation

- `jurisdictions` `us-pr,us-md-montgomery,us-md` searched `["us-md"]`, listed both others in `missing_sources`, and the `note` ended "Absence from our coverage never means no law applies there."
- A no-match search returned `hits: []` (REST HTTP 200), not an error.
- Citation lookup wants the service's own form. `Tex. Property Code 92.103`, `42 USC 3604`, `24 CFR 100.204`, `29 USC 2612`, and `California Civil Code 1950.5` resolved; `Texas Property Code 92.103` and `Md. Code, Real Property 8-203` failed with "Citation not found. Try search with the state and code name." (REST 404; in a batch, that item's `error`). Fall back to search with the state named.

## Honesty rules (verbatim; relay them, never override them)

From https://law.agentlookups.ai/llms.txt, fetched 2026-09-28:

1. "Verbatim text only: we never summarize, characterize, or rank law by importance; the calling agent interprets."
2. "Every record names its official source; the official source is authoritative, we are not."
3. "Absence is never an answer: when we do not host a jurisdiction's text, the response says exactly that and points at where the text lives when known."
4. "Vintage-labeled text is a dated snapshot, not current law; relay the label with the text."
5. "Not legal advice; no individualized guidance."

The `notice` on every response: "GroundRules: Original legal text. Not legal advice." The beta charter (https://law.agentlookups.ai/charter/): "A missing result is never a claim that no law applies." If a response's wording differs from the above, use the response's. The one exception: a stack `honesty` line that says a source is not hosted when the stack or coverage marks it hosted (workflow step 3).

## Output contract

Every answer carries all five, taken from the response, not from memory:

1. **The words**: the operative sentence or subsection, quoted exactly, with the `citation`. Plain-words restatement, if any, comes after the quote and is marked as yours.
2. **The date**: the hit's `current_through`, plus the `vintage` label or the stale warning when present, and "looked up <today's date>". Never "current law" or "today" on its own.
3. **The source**: the publisher from the stack; the GroundRules page (`html_url` from a search hit, or `https://law.agentlookups.ai/law/<id>`); and the official link (`publisher_url` from a search hit, or `source_url` from a section read, replaced by the stack's `official_url` when it is a whole-title or bulk file).
4. **The gaps**: every level or source that was not searched, by name (for example "Austin's city code and the Texas Administrative Code are not in GroundRules"), any `missing_sources`, and where that text lives when the response says.
5. **The limit**: not legal advice; for a dispute or a deadline that matters, a licensed attorney or a legal aid office. Errors go to corrections@agentlookups.ai (the service's report page, https://law.agentlookups.ai/dispute/: "Send us the page link or citation and a short description of the problem.").

## What this cannot answer, and where to send people

- **City or county code outside New York City**: the place's own site, or the libraries the `honesty` line names (Municode https://library.municode.com/, American Legal https://codelibrary.amlegal.com/, eCode360 https://www.generalcode.com/library/), or the city or county clerk. About half of US municipalities publish no code online.
- **State regulations other than Virginia's**: the stack's `official_url` for that state's regulations.
- **How courts read a law, and whether it applies to the user**: a licensed attorney or legal aid.
- **Bills not yet law**: the state legislature or Congress.
- **Leases, HOA rules, contracts**: the documents themselves.
