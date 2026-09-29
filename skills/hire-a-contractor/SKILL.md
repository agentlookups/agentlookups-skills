---
name: hire-a-contractor
description: "Check a US contractor's license record before hiring, with Plumbline (free during beta). For \"is my contractor licensed\", \"check this electrician's license\", \"is this roofer legit\", \"how big a deposit should I give\", \"my contractor walked off the job\". Never a background check on a person or screening of an applicant or tenant; not legal advice or lawsuit help. A no-match never means unlicensed."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Hire a contractor

Help someone who is about to hire a plumber, electrician, roofer, HVAC company,
general contractor, or home improvement contractor check the license record
before they sign or pay a deposit. One service: Plumbline, built on official
licensing-board records. Free during beta (llms.txt: "Lookups are currently free;
features, limits and pricing may change."), no account or key needed; a handful of
requests per hire is normal use. If you see HTTP 429, slow down. Every example
below was run live on 2026-09-28 and returned the shape described; records and
snapshot dates change, so read "returned" as "returned on 2026-09-28".

## When to use

Use it for any contractor license question, even from a buyer, renter, landlord,
or agent. US only, 30+ state and city jurisdictions. Typical asks:

- "is my contractor licensed", "is this plumber licensed"
- "check this electrician's license number"
- "is this roofer legit", "verify my general contractor before I sign"
- "what should I check before I hire a contractor"
- "how big a deposit should I give my contractor"
- "my contractor walked off the job, where do I complain"
- "is the roofer from our inspection licensed"

Not for:

- Records on the house itself: use `home-purchase-due-diligence`,
  `rental-property-check`, or `listing-evaluation-for-agents`.
- Permits or code for a planned job: use `home-project-permits`.
- A background check on a person, or screening an applicant, tenant, or borrower.
- Reviews or ratings.
- Legal advice on suits or liens.

A no-match never means unlicensed; an active record is not insurance, bonding, an
endorsement, or proof it covers this job.

## Never use this skill for

- **A background check on a person.** Some records are licenses individuals hold
  in their own name (journeyman or master trades). You may check the one license
  of the tradesperson the user is hiring, by the number they gave. Never search a
  name to learn about a person, never combine the record with anything else, and
  never use it to screen a job applicant, employee, tenant, borrower, or insured.
  Every response says: "This is NOT a consumer report under the FCRA or state
  consumer-reporting laws, and must not be used for decisions about employment,
  credit, insurance, or housing."
- **A quality verdict or ranking.** "Is this a good roofer" or "which of these
  three bids is best" has no answer here. Plumbline reports records, "not a
  rating, a recommendation, or a determination of fitness."
- **A hire verdict.** Even when the user asks "is this roofer legit", never answer
  "legit", "safe to hire", "OK to hire", "fully licensed", or "good to go". Say
  what the record shows (board, status, class, date) and what it cannot show, and
  leave the decision to the user.
- **Code, safety, or permit rulings.** "Is this wiring safe" or "does this need a
  permit" has no answer in a license record, which says nothing about the work.
  Send the user to the local permit or building office. For what the law says
  about permits for a planned job, use `home-project-permits`.
- **Legal advice** on a contract, a lien, or a dispute. Point to the board, the
  state consumer-protection office, or a lawyer.

## Calls

| Need | REST (GET) | MCP `check_contractor` arguments |
|---|---|---|
| Which jurisdictions are covered | `https://contractors.agentlookups.ai/v1/coverage` | (REST only) |
| Check by license number (best) | `/v1/check?license=<no>&jurisdiction=US-<ST>` | `{"license": "...", "jurisdiction": "CA"}` |
| Check by business name | `/v1/check?name=<name>&jurisdiction=US-<ST>` | `{"name": "...", "jurisdiction": "CA"}` |
| Narrow by the city on the record | add `&city=<city>` | add `"city": "..."` |
| Pick one record from candidates | `/v1/check?entity_id=<entity.id>` | `{"entity_id": "..."}` |
| Page through candidates | add `&offset=<n>` | add `"offset": n` |

MCP endpoint: `https://contractors.agentlookups.ai/mcp` (Streamable HTTP, no auth,
stateless `tools/call` works; over raw HTTP, POST JSON-RPC with headers
`content-type: application/json` and `accept: application/json, text/event-stream`).
The one tool is `check_contractor` (title "Check a contractor's license record",
read-only); every argument is optional, but a call needs a license, a name, or an
`entity_id`. `jurisdiction` takes a state code (`CA` or `US-CA`, any case), a full
state name (`California`, `District of Columbia`), or a local code: `CHI`,
`MD-HOWARD`, `NYC`, `PHL`. The MCP result carries the same JSON as REST in
`structuredContent`, and as a string in `content[0].text`.

Send the license number as printed, spaces included: a Howard County, Maryland
number matched with its space and returned `no_match_in_index` without it. Howard
County records carry no names, so a name search there never matches; ask for the
number.

### Errors

REST answers a bad request with HTTP 400 or 404 and `{"error": "...",
"error_code": "..."}`. MCP returns the same `error` text with `isError: true` and
no `error_code`. Read the text and fix the call; none of these is a result about
the contractor. Observed 2026-09-28:

- `missing_query` (400), no license, name, or `entity_id`: "Nothing to look up:
  send a license number, a business name, or an entity_id from an earlier result.
  jurisdiction and city only narrow those."
- `unrecognized_jurisdiction` (400), for example `jurisdiction=Narnia`: the error
  lists the accepted forms and ends "A city goes in the city field."
- `unknown_entity_id` (404): "No record has entity_id <id>. Entity IDs come from
  the entity.id field of earlier check results; search by name or license number
  instead."
- `entity_id_conflict` (400), when `jurisdiction` or `city` is not the record's
  own: "Entity plb_argkixjllr is in CA, not WA. Leave jurisdiction empty, or set it
  to CA, to get this record."
- `invalid_offset` (400), for a negative, fractional, or non-number offset:
  "offset must be a whole number, 0 or more ..."
- Over MCP, an unknown argument or a wrong type is an error too, for example
  "license must be a string, such as \"1000002\"; got a number."

## Workflow

1. **Get the license number and the state** from the bid, contract, business card,
   or truck. A number beats a name. Give the state too: a bare `license=604196`
   returned three candidates with the message "This license number exists in 3
   covered jurisdictions (CA, CT, NYC); a bare number does not identify the issuing
   state, so pick the state or jurisdiction that issued this license."
2. **Check coverage if unsure.** `GET /v1/coverage` returns `sources` (each with
   `source`, `jurisdiction`, `snapshot_run`, `records`), a total `entities` count,
   and a `note` that includes "Jurisdictions not listed are not covered; a no-match
   is never a determination of unlicensure." The list changes; fetch it, do not
   recite it or guess from memory. Check it before you tell a user a state is not
   covered.
3. **Run the check**, act on `result_type` (examples below), then walk the user
   through the pre-signing checklist.

## Worked examples, and what to do with each result

### A. `match`: by license number

```
GET https://contractors.agentlookups.ai/v1/check?license=604196&jurisdiction=US-CA
```

```json
{"name": "check_contractor", "arguments": {"license": "604196", "jurisdiction": "CA"}}
```

Returned `result_type: "match"`: `entity` (kind `business`, ROTO-ROOTER, Oakdale,
`human_page`); `findings.license` with `status: "active"`, `raw_status: "CLEAR"`,
`classification: "A| C36| C42| HAZ"`, `expires: "2028-10-31"`, a `credential`
block, and `provenance` (`source`, `snapshot_id`, `observed_at`, `raw_hash`); a
`summary` naming the board and "snapshot dated 2026-09-27"; `not_checked` blocks
for courts, discipline, liens, permits, and registration; plus `official_lookup`,
`complaint_route` (CSLB's `online_form`), `terms`, and `dispute_url`.
`jurisdiction=California` returned the same record.

Then: relay it under the output contract, and compare it with the hire: the name
matches the contract, the status is active, and the expiry falls after the job
ends. Show the classification as the board lists it and tell the user to confirm
with the board that it covers this job; do not turn class codes into a yes or no
on scope.

### B. `candidates`: by name, then narrow to one record

```
GET https://contractors.agentlookups.ai/v1/check?name=roto+rooter&jurisdiction=US-CA
GET https://contractors.agentlookups.ai/v1/check?name=roto+rooter&jurisdiction=US-CA&city=novato
GET https://contractors.agentlookups.ai/v1/check?entity_id=plb_argkixjllr
```

The first returned `total_candidates: 12` (each with name, license, city, status,
classification, `entity.id`, `human_page`) and the message "More than one record
matched. Add the exact city listed on the record, or use the license number from
the bid or contract, to pick one." Adding `city=novato` returned a single `match`
(ROTO ROOTER PLUMBERS, license 288461). `entity_id=plb_argkixjllr` selected the
Oakdale record from example A. `offset=10` returned the last two rows and "(showing
11-12 of 12)"; `offset=50` returned an empty `candidates` list and "the requested
offset 50 is past the end of the 12 matched records." MCP takes the same arguments:
`{"name": "roto rooter", "jurisdiction": "CA", "city": "Novato"}`.

Page sizes differ: REST returns up to 40 rows a page, MCP up to 20.
`total_candidates` is always the full count. On MCP, a field with the same value on
every row of the page (for example `source_label`, `as_of`, `status`,
`official_lookup`) moves to one `same_for_all_rows` block, and each row's `entity` block drops what the
row already states. Rows come in relevance order; the first row is not a pick.

Then: never pick one for the user. Ask for the license number on the bid, or
narrow with `city` and select with `entity_id`. From llms.txt: "city is an
optional exact listed-city filter (case-insensitive), applied before pagination. It
is not a service-area search; clear it to include other or missing cities." The
record's city may be an office far from the job (example C lists ATLANTA). Read the
`message` first: when it says "We ignored" a word and searched for another (the
dropped words are also in `ignored_terms`), the candidates may be other businesses
entirely, or individuals' own trade licenses (one relaxed search returned a single
individual's record). Do not present those as the contractor; ask for the number.

A `match` from a name search is not an exact name match, and it may carry no
`message` at all. `name=sunshine+plumbing+llc&jurisdiction=CA` returned a single
`match` for SUNSHINE PLUMBING & ROOTER. Compare `entity.canonical_name` with the
name on the contract before you relay anything, and ask for the license number if
they differ.

### C. A `match` that is a registration, not a trade license

```
GET https://contractors.agentlookups.ai/v1/check?license=HOMEDDU785NU&jurisdiction=US-WA
```

Returned a `match` for HOME DEPOT U.S.A., INC., classification "PLUMBING
CONTRACTOR", `status: "active"` (`raw_status: "ACTIVE"`), but `credential.kind`
is `registration`, label "Contractor registration", with the note: "This
registration does not establish every trade license or endorsement that a
particular job may require. Verify the scope with the issuing agency." Its
`complaint_route` also carries a `guaranty_fund_url` (Washington's Homeowner
Recovery Fund).

Then: call it a registration, relay the note, and send the user to the board to
confirm the trade license the job needs.

### D. `no_match_in_index`

```
GET https://contractors.agentlookups.ai/v1/check?name=zzyzx+quantum+roofing&jurisdiction=US-TX
```

Returned a `message` that begins "We found no license matching this query in our
index. This is NOT a determination that the business is unlicensed.", three
`suggestions` (try the license number on the bid, try the legal name on the
contract, the contractor may be licensed elsewhere), and `official_lookup:
"https://www.tdlr.texas.gov/LicenseSearch/"`.

Then: relay the message, the suggestions, and the link. Say "no match in the index
we checked", never "unlicensed".

Where Plumbline's coverage of a state is partial (Maryland, North Carolina, and
Massachusetts on 2026-09-28), the no-match `message` opens with a warning, for
example "Massachusetts coverage is PARTIAL: the first full roster sweep is pending.
A no-match here means very little yet; use the OPSI lookup directly." A `match` or
`candidates` result there carries the warning in `coverage_note` instead, for
example "The records listed may not be all the boards hold". Relay the warning
whenever it is present.

### E. `jurisdiction_not_covered`

```
GET https://contractors.agentlookups.ai/v1/check?name=acme+plumbing&jurisdiction=US-AZ
```

Returned "We do not yet cover AZ. Use the official state lookup provided with this
response for an authoritative answer. Absence from our index is NOT a
determination of licensure." with `official_lookup` set to the Arizona ROC
contractor search. For `US-NY` (or `New York`) the message says to try `NYC` and
that its link "is the New York Department of State's consumer-protection division,
not a license lookup". For GA, SC, UT, and WI (as tested) there is no link, and the
message says "We do not have its official lookup URL on file; check the
jurisdiction's licensing agency directly."

Then: relay the message wording exactly, and never present the link as a license
lookup unless the message says it is one. The MCP tool description: "the response
message states exactly what the link is, or that none is on file, and for some
jurisdictions the link is a consumer-protection or licensing-board page, not a
license lookup".

## What an active record means, and what it does not

- `status` is Plumbline's grouping (its methodology page: "We also group statuses
  as active, inactive, suspended or revoked to help compare records"); `raw_status`
  is the board's own word (CSLB says "CLEAR"). Give both. `status` can also be
  `unknown` when the board's word does not fit a group (observed 2026-09-28: "Not
  In Good Standing" in OK, "Referred to Enforcement" in DC). Then relay
  `raw_status` exactly and send the user to the board's live page; never read
  unknown as active.
- **Read the credential block before calling anything a license.** From llms.txt:
  "inspect credential.kind, credential.label and credential.note before describing
  a record as a trade license. Business licenses, registrations and bond filings
  are distinct." Kinds observed: `license`, `registration`, `business_license`,
  `bond`, `credential`. Relay the `note` as written; examples seen: "This is a bond
  filing, not a state mechanical contractor license. Check any applicable local
  requirements separately." (Minnesota) and "This registration is separate from
  skilled-trade licenses such as electrical, plumbing and HVAC." (Connecticut home
  improvement).
- **Not an endorsement, and not proof it covers this job.** A `license` record's
  note says: "Check the exact classification and holder. An active record is not an
  endorsement or proof that this credential covers your job." A generic
  `credential` record's note says: "Read the classification and source wording,
  then verify what this credential covers with the issuing agency. This record is
  not an endorsement."
- **Not insurance or bonding.** No record shows a current insurance policy; the
  user needs the certificate of insurance (checklist step 2). A `kind: bond`
  record is a bond filing as of its snapshot date, nothing more.
- **Not a clean history.** Discipline, liens, permits, courts, and business
  registration come back `not_checked` with notes such as "Absence here means
  nothing." Relay that. (`registries` in `/v1/coverage` was empty on 2026-09-28;
  llms.txt: "no_match_in_registry is NOT a determination that a business is
  unregistered".)
- **Not today's status.** `as_of` and `provenance.snapshot_id` date the snapshot
  (`20260927T...` means 2026-09-27). "confidence: verified describes indexed
  source evidence, not an endorsement or a fresh agency check."
- `entity.kind` is `business` or `individual`. If the board publishes contact
  details for a business, llms.txt says to relay them as what "the board's record
  lists", "never as verified-current; it is for the user to reach the business,
  not a lead." Individuals' records never carry contact details.

## Before signing: the site's checklist

Source: `https://contractors.agentlookups.ai/how-to-check-a-contractor/` (record
pages link to it). Its steps, in short; send the user the page for the full text:

1. Look up the license, then confirm on the board's live page. "A business
   registration is different from a trade license, and a license may cover only
   certain work."
2. Get the certificate of insurance before signing; it should name the same
   contractor as the license, and dates that cover the job. Call the insurance
   agent on it to confirm the policy is in force.
3. Confirm who pulls the permit. "Be wary if a contractor asks you to pull the
   permit yourself."
4. Match the names: "the name on the license, the name on the contract, and the
   name you make payments to."
5. Put the license number on the contract.
6. Keep the deposit small, pay by check or card, tie payments to finished stages,
   "Never pay in full up front." Deposit limits are state law. The page's one
   verified example, quoting California's board: "The down payment cannot be more
   than $1,000 or 10 percent of the contract price, whichever is less, for a home
   improvement job or swimming pool, excluding finance charges" (CSLB, checked
   2026-08-05). For other states: "ask the licensing board behind your state's
   official lookup; we do not guess at laws we have not checked." Never state
   another state's limit from memory. To find it, use `find-the-law`, which quotes
   the statute with its citation and current-through date, or ask the board. Use
   the GroundRules MCP tools (`law_search`, then `law_get_section` for the full
   text) so the statute comes back verbatim; REST through a web-fetch tool may
   summarize it, so use REST only as a fallback. Observed 2026-09-28: `law_search`
   with `{"q": "home improvement contract deposit", "jurisdictions": "us-md"}` (and
   the REST fallback `GET https://law.agentlookups.ai/v1/search?q=home+improvement+contract+deposit&jurisdictions=us-md`)
   led with `Md. Code, Business Regulation § 8–617`.
7. Check the license again the day you sign, and before the final payment if the
   job runs long.

And: "A license does not tell you how well someone works. Ask for references from
recent customers, and look at a finished job if you can."

## If the job goes wrong

- A `match` carries `complaint_route` where Plumbline has one: `url`, `kind`
  (`online_form`, `pdf_form`, `instructions_page`, or
  `consumer_protection_fallback`), sometimes a `note`, and in some states a
  `guaranty_fund_url` for a homeowner recovery or guaranty fund (observed
  2026-09-28 in CT, FL, MA, MD, MN, NC, NYC, VA, WA). Relay the link and the note.
  Do not promise a payout; the fund's own rules decide. The record page
  (`human_page`) shows the complaint link but may not show the fund link, so relay
  `guaranty_fund_url` from the API response. A missing `guaranty_fund_url` does not
  mean the state has no fund.
- `consumer_protection_fallback` comes with the note "The licensing program itself
  publishes no complaint process; this is the state consumer-protection intake."
- A `match` can come back with no `complaint_route` key at all (observed
  2026-09-28 in KY, NV, ND, OH, VT, and MD-HOWARD). Then say no route is on file
  and point to the board behind `official_lookup`.
- **Wrong data is different from a bad contractor.** If Plumbline's record itself
  is wrong or stale, use `dispute_url` (`https://contractors.agentlookups.ai/dispute`)
  or email support@agentlookups.ai. Complaints about the contractor's work go to the
  board, not to Plumbline.

## Honesty rules (verbatim, relay these, never override them)

From `https://contractors.agentlookups.ai/llms.txt`, fetched 2026-09-28:

- "A no_match_in_index result is NOT a determination that a business is unlicensed."
- "We report what the public record says, with its date; the issuing agency's live
  page is authoritative for today."
- "We report occupational-license records, including licenses individuals hold in
  their own name (journeyman/master trades). We NEVER build person dossiers: an
  individual's page is their one license record as the board publishes it, with no
  aggregation beyond it, and it is never for any FCRA purpose."

From every API response's `terms` field: "Public occupational-license records
(business and individually held), republished with sources. This is NOT a consumer
report under the FCRA or state consumer-reporting laws, and must not be used for
decisions about employment, credit, insurance, or housing. We never aggregate
beyond the license record. The issuing agency's live page is authoritative."

When a response's own wording differs from this page, use the response's wording.

## Output contract

Every answer about a contractor carries all of:

1. **Source**: the board, from `source_label` (for example "California Contractors
   State License Board (CSLB)"), and the credential label. See below for results
   that carry no `source_label`.
2. **Snapshot date**: from `as_of` or the `summary`, said as "board snapshot dated
   2026-09-27", never "currently". Never make up a date.
3. **Coverage caveat**: what the check could not see (the `not_checked` blocks, the
   credential note, any `coverage_note` or partial-coverage warning, the no-match
   rule) in the answer, not in a footnote.
4. **Official link**: `official_lookup` (and `human_page` on a match), with the
   instruction to check the board's live page on the day they sign.
5. **On a match, the complaint route** (`complaint_route.url`, and
   `guaranty_fund_url` if present), so the user knows where to go if the job fails.
6. **When `entity.kind` is `individual`**, the `terms` sentence verbatim: "This is
   NOT a consumer report under the FCRA or state consumer-reporting laws, and must
   not be used for decisions about employment, credit, insurance, or housing."

Where the source and date come from, by result type:

- `match`: `source_label` and `as_of` (or the `summary`).
- `candidates`: each row's `source_label` and `as_of`; over MCP, when every row
  shares them, they sit once in `same_for_all_rows`.
- `no_match_in_index`: the response has no `source_label`, `as_of`, or `summary`.
  Name the source as "Plumbline's index of <ST> board records" plus
  `official_lookup`, and take the date from that jurisdiction's `snapshot_run` in
  `/v1/coverage` (as of 2026-09-28, both TX sources read `20260925T...`: "index
  snapshot dated 2026-09-25").
- `jurisdiction_not_covered`: say Plumbline holds no records for that place, so
  there is no date; relay the `message` and the link as worded.

A short relay for example A: "California's contractor board (CSLB), snapshot
dated 2026-09-27: license 604196, ROTO-ROOTER of Oakdale, status CLEAR (active),
classes A, C36, C42, HAZ, expires 2028-10-31. This is the license record only. It
does not show insurance, discipline, liens, or lawsuits, and it does not prove the
license covers your job. Check the board's live page before you sign:
https://www.cslb.ca.gov/onlineservices/checklicenseII/checklicense.aspx.
Complaints go to https://www.cslb.ca.gov/Consumers/Filing_A_Complaint/."

## What this cannot answer, and where to send the user

- **Insurance in force**: the certificate of insurance and a call to the agent on it.
- **Permits for this job**: the city or county permit office decides;
  `home-project-permits` quotes the law on permits for the job.
- **Workmanship**: references and finished work; no score exists here.
- **Deposit rules outside California**: `find-the-law` for the state statute, or
  the state's licensing board.
- **Discipline, liens, lawsuits**: the state's licensing board; the county
  recorder or courts for liens and suits.
- **Anything about a person** beyond the one license record the board publishes:
  decline, and say these services never build person dossiers.
