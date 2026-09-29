---
name: maryland-assessment-appeal
description: "Check if a Maryland home's property tax assessment is fair next to similar homes, and when and how to appeal it, with Overassessed (free during beta). Use for \"is my assessment too high\", \"should I appeal my reassessment notice\", \"SDAT appeal deadline\", \"make me an appeal packet\". Maryland only. Never who owns a home or anything about a person; not an appraisal; not legal or tax advice."
license: MIT
metadata:
  author: TopHat Monkey Software LLC
---

# Maryland assessment appeal

Overassessed compares one Maryland home's assessment with genuinely similar homes on
the state's own roll ("SDAT, 2.4M parcels, refreshed monthly", per its llms.txt),
says where the home stands, and gives Maryland's appeal windows and filing links.
It is free during beta, needs no account and no key, and is read-only; its home page
says "Currently free in beta. Features and pricing may change." It stores no owner
names and takes no name search. Every example below was run live on 2026-09-28 and
returned the shape shown; the roll refreshes monthly, so numbers can drift. Normal use
is a few lookups per question. Its docs say "Rate limits apply."; a burst of 30
parallel calls drew HTTP 429 on 9 of them (2026-09-28), so slow down if you see it.

This skill covers the assessment question alone, whoever asks it. If the user wants
the full set of checks on a house they are buying, renting, or listing, use
`home-purchase-due-diligence`, `rental-property-check`, or
`listing-evaluation-for-agents`, and use this skill for the assessment part.

Most homes that get a verdict land in the normal range, and "Not enough similar homes"
is a common answer. When the home is in range, say so plainly; do not talk the user
into an appeal.

## When to use

Homeowners, buyers, landlords, and agents all ask it. Typical requests:

- "Is my property assessment too high?" "Is my house (or rental) over-assessed?"
- "My reassessment notice went up, should I appeal?" "My property taxes keep going up."
- "How do I appeal my SDAT assessment?" "What's the deadline to appeal my assessment?"
- "I just bought in Maryland, can I appeal?"
- "The taxes on the house we're buying look high." "Does this listing's assessment
  look high?"
- "Make me an appeal packet."

Not for:

- A property in any other state: say it is not covered and stop.
- A full check of a house being bought, rented, or listed: use the property skills
  named above, and this skill for the assessment part only.
- What a home would sell for: this is never an appraisal.
- Tax rates, bills, or credits.
- Legal or tax advice, or the odds of winning an appeal.
- Anything about a person, like who owns a home: this is not a consumer report under
  the FCRA.

A no-match proves nothing.

## Never use this skill for

- **A property outside Maryland.** Say "Overassessed covers Maryland only" and stop;
  point to that county's assessor. Do not run the check anyway: without a zip, an
  address outside Maryland can match a Maryland parcel with the same street string.
  Observed 2026-09-28: `1600 PENNSYLVANIA AVE`, the White House's street address, sent
  with no zip, matched a parcel in Baltimore City.
- **Anything about a person**: who owns a house, what someone owns. Never add an owner
  name from another source. The shared terms, verbatim: "This is not a consumer report
  under the Fair Credit Reporting Act or any state consumer-reporting law. Do not use
  it to decide on anyone's employment, credit, insurance, or housing."
- **A value estimate.** The service is "never an appraisal".
- **Legal or tax advice**, tax bill math, tax credits, or whether an appeal will win.
- **A score or grade.** Relay the branch in the service's words; never turn it into a
  number, grade, or odds.

## Calls

| Step | URL (GET) |
|---|---|
| Check by address | `https://overassessed.agentlookups.ai/v1/check?address=<STREET AS THE ROLL WRITES IT>&zip=<ZIP>` |
| Check by account | `https://overassessed.agentlookups.ai/v1/check?acct=<ACCOUNT_ID>` |
| Verdict page for humans, with this home's own appeal dates | `https://overassessed.agentlookups.ai/check?acct=<ACCOUNT_ID>` |
| Print-ready evidence packet | `https://overassessed.agentlookups.ai/packet?acct=<ACCOUNT_ID>` |

MCP: `https://overassessed.agentlookups.ai/mcp` (Streamable HTTP, no auth; server
`overassessed` in this plugin). One read-only tool, `check_assessment`, titled "Check
fairness of a Maryland property assessment", with three optional string arguments,
described by `tools/list` as: `address` "Street address in the form the roll uses,
e.g. 1 STATE CIR, without unit number or city"; `zip` "5-digit zip; optional, but the
same street address occurs in several Maryland towns, and a zip narrows the match";
`acct` "SDAT account id, as returned in parcel.account_id or in a multiple_matches
result; when present it takes precedence over address". A bare `tools/call` works
without `initialize`:

```
curl -sS -X POST https://overassessed.agentlookups.ai/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"check_assessment","arguments":{"address":"1 STATE CIR","zip":"21401"}}}'
```

The result holds the REST JSON twice, in `content[0].text` and `structuredContent`.
`isError` stays `false` even on a no-match or a multiple match; look for an `error` or
`multiple_matches` key instead. With no `address` and no `acct`, the tool returns
`isError` `true` and only the text "provide address (plus zip), or acct from a prior
multiple_matches result".

## Workflow

### 1. Get the address the way the roll writes it

Street only (number, name, suffix) plus the zip as its own field, every time. No unit
number, no city, no state. Observed: case, periods, and "Street" vs "ST" do not matter
("1 State Circle" matched). A city or unit in the string gives a no-match, and so can
a direction word the roll leaves out: Baltimore City Hall is `100 HOLLIDAY ST` on the
roll, and `100 N HOLLIDAY ST` did not match. When stuck, ask the user to find the home
in SDAT's own search (https://sdat.dat.maryland.gov/RealProperty/) in a browser and
copy its spelling; that site answered scripted fetches with HTTP 403 and a Cloudflare
block page (2026-09-28), so do not rely on fetching it yourself.

### 2. Run the check

Worked example with the Maryland State House, a public building and the service's own
sample address (substitute the user's street and zip):

```
GET https://overassessed.agentlookups.ai/v1/check?address=1+STATE+CIR&zip=21401
```

HTTP 200 with five keys:

- `parcel`: `account_id` ("020600002182004" here), `address`, `city`, `county`, `zip`,
  `market_value` (full market value, "what an appeal argues about" per the verdict
  page), `land_value`, `improvements_value`, `taxable_now`, `sqft`, `acres`,
  `year_built`, `reassessed_cycle` ("2024.01", which the verdict page words as "Last
  reassessed January 2024"). No owner field.
- `uniformity`: `Branch`, `BandSize` (similar homes found), `PctFMV` (percent of them
  assessed lower), `MedianFMV`, `MedianSalesRatio`, `RatioSales` (sales behind that
  ratio), `COD`, and more. Here: `Branch` "insufficient", `BandSize` 14, every other
  number 0. Per `tools/list`, insufficient "means fewer than 25 similar homes exist for
  a fair comparison, which is no finding either way."
- `plain`: sentences to relay (step 3). `appeals`: windows and links (step 4).
  `honesty`: two sentences to relay verbatim.

`GET https://overassessed.agentlookups.ai/v1/check?acct=020600002182004` returns the
same record. `acct` takes the id exactly as the service printed it, digits only; a
dashed form did not match. Before relaying, check that `parcel.city`, `county`, and
`zip` match the home the user meant.

Other outcomes:

- **HTTP 300, `multiple_matches`**: several parcels share the street string. Observed:
  `GET https://overassessed.agentlookups.ai/v1/check?address=100+MAIN+ST` (no zip)
  returned 10 parcels in towns across the state, each with `account_id`, `address`,
  `city`, `zip`, `land_use`, `market_value`, `sqft`, `year_built`, and a `hint`:
  "several parcels share this address; re-query with acct=<account_id>". Show the user
  city, zip, land use, size, and year built; ask which is theirs; re-query with
  `acct=`. Condo buildings return this even with the zip, since units share the street
  address and the list shows no unit numbers. In the words of `tools/list`, the list
  then "holds 10 of them, with no unit numbers and no total count, and a zip does not
  narrow it further; the official SDAT record has a record for every unit." So a unit
  may be missing; then have the user get the account from SDAT's search. Outside
  Baltimore City, every id observed was the 2-digit county code, the 2-digit district,
  and the account number, run together. Baltimore City ids carry no county code: City
  Hall's is `04121302001`, the ward, section, block, and lot run together.
- **Condo units**: every unit tried (48 units in 8 buildings across 4 counties,
  2026-09-28; a later spot check of 9 units the same day agreed) came back
  `insufficient` with `BandSize` 0. Tell a condo owner this before they hunt for their
  unit; the appeal windows and the SDAT record still apply.
- **HTTP 404, `error`**: "no parcel matched that address; include the zip, and write it
  as the assessment roll does (e.g. 1 STATE CIR)". Rewrite per step 1 and retry. An
  `acct=` that matches nothing returns the same text, although no address was sent;
  then recheck the account digits. A no-match says nothing about the home or its
  assessment.

### 3. Relay the verdict, branch by branch

Lead with `plain.headline` word for word, then whichever other `plain` sentences are
present.

| `Branch` | `plain.headline`, verbatim | Then |
|---|---|---|
| `fair` | "This assessment sits in the normal range for similar homes." | Say so plainly. The packet page is titled "Assessment comparison sheet". |
| `review` | "This assessment is on the high side for similar homes: worth a closer look." | Relay the windows (step 4) and the packet ("Appeal evidence packet"). Filing is the user's call. |
| `elevated` | "This assessment is higher than almost all similar homes." | Same as review. |
| `insufficient` | "Not enough similar homes to compare fairly." | No verdict either way. Relay `plain.body`, e.g. the State House's "We found only 14 homes similar enough (same reassessment group, similar size, lot, and age, same zip). An honest answer needs more; the official record link below is the next stop." With `BandSize` 0 it opens "We couldn't find any other homes similar enough" instead. The packet page says "We can't build an honest packet here". |

The `insufficient` branch carries only `headline` and `body`. On the other three,
more `plain` keys appear when they apply. As worded on 2026-09-28: `position` ("Out of
237 similar homes, about 1 in 10 are assessed lower than this one and 9 in 10
higher."); `level`, only when the group has enough recent sales ("As a group, these
homes are assessed at about 80% of what they actually sell for (based on 20 recent
sales)."); `phase_in`, when `taxable_now` is below `market_value` ("The assessment is
still phasing in: increases spread over three years, so the taxable amount steps up
again the next year or two even with no new assessment. Decreases apply
immediately.").

Zeros in `uniformity`: on the `insufficient` branch every number but `BandSize` reads
0, meaning not computed. On the other branches, `MedianSalesRatio` and `COD` read 0
"when RatioSales is below 10, meaning no ratio is published" (`tools/list`); `level`
is then absent, and `RatioSales` still shows the small count, such as 8. `PctFMV` 0 on
a fair, review, or elevated parcel is real: no similar home is assessed lower than
this one. Set `market_value` beside `MedianFMV` only when `MedianFMV` is above 0.

Most commercial parcels tried (14 of 16) and every tax-exempt parcel tried came back
`insufficient` (2026-09-28); a few commercial parcels get a verdict, still worded as "similar
homes". "Similar", as the packet defines it: "same reassessment group, same zip code,
same land-use class, living area within 30%, lot size within 50%, and year built
within 20 years."

### 4. Appeal windows and where to file

Relay the `appeals` block as the response words it. Observed 2026-09-28:

- `window_notice`: "within 45 days of the date on a reassessment notice (notices mail in late December)"
- `window_petition`: "for the two years a home is not reassessed: a Petition for Review filed by the first working day after January 1"
- `window_purchase`: "within 60 days of the transfer, for a purchase transferred between January 1 and June 30"
- `portal`: https://assessmentappeals.dat.maryland.gov/start.aspx
- `petition_pdf`: https://dat.maryland.gov/SDAT%20Forms/petitnrv.pdf
- `process`: https://dat.maryland.gov/realproperty/Pages/Assessment-Appeal-Process.aspx

The JSON gives only statewide rules. For "can I still appeal, and by when", fetch
`https://overassessed.agentlookups.ai/check?acct=<ACCOUNT_ID>` and quote its "For this
home:" paragraph. It works out, from the reassessment cycle, this home's last notice,
when that 45-day window closed, and the next option. It offers a Petition for Review
only in the two years the home is not reassessed; in the winter its next reassessment
notice mails, it offers only that notice. As of 2026-09-28:

- The State House (`acct=020600002182004`, last reassessed January 2024): "For this
  home: This home's last reassessment notice mailed in late December 2023, so its
  45-day appeal window closed in February 2024. The next free option: the next
  reassessment notice should mail in late December 2026, and the date printed on it
  starts a fresh 45-day appeal window."
- Baltimore City Hall (`acct=04121302001`, last reassessed January 2025): "For this
  home: This home's last reassessment notice mailed in late December 2024, so its
  45-day appeal window closed in February 2025. The next free option: file a Petition
  for Review by January 4, 2027. The next reassessment notice should mail in late
  December 2027, starting a fresh 45-day window."

Then tell the user these dates are the service's estimate. The service's explainer
words the first window "Within 45 days of the date on your reassessment notice", so the
date printed on the user's notice and SDAT's `process` page decide. For a purchase,
the 60 days count from the transfer, a date the service does not know.

Take only that paragraph from the page; every number and the verdict come from the
JSON. On an `insufficient` parcel with similar homes found, the page can still draw a
similar-homes median and range that the JSON leaves at 0 (as of 2026-09-28 the State
House page showed "similar-homes median $758,600"). Never relay them.

The page calls the three windows "all free to file". What follows a filing, verbatim
from the service's explainer (`/is-my-maryland-property-assessment-too-high`): "The
first stop is an informal hearing with an assessor, usually about 15 minutes. If you
disagree with the result, you can appeal to your local Property Tax Assessment Appeals
Board within 30 days of the date on the final notice from that hearing, then to the
Maryland Tax Court within 30 days of the date of the board's order. Neither charges a
fee."

### 5. Hand over the packet

The packet URL is `https://overassessed.agentlookups.ai/packet?acct=<ACCOUNT_ID>`. What
it holds follows the branch:

- **review or elevated**: an "Appeal evidence packet". Give the user the link. It is an
  HTML page with the property record, the verdict in plain words, the 10 similar homes
  closest in size (address, total value, size, year built, land, buildings, dollars per
  square foot; no owners), this home's appeal dates, and the filing links. In its
  words: "Your browser's print button turns it into the paper or PDF exhibit you attach
  to Maryland's Petition for Review or bring to a hearing. Printing happens in your
  browser; we keep no copy."
- **fair**: the same page headed "Assessment comparison sheet". Offer it only if the
  user wants it.
- **insufficient**: never hand it over. The page is a refusal headed "We can't build
  an honest packet here" (its browser tab still reads "Appeal evidence packet"). The
  State House shows it:
  `https://overassessed.agentlookups.ai/packet?acct=020600002182004`.

## Honesty rules (verbatim; relay them, never soften or strengthen them)

From every `/v1/check` response, `honesty` array:

- "This compares the official assessment record with similar homes in the same reassessment group; it is a fairness snapshot of public data, never an appraisal."
- "Similar-home comparisons use size, lot, age, and neighborhood bands; no comparison is perfect. The official record is authoritative: https://sdat.dat.maryland.gov/RealProperty/"

From the MCP server's `initialize` instructions: "Each single-parcel result carries
honesty notes stating that the check is a fairness snapshot of the public record, not
an appraisal, and that the official SDAT record is authoritative. The check is not
legal advice." From the fair, review, and elevated packets: "It is not an appraisal
and not legal advice." From the shared terms (https://agentlookups.ai/terms/): "Not
advice. Nothing here is legal, financial, medical, or professional advice." and "A
\"no records found\" result proves nothing."

When a live response words these differently, use the live wording.

## Output contract

Every answer carries all four, taken from the response, not from memory:

1. **Source**: Overassessed, from Maryland's SDAT assessment roll; the account id,
   county, and last reassessment.
2. **Snapshot date**: the JSON has none. Quote the "Roll data as of <Month YYYY>" line
   from the home page (https://overassessed.agentlookups.ai/), which shows it for any
   branch, from the MCP server's `initialize` instructions, or from a fair, review, or
   elevated packet (all three read "Roll data as of September 2026" on 2026-09-28); the
   verdict page and the insufficient packet carry none. If you fetched none of them,
   "the roll refreshes monthly; checked <today's date>". Never "currently".
3. **Coverage caveat**: both `honesty` sentences, `BandSize`, the sales count only when
   `plain.level` is present (that sentence carries it), and "not legal or tax advice".
4. **Official links**: https://sdat.dat.maryland.gov/RealProperty/, the `appeals`
   links, and the packet link for review or elevated.

Pick the template by `Branch`. Leave out any line marked "only" when its condition
fails; never print a 0 or an empty slot.

Fair, review, or elevated:

```
<plain.headline> (Overassessed, Maryland SDAT roll; account <account_id>, <county>)
<plain.position> <plain.level, only if present> <plain.phase_in, only if present>
Assessed at $<market_value>, last reassessed <Month YYYY>; similar-homes median $<MedianFMV>.
Appeal: <"For this home:" paragraph>. These dates are the service's estimate; the date on your notice and SDAT decide. File at <portal> or with <petition_pdf>; steps: <process>.
Packet to print (only for review or elevated, or fair if asked): https://overassessed.agentlookups.ai/packet?acct=<account_id>
Roll data as of <Month YYYY> (or: roll refreshes monthly; checked <date>).
"<honesty 1>" "<honesty 2>" Not legal or tax advice.
```

Insufficient (no position, median, or packet):

```
<plain.headline> (Overassessed, Maryland SDAT roll; account <account_id>, <county>)
<plain.body>
Assessed at $<market_value>, last reassessed <Month YYYY>. Official record: https://sdat.dat.maryland.gov/RealProperty/
Appeal: <"For this home:" paragraph>. These dates are the service's estimate; the date on your notice and SDAT decide. File at <portal> or with <petition_pdf>; steps: <process>.
Roll data as of <Month YYYY> (or: roll refreshes monthly; checked <date>).
"<honesty 1>" "<honesty 2>" Not legal or tax advice.
```

## What this cannot answer, and where to send people

- **What the home would sell for.** The explainer says that question "needs an
  appraisal or recent sales"; this service checks only the similar-homes question.
  Send them to an appraiser or recent local sales.
- **Tax rates, the bill, tax credits** (homestead, homeowners'): the county finance
  office and SDAT.
- **Whether to appeal or what to argue**: the user's call; for advice, a real estate
  attorney. The `process` link explains each step.
- **A roll record that looks wrong** (size, lot): the SDAT record is authoritative;
  raise it with SDAT. A wrong record in this service: corrections@agentlookups.ai.
- **Any other state**: not covered; the county assessor's own record.
- **What is near the home** (environmental records, for a buyer): the
  `whats-near-this-address` skill. The verdict page links there too ("Buying rather
  than appealing? Check what's near this address").
- **Anything about a person**: decline (see Never use this skill for).
