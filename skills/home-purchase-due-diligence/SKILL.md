---
name: home-purchase-due-diligence
description: "Run every public-record check on a US house someone is buying: EPA sites and tap water nearby, school and hazard layers, Maryland assessment fairness, the inspector's license. Use for \"we're putting in an offer\", \"anything I should know about this address\", \"we're under contract, what should we check\". Never a background check on a person; not legal or financial advice; no \"is it safe\" verdicts."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Home purchase due diligence

Run the checks a careful buyer would run on one address before making or firming an offer. All data comes from agentlookups.ai services built on official public records. They are free during beta with no account needed; that is not a permanent promise, and features and pricing may change. Keep call rates reasonable. One privacy policy (https://agentlookups.ai/privacy/) and one set of terms (https://agentlookups.ai/terms/) cover every service.

Every REST endpoint below answers a plain GET. Each service also has an MCP endpoint at `<site>/mcp` (Streamable HTTP, no auth); this plugin's `.mcp.json` connects them. Without the plugin, POST JSON-RPC with headers `content-type: application/json` and `accept: application/json, text/event-stream`, for example:

```
curl -s -X POST https://env.agentlookups.ai/mcp \
  -H 'content-type: application/json' -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"drinking_water","arguments":{"state":"MD","county":"Howard"}}}'
```

The buyer is usually mid-purchase and short on time. Lead with findings, keep each finding to source + date + what it means, and put caveats inline, not in a wall at the end.

## When to use

The user is buying a US house and wants it checked before an offer or during the contract period. Typical asks:

- "We're putting in an offer this weekend."
- "Anything I should know about this address?"
- "Run every check you have on this house before we offer."
- "We're under contract, what public records should we check?"
- "How do I vet the home inspector?"
- Environmental problems, water-quality records, schools, or hazards near a house.
- Whether a Maryland property's assessment (the property tax basis) is fair.
- The license of a home inspector, or of a contractor quoting repairs from the inspection report.

Coverage is US only and uneven (as of 2026-09-28): EPA environmental and drinking-water records nationwide; due-diligence layer scores with partial coverage (schools nationwide; crime and most hazard layers in a few places only); assessment fairness in Maryland only; contractor licenses in 34 jurisdictions.

Not for:

- A background check on any person (seller, neighbor, contractor as a person). These services never build person dossiers and are never FCRA consumer reports. No tenant, employment, or borrower screening.
- Legal, financial, tax, or health advice, or an appraisal of what the house is worth.
- Safety ratings, or "is it safe" or "good neighborhood" verdicts.

If asked for those, say plainly that these are public-record lookups and cannot do that. A no-match never means unlicensed, and no records never means clean.

When the buyer's question narrows to one topic, hand off to the skill built for it:

- **whats-near-this-address**: one question about EPA sites or tap water.
- **maryland-assessment-appeal**: the full Maryland appeal walk-through (windows, packet, what to file).
- **hire-a-contractor**: a full contractor check before signing, including deposits and complaints.
- **home-project-permits**: work the buyer plans after closing (deck, fence, panel, water heater) and whether it needs a permit.
- **find-the-law**: what state, county, or city law says about something at this address.

## Workflow

For a full address check, run steps 1 to 3 in parallel; they are independent. Steps 4 and 5 only when they apply.

1. **Environmental records near the address** (GroundTruth `/v1/near` or MCP `environment_near`): Superfund/NPL sites, TRI toxic-release facilities, and enforcement-flagged facilities within a radius. Works for any US address.
2. **Drinking water** (GroundTruth `/v1/water` or MCP `drinking_water`): public water systems serving the county, with health-based violation history. Works for any US state.
3. **Due-diligence layers** (GroundTruth `/v1/diligence` or MCP `due_diligence`): school quality, crime, hazards, and other layer scores. Coverage differs by layer and changes over time, so read the `coverage` field in each response rather than assuming. Still worth running: the response flags its own gaps machine-readably.
4. **Assessment fairness, Maryland only** (Overassessed `/v1/check` or MCP `check_assessment`): whether the assessment is in line with genuinely similar homes. It matters to a buyer because the assessment sets the property tax they will pay, and because Maryland gives a new owner an appeal window that counts from the transfer. On 2026-09-28 the response's `appeals.window_purchase` read "within 60 days of the transfer, for a purchase transferred between January 1 and June 30"; quote the response, not this line.
5. **Vet the inspector or a contractor** (Plumbline `/v1/check` or MCP `check_contractor`): license record checks when the buyer names an inspector or a contractor quoting repairs. Check `/v1/coverage` for the current jurisdiction list before you promise anything.

Plug-in (balcony) solar rules are not covered by these services; the buyer should check their state's law and their utility.

## Worked examples

All of these ran live on 2026-09-28 and returned the shapes described; counts and dates will move. Address examples use public buildings: the Howard County government building at 3430 Court House Dr, Ellicott City, MD 21043, and the Maryland State House. Substitute the buyer's address, county, and contractor.

### 1. Environmental records near the address

```
GET https://env.agentlookups.ai/v1/near?lat=39.276032&lon=-76.805841&radius_km=10
```

MCP (endpoint `https://env.agentlookups.ai/mcp`, tool `environment_near`) also takes a street address and geocodes it (the coordinates above are that geocode):

```json
{"name": "environment_near", "arguments": {"address": "3430 Court House Dr, Ellicott City, MD 21043", "radius_km": 10}}
```

Response: `query` (the `Lat`, `Lon`, and `RadiusKM` actually searched, plus `matched_address` when you sent an address), then `superfund_sites`, `toxic_release_facilities`, `enforcement_flagged_facilities`, each record with name, `distance_km`, and dates; plus `honesty` (relay these), `provenance` (per-dataset `as_of` snapshot dates), `terms`, and `dispute_url`. A category with no records comes back as `null`: that means nothing in the index within the radius, never a clean bill of health. Each category caps at 25 records; when the cap bites, `truncation.<category>` reports `shown` vs `total_within_radius`, so never present a capped list as complete.

Observed 2026-09-28 for the example: `superfund_sites` null; 1 TRI facility (reporting year 2024, 9.9 km); 6 enforcement-flagged facilities, the nearest 2.6 km; every dataset `as_of` 2026-09-05; no truncation.

Guard the inputs yourself; do not count on the service to reject bad ones:

- Keep `radius_km` at 50 or below (the default is 10), and read `query.RadiusKM` back from the response before you state the radius. On 2026-09-28 a request for 80 came back searched at 10, with no error.
- Check lat/lon before calling: latitudes in the 50 states run from about 18 to 72, and the tool's schema expects a negative longitude. On 2026-09-28 impossible coordinates (latitude 139) and a flipped longitude sign both came back as ordinary responses with no records, which reads like a clean result. The address form avoids this; confirm `matched_address` is the buyer's house.

### 2. Drinking water for the county

```
GET https://env.agentlookups.ai/v1/water?state=MD&county=Howard
```

MCP tool `drinking_water` on the same endpoint (`state` required, `county` and `city` optional):

```json
{"name": "drinking_water", "arguments": {"state": "MD", "county": "Howard"}}
```

Response: up to the 10 largest matching `water_systems`, each with `pwsid`, `population_served`, `health_based_violations_5y`, `open_violations`, and, when there are any, `recent_violations` (contaminant, dates, `health_based`, and a status such as "Resolved"); a `truncation` object reports the full match count when more exist.

Observed 2026-09-28 for the example: 4 systems, no truncation. The largest, HOWARD COUNTY D.P.W. DISTRIBUTION (MD0130002, 286,158 served), had 0 health-based violations in 5 years; two small systems had 1 each, both resolved.

Send `state` as the two-letter code and `county` without the word "County". On 2026-09-28, `state=Maryland` and `county=Howard County` each returned `water_systems: null`, the same as a county with no systems. If you get `null`, check the spelling in `query` before you relay anything.

### 3. Due-diligence layers

```
GET https://env.agentlookups.ai/v1/diligence?address=3430+Court+House+Dr,+Ellicott+City,+MD+21043
```

MCP tool `due_diligence` takes `address` or `lat`/`lon`. Response: `scores` from 0 to 1 (1.0 favorable), where `null` means the service cannot score that layer here; plus `caveats`, `no_coverage`, `provenance` (a date per layer), `coverage`, and `honesty`. Read `no_coverage` and `caveats` before relaying any score: some layers return a sentinel value outside their coverage (for example `air_quality: 0.5` outside California means no data, not average). Never relay a score whose caveat says treat as no-data.

Observed 2026-09-28 for the example address: `school_quality` 0.892 (as of 2026-08-28); `crime` null, with the caveat "no incident feed for this location; the underlying grid reads 'safest' where the truth is 'no data' (feeds: SF, Oakland, Chicago)"; `air_quality`, `groundwater`, and `natural_hazards` flagged in `no_coverage` as sentinels.

**School and crime scores are source data, not your judgment.** Relay them the way the service presents them: attributed to GroundTruth's layer, with the layer's date and the response's coverage and caveat text. For example: "GroundTruth's school-quality layer scores this point 0.892 on a 0 to 1 scale, 1.0 favorable (layer data as of 2026-08-28)." A `null` crime score means no data for this place, never a low-crime area. Do not turn a score into your own words about the neighborhood ("good schools", "safe area", "rough part of town"); the buyer draws the conclusions. For the schools behind the score, `GET https://env.agentlookups.ai/v1/diligence/detail/school_quality?lat=39.276032&lon=-76.805841` returns a `detail` list of nearby schools with `name`, `level`, `city`, `distance_miles`, and `quality_score` (12 schools for this point on 2026-09-28). It takes lat/lon only (an address gets HTTP 400 "lat and lon are required"); use the `query` coordinates from the diligence response.

### 4. Maryland assessment fairness

The placeholders below are not addresses; fill them from the house.

```
GET https://overassessed.agentlookups.ai/v1/check?address=<STREET AS THE ROLL WRITES IT>&zip=<ZIP>
```

MCP (endpoint `https://overassessed.agentlookups.ai/mcp`, tool `check_assessment`; write the address the way the roll does, no unit numbers, no city; zip recommended):

```json
{"name": "check_assessment", "arguments": {"address": "<STREET AS THE ROLL WRITES IT>", "zip": "<ZIP>"}}
```

Response: the `parcel` record (with `account_id`), a `uniformity` block whose `Branch` is one of fair / review / elevated / insufficient, a `plain` block with ready-to-relay sentences, `appeals` with Maryland's windows and official filing links, and `honesty`. If several parcels share the address you get `multiple_matches` with account ids; re-query with `acct=<ACCOUNT_ID>`. An address that matches nothing returns `"error": "no parcel matched that address; include the zip, and write it as the assessment roll does (e.g. 1 STATE CIR)"` (HTTP 404 on REST; on MCP the same `error` field arrives in the tool result).

Observed 2026-09-28: `address=1 STATE CIR&zip=21401` (the Maryland State House) matched account 020600002182004 with `Branch` "insufficient" and `BandSize` 14, and `plain.headline` "Not enough similar homes to compare fairly."

The evidence packet, `https://overassessed.agentlookups.ai/packet?acct=<ACCOUNT_ID>`, follows the branch:

- **review or elevated**: give the buyer the link, and say the new-owner window counts from the transfer, so the packet is for after closing.
- **fair**: there is nothing to argue; offer the link only if the buyer asks.
- **insufficient**: do not hand over a packet. The page says "We can't build an honest packet here" (seen 2026-09-28 for the State House account); relay `plain.body` and the official SDAT link instead.

The packet and the comparison describe that parcel's public record, not the buyer's: until closing the house belongs to the seller. The verdict page `https://overassessed.agentlookups.ai/check?acct=<ACCOUNT_ID>` states that home's own appeal timing under "For this home". For the full appeal walk-through, use maryland-assessment-appeal.

### 5. Contractor or inspector license check

Give `jurisdiction` as a state code (`CA` or `US-CA`, any case), a full state name (`California`), or a local code from `/v1/coverage` (`NYC`, `CHI`, `PHL`, `MD-HOWARD`). Text that names no US state or local code returns HTTP 400 with `error_code` `unrecognized_jurisdiction` and a message listing the accepted forms.

By name (returns candidates to disambiguate; on 2026-09-28 this query returned 12, the first ROTO-ROOTER, license 604196, Oakdale):

```
GET https://contractors.agentlookups.ai/v1/check?name=roto+rooter&jurisdiction=US-CA
```

By license number (the number on the bid or contract):

```
GET https://contractors.agentlookups.ai/v1/check?license=604196&jurisdiction=US-CA
```

MCP (endpoint `https://contractors.agentlookups.ai/mcp`, tool `check_contractor`):

```json
{"name": "check_contractor", "arguments": {"license": "604196", "jurisdiction": "CA"}}
```

On 2026-09-28 the license query matched ROTO-ROOTER in the California CSLB snapshot dated 2026-09-27, status "CLEAR", expiring 2028-10-31.

The response is a discriminated union on `result_type`:

- `match`: `entity` (name, jurisdiction, and the `human_page` link), a plain-sentence `summary`, and `findings`. `findings.license` holds `status`, `classification`, `expires`, `credential` (read `credential.kind` and `credential.label` before calling it a trade license), and `provenance` (source, snapshot, raw record hash). The other findings (`courts`, `discipline`, `liens`, `permits`, `registration`) may read `not_checked` with notes like "Absence here means nothing"; relay that, and never present a clean license as a clean record. Top-level `official_lookup` points to the issuing board, and `complaint_route` to its complaint page.
- `candidates`: several possible records, `total_candidates` for the full count, and a `message`. A page holds up to 40 rows on REST and 20 on MCP (MCP gives a field shared by every row once, in `same_for_all_rows`). Narrow with `city` (the exact city on the record), re-query by license number or `entity_id`, or page with `offset`. An `entity_id` no record carries returns HTTP 404 `unknown_entity_id`; one sent with a jurisdiction that is not the record's own returns HTTP 400 `entity_id_conflict`, and the message names the record's jurisdiction; a negative or fractional offset returns HTTP 400 `invalid_offset`.
- `no_match_in_index`: NOT a determination of unlicensure (see honesty rules below). The response includes `suggestions` and, where available, an `official_lookup` URL; relay both. A search with no jurisdiction says how many covered jurisdictions it searched.
- `jurisdiction_not_covered`: the index does not cover that jurisdiction; the message says what official pointer exists, or that none is on file. Relay its exact wording.

Two fields to relay when present. `coverage_note`: on 2026-09-28, Maryland, North Carolina, and Massachusetts results carried one saying coverage is PARTIAL and naming the board lookup for a full check. `ignored_terms`: the search dropped words that matched nothing and the message says which; confirm the record is the business the buyer meant before you describe it.

Coverage before you promise anything: `GET https://contractors.agentlookups.ai/v1/coverage` lists every source with jurisdiction, record count, and snapshot run (observed 2026-09-28: 47 sources across 34 jurisdictions, 29 states and DC plus the local codes CHI, MD-HOWARD, NYC, and PHL). Note for inspector checks: many states license home inspectors under boards this index does not yet carry; a no-match on an inspector is exactly as inconclusive as any other no-match. Point the buyer to the state's own inspector-license lookup. For a full contractor check before signing, use hire-a-contractor.

## Honesty rules (verbatim, relay these, do not override them)

From GroundTruth (env.agentlookups.ai/llms.txt, fetched 2026-09-28):

- "We publish records with dates and distances. Category scores are model estimates, not safety ratings."
- "Absence of records is NEVER a clean bill of health."
- "Due-diligence layers return null where they cannot score; an honest no-data beats a made-up score, and a null is never coerced to a number."
- "TRI figures are lawful self-reported releases; quantity is not toxicity."
- "Water-system matching by geography is approximate; the water bill is authoritative."

From Plumbline (contractors.agentlookups.ai/llms.txt, fetched 2026-09-28):

- "A no_match_in_index result is NOT a determination that a business is unlicensed."
- "We report what the public record says, with its date; the issuing agency's live page is authoritative for today."
- "We report occupational-license records, including licenses individuals hold in their own name (journeyman/master trades). We NEVER build person dossiers: an individual's page is their one license record as the board publishes it, with no aggregation beyond it, and it is never for any FCRA purpose."

From Overassessed (response `honesty` field, fetched 2026-09-28):

- "This compares the official assessment record with similar homes in the same reassessment group; it is a fairness snapshot of public data, never an appraisal."
- "Similar-home comparisons use size, lot, age, and neighborhood bands; no comparison is perfect. The official record is authoritative: https://sdat.dat.maryland.gov/RealProperty/"

Each response also carries its own `honesty` and `terms` text; when a response's wording differs from the above, use the response's wording.

## Output contract

Every finding you relay to the buyer carries all four of:

1. **Source**: which public dataset, agency, or layer (for example "EPA TRI 2024 reporting year", "California CSLB", "Maryland SDAT assessment roll", "GroundTruth school-quality layer").
2. **Snapshot date**: the `as_of` / `provenance` date from the response, stated as "as of", never as "currently".
3. **Coverage caveat**: what the check could not see (categories `not_checked`, layers with `no_coverage` or a null score, a `coverage_note`, jurisdictions not in the index, truncated lists).
4. **Official-source link**: the `human_page`, `official_lookup`, agency page, or dispute URL from the response, so the buyer can verify today's state themselves.

A finding missing any of the four is not ready to relay.

## What this cannot answer, and where to send the buyer

- **"Is this house safe?" or "Is this a good neighborhood?"** No product here rates safety or neighborhoods. Relay records and layer scores with their sources and dates, and say so.
- **On-site conditions**: radon, lead paint, mold, well and septic condition, structure. That is the licensed home inspector's job and, for wells, the local health department.
- **A Phase I environmental site assessment**, title search, lien search, or permit history for the property itself. Those come from an environmental professional, a title company, and the local permit office.
- **Private well water**: the SDWA data covers public water systems. A house on a well needs a lab test, not a lookup.
- **Whether to offer, how much, or contract terms**: the buyer's agent and a real estate attorney.
- **Assessment fairness outside Maryland**: not covered; the county assessor's own record is the starting point.
- **Anything about a person** beyond a single license record as the issuing board publishes it: refuse; these services never build person dossiers and are never FCRA consumer reports.
