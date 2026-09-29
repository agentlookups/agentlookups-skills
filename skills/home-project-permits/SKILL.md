---
name: home-project-permits
description: "Need a permit for a home project, or can I do it myself? Quotes the permit and licensing law GroundRules holds for a US address, verbatim with citation, and checks the contractor's license. Asks like \"do I need a permit to build a deck\", \"can I replace my own water heater\", \"does my contractor need a license\". Not a permit decision, legal or safety advice; never a background check on a person."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Home project permits

For someone planning a deck, fence, shed, water heater, electrical panel, roof, or
remodel who asks "do I need a permit?", "can I do this myself?", or "does my
contractor need a license?". Two read-only services, free during beta, no account
needed:

- **GroundRules** (https://law.agentlookups.ai) turns an address into its state,
  county, and city, and returns law text word for word with citation and
  current-through date.
- **Plumbline** (https://contractors.agentlookups.ai) returns the contractor's
  license record from the state or city board.

## When to use

Any question about whether a home project needs a permit, whether the owner may do
the work, or whether the contractor needs a license for it. Also for buyers,
renters, and agents, even on a listing or a job done in the past. US only. Typical
asks:

- "Do I need a permit to build a deck?"
- "Does my fence need a permit?" / "Do I need a permit for a shed?"
- "Can I replace my own water heater?"
- "Can I do my own electrical or plumbing?"
- "Do I need a permit for a new electrical panel?"
- "Did this addition need a permit?"
- "Does my contractor need a license?"

Not for: a license check alone (use `hire-a-contractor`, which also covers
insurance, deposits, and complaints); other law questions (use `find-the-law`);
a permit decision; legal advice; electrical, gas, or structural safety advice;
contractor ratings; a background check on a person. Only local building and
zoning offices decide if a job needs a permit; a quote is not that decision, and
a missing code or a no-match proves nothing.

## Three facts that shape every answer

Say them plainly:

1. **The local building and zoning offices decide whether a job needs a permit.**
   A quoted section is not a permit decision. A building-code exemption can still
   leave a zoning approval (height, setbacks, location); fences, sheds, decks, and
   additions are the usual cases.
2. **GroundRules holds almost no local code.** On 2026-09-28 `law_coverage` listed
   one, New York City's (`us-nyc`); check it for the current list. Elsewhere it
   names the county and city, holds the state's statutes where `law_coverage` lists
   them (and, for Virginia, the statewide building code in the Virginia
   Administrative Code, `us-va-vac`), and points to where a local code might live.
   About half of US towns publish no code online; then say so and send the user to
   the building department.
3. **A state license exemption is not a permit exemption.** "You may do your own
   plumbing" and "you need no permit" are different rules, often in different codes.

Every example below was run live on 2026-09-28 and returned what is described.

## Never use this skill for

- **A permit decision or legal advice.** GroundRules: "Not legal advice; no
  individualized guidance." Never say "you don't need a permit", "you can do this
  yourself", "this is legal", "this is safe", or "this license covers the job".
  Say what the quoted text says, which chapter it is in, and who decides.
- **Treating absence as an answer.** A search that finds no exemption, a code
  GroundRules does not hold, or hits that miss the job do not mean no permit rule
  exists. Search always returns ranked hits, often off topic, and outside New York
  City the local code was not searched. Plumbline's `findings.permits`: "Absence
  here means nothing."
- **Electrical, gas, or structural safety advice.** Whether a panel swap, gas line,
  or beam change is safe, or how to do it, is for a licensed tradesperson and the
  building inspector. The shared terms (https://agentlookups.ai/terms/): "Not
  advice. Nothing here is legal, financial, medical, or professional advice."
- **Scoring or ranking contractors.** Plumbline: "Results are records, not ratings
  or recommendations." No safety or quality score exists here; do not make one up.
- **A background check on a person.** Check only the business the user is hiring,
  by the license number on the bid. Plumbline's `terms`: "This is NOT a consumer
  report under the FCRA or state consumer-reporting laws, and must not be used for
  decisions about employment, credit, insurance, or housing."

## Calls

Use the MCP tools first. They return the law text as data, and every quote must be
word for word. REST is the fallback; if you read it through a web-fetch tool that
summarizes pages, the law text may come back reworded, so never quote from a
summary: fetch the raw JSON or say you could not get the exact text.

| Need | MCP tool and arguments (first choice) | REST fallback (GET unless noted) |
|---|---|---|
| State, county, city for an address | `law_for_location` `{"address": "..."}` or `{"lat": .., "lon": ..}` | `https://law.agentlookups.ai/v1/resolve?address=<street, city, state>` |
| Find permit sections | `law_search` `{"q": "...", "jurisdictions": "us-va", "limit": 5}`; add `"include_text": 1` (up to 3) for the top hits' full text | `https://law.agentlookups.ai/v1/search?q=<words>&jurisdictions=<ids>` |
| Full text before quoting | `law_get_section` `{"id": "..."}` or `{"citation": "..."}`, or `{"references": [...]}` for up to 10 | `https://law.agentlookups.ai/v1/section/<id>`; by citation: `/v1/lookup?citation=<cite>`; batch: POST `/v1/sections` with `{"references": [...]}` |
| What GroundRules holds, and what is stale | `law_coverage` `{}` | `https://law.agentlookups.ai/v1/coverage` |
| Contractor license (Plumbline) | `check_contractor` `{"license": "...", "jurisdiction": "VA"}` | `https://contractors.agentlookups.ai/v1/check?license=<no>&jurisdiction=US-<ST>` |

MCP endpoints: `https://law.agentlookups.ai/mcp` and
`https://contractors.agentlookups.ai/mcp`, Streamable HTTP, no auth. A bare
`tools/call` POST (headers `content-type: application/json` and
`accept: application/json, text/event-stream`) worked with no `initialize`.
GroundRules MCP results put the payload under `structuredContent.result` (REST puts
it under `data`); a section's text is in `body`, and a search hit fetched with
`include_text` has it in `text`. A `references` batch returns one item per
reference, each with its `section` or its own `error` (an unknown id got "section
not found in hosted sources; we may not host this jurisdiction's text: see coverage
(/v1/coverage or the law_coverage tool)"). Plumbline's `structuredContent` is the
REST JSON.

Plumbline's `jurisdiction` takes a two-letter code (`VA` or `US-VA`) or a full
state name (`Virginia` returned the same match as `VA`). A value it cannot place
returns HTTP 400 `unrecognized_jurisdiction` (over MCP, `isError` with the same
text): the message says the value "is not a US state or a local code this index
uses", lists the accepted forms, and ends "A city goes in the city field." A call
with no license, name, or `entity_id` returns HTTP 400 `missing_query`: "Nothing to look
up: send a license number, a business name, or an entity_id from an earlier result.
jurisdiction and city only narrow those."

Pace GroundRules calls. One answer takes several (a resolve, a search or two, a
section fetch or two), and its llms.txt says "No auth is currently required for
reasonable rates; automated clients are supported." Make them one after another,
not in parallel bursts, and fetch several sections in one `references` call.

## Workflow

1. **Get the street address and the job**: what the work is, and whether the user
   will do it or hire it out. If the geocoder misses ("the Census geocoder did not
   match this location; check the address, or pass lat= and lon= instead", an MCP
   `isError` or a REST 404), retry with `lat` and `lon`.
2. **Resolve the address** with `law_for_location`. Note the state `id` and
   whether there is a `place` (a city or town). A `census_designated_place` or no
   place means only that the address is outside an incorporated city; `honesty`
   then says "The county's code is the local code that applies". In most states the
   county's building department then decides. The resolver has no town or township
   level, so in the twelve town and township states (CT, MA, ME, MI, MN, NH, NJ,
   NY, PA, RI, VT, WI) do not relay "the county decides": there the town or
   township is the local government and most likely runs the building department,
   and some counties have no government at all. Say that, name the county only as a
   fallback, and send the user to confirm with the town or township. On 2026-09-28
   the Edison, NJ municipal complex (`100 Municipal Blvd, Edison, NJ 08817`) and the
   Lexington, MA town office building (`1625 Massachusetts Ave, Lexington, MA
   02420`, "Lexington CDP") both got the county line. Relay the other `honesty`
   lines, but where one contradicts the stack, go by the stack and the hits: for
   New York City, see example A; for Georgia, on 2026-09-28 the Atlanta resolve
   said of the Official Code of Georgia Annotated "We do not host or fetch its
   text" while its stack row read `hosted: true`, and Georgia hits carry the
   `vintage` label "Public.Resource.Org bulk O.C.G.A., Release 86 (2022-11),
   retrieved 2026-09-17". Relay that label with any Georgia quote (step 6).
3. **Search with explicit ids**: the state `id` (`us-tx`), plus `us-nyc` in New
   York City. An `address` scope also works now: it searches the stack's hosted
   sources (on 2026-09-28, Baltimore City Hall at `100 N Holliday St, Baltimore, MD
   21202` searched `["us","us-cfr","us-md"]`). But it always adds federal law, and
   federal permit rules crowd out the building code: `q` "do I need a permit" with
   `address` `2 Lafayette St, New York, NY 10007` searched
   `["us","us-cfr","us-ny","us-nyc"]` and all five hits were CFR sections (National
   Park Service, FDA, and BLM permits). Explicit ids without `us` keep federal hits
   out. With no scope at all, a query that names one state searches that state
   plus federal law, and one that names none searches every hosted source. Read
   `searched_sources` every time.
4. **Query with the job and the word permit** ("permit required fence shed deck",
   "work exempt from permit"); for do-it-yourself questions, the trade plus
   "homeowner" or "property owner". Results are ranked, not filtered: check each
   hit's `path` to confirm it is the building code, not pool or day-care rules.
5. **Fetch the full section** with `law_get_section` and quote the lead-in, the one
   item that fits the job, and every condition or exception that touches it.
   Snippets are fragments.
6. **Label each quote** with `citation` and `current_through` as written. Take
   `current_through`, `vintage`, and `stale` from the hit or section you quote (or
   `law_coverage`), not from memory. If `vintage` is not empty, llms.txt says
   "vintage-labeled text is NEVER current law"; relay the label. If `stale` is
   true, say the copy may lag the official text. Link the official text: the hit's
   `publisher_url` or the section's `source_url`, or the stack's `official_url`
   when that is a bulk file (NYC's is a `.zip`). Say the official text governs.
   `html_url` is an optional readable copy; only search hits carry it, so after a
   fetch with no search, use `https://law.agentlookups.ai/law/<id>`.
7. **Name the gap.** Outside New York City the city or county code was not searched,
   and zoning ordinances are local code too: GroundRules holds none (for New York
   City it holds only the Charter and Administrative Code, not the Zoning
   Resolution). Relay the `honesty` line that names Municode
   (https://library.municode.com/), American Legal
   (https://codelibrary.amlegal.com/), and eCode360
   (https://www.generalcode.com/library/). If the code is in none of them, the town
   may publish none online; send the user to its building and zoning offices.
8. **If hiring**, check the license (example D), then follow `hire-a-contractor` for
   the rest, including who pulls the permit.

## Worked examples

### A. New York City: "can I swap my gas water heater without a permit?"

```
law_for_location  {"address": "2 Lafayette St, New York, NY 10007"}
law_search        {"q": "work exempt from permit", "jurisdictions": "us-ny,us-nyc", "limit": 5, "include_text": 1}
law_get_section   {"references": ["us-nyc/n.y.c.-admin.-code-28-105.4.4", "us-nyc/n.y.c.-admin.-code-28-105.4.2"]}
```

REST fallback: `/v1/resolve?address=2+Lafayette+St,+New+York,+NY+10007`,
`/v1/search?q=work+exempt+from+permit&jurisdictions=us-ny,us-nyc&limit=5`, and
`/v1/section/us-nyc/n.y.c.-admin.-code-28-105.4.4`.

An office building near City Hall. Resolve: place `us-nyc`, "New York City Charter
and Administrative Code" `hosted: true`, `current_through` "Local Law 2026/147
(enacted September 12, 2026)"; "Rules of the City of New York (RCNY)"
`hosted: false`. Its last `honesty` line is generic ("We name county and place
jurisdictions but do not host their code text in v1."); here the Charter and
Administrative Code is hosted and the RCNY is not, so relay it with that
correction. Search (`searched_sources` `["us-ny","us-nyc"]`) led with
`N.Y.C. Admin. Code § 28-105.4`, "§ 28-105.4 Work exempt from permit.", with its
full `text`: the list opens "Unless otherwise indicated, permits shall not be
required for the following:" and ends "11. Other categories of work as described
in department rules, consistent with public safety." Those rules are the RCNY,
which GroundRules does not hold.

§ 28-105.4.4 opens "The following ordinary plumbing work may be performed without a
permit, provided that the licensed plumber performing such work: (i) provides a
monthly report ...". Its item 5: "In buildings classified as residential occupancy
groups occupied by five families or fewer, the replacement of a gas water heater,
gas furnace, or a gas-fired boiler with a capacity of 350,000 BTU (103 kW) or less
where the existing appliance shutoff valve is not moved, provided that the plumber
has inspected the chimney and found it to be in good operational condition." The
same batch returned § 28-105.4.2: "A permit shall not be required for minor
alterations and ordinary repairs."

Then: quote the lead-in and item 5. The text ties the no-permit route to "the
licensed plumber performing such work"; never turn that into "you can do it
yourself". Say the department rules were not searched.

### B. Unincorporated Fairfax County, Virginia: "does my fence or shed need a permit?"

```
law_for_location  {"address": "12000 Government Center Pkwy, Fairfax, VA 22035"}
law_search        {"q": "permit required fence shed deck", "jurisdictions": "us-va", "limit": 5}
law_get_section   {"references": ["us-va/13vac5-63-80", "Va. Code § 15.2-2280"]}
```

REST fallback: `/v1/resolve?address=12000+Government+Center+Pkwy,+Fairfax,+VA+22035`,
`/v1/search?q=permit+required+fence+shed+deck&jurisdictions=us-va&limit=5`,
`/v1/section/us-va/13vac5-63-80`, and
`/v1/lookup?citation=Va.+Code+%C2%A7+15.2-2280`.

The county government center. Resolve: county "Fairfax County",
`census_designated_place` "Fair Oaks CDP", no place level, and the `honesty` line
"... The county's code is the local code that applies; there is no city or town
code for this address." Search (`searched_sources` `["us-va","us-va-vac"]`) led
with `13VAC5-63-80`, "Section 108 Application for permit", `path` including
"Chapter 63. Virginia Uniform Statewide Building Code", `current_through` "2026
Regular Session (effective July 1, 2026)", `stale: false`. Hits 4 and 5 were
public-pool and family-day-home rules.

Its 108.1 says "a permit shall be obtained prior to the commencement of" listed
work, including "Construction or demolition of a building or structure" and
"Installations or alterations involving ... (vi) electric wiring". 108.2 opens:
"Notwithstanding the requirements of Section 108.1, application for a permit and
any related inspections shall not be required for the following; however, this
section shall not be construed to exempt such activities from other applicable
requirements of this code." Items include "2. One story detached structures used as
tool and storage sheds, playhouses, or similar uses, provided the building area does
not exceed 256 square feet (23.78 m2) and the structures are not classified as a
Group F-1 or H occupancy." and "5. Fences of any height unless required for
pedestrian safety as provided for by Section 3306 or used for the barrier for a
swimming pool." An exception: "Application for a permit may be required by the
building official for any items exempted in this section that are located in a
special flood hazard area." 108.4 requires "satisfactory proof to the building
official that the person is duly licensed or certified" or a written statement that
the person is not subject to licensure.

108.2's caveat covers only "this code", the building code. Zoning is separate:
`Va. Code § 15.2-2280`, "Zoning ordinances generally" (`current_through`
"9/28/2026" when fetched on 2026-09-28, `stale: false`), lets any locality regulate
"2. The size, height, area, bulk, location, erection, construction, reconstruction,
alteration, repair, maintenance, razing, or removal of structures;". So a county
zoning rule can reach a fence or shed that 108.2 exempts.

Then: quote the lead-in, the item, and the flood-area exception; say the county's
own code, zoning included, is not in GroundRules and the county's building and
zoning offices decide. The two sections come from different sources with different
dates (`source_id` `us-va-vac` for 13VAC5-63-80, `us-va` for § 15.2-2280), so label
each quote with its own `current_through`.

### C. Austin, Texas: "can I do my own plumbing or electrical?"

```
law_for_location  {"address": "301 W 2nd St, Austin, TX"}
law_search        {"q": "plumbing homeowner homestead exemption license", "jurisdictions": "us-tx", "limit": 8}
law_search        {"q": "electrical work homeowner dwelling exemption", "jurisdictions": "us-tx", "limit": 8}
law_get_section   {"references": ["us-tx/tex.-occupations-code-1301.051", "us-tx/tex.-occupations-code-1305.003"]}
```

City Hall. Resolve: place "Austin city" `hosted: false`; `honesty`: "Austin city is
an incorporated municipality, but we do not host municipal code text (New York City
is the only hosted exception in v1)." The plumbing search led with
`Tex. Occupations Code § 1301.051`, "PLUMBING BY PROPERTY OWNER IN HOMESTEAD.": "A
property owner is not required to be licensed under this chapter to perform plumbing
in the property owner's homestead." The electrical search led with
`Tex. Occupations Code § 1305.003`, whose (a) opens "This chapter does not apply
to:" and lists "(6) work not specifically regulated by a municipal ordinance that
is performed in or on a dwelling by a person who owns and resides in the dwelling;".
Both `current_through` "89th 2nd Called Legislative Session, 2025".

Then: say these are state licensing rules, not permit rules, and § 1301.051 lifts
only the license "under this chapter"; never say "Texas lets you do your own
plumbing". § 1305.003(a)(6) turns on "a municipal ordinance", which means Austin's
code, and GroundRules does not hold it. Send the user to the libraries in `honesty`
and to Austin's building department.

### D. The contractor's license (Plumbline)

```
check_contractor  {"license": "068841", "jurisdiction": "VA"}
```

REST fallback:
`https://contractors.agentlookups.ai/v1/check?license=068841&jurisdiction=US-VA`.
Returned `result_type: "match"` for HOME DEPOT USA. INC (`jurisdiction`
"Virginia" gave the same match), `credential.kind` `license`, `credential.note`
"Check the exact classification and holder. An active record is not an endorsement
or proof that this credential covers your job.", classification "PLB ELE CIC HIC
GFC HVA", `expires` "2028-05-31", the summary "The Virginia Department of
Professional and Occupational Regulation (DPOR) snapshot dated 2026-09-25 shows
license 068841 (PLB ELE CIC HIC GFC HVA) listed as active in the board's public
list of active licenses, expiring 2028-05-31. The issuing agency's live page is
authoritative for today.", `findings.permits` `not_checked` with "Permit activity is
not yet indexed for this jurisdiction. Absence here means nothing.",
`official_lookup` https://www.dpor.virginia.gov/LicenseLookup, and a
`complaint_route` with a `guaranty_fund_url`.

Then: relay it under `hire-a-contractor`'s output contract, which also covers
`candidates` and `no_match_in_index` results. Relay `credential.note` as written;
never turn the classification into "your contractor is licensed for this work". A
license record does not show whether this job has a permit; ask the contractor for
the permit and check it with the building department.

## Honesty rules (verbatim; relay them, never override them)

From https://law.agentlookups.ai/llms.txt, fetched 2026-09-28:

1. "Verbatim text only: we never summarize, characterize, or rank law by importance;
   the calling agent interprets."
2. "Every record names its official source; the official source is authoritative,
   we are not."
3. "Absence is never an answer: when we do not host a jurisdiction's text, the
   response says exactly that and points at where the text lives when known."
4. "Vintage-labeled text is a dated snapshot, not current law; relay the label with
   the text."
5. "Not legal advice; no individualized guidance."

Every GroundRules response carries the `notice` "GroundRules: Original legal text.
Not legal advice."

From https://contractors.agentlookups.ai/llms.txt, fetched 2026-09-28:

- "A no_match_in_index result is NOT a determination that a business is unlicensed."
- "We report what the public record says, with its date; the issuing agency's live
  page is authoritative for today."
- "We report occupational-license records, including licenses individuals hold in
  their own name (journeyman/master trades). We NEVER build person dossiers: an
  individual's page is their one license record as the board publishes it, with no
  aggregation beyond it, and it is never for any FCRA purpose."

When a response's own wording differs from this page, use the response's wording.

## Output contract

Every answer carries all five, taken from the responses, not from memory:

1. **Who decides**: the matched address; whether it is in a city, a town or
   township, or an unincorporated county; and that this local government's
   building and zoning offices decide whether the job needs a permit. In the twelve
   town and township states, name the town or township as the likely office and the
   county only as a fallback (step 2).
2. **The words**: each section quoted exactly (lead-in, the matching item, its
   conditions and exceptions) with `citation`, `current_through` (plus the `vintage`
   label or stale warning when present), "looked up <today's date>", the official
   link (`publisher_url` or `source_url`, or the stack's `official_url` when that
   is a `.zip`), and the words "the official text governs". The `html_url` copy is
   optional.
3. **The gap**: what was not searched (the city or county code, zoning included,
   outside New York City; the RCNY and zoning inside it) and where it may live, or
   that the town may publish no code online.
4. **The limits**: a quoted section is not a permit decision; not legal advice; not
   electrical, gas, or structural safety advice.
5. **If a contractor was checked**: board, snapshot date, credential label and
   `credential.note`, official link, complaint route, and that the agency's live
   page is authoritative for today, as in `hire-a-contractor`.

A short relay for example B: "This address is in unincorporated Fairfax County, so
Fairfax County's building and zoning offices decide. Virginia's statewide building
code, 13VAC5-63-80 (current through 2026 Regular Session (effective July 1, 2026);
looked up 2026-09-28), says "application for a permit and any related inspections
shall not be required for the following", and lists: "Fences of any height unless
required for pedestrian safety as provided for by Section 3306 or used for the
barrier for a swimming pool." The same section says the building official may
require a permit for exempt items "located in a special flood hazard area", and
that the exemption does not lift "other applicable requirements of this code."
Fairfax County's own code, including its zoning rules, is not in this service.
Official text (it governs):
https://law.lis.virginia.gov/admincode/title13/agency5/chapter63/section80/.
Readable copy: https://law.agentlookups.ai/law/us-va/13vac5-63-80. This is a quote,
not a permit decision, and not legal advice."

Errors in the law text go to corrections@agentlookups.ai; errors in a license record
go to https://contractors.agentlookups.ai/dispute.

## Where to send people

- **The permit decision, fees, inspections**: the city's (or county's) building
  department and zoning office; in the twelve town and township states (step 2),
  the town's or township's first.
- **Local code text outside New York City**: the three libraries above; failing
  those, the building department or the city, town, or county clerk.
- **New York City department rules (RCNY)**: the stack's `official_url`,
  https://codelibrary.amlegal.com/codes/newyorkcity/latest/overview.
- **Whether the work is safe, or how to do it**: a licensed electrician, plumber, or
  engineer, and the building inspector.
- **HOA covenants and lease terms**: the documents, and the board or landlord.
- **Contractor insurance, deposits, complaints**: `hire-a-contractor`.
