# AgentViva

AgentViva watches AI tools, agents and assistants from the outside and answers one question per target:
**is it alive, and since when.**

This repository is the public reference. The service runs at [agentviva.dev](https://agentviva.dev); its
source is private.

**New here? Start with [GETTING-STARTED.md](GETTING-STARTED.md)** (also at <https://agentviva.dev/docs>): how to
get access, add what to watch (dashboard, catalog, API or your AI agent), get alerts and show the status.

| File | What it is |
| --- | --- |
| [GETTING-STARTED.md](GETTING-STARTED.md) | How to connect and work with AgentViva, step by step |
| [api.md](api.md) | The API reference. The only document an integration should rely on |
| [openapi.yaml](openapi.yaml) | The same contract as OpenAPI 3.1, for tooling and generated clients |
| [skills/agentviva](skills/agentviva/SKILL.md) | A Claude Code skill, so a coding agent knows when to call us |
| [CHANGELOG.md](CHANGELOG.md) | Every change to the public API, newest first |
| [SECURITY.md](SECURITY.md) | How to report a vulnerability |

## Three ways to ask

- **`page`** — loads the site and follows redirects. Catches parked domains, dead hosts, bot walls and quiet
  moves to a new domain.
- **`json`** — calls a health endpoint and runs your rules on the body: a field equals a value, a number stays
  under a limit, a key is present.
- **`canary`** — sends a real prompt and reads the reply. A judge model decides whether that was a working
  answer or an apology.

## We never cry wolf

A single failed probe changes nothing. The same verdict has to repeat, ten minutes apart, before AgentViva
writes it down or wakes anybody up — and the record says how many probes agreed.

If a target simply refuses us, that is `blocked`, not `down`. A tool does not get declared dead for disliking
our user agent.

States are `unknown`, `up`, `degraded`, `down` and `blocked`.

## The whole API, in three calls

```bash
curl -X POST https://agentviva.dev/v1/monitors -H "Authorization: Bearer av_live_..." -H "content-type: application/json" -d '{"type":"page","name":"Notion","target":{"url":"https://www.notion.so"},"public":true,"public_slug":"notion"}'

curl "https://agentviva.dev/v1/changes?since=0" -H "Authorization: Bearer av_live_..."

curl https://agentviva.dev/v1/public/aigregator/notion.json
```

No agent to install, no SDK — one HTTP call per monitor.

## Many tools? Connect a catalog

A directory does not need to create thousands of monitors itself. Publish one JSON or CSV file with an id, a name
and a URL for each tool, and connect its link once:

```bash
curl -X PUT https://agentviva.dev/v1/catalog -H "Authorization: Bearer av_live_..." -H "content-type: application/json" -d '{"url":"https://example.com/tools.json"}'
```

AgentViva reads the list every day. A new tool gets a monitor, a renamed or moved tool is updated, and a tool that
left the list is paused, with its history kept. Then ask which ones stopped working:
`GET /v1/monitors?state=down&source=catalog`. See [Catalog in the API reference](api.md#catalog).

## MCP server

An AI agent can use AgentViva without writing REST calls. The MCP server at `https://agentviva.dev/mcp` has eight
tools: list and check monitors, create one, probe now, read recent changes, and connect and sync a catalog. In
Claude Code:

```bash
claude mcp add --transport http agentviva https://agentviva.dev/mcp --header "Authorization: Bearer av_live_..."
```

It needs a client that can send an `Authorization` header (Claude Code, Cursor, Codex). claude.ai custom connectors
and browser-based inspectors do not work yet: they need OAuth, which is not built. See the
[MCP section of the API reference](api.md#mcp-server).

## Public surfaces, no key needed

```text
GET https://agentviva.dev/v1/public/<team>/<external_id>.json
GET https://agentviva.dev/badge/<team>/<slug>.svg      180x48 SVG, cached 5 minutes
GET https://agentviva.dev/s/<team>/<slug>              HTML status page, 90-day history, no JavaScript
GET https://agentviva.dev/pricing.md                    plans and prices in Markdown, for agents
GET https://agentviva.dev/v1/public/search?q=notion     the public tools we already watch, by name or address
```

A catalog tool's `<slug>` is its id from your list (lower case, with dashes, when the id is not a valid link), so one
line in the template of your tool pages gives every tool its badge. The Catalog page in the dashboard shows it.

## Getting a key

Signup is not open yet. Email [support@agentviva.dev](mailto:support@agentviva.dev) with what you want to
watch and we will tell you whether we can help. Keys are issued per team and shown once.

## Install the plugin (skill and MCP)

In Claude Code, the plugin brings the skill and the MCP connection. It asks for your API key once (stored in your
system's secure credential store). No key yet? Leave it empty: you can still ask "is Notion up?", answered from the
tools AgentViva already watches (`lookup_tool`):

```bash
claude plugin marketplace add AgentViva/docs
claude plugin install agentviva@agentviva
```

Installed it before version 1.2? Run `claude plugin marketplace update agentviva` and
`claude plugin update agentviva@agentviva`, then add the key in Claude Code: `/plugin` → AgentViva →
**Configure options**.

Or with [skills.sh](https://skills.sh), which works with any agent that reads `SKILL.md`:

```bash
npx skills add AgentViva/docs
```

To read the skill once without installing it:

```bash
npx skills use AgentViva/docs@agentviva
```

## Our crawler

AgentVivaBot fetches one page per monitored site at most once an hour (most sites are checked once a day), more
often only while confirming a change, never more than 2 MB, and never posts
data except to assistant endpoints whose owners configured a canary question. The policy, and how to opt out,
is at [agentviva.dev/bot](https://agentviva.dev/bot).

## Licence

Example code and the OpenAPI spec are MIT — see [LICENSE](LICENSE). The prose documentation is
[CC BY 4.0](LICENSE-DOCS). Attribute it to AgentViva with a link to this repository.
