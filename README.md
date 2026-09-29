# agentlookups skills

Agent Skills that teach an AI agent to use the public-record lookup
services at [agentlookups.ai](https://agentlookups.ai). Packaged as a
Claude Code plugin; the skills follow the open
[Agent Skills](https://agentskills.io) format, so other agent runtimes can
use them too.

The services are free during beta, with no account needed. Features,
access, and pricing may change.

## The services

Five read-only lookup services, run by TopHat Monkey Software LLC:

| Service | What it answers | Site | Machine index |
| --- | --- | --- | --- |
| Plumbline | Is this contractor licensed? The licensing-board record, with its source and snapshot date, in the states and cities it covers. | [contractors.agentlookups.ai](https://contractors.agentlookups.ai) | [llms.txt](https://contractors.agentlookups.ai/llms.txt) |
| CounterScript | Is this cash price for a prescription fair? What pharmacies pay to buy the drug (CMS NADAC, a benchmark, not a price anyone owes), a fair cash estimate, FDA generic equivalents, and Medicare negotiated prices. | [rx.agentlookups.ai](https://rx.agentlookups.ai) | [llms.txt](https://rx.agentlookups.ai/llms.txt) |
| GroundTruth | What do public records say about this US address? Superfund/NPL sites, toxic-release (TRI) facilities, and enforcement-flagged facilities nearby; drinking-water systems with violations; and layer scores for schools, hazards, contamination, noise, walkability, crime, and climate. | [env.agentlookups.ai](https://env.agentlookups.ai) | [llms.txt](https://env.agentlookups.ai/llms.txt) |
| GroundRules | What does the law say about this where I live? Maps a US address to its federal, state, county, and city layers, names each layer's code sources, and quotes the text it hosts verbatim, with citation, source link, and current-through date. | [law.agentlookups.ai](https://law.agentlookups.ai) | [llms.txt](https://law.agentlookups.ai/llms.txt) |
| Overassessed | Is this Maryland property assessed fairly next to similar homes? The state's own assessment roll (SDAT). | [overassessed.agentlookups.ai](https://overassessed.agentlookups.ai) | [llms.txt](https://overassessed.agentlookups.ai/llms.txt) |

Coverage differs by service and changes over time. Each service's
llms.txt says where to check what it covers today.

## What the services say about their results

Each service publishes rules about what its results mean, in its
llms.txt or in every response. The skills pass them on. In short:

- A result shows what a public record said on the date shown. The agency
  or official source that issued it is the authority for today.
- An empty result proves nothing. Plumbline finding no match is not a
  finding that a contractor is unlicensed. GroundTruth finding no records
  is never a clean bill of health. When GroundRules does not host a
  place's law, it says so, and that never means no law applies.
- Plumbline results are records, not ratings or recommendations.
  GroundTruth's layer scores are model estimates, not safety ratings, and
  a layer it cannot score comes back null or as a flagged placeholder,
  never a made-up number.
- GroundRules quotes law word for word and never sums it up. Text it
  labels with a vintage is a dated snapshot, not current law.
- Overassessed gives a fairness snapshot of the public record, never an
  appraisal.
- Nothing here is legal, financial, medical, or professional advice. None
  of it is a background check or a consumer report under the Fair Credit
  Reporting Act.

## Skills in this plugin

| Skill | For | Uses |
| --- | --- | --- |
| `hire-a-contractor` | anyone about to hire a contractor who wants the license record first | Plumbline |
| `prescription-price-check` | anyone asking if a US prescription price is fair, and how to ask for less | CounterScript |
| `maryland-assessment-appeal` | a Maryland owner or buyer asking if an assessment is fair, and how to appeal | Overassessed |
| `whats-near-this-address` | one question about EPA sites or water-system violations near a US place | GroundTruth |
| `find-the-law` | finding and quoting a US law, with its citation and date | GroundRules |
| `home-project-permits` | a homeowner asking if a project needs a permit, or if they can do the work themselves | GroundRules, Plumbline |
| `home-purchase-due-diligence` | a homebuyer checking a house before an offer | GroundTruth, Plumbline, Overassessed (Maryland) |
| `listing-evaluation-for-agents` | a realtor or buyer's agent vetting a listing | GroundTruth, Plumbline, Overassessed (Maryland) |
| `rental-property-check` | a tenant or small landlord checking a rental | GroundTruth, Plumbline, Overassessed (Maryland) |

The first six answer one question with one or two services. The last
three run the full set of checks on one property and hand law and permit
questions to `find-the-law` and `home-project-permits`. Each skill lives
at `skills/<name>/SKILL.md`.

## MCP servers in this plugin

Installing the plugin connects all five services' MCP servers, listed in
[`.mcp.json`](.mcp.json). They are read-only and need no sign-in during
beta.

| Server | Service | Endpoint | Tools |
| --- | --- | --- | --- |
| `plumbline` | Plumbline | `https://contractors.agentlookups.ai/mcp` | `check_contractor` |
| `counterscript` | CounterScript | `https://rx.agentlookups.ai/mcp` | `drug_price` |
| `groundtruth` | GroundTruth | `https://env.agentlookups.ai/mcp` | `environment_near`, `drinking_water`, `due_diligence` |
| `overassessed` | Overassessed | `https://overassessed.agentlookups.ai/mcp` | `check_assessment` |
| `groundrules` | GroundRules | `https://law.agentlookups.ai/mcp` | `law_for_location`, `law_search`, `law_get_section`, `law_coverage` |

The skills also show the plain HTTP form of their lookups, so they work
in runtimes without MCP.

## Use it in Claude Code

```
/plugin marketplace add agentlookups/agentlookups-skills
/plugin install agentlookups@agentlookups
```

Or from a local clone:

```bash
git clone https://github.com/agentlookups/agentlookups-skills
claude --plugin-dir ./agentlookups-skills
```

Skills load namespaced as `/agentlookups:<skill-name>`.

Give a state as its two-letter code (MD, TX), and an address with street,
city, state, and ZIP, such as `1100 Congress Ave, Austin, TX 78701`.

## Privacy, terms, and contact

One privacy page and one terms page are shared by every agentlookups.ai
service:

- Privacy: <https://agentlookups.ai/privacy/>
- Terms: <https://agentlookups.ai/terms/>

This plugin holds only instructions and server addresses; it runs no code
of its own. Your agent sends lookups straight to the services.

Report a wrong record to corrections@agentlookups.ai. Anything else:
support@agentlookups.ai.

## License

MIT. See [LICENSE](LICENSE).
