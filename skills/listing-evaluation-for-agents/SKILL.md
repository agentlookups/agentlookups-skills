---
name: listing-evaluation-for-agents
description: "Run every public-record check on a listing for an agent's client (US, free during beta): nearby EPA sites and water, schools, Maryland assessment fairness, contractor licenses. Triggers: \"run the checks on this listing\", \"anything near this listing I should disclose\", \"what should I flag on this listing\". Never a background check on a person, legal advice, or an appraisal."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Listing evaluation for agents

You are helping a real-estate professional check a listing against official
public records so they can forward facts to a client without overstating.
Their license depends on not overclaiming, so every fact you relay must
carry its source, its snapshot date, and its limits. These services are
free during beta, need no account or key, and welcome automated clients
at reasonable rates (a handful of requests per property is normal use).

## When to use

A realtor or buyer's agent asks things like:

- "Run the standard checks on this listing", "pull the public records on
  this listing for my client", "anything near this listing I should
  disclose", or "what should I flag on this listing": environmental
  records, then drinking water.
- "Does this assessment look high" (Maryland only): assessment fairness.
- "Is this contractor licensed" when advising a client on repairs:
  contractor license check.

For one question alone, use the task skill instead:
whats-near-this-address, maryland-assessment-appeal, hire-a-contractor,
home-project-permits, or find-the-law.

Coverage is US only. As of 2026-09-28: environmental and drinking-water
records nationwide; contractor licenses in 34 state and local
jurisdictions; assessment fairness in Maryland only. Each service's
/llms.txt and coverage endpoint give the current numbers.

Not for: appraisals or valuations, legal or financial advice, safety or
crime ratings, describing a neighborhood or its schools in your own
words, background checks on people, tenant or employment screening or
any other FCRA purpose (results are public-record lookups, not consumer
reports), or anything outside the US.

## The services

| Question | Service | Base | MCP |
| --- | --- | --- | --- |
| Federal environmental records near the address | GroundTruth | https://env.agentlookups.ai | POST /mcp, tools: environment_near, drinking_water, due_diligence |
| Drinking-water systems and violations | GroundTruth | https://env.agentlookups.ai | same |
| Is this Maryland assessment fair | Overassessed | https://overassessed.agentlookups.ai | POST /mcp, tool: check_assessment |
| Is this contractor licensed | Plumbline | https://contractors.agentlookups.ai | POST /mcp, tool: check_contractor |

MCP endpoints are Streamable HTTP, no auth, and answer stateless
tools/call requests. With the agentlookups plugin installed, these
servers are already connected; prefer their tools, since a web-fetch
tool may summarize a REST response and drop its caveats. Each site
publishes a machine index at /llms.txt and honesty rules there; fetch it
for current coverage or cadence details rather than trusting a number
written here.

## Workflow for one listing

Run steps 1 and 2 for every listing. Steps 3 to 5 only when the question
comes up. The examples use a public building, the Howard County
government office at 3430 Court House Dr, Ellicott City, MD 21043;
replace it with the listing address.

Plug-in (balcony) solar rules are not covered by these services; the
client should check their state's law and their utility.

### 1. Environmental records near the address

```
GET https://env.agentlookups.ai/v1/near?address=3430+Court+House+Dr,+Ellicott+City,+MD+21043&radius_km=5
```

Also accepts lat and lon. radius_km defaults to 10. The response has
three record lists: superfund_sites, toxic_release_facilities,
enforcement_flagged_facilities, each entry with name, distance_km, and
identifiers, plus a provenance block with a per-dataset as_of date and
an honesty list to relay. An empty list can come back as null: that
means no records in the index within the radius, which is still not a
clean bill of health. Categories cap at 25 records; when a cap bites, a
truncation object reports shown versus total_within_radius, so never
present a capped list as complete.

Bad input gets an error, not a result (as of 2026-09-29: HTTP 400 with
an error field over REST, isError with the same text over MCP). Fix the
input and retry; an error is not "no records".

- radius_km must be more than 0 and at most 50. A larger value got
  "radius_km must be more than 0 and at most 50 (default 10); got 80".
  Read query.RadiusKM back before saying what radius you searched.
- lat=91 got "lat must be between -90 and 90; got 91", and a point
  outside the US got "lat/lon must be in the 50 states, DC, PR, VI,
  Guam, NMI or American Samoa". Send address or lat and lon, not both.
  The check is a coarse box: a point in Tijuana, Mexico passed it and
  got San Diego records, so confirm the point is the listing.

As of 2026-09-28 the example returned no Superfund or toxic-release
records within 5 km (both null), four enforcement-flagged facilities
(nearest 2.6 km), and an as_of of 2026-09-05 for each EPA dataset.

### 2. Drinking water for the county

```
GET https://env.agentlookups.ai/v1/water?state=MD&county=Howard
```

Give state as a two-letter code. As of 2026-09-29, state=Maryland got
HTTP 400 with the error `state must be a two-letter code such as MD;
got "Maryland"`. Returns the 10 largest matching systems with
population_served, health_based_violations_5y, open_violations, and
recent_violations when there are any; a truncation object reports the
full match count when more than 10 match. The system serving the listing
is usually the largest one for the city or county, but matching by
geography is approximate: tell the client the water bill names the
actual system.

As of 2026-09-28 the example returned four Howard County systems; the
largest, HOWARD COUNTY D.P.W. DISTRIBUTION, served 286,158 people with 0
health-based violations in five years (SDWA as_of 2026-09-05).

### 3. Optional context layers: schools, hazards, crime and more

```
GET https://env.agentlookups.ai/v1/diligence?address=3430+Court+House+Dr,+Ellicott+City,+MD+21043
```

Returns 0-to-1 scores (1.0 favorable) per layer, such as superfund,
school_quality, natural_hazards, walkability, and crime. Each response
carries its own coverage field (which layers cover which places),
per-layer caveats, and a per-layer provenance date. Read them on every
call, since coverage changes. Before relaying anything:

- null means cannot score. Say "no data", never "no problem".
- A layer set to true in no_coverage carries a placeholder (sentinel)
  score, not a real one. Never quote a score for that layer.
- Relay school and crime layer data as GroundTruth's source data, with
  its caveat and provenance date, and do not describe the neighborhood
  yourself (no "good schools", "safe area", or "rough part of town").
  Under fair-housing rules, agents should forward the source data with
  its attribution and caveats, and not describe neighborhoods or schools
  themselves.
- For the schools behind a school_quality score, call
  /v1/diligence/detail/school_quality?lat=<lat>&lon=<lon> with the lat
  and lon from the response's query block. It lists nearby schools with
  level, distance_miles, NCES id (ncessch), and each school's
  quality_score. The detail carries no provenance date of its own; use
  the school_quality date from the main response.

As of 2026-09-28 the example returned school_quality 0.892 (provenance
2026-08-28); crime null, with the caveat "no incident feed for this
location; the underlying grid reads 'safest' where the truth is 'no
data' (feeds: SF, Oakland, Chicago)"; and air_quality, groundwater, and
natural_hazards flagged in no_coverage, with caveats such as "0.5 is
this layer's outside-coverage sentinel (CalEnviroScreen, CA only); treat
as no-data".

### 4. Maryland only: assessment fairness

```
GET https://overassessed.agentlookups.ai/v1/check?address=<STREET AS THE ROLL WRITES IT>&zip=<ZIP>
```

Write the street the way the roll does (number and street name, no
city, no unit), and include the zip. An error reading "no parcel matched
that address in that zip; write the street as the assessment roll does
(e.g. 1 STATE CIR), with no unit, city or state, and check that the zip
is the one the roll records for the parcel" (HTTP 404 over REST, an
error field over MCP) means rewrite the street or check the zip, not
that the parcel is missing. As of 2026-09-29 the example office, sent as
3430 COURT HOUSE DR with zip 21043, returned that error; the Maryland
State House, 1 STATE CIR with zip 21401, matched account
020600002182004. A zip outside Maryland gets "that zip is outside
Maryland; Overassessed covers Maryland only, from the state's own
assessment roll (SDAT)".

Returns the SDAT parcel record (with parcel.account_id), a uniformity
analysis with a verdict branch (fair, review, elevated, or
insufficient), a plain-language verdict, Maryland's appeal windows with
official filing links, honesty notes, and a roll object whose as_of is
the date the state last updated the roll (2026-09-04 on 2026-09-29);
give that date with the verdict. The API returns HTTP 300 with
multiple_matches when several parcels share an address; re-query with
acct=<ACCOUNT_ID>.

Hand the client the evidence packet link only when the branch is review
or elevated:

```
https://overassessed.agentlookups.ai/packet?acct=<ACCOUNT_ID>
```

(<ACCOUNT_ID> is parcel.account_id from the check response, exactly as
printed.) On insufficient, never hand it over: the packet page reads
"We can't build an honest packet here". The State House came back
insufficient on 2026-09-28 ("Not enough similar homes to compare
fairly."). For a buyer, the check describes a house they do not own
yet; the response's window_purchase gives the new-owner window: "within
60 days of the transfer, for a purchase transferred between January 1
and June 30". For appeal steps and deadlines, hand off to the
maryland-assessment-appeal skill.

### 5. Contractor vetting for repair advice

```
GET https://contractors.agentlookups.ai/v1/check?name=roto+rooter&jurisdiction=CA
GET https://contractors.agentlookups.ai/v1/check?license=604196&jurisdiction=CA
```

Give jurisdiction as a two-letter state code (CA or US-CA, any case), a
full state name (California), or a local code from /v1/coverage (such as
NYC or MD-HOWARD); a city goes in the city field. A license-number check
is more precise than a name check; use the number when the contractor's
card or bid shows one. The response is a discriminated union on
result_type:

- match: one record, with findings.license (status, classification,
  expiry, credential) and provenance (source, snapshot date, record
  hash). Read credential.kind and credential.label before calling a
  record a trade license; business licenses, registrations, and bond
  filings differ.
- candidates: several plausible records, with total_candidates for the
  full count. Show the list and let the human pick, then re-query with
  entity_id=<entity.id> to get that one record. Name matching is fuzzy;
  verify the city, or add city=<the city on the record> to narrow. When
  there are more records than one page holds, the message says so (for
  example "showing 20 of 11777") and offset pages through. Over MCP,
  fields that are the same on every row of a page (such as source,
  status, and snapshot date) sit once in same_for_all_rows; read them
  with each row.
- no_match_in_index: see the honesty rules below before saying
  anything.
- jurisdiction_not_covered: the message names a covered local code when
  one exists (for PA, "try jurisdiction PHL"), and official_lookup gives
  the state's own lookup when the service has one on file. When it does
  not (GA and WY as of 2026-09-28: "We do not have its official lookup
  URL on file"), send the client to that state's licensing agency.

Where a state's index is partial (MD, NC, and MA as of 2026-09-28),
match and candidates results carry a coverage_note, such as "Maryland
coverage is PARTIAL: ... for a full check, use the Maryland Department
of Labor (DLLR) board lookups directly." Relay it with the result.

A request the service cannot run returns an error that says how to fix
it (HTTP 400 or 404 with error_code over REST, isError with the same text
over MCP): unrecognized_jurisdiction lists the accepted forms;
missing_query means no name, license, or entity_id was sent;
unknown_entity_id (404) means the id did not come from an earlier
result; entity_id_conflict means the jurisdiction does not match that
record's; invalid_offset means offset was not a whole number, 0 or more.

As of 2026-09-28 the name check returned candidates (total_candidates
12); the license check returned a match for ROTO-ROOTER, CSLB snapshot
dated 2026-09-27, status active (the board's status "CLEAR"), expiring
2028-10-31.

Note the not_checked blocks in a match (courts, discipline, liens,
permits, and registration where no business registry is indexed): the
service does not index those records, and it says so. Never present a
match as "clean" beyond the license status itself. For the current list
of covered jurisdictions, check:

```
GET https://contractors.agentlookups.ai/v1/coverage
```

For a full hiring check, hand off to the hire-a-contractor skill.

## MCP examples

Any MCP client can connect to the /mcp URLs directly. Raw HTTP works
too; these stateless calls were run live on 2026-09-28. Send the same
two headers on every POST:

```
POST https://contractors.agentlookups.ai/mcp
Content-Type: application/json
Accept: application/json, text/event-stream

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"check_contractor","arguments":{"name":"roto rooter","jurisdiction":"CA"}}}
```

```
POST https://env.agentlookups.ai/mcp

{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"due_diligence","arguments":{"address":"3430 Court House Dr, Ellicott City, MD 21043"}}}
```

```
POST https://overassessed.agentlookups.ai/mcp

{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"check_assessment","arguments":{"address":"1 STATE CIR","zip":"21401"}}}
```

environment_near takes {address | lat+lon, radius_km}; drinking_water
takes {state, county, city}; due_diligence takes {address | lat+lon};
check_contractor takes {name | license | entity_id, jurisdiction, city,
offset}; check_assessment takes {address, zip, acct}. Two-letter state
codes work in every tool, and drinking_water needs one.

## Honesty rules: relay these, never override them

These are conditions of use, quoted verbatim from each service's
/llms.txt or response honesty text:

Contractor checks (Plumbline):
- "A no_match_in_index result is NOT a determination that a business
  is unlicensed." Licenses may be held under a different name, another
  jurisdiction, or a board not in the index. Say "no match in the
  index we checked", never "unlicensed".
- "We report what the public record says, with its date; the issuing
  agency's live page is authoritative for today."
- "We report occupational-license records, including licenses
  individuals hold in their own name (journeyman/master trades). We
  NEVER build person dossiers: an individual's page is their one
  license record as the board publishes it, with no aggregation beyond
  it, and it is never for any FCRA purpose." Do not use contractor
  lookups to screen people.

Environmental records and layers (GroundTruth):
- "We publish records with dates and distances, never a safety score."
- "Absence of records is NEVER a clean bill of health."
- "Due-diligence layers return null where they cannot score; an honest
  no-data beats a made-up score, and a null is never coerced to a
  number."
- "TRI figures are lawful self-reported releases; quantity is not
  toxicity."
- "Water-system matching by geography is approximate; the water bill
  is authoritative."

Assessment checks (Overassessed, from its response honesty text):
- "This compares the official assessment record with similar homes in
  the same reassessment group; it is a fairness snapshot of public
  data, never an appraisal."
- "Similar-home comparisons use size, lot, age, and neighborhood bands;
  no comparison is perfect. The official record is authoritative:
  https://sdat.dat.maryland.gov/RealProperty/"

## Output contract

Every fact you forward to the client carries four things:

1. Source: the official record set it came from (for example "CA CSLB
   licensing board", "EPA TRI", "Maryland SDAT roll", "GroundTruth
   school layer").
2. Snapshot date: the as_of or provenance date from the response, said
   plainly ("board snapshot dated 2026-09-27").
3. Coverage caveat: the matching honesty rule above, in the summary,
   not a footnote. At minimum: absence of records is never a clean
   bill of health, and a no-match is not "unlicensed".
4. Official link: the human_page, official_lookup, or agency URL from
   the response, so the client can see today's record themselves.

Never total, rank, or score across records ("this street scores 3/10"),
never sum up a neighborhood in your own words, and never soften a
caveat to make a listing look better or worse.

## Hand-offs to task skills

When the client's question narrows to one topic, the task skill has the
full workflow:

- whats-near-this-address: one question about EPA sites or a water
  system.
- maryland-assessment-appeal: the Maryland appeal path, deadlines, and
  evidence packet.
- hire-a-contractor: vetting a contractor before the client signs.
- home-project-permits: work the client plans after closing (a deck, a
  fence, a water heater) and whether it needs a permit.
- find-the-law: quoting a state or local law, such as seller-disclosure
  or security-deposit rules, with citation and current-through date. It
  quotes the law; it does not decide what the professional must
  disclose.

## What this cannot answer

- Title, deed, easements, permit history, mechanics liens: title
  company or the county land records office.
- Structural condition, radon, lead paint, mold: a licensed home
  inspector; these are records services, not inspections.
- Flood zone and insurance rating: FEMA flood maps
  (https://msc.fema.gov/portal/home) and an insurance agent.
- Crime and safety verdicts: the crime layer carries incident-feed data
  only where its coverage field says so, and null elsewhere. Relay it
  with its source; never turn it, or any other layer, into a safety
  verdict.
- Whether to buy, offer price, appeal strategy, disclosure
  obligations: the professional's judgment and, where it matters,
  a lawyer. These lookups inform that judgment; they do not make it.
