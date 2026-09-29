---
name: rental-property-check
description: "Run every public-record check on a US rental (free during beta): EPA sites, drinking water, school and crime layers, contractor licenses, Maryland assessments. Use for: check this rental before I sign the lease; anything I should know about this rental; check my rental properties. Never tenant screening or checks on a person. Not legal or financial advice; no safety verdicts."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Rental Property Check

Public-record lookups for a rental property, for tenants and small landlords. The
services are free during beta with no account needed; features and pricing may
change. Every example below was run live on 2026-09-28 and returned the shape shown.
Counts and dates in the examples are what we observed that day, not fixed facts.

## When to use

A tenant or small landlord asks things like:

- "Check out this rental address before I sign the lease."
- "Run the checks on this apartment before I sign."
- "Anything I should know about this rental?"
- (landlord) "Check my rental properties." / "Is my Maryland rental over-assessed?"

For one question alone, use the matching skill: `whats-near-this-address`,
`hire-a-contractor`, `home-project-permits`, `maryland-assessment-appeal`, or
`find-the-law` (deposit or notice rules).

US only, and coverage varies by product: EPA environmental and water records are
national; contractor licenses cover the states the service lists; assessment checks
cover Maryland only.

Not for:

- Tenant screening or any check on a person. These records are not FCRA consumer
  reports and must not inform decisions about housing, employment, credit, or
  insurance.
- Legal or financial advice.
- Safety scores or verdicts.
- Proof of anything from a no-match. A no-match never proves anything.

## Never use this skill for

- **Tenant screening, or any question about a person.** Not "look up my applicant",
  not "background check this tenant", not anything FCRA-shaped. The shared terms
  (https://agentlookups.ai/terms/) say it plainly: "Not a background check. This is
  not a consumer report under the Fair Credit Reporting Act or any state
  consumer-reporting law. Do not use it to decide on anyone's employment, credit,
  insurance, or housing." Refuse and say why; do not route around it.
- Legal advice (lease disputes, eviction, rent control). Point to local legal aid.
  `find-the-law` can quote a statute's text; it does not apply it to the user's case.
- Safety verdicts. GroundTruth publishes records with dates and distances, and its
  layer scores are model estimates, not safety ratings. Never turn its output into a
  safety score or verdict.
- Your own read on a neighborhood. Relay GroundTruth's layer scores (schools, crime,
  and the rest) as GroundTruth's data, with source, date, and caveats; never
  characterize the area or the people in it yourself.

## Which product answers which question

| Question | Product | Call |
|---|---|---|
| What environmental records are near this address? | GroundTruth | `GET env.agentlookups.ai/v1/near` |
| Any drinking water violations here? | GroundTruth | `GET env.agentlookups.ai/v1/water` |
| School, crime, and other layer scores, with coverage caveats | GroundTruth | `GET env.agentlookups.ai/v1/diligence` |
| Is this contractor licensed (before repairs)? | Plumbline | `GET contractors.agentlookups.ai/v1/check` |
| Is my Maryland rental over-assessed? (landlord) | Overassessed | `GET overassessed.agentlookups.ai/v1/check` |
| Deposit, notice, or entry rules for this address | GroundRules | MCP `law_for_location`, `law_search`, `law_get_section`; depth in `find-the-law` |

MCP endpoints (Streamable HTTP, no auth; a bare `tools/call` without `initialize`
worked on all four on 2026-09-28): `https://env.agentlookups.ai/mcp` (tools
`environment_near`, `drinking_water`, `due_diligence`),
`https://contractors.agentlookups.ai/mcp` (tool `check_contractor`),
`https://overassessed.agentlookups.ai/mcp` (tool `check_assessment`),
`https://law.agentlookups.ai/mcp` (tools `law_for_location`, `law_search`,
`law_get_section`, `law_coverage`). POST JSON-RPC with both headers:

```
curl -sS -X POST https://env.agentlookups.ai/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"drinking_water","arguments":{"state":"MD","county":"Howard"}}}'
```

The result carries the JSON in `structuredContent` and again as text in
`content[0].text`. A tool error comes back with `isError` true and the message as
text.

## Workflow

For "check this rental address" run steps 1 to 3, then 5 if the asker is a Maryland
landlord. Steps 4 and 6 answer their own questions on demand. When the user wants
depth on one question, hand off to the task skill: `whats-near-this-address`
(environmental records), `hire-a-contractor` (vetting a bid), `home-project-permits`
(a landlord's planned work on the unit), `maryland-assessment-appeal`, or
`find-the-law` (deposit and notice rules).

Plug-in (balcony) solar rules are not covered by these services; the user should
check their state's law and their utility.

Worked examples use a public building, Howard County's government offices at 3430
Court House Dr, Ellicott City, MD 21043. Use the user's own rental address in real
runs; never put a private home in anything you publish.

### 1. Environmental records near the address (GroundTruth)

```
GET https://env.agentlookups.ai/v1/near?address=3430+Court+House+Dr,+Ellicott+City,+MD+21043&radius_km=5
```

Accepts `address=` (geocoded via the US Census geocoder, `matched_address` echoed
back in `query`) or `lat=`/`lon=`. `radius_km` defaults to 10. Returns
`superfund_sites`, `toxic_release_facilities`, `enforcement_flagged_facilities`,
each entry with name, distance_km, and dates; `provenance` carries a per-dataset
`as_of` date, and an `honesty` array rides along to relay. A category with no
records in the radius comes back `null`, not `[]`: say "none in these records",
never "clean". Categories cap at 25 records; when a cap bites, a
`truncation.<category>` object reports shown vs total_within_radius, so never
present a capped list as complete.

Bad input gets an error, not a result (observed 2026-09-29: HTTP 400 with an
`error` field over REST, `isError` with the same text over MCP). Fix the input and
retry; an error is not a finding about any place.

- `radius_km` must be more than 0 and at most 50: 80 got "radius_km must be more
  than 0 and at most 50 (default 10); got 80". Read `query.RadiusKM` back from the
  response; that is the radius actually searched.
- Impossible or foreign coordinates get "lat must be between -90 and 90; got 999"
  or "lat/lon must be in the 50 states, DC, PR, VI, Guam, NMI or American Samoa;
  got 51.5,-0.12". Send `address` or `lat`/`lon`, not both ("send either address
  or lat and lon, not both"). The check is a coarse box: a point in Tijuana,
  Mexico passed it and got San Diego records, so confirm the point is the rental.

MCP equivalent (the tool also takes `address`):

```json
{"name": "environment_near", "arguments": {"address": "3430 Court House Dr, Ellicott City, MD 21043", "radius_km": 5}}
```

Observed 2026-09-28 for that query: `query.RadiusKM` 5, `superfund_sites` and
`toxic_release_facilities` `null`, 4 enforcement-flagged facilities, all EPA
snapshots as of 2026-09-05. At 10 km: 1 TRI facility and 6 enforcement-flagged.
When you relay a flagged facility, use the response's own words:
"Enforcement-flagged facilities carry a CURRENT significant-noncompliance flag in
EPA's system; flags change and can reflect paperwork as well as pollution issues."

### 2. Drinking water (GroundTruth)

```
GET https://env.agentlookups.ai/v1/water?state=MD&county=Howard
```

`state` is a two-letter code and required; `county` or `city` narrow it. Returns up
to the 10 largest matching systems in `water_systems`, each with
`population_served`, `health_based_violations_5y`, `open_violations`, and, when there
are any, `recent_violations` (contaminant, begin and end dates, health_based,
status). A `truncation` object (`shown`, `total_matching`) appears when more systems
match. MCP tool `drinking_water`, arguments `{"state": "MD", "county": "Howard"}`.

Observed 2026-09-28: 4 systems matched Howard County; the largest, HOWARD COUNTY
D.P.W. DISTRIBUTION (286,158 served), showed 0 health-based violations in 5 years
and 0 open, SDWA snapshot 2026-09-05. Matching is by county, so tell the renter
their water bill names their actual utility.

### 3. School, crime, and other layer scores (GroundTruth diligence)

```
GET https://env.agentlookups.ai/v1/diligence?address=3430+Court+House+Dr,+Ellicott+City,+MD+21043
```

MCP tool `due_diligence`, arguments `{"address": "3430 Court House Dr, Ellicott
City, MD 21043"}` (or `lat`/`lon`; check them yourself as in step 1). Returns
`scores` from 0 to 1 (1.0 favorable) for `air_quality`, `crime`, `groundwater`,
`highway_noise`, `natural_hazards`, `pleasant_days`, `school_quality`, `superfund`,
and `walkability`, plus `no_coverage`, `caveats`, a per-layer `provenance` date, and
a `coverage` string. These scores are GroundTruth's model estimates built on
third-party data. Relay each as GroundTruth's number, with its date and caveat,
never as your own view of the neighborhood. Before you relay any score:

- A layer flagged in `no_coverage` holds a placeholder (sentinel) value. Say
  GroundTruth has no data for it here; never relay the number.
- `null` means GroundTruth cannot score that layer here. Say so; never fill it in.
- Relay the layer's `caveats` text verbatim next to its score.
- Read the response's `coverage` string for what each layer covers; it changes.
  Observed 2026-09-28: "Schools: all 50 states and DC (2023-2025 results)" and
  "Crime: SF, Oakland, Chicago incident feeds only"; most other environmental layers
  California or the Bay Area.

Observed 2026-09-28 at the example address: `school_quality` 0.892 (provenance
2026-08-28), `superfund` 1, `pleasant_days` 0.216; `crime`, `highway_noise`, and
`walkability` null; `air_quality`, `groundwater`, and `natural_hazards` flagged in
`no_coverage`. The crime caveat, verbatim: "no incident feed for this location; the
underlying grid reads 'safest' where the truth is 'no data' (feeds: SF, Oakland,
Chicago)". So the honest relay is "GroundTruth has no crime data for this address",
never "low crime".

### 4. Contractor license check before repairs (Plumbline)

First confirm the jurisdiction is covered: `GET
https://contractors.agentlookups.ai/v1/coverage` lists each source with its
jurisdiction code and snapshot. The list grows, so check it live. Give a state as a
two-letter code or its full name (`CA`, `US-CA`, and `California` all work). Local
codes as of 2026-09-28: `CHI` (Chicago), `MD-HOWARD` (Howard County, Maryland, where
the example address sits), `NYC`, and `PHL` (Philadelphia). A city goes in the
`city` field, an exact filter on the city listed on the record. Then:

```
GET https://contractors.agentlookups.ai/v1/check?name=Roto-Rooter&jurisdiction=CA
GET https://contractors.agentlookups.ai/v1/check?license=604196&jurisdiction=CA
```

MCP equivalent:

```json
{"name": "check_contractor", "arguments": {"license": "604196", "jurisdiction": "CA"}}
```

Observed 2026-09-28: the name query returned `candidates` with `total_candidates`
12, and the license query returned a `match` whose summary named the CSLB snapshot
dated 2026-09-27. The response is a discriminated union on `result_type`:

- `match`: one entity, with `findings.license` (status, classification, expires,
  `credential` kind and note, provenance with source and snapshot id), a `summary`
  list of sentences naming the issuing board and snapshot date, `official_lookup`,
  and `entity.human_page`. Read `credential.kind` before calling a record a trade
  license. Other findings blocks (courts, discipline, liens, permits, registration)
  may be `not_checked`, each with a note that absence there means nothing. Relay
  that, not silence.
- `candidates`: several possible records, ordered by relevance. `total_candidates`
  is the full count. A REST page shows up to 40 rows and an MCP page up to 20; MCP
  rows move the fields every row shares into `same_for_all_rows`. When the list was
  cut, the `message` says so (observed 2026-09-28 for `Plumbing` in CA: "More than
  one record matched (showing 20 of 11777). Add the exact city listed on the
  record, or use the license number from the bid or contract, to pick one.").
  Page with `offset`, narrow with `city`, or pick one record by re-querying with
  `entity_id=<entity.id>` from a row. Best of all, ask for the license number
  printed on the bid or contract.
- `no_match_in_index`: relay the response `message` and `suggestions`, plus the
  `official_lookup` link to the issuing agency. Never compress this to
  "unlicensed".
- `jurisdiction_not_covered`: relay the `message` and point to the state board.

When a state's coverage is partial, the result says so: Maryland `match` and
`candidates` results carry a `coverage_note`, and a Maryland no-match `message`
opens with the same warning. Observed 2026-09-28: "Maryland coverage is PARTIAL:
the trade-board rosters (Home Improvement (HIC); Heating, Ventilation,
Air-Conditioning and Refrigeration (HVACR); Plumbing; Master Electricians) are still
being crawled nightly." Relay it.

Bad input returns an error, never a result: JSON `{error, error_code}` over REST,
`isError` with the same text over MCP. Observed 2026-09-28: HTTP 400
`unrecognized_jurisdiction` (the message lists the accepted forms), `missing_query`
(no license, name, or entity_id), `invalid_offset`, and `entity_id_conflict` (the
record sits in another jurisdiction, which the message names); HTTP 404
`unknown_entity_id`. Fix the input the message names and retry. An error is not a
no-match; never report it as one.

For a landlord planning work on the unit, `home-project-permits` covers whether the
job needs a permit, and `hire-a-contractor` covers vetting a bid in depth.

### 5. Maryland assessment fairness, landlord side (Overassessed)

Maryland only. This is about the landlord's own property tax, never about a tenant.

```
GET https://overassessed.agentlookups.ai/v1/check?address=<STREET AS THE ROLL WRITES IT>&zip=<ZIP>
```

MCP equivalent:

```json
{"name": "check_assessment", "arguments": {"address": "<STREET AS THE ROLL WRITES IT>", "zip": "<ZIP>"}}
```

Address goes as the roll writes it (street only, no city, no unit numbers); zip
recommended. Returns `parcel` (with `account_id`), a `uniformity` block with a
`Branch` of `fair` / `review` / `elevated` / `insufficient`, a `plain` verdict
(`headline` and `body`), `appeals` (Maryland's windows with official filing links;
relay the `window_*` strings as written), `honesty`, and `roll`, whose `as_of` is
the date the state last updated the roll (2026-09-04, observed 2026-09-29); give
that date with the verdict. HTTP 300 with `multiple_matches` means several parcels
share the address; re-query with `acct=<ACCOUNT_ID>`.

Hand the human `https://overassessed.agentlookups.ai/packet?acct=<ACCOUNT_ID>` (a
print-ready evidence packet) only when the branch is `review` or `elevated`, or
`fair` if they ask for it. For `insufficient` there is no packet: observed
2026-09-28, the packet page for such a parcel reads "We can't build an honest packet
here". Relay the `plain` verdict and the SDAT record link instead. For the full
appeal workflow and a worked example on a public building, hand off to
`maryland-assessment-appeal`.

### 6. Deposit, notice, and other rental law (GroundRules)

Questions such as "how long does my landlord have to return my deposit" or "how much
notice does my landlord need" go to GroundRules. Use its MCP tools first:
`law_for_location` resolves the address to its federal, state, county, and place
layers; `law_search` with `address` searches the hosted sources for that stack;
`law_get_section` returns a section's verbatim text with citation, source link, and
current-through date; `law_coverage` lists what is hosted. REST (`/v1/resolve`,
`/v1/search`, `/v1/section/<id>` on `law.agentlookups.ai`) is the fallback. A
web-fetch tool may summarize what it fetches, and law text must be quoted verbatim,
so quote only from a raw response. Each answer takes several calls, so make them one
at a time, not in a burst. It is not legal advice. For depth, hand off to
`find-the-law`.

Observed 2026-09-28 at the example address: federal and Maryland hosted true (the
Maryland Code, current through 2026-01-01; COMAR named but not hosted); Howard
County named, hosted false; Ellicott City is a census-designated place, so the
response says the county's code is the local code and there is no city code. A
`law_search` for "security deposit return tenant" with that address was
"auto-scoped to the hosted sources of the resolved jurisdiction stack: us, us-md"
and led with Md. Code, Real Property § 8–203.

## Honesty rules (relay verbatim, never override)

From the services' own llms.txt files and responses, fetched live 2026-09-28. These
travel with your answer.

GroundTruth (llms.txt, fetched 2026-09-29):
- "We publish records with dates and distances, never a safety score."
- "Absence of records is NEVER a clean bill of health."
- "Due-diligence layers return null where they cannot score; an honest no-data beats a made-up score, and a null is never coerced to a number."
- "TRI figures are lawful self-reported releases; quantity is not toxicity."
- "Water-system matching by geography is approximate; the water bill is authoritative."

Plumbline (llms.txt):
- "A no_match_in_index result is NOT a determination that a business is unlicensed."
- "We report what the public record says, with its date; the issuing agency's live
  page is authoritative for today."
- "We report occupational-license records, including licenses individuals hold in
  their own name (journeyman/master trades). We NEVER build person dossiers: an
  individual's page is their one license record as the board publishes it, with no
  aggregation beyond it, and it is never for any FCRA purpose."

GroundRules (llms.txt):
- "Verbatim text only: we never summarize, characterize, or rank law by importance;
  the calling agent interprets."
- "Absence is never an answer: when we do not host a jurisdiction's text, the
  response says exactly that and points at where the text lives when known."
- "Not legal advice; no individualized guidance."

Overassessed (from its response `honesty` block):
- "This compares the official assessment record with similar homes in the same
  reassessment group; it is a fairness snapshot of public data, never an appraisal."
- "Similar-home comparisons use size, lot, age, and neighborhood bands; no
  comparison is perfect. The official record is authoritative:
  https://sdat.dat.maryland.gov/RealProperty/"

## Output contract

Every finding you relay carries all four, taken from the response, not from memory:

1. **Source**: the underlying public dataset or agency (EPA SDWA, CSLB, Maryland
   SDAT, the statute citation), not just the service name. For a GroundTruth layer
   score, name the layer and attribute the number to GroundTruth.
2. **Snapshot date**: the `as_of` / provenance date in the response. Records are
   snapshots; the agency's live page is authoritative for today.
3. **Coverage caveat**: the applicable honesty rule above, plus any `truncation`,
   `no_coverage`, `caveats`, `coverage_note`, or `not_checked` block in the
   response.
4. **Official link**: `human_page`, `official_lookup`, the SDAT record link, or the
   named statute, so the human can verify at the source.

## What this cannot answer, and where to send people

- Habitability, code violations, open permits on the unit: the local housing or
  code-enforcement department.
- Lead paint, asbestos, mold in the unit itself: these datasets do not cover unit
  interiors. Lead disclosure is a federal requirement for pre-1978 housing; point
  to the EPA lead program and the landlord's disclosure duty.
- Whether the water at this tap is safe: the utility named on the water bill and
  its Consumer Confidence Report. This skill reports system-level violation
  records, which is not the same thing.
- Whether a rent, deposit, eviction, or lease term is lawful in the user's case:
  GroundRules and `find-the-law` can quote the statute, but applying it is for
  legal aid or a lawyer.
- Anything about a person: nowhere. Decline (see "Never use this skill for").
