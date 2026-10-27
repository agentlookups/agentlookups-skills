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
names and takes no name search. Every example below was rerun live on 2026-10-27,
returning the shape shown, unless it carries an earlier date. The roll refreshes
monthly, so numbers can drift. Normal use is a few lookups per question.
Its docs say "Rate limits apply."; a burst of 30 parallel calls drew HTTP 429 on 9 of
them (2026-09-28), so slow down if you see it. Since 2026-10-27 a 429 is JSON (an
`error` sentence and `retry_after_seconds` 1) with a `Retry-After: 1` header: wait that
long before the next call.

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
  point to that county's assessor. Do not run the check anyway. The service refuses
  most such calls: `1600 PENNSYLVANIA AVE`, the White House's street address, sent
  alone got HTTP 400 "Overassessed covers Maryland only; add the zip, or the Maryland
  city or county, to the street (e.g. 1 STATE CIR, ANNAPOLIS or zip 21401), so the
  match stays in Maryland"; `1600 Pennsylvania Ave, Washington, DC` got HTTP 404
  "Overassessed covers Maryland only; District of Columbia is not covered yet"; with
  its zip, 20500, it got HTTP 404 "that zip is outside Maryland; Overassessed covers
  Maryland only, from the state's own assessment roll (SDAT)". But the same street
  sent with MD, or with a Maryland town of the same name, matches Maryland parcels.
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
| Check by address | `https://overassessed.agentlookups.ai/v1/check?address=<STREET AS THE ROLL WRITES IT>&zip=<ZIP>`, or the town in the address: `address=<STREET>, <TOWN>, MD <ZIP>` |
| Check by account | `https://overassessed.agentlookups.ai/v1/check?acct=<ACCOUNT_ID>` |
| Verdict page for humans, with this home's own appeal dates | `https://overassessed.agentlookups.ai/check?acct=<ACCOUNT_ID>` |
| Print-ready evidence packet | `https://overassessed.agentlookups.ai/packet?acct=<ACCOUNT_ID>` |

MCP: `https://overassessed.agentlookups.ai/mcp` (Streamable HTTP, no auth; server
`overassessed` in this plugin). One read-only tool, `check_assessment`, titled "Check
fairness of a Maryland property assessment", with three optional string arguments,
described by `tools/list` as (2026-10-27): `address` "Maryland street address in the
form the roll uses (e.g. 1 STATE CIR), with any unit after the street (100 HARBORVIEW
DR UNIT 204), followed by its Maryland city or county, the state, or the zip (e.g. 1
STATE CIR, ANNAPOLIS, MD 21401); an address with no Maryland zip, city, county or
state, here or in the zip input, is not matched. An SDAT account id sent here alone
(0304121302 001, 160500305131) is read as acct"; `zip` "5-digit zip; a ZIP+4 is read
as its first five digits, and a whole JSON number up to 99999 is read as a zip with
its leading zeros restored (1234 is 01234). Optional when the address carries a
Maryland zip, city, county or state; a zip here takes precedence over one in the
address. The same street address occurs in several Maryland towns, and a zip narrows
the match"; `acct` "SDAT account id as returned in parcel.account_id or in a
multiple_matches result: letters and digits (e.g. 020600002182004), with one space
before the lot in Baltimore City (e.g. 0304121302 001); spaces in it are ignored, so
0304121302001 reads the same; when present it takes precedence over address. A JSON
number is read as its digits, or with the leading zero a number cannot hold when that
is the id on the roll; a number that names no single parcel is an error asking for the
id as a string". Send `acct` as a JSON string all the same: a number cannot carry
Baltimore City's space. A bare `tools/call` works without `initialize`:

```
curl -sS -X POST https://overassessed.agentlookups.ai/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"check_assessment","arguments":{"address":"1 STATE CIR","zip":"21401"}}}'
```

A result (one parcel, or a `multiple_matches` list) holds the REST JSON twice, in
`content[0].text` and `structuredContent`, with `isError` `false`. Since 2026-10-27 a
refusal or a no-match returns `isError` `true` and only the error sentence as text, the
one REST puts in its `error` field; with no `address` and no `acct` that is "provide
address (plus zip), or acct from a prior multiple_matches result".

## Workflow

### 1. Get the address the way the roll writes it

The street (number, name, suffix), any unit after it, then the zip, or the Maryland
city or county, in the address or with the zip as its own field. Observed: case,
periods, and "Street" vs "ST" do not matter ("1 State Circle" with zip 21401
matched); the whole line `100 Holliday St, Baltimore, MD 21202` matched, and so did
`101 N COURT ST, FREDERICK` with no zip. A street with no Maryland zip, city, county
or state gets HTTP 400 (step 2); MD alone searches the whole state. Spelled-out
directions and street types read as the roll's short forms (`4620 North Park Ave` with
zip 20815 finds 4620 N PARK AVE), and an account number from the notice, typed alone as
the address, finds its parcel. A direction word the roll leaves out gives a no-match: Baltimore City Hall is `100 HOLLIDAY ST` on the
roll, and `100 N HOLLIDAY ST` did not match. If a town misses, try the zip: the roll
may record the parcel under another town name. When stuck, ask the user to find the home
in SDAT's own search (https://sdat.dat.maryland.gov/RealProperty/) in a browser and
copy its spelling; that site answered scripted fetches with HTTP 403 and a Cloudflare
block page (2026-09-28), so do not rely on fetching it yourself.

### 2. Run the check

Worked example with the Maryland State House, a public building and the service's own
sample address (substitute the user's street and zip):

```
GET https://overassessed.agentlookups.ai/v1/check?address=1+STATE+CIR&zip=21401
```

HTTP 200 with six keys (2026-10-27):

- `parcel`: `account_id` ("020600002182004" here), `address`, `city`, `county`, `zip`,
  `market_value` (full market value, "what an appeal argues about" per the verdict
  page), `land_value`, `improvements_value`, `taxable_now`, `sqft`, `acres`,
  `year_built`, `reassessed_cycle` ("2024.01", which the verdict page words as "Last
  reassessed January 2024"). No owner field. A condo unit can also carry `unit`.
- `uniformity`: `Branch`, `BandSize` (similar homes found), `PctFMV` (percent of them
  assessed lower), `MedianFMV`, `MedianSalesRatio`, `RatioSales` (sales behind that
  ratio), `RatioFromYear` and `RatioToYear` (the first and last years of the sales
  window, 2024 and 2026 here), `COD`, `NotCompared`, and more. Here: `Branch` "fair",
  `BandSize` 26, `PctFMV` 69.2, `MedianFMV` 537150, `RatioSales` 4 (so
  `MedianSalesRatio` and `COD` read 0), and `NotCompared` "year built": the roll
  records no year built for the State House, so that test was skipped. Per
  `tools/list`, NotCompared names a skipped test "(lot size or year built; both as lot
  size, year built)" and "is absent when every test ran".
- `roll`: the roll's dates. Here `{"as_of": "2026-10-06", "pulled": "2026-10-27",
  "overdue": false}`. Per `tools/list`, "as_of is the date the state last updated the
  roll and pulled is the date this copy was taken (both YYYY-MM-DD), overdue is true
  when this copy is more than 35 days old, and a field the server does not know is left
  out." 
- `plain`: sentences to relay (step 3). `appeals`: windows and links (step 4).
  `honesty`: two sentences to relay verbatim.

`GET https://overassessed.agentlookups.ai/v1/check?acct=020600002182004` returns the
same record. `acct` takes the id as the service printed it; spaces in it are
ignored. A dashed form (`acct=02-06-00002182004`) got HTTP 400: "acct is not an SDAT
account id: an id is letters and digits, as parcel.account_id or a multiple_matches
result prints it, and spaces in it are ignored (e.g. 020600002182004, or 0304121302
001 in Baltimore City)". Before relaying, check that
`parcel.city`, `county`, and `zip` match the home the user meant.

Other outcomes:

- **HTTP 300, `multiple_matches`**: several parcels share the street string. Observed:
  `GET https://overassessed.agentlookups.ai/v1/check?address=100+MAIN+ST%2C+MD` (MD
  alone, so statewide) returned `total` 13 and the first 10 parcels, in towns across
  the state, each with `account_id`, `address`, `city`, `zip`, `land_use`,
  `market_value`, `sqft`, `year_built`, and a `hint`: "13 parcels share this address;
  this list holds the first 10; re-query with acct=<account_id>" (with 10 or fewer,
  "several parcels share this address; re-query with acct=<account_id>"). The same
  `roll` object comes with the list. Show the user city, zip, land use, size, and year built; ask which is theirs;
  re-query with `acct=`. Condo buildings return this even with the zip, since units
  share the street address. In the words of `tools/list`, "The list is in unit order,
  each option carries unit (the unit as the roll writes it, such as UNIT 204) when the
  roll records one, and total gives how many parcels share the address; when more
  than 10 do (a condominium building, for instance), the list holds the first 10." So
  ask a condo owner for the unit and send it after the street (`100 HARBORVIEW DR UNIT
  204` or `100 HARBORVIEW DR #204`). Outside Baltimore City, every id observed was the
  2-digit county code, the 2-digit district, and the account number, run together.
  Baltimore City ids changed with the October 2026 roll: City Hall's was
  `04121302001` and is now `0304121302 001`, the county code 03, then the old ward,
  section, and block, one space, and the lot. An id saved in the old form gets the
  no-parcel error below; look the address up again.
- **Condo units** usually get a verdict, compared without the lot test, since the
  roll records no lot for most units: 25 of 27 residential condo units in three
  buildings in Baltimore City and Montgomery County came back `fair` or `review`
  (2026-10-27), most with `NotCompared` "lot size".
- **HTTP 400 or 404, `error`** (texts as of 2026-10-27): with no Maryland zip, city,
  county or state, HTTP 400 "Overassessed covers Maryland only; add the zip, or the
  Maryland city or county, to the street (e.g. 1 STATE CIR, ANNAPOLIS or zip 21401),
  so the match stays in Maryland". The rest are HTTP 404. With MD alone, "no parcel
  matched that address; include the zip, and write it as the assessment roll does
  (e.g. 1 STATE CIR)"; with a zip, "no parcel matched that address in that zip; write
  the street as the assessment roll does (e.g. 1 STATE CIR), with any unit after it
  (100 HARBORVIEW DR UNIT 204), and check that the zip is the one the roll records for
  the parcel"; with a town or county, "no parcel matched that address in <place>;
  write the street as the assessment roll does (e.g. 1 STATE CIR), with any unit after
  it (100 HARBORVIEW DR UNIT 204), or give the zip instead, since the roll may record
  the parcel under another town name". Rewrite per step 1 and retry. A unit the roll
  does not record at an address with units gets "the roll records units at that
  address, but none is <unit>; send the address without the unit to list the parcels
  there, each with its unit". An `acct=` that matches nothing gets "no parcel on the
  roll has that account id; ids come from parcel.account_id or a multiple_matches
  result"; then recheck the account digits. A zip that is not 5 digits gets HTTP 400
  "zip must be 5 digits, such as 21202; a ZIP+4 such as 21202-1234 also works". A
  no-match says nothing about the home or its assessment.

### 3. Relay the verdict, branch by branch

Lead with `plain.headline` word for word, then whichever other `plain` sentences are
present.

| `Branch` | `plain.headline`, verbatim | Then |
|---|---|---|
| `fair` | "This assessment sits in the normal range for similar homes."; below the 25th percentile "This assessment is on the low side for similar homes.", below the 10th "This assessment is lower than almost all similar homes." | Say so plainly. The packet page is titled "Assessment comparison sheet". |
| `review` | "This assessment is on the high side for similar homes: worth a closer look." | Relay the windows (step 4) and the packet ("Appeal evidence packet"). Filing is the user's call. |
| `elevated` | "This assessment is higher than almost all similar homes." | Same as review. |
| `insufficient` | "Not enough similar homes to compare fairly." | No verdict either way. Relay `plain.body`, e.g. Frederick City Hall's "We found only 2 homes similar enough (same reassessment group, similar size, lot, and age, same zip). An honest answer needs more; the official record link below is the next stop." With `BandSize` 0 it opens "We couldn't find any other homes similar enough" instead, and a skipped test drops out of the parenthesis ("similar size and lot" when year built was not compared). The packet page says "We can't build an honest packet here". |

The `insufficient` branch carries `headline` and `body`, plus `not_compared` when a
test was skipped. On the other three, more `plain` keys appear when they apply. As
worded on 2026-10-27: `position` ("Out of 1,196 similar homes, about 5 in 10 are
assessed lower than this one and 5 in 10 higher."); `level`, only when the group has
at least 10 open-market sales of a single home in the window ("As a group, these homes
are assessed at about 93% of what they actually sell for (based on 111 open-market
sales of a single home from 2024 through 2026)."; the years are `RatioFromYear` and
`RatioToYear`; foreclosures, gifts, auctions, multi-parcel and vacant-lot sales are
left out); `size`, when a high or low position comes mostly from the home's size
("Much of this is size: this home is larger than about 7 in 10 of them, and per square
foot its assessment is higher than about 5 in 10.", 11701 Berwick Rd 20904, which reads
"on the high side"; relay it, since it bears on whether an appeal is worth it); `phase_in`, when `taxable_now` is below
`market_value` ("The assessment is still phasing in: increases spread over three
years, so the taxable amount steps up again the next year or two even with no new
assessment. Decreases apply immediately."); `not_compared`, when the roll records no
lot or no year built ("The assessment roll records no year built for this home, so
year built was not compared.", the State House's).

Zeros in `uniformity`: on the `insufficient` branch every number but `BandSize`,
`RatioFromYear`, and `RatioToYear` reads 0, meaning not computed. On the other
branches, `MedianSalesRatio` and `COD` read 0 "when RatioSales is below 10, meaning
no ratio is published" (`tools/list`); `level`
is then absent, and `RatioSales` still shows the small count, such as the State
House's 4. `PctFMV` 0 on
a fair, review, or elevated parcel is real: no similar home is assessed lower than
this one. Set `market_value` beside `MedianFMV` only when `MedianFMV` is above 0.

Most commercial parcels tried (14 of 16) and every tax-exempt parcel tried came back
`insufficient` (2026-09-28); a few get a verdict, still worded as "similar homes". The
State House, tax-exempt, has got `fair` since the roll's missing year built stopped
counting against it (2026-10-26). "Similar", as the packet defines it: "same
reassessment group, same zip code, same land-use class, living area within 30%, lot
size within 50%, and year built within 20 years.", less any test `NotCompared` names.

### 4. Appeal windows and where to file

Relay the `appeals` block as the response words it. Observed 2026-09-29:

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
notice mails, it offers only that notice. As of 2026-10-27:

- The State House (`acct=020600002182004`, last reassessed January 2024): "For this
  home: This home's last reassessment notice mailed in late December 2023, so its
  45-day appeal window closed in February 2024. The next free option: the next
  reassessment notice should mail in late December 2026, and the date printed on it
  starts a fresh 45-day appeal window."
- Baltimore City Hall (`acct=0304121302 001`, last reassessed January 2025): "For this
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
similar-homes median and range that the JSON leaves at 0 (as of 2026-10-27 the
Frederick City Hall page, 2 similar homes, showed "similar-homes median $3,499,400").
Never relay them.

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
  an honest packet here" (its browser tab still reads "Appeal evidence packet").
  Frederick City Hall shows it:
  `https://overassessed.agentlookups.ai/packet?acct=1102031795`.

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
2. **Snapshot date**: `roll.as_of`, the date the state last updated the roll
   ("2026-10-06" on 2026-10-27), and `roll.pulled`, the date this copy was taken, when
   present ("2026-10-27"). The home page, the verdict page, every packet, and the MCP
   `initialize` instructions word the month as "Roll data as of October 2026"
   (2026-10-27); the instructions add "The state last updated the roll on October 6,
   2026; this copy was taken on October 27, 2026." When
   `roll.overdue` is true, say this copy is more than 35 days old. If `roll` is null,
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
<plain.position> <plain.level, only if present> <plain.phase_in, only if present> <plain.not_compared, only if present>
Assessed at $<market_value>, last reassessed <Month YYYY>; similar-homes median $<MedianFMV>.
Appeal: <"For this home:" paragraph>. These dates are the service's estimate; the date on your notice and SDAT decide. File at <portal> or with <petition_pdf>; steps: <process>.
Packet to print (only for review or elevated, or fair if asked): https://overassessed.agentlookups.ai/packet?acct=<account_id>
Roll data as of <Month YYYY>; the state last updated the roll on <roll.as_of>. <"This copy is more than 35 days old.", only if roll.overdue is true>
(If roll is null: roll refreshes monthly; checked <date>.)
"<honesty 1>" "<honesty 2>" Not legal or tax advice.
```

Insufficient (no position, median, or packet):

```
<plain.headline> (Overassessed, Maryland SDAT roll; account <account_id>, <county>)
<plain.body> <plain.not_compared, only if present>
Assessed at $<market_value>, last reassessed <Month YYYY>. Official record: https://sdat.dat.maryland.gov/RealProperty/
Appeal: <"For this home:" paragraph>. These dates are the service's estimate; the date on your notice and SDAT decide. File at <portal> or with <petition_pdf>; steps: <process>.
Roll data as of <Month YYYY>; the state last updated the roll on <roll.as_of>. <"This copy is more than 35 days old.", only if roll.overdue is true>
(If roll is null: roll refreshes monthly; checked <date>.)
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
