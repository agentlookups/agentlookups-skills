---
name: whats-near-this-address
description: "Any Superfund sites, toxic-release factories, or EPA-flagged facilities near a US address, or violations at my water system? Answers one question from GroundTruth's EPA records (free during beta). Asks like \"any Superfund sites near me\", \"does my water system have violations\", \"any toxic sites near the house we're buying\". No safety verdict or health advice; never a background check on a person."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# What's near this address

Answer one question about one US place, such as "any Superfund sites near me" or
"does my water system have violations", from GroundTruth
(https://env.agentlookups.ai). It republishes four EPA datasets with names,
distances, and dates. It is free during beta, with no account and no key; the
shared front page (https://agentlookups.ai/) says "The services are in beta and
currently free to use. Features, access and pricing may change as they develop."
GroundTruth's llms.txt: "No auth for reasonable rates; we never challenge automated
clients." GroundTruth re-snapshots the EPA data monthly, around the 5th.
Examples A to D were run live on 2026-09-29; A to C read the EPA snapshot dated
2026-09-05, and D's layer scores carry their own dates, one per layer. Later
snapshots will change the counts, so check
`https://env.agentlookups.ai/v1/coverage` for the current snapshot dates and trust
each response over the numbers here.

This skill answers a single question. For a full check of a home someone is
buying, renting, or listing, use `home-purchase-due-diligence`,
`rental-property-check`, or `listing-evaluation-for-agents`.

## When to use

US addresses only. Fine for one such question from a buyer, renter, or agent,
even about a listing. Asks like:

- "any Superfund sites near me", "how far is the nearest Superfund site"
- "what factories near this address release toxic chemicals"
- "is there contamination near this school", "any EPA violations near this park"
- "does my water system have violations"
- "any toxic sites near the house we're buying"

Not for:

- A full home check before buying, renting, or listing: use
  `home-purchase-due-diligence`, `rental-property-check`, or
  `listing-evaluation-for-agents`.
- "Is it safe" verdicts, safety scores, or rankings; health, legal, or financial
  advice; private wells; anything about a person (see the next section).
- Reading an empty result as "clean": no records never means clean.

## Never use this skill for

- **A safety verdict, score, or ranking.** "Is it safe to live here" has no
  answer here. Under "What we refuse to compute", the methodology page
  (https://env.agentlookups.ai/methodology/) lists: "Composite scores, rankings,
  and "is it safe" verdicts."
- **Health advice.** Whether water or air will make someone sick is for a doctor
  or the local health department. The `terms` field on `environment_near` and
  `drinking_water` responses: "NOT a safety assessment, health advice, or a
  property evaluation."
- **Anything about a person** (who owns or lives at an address). The shared terms
  (https://agentlookups.ai/terms/): "This is not a consumer report under the Fair
  Credit Reporting Act or any state consumer-reporting law. Do not use it to
  decide on anyone's employment, credit, insurance, or housing."
- **Legal or financial advice**, including what a record does to a property's value.

## Calls

| Question | MCP tool (server `groundtruth`) | REST (GET) |
|---|---|---|
| Superfund sites, TRI factories, EPA-flagged facilities near a place | `environment_near` `{"address": "...", "radius_km": 10}` | `https://env.agentlookups.ai/v1/near?address=<street, city, state>&radius_km=10` |
| Public water systems and violations | `drinking_water` `{"state": "MD", "county": "Howard"}` (`state` required, as a two-letter code; `county` and/or `city` to narrow) | `https://env.agentlookups.ai/v1/water?state=MD&county=Howard` |
| Layer scores (schools, hazards, noise, walkability, crime, climate) | `due_diligence` `{"address": "..."}` | `https://env.agentlookups.ai/v1/diligence?address=<street, city, state>` |
| Dataset counts and snapshot dates | none | `https://env.agentlookups.ai/v1/coverage` |

`environment_near` and `due_diligence` take either `address` or `lat` and `lon`,
not both: sending both gets the error "send either address or lat and lon, not
both". `tools/list` marks all three tools read-only, with the titles "Find
federal environmental records near an address", "Look up public water systems
for an area", and "Score a property's due-diligence risk layers".
MCP endpoint `https://env.agentlookups.ai/mcp`: Streamable HTTP, no auth; a plain
POST of `tools/call` worked with no `initialize` step (headers
`content-type: application/json` and `accept: application/json, text/event-stream`).
The result carries the same JSON as REST in `structuredContent`, and as a string
in `content[0].text`. Errors differ: on MCP they come back as HTTP 200 with
`isError: true`, the message in `content[0].text`, and no `structuredContent`;
the REST equivalent is HTTP 400 with an `error` field.

## Workflow

1. **Get a full street address with city and state**, or lat/lon. Check
   `matched_address` (under `query` for `environment_near`, at the top level for
   `due_diligence`) against what the user meant; for a street address the
   `honesty` text says "geocoding is approximate". A lat/lon query has no
   `matched_address`. A city alone (`address=Mountain+View,+CA`) works, but the
   response then says: "These results are for the geographic center of Mountain
   View city, CA, not for a specific property." An address the geocoder cannot
   place gets an error that begins "no match for" (REST: HTTP 400 `error`; MCP:
   `isError: true`, text in `content[0].text`); ask for the full address or pass
   lat/lon. Send latitude first. The service rejects a pair it cannot place in
   the US (REST HTTP 400, MCP `isError`): `lat=999&lon=-999` got "lat must be
   between -90 and 90; got 999", a London point got "lat/lon must be in the 50
   states, DC, PR, VI, Guam, NMI or American Samoa; got 51.5,-0.12", and a
   swapped Baltimore pair added "(lat and lon look swapped)" (2026-09-29). Fix
   the pair and retry; never read the error as a finding. The check is coarse (see
   the note after this list).
2. **Pick the call from the question.**
   - Superfund, factories, toxic chemicals, EPA violations near a place, school,
     or office: `environment_near`.
   - "My water": ask for the utility name on the water bill, then call
     `drinking_water` with the state and the county or city.
   - School quality, flood/fire/quake zones, noise, walkability, crime, climate:
     `due_diligence`, read under its own rules below. On 2026-09-28 flood/fire/quake
     data existed only for parts of California (check the response's `coverage`
     string and `no_coverage`); elsewhere send flood questions to FEMA's flood map
     (https://msc.fema.gov/portal/home).
3. **Set the radius on purpose.** The default is 10 km (6.2 mi), the maximum 50.
   A `radius_km` of 0 or less, or above 50, gets an error instead of a search
   (2026-09-29, REST and MCP): "radius_km must be more than 0 and at most 50
   (default 10); got 80". Read `query.RadiusKM` back from the response and state
   that radius. If the user names a distance, convert (1 mi = 1.609 km); for more
   than 50 km (31 mi), search 50 km and tell the user that is the limit.
4. **Relay under the output contract.**

The coordinate check is a coarse box, not a border. On 2026-09-29 a point in
Tijuana, Mexico (`lat=32.5149&lon=-117.0382`) passed it: `/v1/near` listed San
Diego facilities 6.9 to 8.2 km away, and `/v1/diligence` returned layer scores
(`school_quality` 0.09) for a point outside the US. A point the check lets
through is not proof that it lies in the US: confirm the point is the property,
and say GroundTruth covers the US only.

## Worked examples

### A. Nothing on the Superfund list nearby (MCP)

```json
{"name": "environment_near", "arguments": {"address": "3430 Court House Dr, Ellicott City, MD 21043", "radius_km": 10}}
```

```
GET https://env.agentlookups.ai/v1/near?address=3430+Court+House+Dr,+Ellicott+City,+MD+21043&radius_km=10
```

Both returned `matched_address` "3430 COURT HOUSE DR, ELLICOTT CITY, MD, 21043",
`RadiusKM` 10, and `superfund_sites: null`. One TRI facility: NORTHROP GRUMMAN
SYSTEMS CORP. - TROY HILL 4, Elkridge, `distance_km` 9.9, `reporting_year` 2024,
`chemicals` ["Lead"], `total_release_lbs` 0.3 (`air_release_lbs` 0.1). Six
enforcement-flagged facilities from 2.6 to 9.8 km, all with `snc_flags` "CWA";
among them CCBC - CATONSVILLE CAMPUS (a Community College of Baltimore County
campus), 6.9 km, `programs` "CWA", `frs_registry_id` 110012669264. Every
`provenance` entry had `as_of` "2026-09-05".

Then: "no Superfund sites within 10 km in EPA's list as of 2026-09-05", with the
absence rule in the same sentence. Never "no contamination".

### B. Superfund sites, and a capped list

```
GET https://env.agentlookups.ai/v1/near?address=500+Castro+St,+Mountain+View,+CA+94041&radius_km=5
```

Returned 8 Superfund sites, each with `epa_id`, `name`, `npl_status`,
`npl_listing_date`, and `distance_km`. Nearest: JASCO Chemical Corp., 1.1 km,
"Deleted NPL Site", listed 1989-10-04. The other seven read "NPL Site"
(Spectra-Physics, Inc., 2 km, listed 1991-02-11). Also 2 TRI facilities and 5
flagged facilities.
The same address at `radius_km=20` returned 21 sites and a `truncation` object:
`"toxic_release_facilities": {"truncated": true, "shown": 25, "total_within_radius": 52, "note": "nearest 25 shown of 52 within this radius"}`,
and 25 of 51 for flagged facilities. Relay the `note`; never present a capped list
as complete.

### C. Water systems for a county

```
GET https://env.agentlookups.ai/v1/water?state=MD&county=Howard
```

```json
{"name": "drinking_water", "arguments": {"state": "MD", "county": "Howard"}}
```

Returned 4 systems, largest first: HOWARD COUNTY D.P.W. DISTRIBUTION (`pwsid`
MD0130002), `population_served` 286158, `health_based_violations_5y` 0,
`open_violations` 0. Two smaller systems had 1 each, listed in `recent_violations`
as "LEAD AND COPPER RULE REVISIONS", begun 2024-10-17, status "Resolved". Systems
with none carried no `recent_violations` field.

- Pass the bare county name. `county=Howard+County` returned no systems, and so did
  `city=Ellicott+City`, each with the `message` "No public water systems matched
  this geography in the SDWA data. This does NOT mean the area lacks regulated
  water: check spelling, try county without city, or consult your water bill for
  the utility name."
- Pass the two-letter state code. `state=Maryland&county=Howard` got HTTP 400
  (MCP: `isError`) with the error `state must be a two-letter code such as MD;
  got "Maryland"` (2026-09-29); `state=md` worked.
- With violations: `https://env.agentlookups.ai/v1/water?state=MD&county=Baltimore+city`
  listed CITY OF BALTIMORE (MD0300002, 1600000 served) with 2 health-based
  violations in 5 years, 0 open: "TTHM", 2026-01-01 to 2026-03-31, status
  "Archived"; "Revised Total Coliform Rule", 2022-09-01 to 2022-09-30, "Resolved".
- Capped: `state=CA&county=Santa+Clara` returned 10 systems and the note "10
  largest systems shown of 74 matching this geography". Narrow with `city`:
  `state=CA&city=Mountain+View` returned one, CITY OF MOUNTAIN VIEW.

Then: find the utility named on the user's bill in the list and report its row,
as "0 health-based violations in 5 years", never "no violations" (see Water
below). If it is not listed, say so; that is not a finding about the utility.

### D. Layer scores, only when asked

```
GET https://env.agentlookups.ai/v1/diligence?address=3430+Court+House+Dr,+Ellicott+City,+MD+21043
```

Returned `scores` with `school_quality` 0.892, `superfund` 1, `pleasant_days`
0.216; `air_quality` 0.5, `groundwater` 0.7, and `natural_hazards` 0.7, all three
listed in `no_coverage` with caveats such as "0.5 is this layer's outside-coverage
sentinel (CalEnviroScreen, CA only); treat as no-data"; `crime`, `highway_noise`,
and `walkability` null. The `crime` caveat: "no incident feed for this location;
the underlying grid reads 'safest' where the truth is 'no data' (feeds: SF,
Oakland, Chicago)". `school_quality` had no caveat. MCP `due_diligence` for 500
Castro St, Mountain View, CA scored every layer but `crime` and had no
`no_coverage` block. The response's `coverage` string, verbatim as observed on
2026-09-29 (read the live one in each response; it can change):
"Superfund/toxic-release proximity: national (EPA NPL + TRI, all US). Other
environmental layers (groundwater, flood/fire/quake zones, air, noise,
walkability): CA/Bay Area. Schools: all 50 states and DC (2023-2025 results).
Climate (pleasant_days): CONUS, 4km (no AK/HI). Crime: SF, Oakland, Chicago
incident feeds only. Scores are 0-1 with 1.0 = favorable; null = cannot score this
point."

A relay of the school and crime layers for this address:

> GroundTruth's school layer, which scores the closest public schools compared
> across states, gives 0.892 on a 0 to 1 scale (1 is favorable), data dated
> 2026-08-28. It does not say which schools serve this address; ask the district.
> GroundTruth has no crime data for this point. Its note: "no
> incident feed for this location; the underlying grid reads 'safest' where the
> truth is 'no data' (feeds: SF, Oakland, Chicago)".

Dates here are per layer, not one EPA snapshot. `provenance` is a map from layer
to date, with no `as_of` key: for this address, `air_quality`, `crime`,
`groundwater`, `highway_noise`, and `natural_hazards` "2026-08-05";
`pleasant_days` and `superfund` "2026-08-26"; `school_quality` "2026-08-28";
`walkability` "2026-08-16". Give each score with its own layer date. The response
has no `terms` or `dispute_url` field.

## How to read the results

- **Superfund.** Relay `npl_status` as written ("NPL Site", "Deleted NPL Site",
  or "Proposed NPL Site"), with `npl_listing_date` when present. Some records
  have no `npl_listing_date`: a "Proposed NPL Site" has none because EPA has
  proposed it but not yet listed it (68th Street Dump, Baltimore, MDD980918387),
  and a few deleted sites lack one too. Give the status, say no listing date is on
  file, and never supply one. The `environment_near` honesty array: "A Superfund
  site 'deleted from the NPL' means EPA determined cleanup was completed; 'final
  NPL' sites are under active federal cleanup management." The methodology page:
  "an NPL listing is a cleanup-process fact." Never say a deleted site is clean or
  safe. Deleted sites can still carry use limits and contamination: at JASCO
  (deleted 2020), EPA's cleanup page says PCE "remains in groundwater" (EPA traces
  it to "an unknown source, not JASCO") and "institutional controls are in place".
  Send the user to EPA's live site profile. Each site has a page at
  `https://env.agentlookups.ai/superfund/<epa_id>-<name>`: lowercase, turn each run
  of characters other than letters and digits into one hyphen, then drop any
  leading or trailing hyphen. "JASCO Chemical Corp." (CAD009103318) becomes
  `https://env.agentlookups.ai/superfund/cad009103318-jasco-chemical-corp`, not
  `...-corp-`, which returns 404. The page says "EPA's live site profile is
  authoritative for cleanup status and history." and links to it.
- **TRI.** Give `reporting_year`, `chemicals`, and pounds. The methodology page:
  "TRI pounds are not toxicity".
- **Enforcement flags.** Give `snc_flags` and name each law: CAA the Clean Air Act,
  CWA the Clean Water Act, RCRA the federal hazardous-waste law (RCRA), SDWA the
  Safe Drinking Water Act, TSCA the Toxic Substances Control Act. A value can join
  several with "/", as in "CWA/RCRA". Say the flag is current as of the snapshot.
  The methodology page: "an enforcement flag is not a conviction". This list holds
  only facilities with a current significant-noncompliance flag (the methodology
  page: "we index ONLY facilities carrying a current significant-noncompliance
  flag"). Facilities with lesser or past violations do not appear, so an empty
  list does not mean no EPA violations nearby.
- **EPA's own report** on any facility or water system:
  `https://echo.epa.gov/detailed-facility-report?fid=<id>`, with the
  `frs_registry_id`, `tri_facility_id`, or `pwsid`. The GroundTruth front door
  builds these links, and EPA's report service resolved all three kinds of ID.
- **Distances** are kilometers (`distance_km`), great-circle from the geocoded
  point. Give miles too (km x 0.621). The methodology page: "Records without
  coordinates cannot appear in radius results (a stated coverage gap)."
- **Water.** The methodology page: "the biggest matching system is listed first
  as the most likely server of an address, and your water bill is authoritative."
  Relay violation `status` as written. The counts and `recent_violations` cover
  health-based violations only (the tool: "health-based SDWA violation
  summaries"), so always say "health-based violations", never just "violations".
  Also say: this list leaves out monitoring, reporting, and other violations; EPA's
  report at `https://echo.epa.gov/detailed-facility-report?fid=<pwsid>` shows every
  kind for about the last three years. For CITY OF BALTIMORE it listed 2023
  monitoring violations (category "MR") under the Nitrate Rule and Synthetic
  Organic Chemicals, and a 2025 reporting violation (category "RPT") under the
  Lead and Copper Rule Revisions, all of which GroundTruth leaves out. HOWARD COUNTY D.P.W. shows 0 here,
  yet EPA's SDWIS data lists a 2022 Consumer Confidence Rule violation (category
  "Other", resolved). Both observed 2026-09-28.
- **Layer scores.** A layer in `no_coverage` has no data at that point: say so and
  quote its caveat (for `natural_hazards`, see the next bullet); never relay its
  number. Never turn a null into a number. Relay caveats verbatim; the tool
  description says `caveats.<layer>` states the limits "the score alone does not
  show". Give each score with its layer name, its own `provenance`
  date, and the source its caveat or the `coverage` string names
  (CalEnviroScreen, GAMA wells, gridMET); never label a layer score "EPA records".
  Never average layers, grade them, or call them safe or unsafe.
- **School and crime layers.** Relay them when asked, as GroundTruth's source data
  with its own caveats, never as our view of a neighborhood: no "good schools",
  "safe area", or "rough block" in your own words. The GroundTruth front door
  (https://env.agentlookups.ai/) describes them (2026-09-29). `school_quality`:
  "How the closest public schools score, compared fairly across states." and "Ask
  the district which schools serve this address." `crime`: reported-incident
  density only where a city publishes a feed; outside them the page says "Only
  some cities publish the incident feeds we use (San Francisco, Oakland, Chicago);
  this point is outside them." Outside the crime feeds, `crime` is null and its
  caveat says why; quote it, and never read the null as "safe". Check the
  `coverage` string for which places each layer covers. For a real estate agent, fair-housing rules apply: forward the
  source data with its attribution and caveats, and do not describe the
  neighborhood or its schools in your own words.
- **Superfund score.** For a Superfund question, lead with the `environment_near`
  records, not the `superfund` score (its layer date can trail the EPA snapshot),
  which the methodology page says "distance-weights every federal cleanup (NPL)
  site and every toxic-release (TRI) facility within 5 miles of the point."
- **Flood, fire, and quake.** On 2026-09-28 these answers existed only for parts
  of California. `natural_hazards` came back in `no_coverage` at the city halls of
  New Orleans, Miami, and Houston and in downtown Los Angeles, yet its caveat ends
  "(flood is nationwide)". That clause is wrong for any point in `no_coverage`: say
  there is no flood, fire, or quake data for the point, and send flood questions
  to FEMA's Flood Map Service Center (https://msc.fema.gov/portal/home).

## Honesty rules (verbatim; relay them, never override them)

From https://env.agentlookups.ai/llms.txt, fetched 2026-09-29:

- "We publish records with dates and distances, never a safety score."
- "Absence of records is NEVER a clean bill of health."
- "Due-diligence layers return null where they cannot score; an honest no-data
  beats a made-up score, and a null is never coerced to a number."
- "TRI figures are lawful self-reported releases; quantity is not toxicity."
- "Water-system matching by geography is approximate; the water bill is authoritative."

From the response `honesty` fields, same date:

- `environment_near`: "These are federal records near the queried location, stated
  with dates and distances. We never compute a safety score; a record is a fact to
  interpret, not a verdict." / "Absence of records is NEVER a clean bill of health:
  datasets have coverage gaps, reporting thresholds, and publication lag." / "TRI
  figures are facility self-reports of lawful, permitted releases, in pounds;
  quantity is not toxicity, and presence on the list is not a violation." /
  "Enforcement-flagged facilities carry a CURRENT significant-noncompliance flag in
  EPA's system; flags change and can reflect paperwork as well as pollution issues."
- `drinking_water`: "Water systems are matched by county/city served, which is
  APPROXIMATE: service-area boundaries are not public data. Your water bill names
  your actual utility; confirm there." / "Absence of records is NEVER a clean
  bill of health: datasets have coverage gaps, reporting thresholds, and
  publication lag."
- `due_diligence`: "Layer scores summarize public datasets with known coverage gaps.
  Never a safety assessment; absence of adverse signals is not a clean bill of health."

When a response's wording differs from the above, use the response's.

## Output contract

Every answer carries all of:

1. **The records**: each by name, distance (km and miles), and date (NPL status,
   and listing date when present; TRI reporting year, chemicals, pounds; the
   flag's law), or the water system's name, people served, and health-based
   violations with dates and status, named as health-based.
2. **Source**: the EPA dataset (Superfund NPL, Toxics Release Inventory, ECHO
   significant noncompliance, Safe Drinking Water Act), not just "GroundTruth".
   For a layer score, the source its caveat or the `coverage` string names (for
   `school_quality` and `crime`, GroundTruth's layer as its front door describes
   it), never "EPA records".
3. **Snapshot date**: for `environment_near` and `drinking_water`, the `as_of` in
   this response's `provenance`, said as "EPA records as of <as_of>", never
   "currently". For `due_diligence`, each score with its own layer date,
   `provenance.<layer>`.
4. **Coverage caveat** in the answer itself: the radius used, any `truncation`
   note, the absence rule, approximate water matching, any `no_coverage` layer,
   and, when relevant, that the flag list holds only current significant
   noncompliance and the water list only health-based violations.
5. **Links**: https://env.agentlookups.ai/ (the user types the address to see the
   same records), plus the Superfund page or EPA facility report for any record
   named. Wrong or stale data: corrections@agentlookups.ai
   (https://env.agentlookups.ai/dispute/).

A short relay for example A (a full answer names all six flagged facilities;
this one names one to stay short): "Within 10 km (6.2 mi) of 3430 Court House Dr,
Ellicott City, MD, EPA records as of 2026-09-05 list no sites on the Superfund
National Priorities List. That is not a clean bill of health; the datasets have
gaps and lag. One factory reported releases to EPA's Toxics Release Inventory:
NORTHROP GRUMMAN SYSTEMS CORP. - TROY HILL 4 in Elkridge, 9.9 km (6.2 mi) away,
0.3 lbs of lead in reporting year 2024, a lawful self-report (pounds are not
toxicity). Six facilities, from 2.6 to 9.8 km (1.6 to 6.1 mi) away, carry a
current significant-noncompliance flag under the Clean Water Act in EPA's ECHO
system, among them CCBC - CATONSVILLE CAMPUS, a Community College of Baltimore
County campus 6.9 km (4.3 mi) away. Flags change and can reflect paperwork, and a
flag is not a conviction. See all six at https://env.agentlookups.ai/ and EPA's
reports at https://echo.epa.gov/detailed-facility-report?fid=2107WNRTHR715TR
(Northrop Grumman) and
https://echo.epa.gov/detailed-facility-report?fid=110012669264 (CCBC Catonsville).
Wrong or stale data: corrections@agentlookups.ai."

## What this cannot answer, and where to send people

- **"Is it safe?"** or **"will this make my kids sick?"**: no verdict exists here;
  a doctor or the local health department.
- **Today's cleanup status** of a site: EPA's live site profile, linked from the
  GroundTruth Superfund page.
- **Water at one tap, lead pipes, or a private well**: the SDWA data covers public
  water systems only. Ask the utility about its own system; a well needs a lab
  test.
- **Soil, radon, or mold at one property**: an inspector or environmental
  professional.
- **A full home check before buying, renting, or listing**: the three property
  skills named at the top.
- **Anything about a person**: decline; these lookups cover places, and the
  shared terms say they are not a consumer report under the FCRA.
