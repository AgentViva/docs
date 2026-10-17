# How to use AgentViva

AgentViva checks your AI tools, sites and APIs from the outside and tells you when one stops working. This guide
shows how to start, step by step. The same guide is on the website: <https://agentviva.dev/docs>.

1. [Get access](#1-get-access)
2. [Add what to watch](#2-add-what-to-watch)
3. [Get told when something breaks](#3-get-told-when-something-breaks)
4. [Show the status on your site](#4-show-the-status-on-your-site)
5. [See what is happening](#5-see-what-is-happening)
6. [What the states mean](#6-what-the-states-mean)

## 1. Get access

1. Go to <https://agentviva.dev/request-access> and sign in with your Google account. We record your request.
2. We email you when your invite is ready.
3. Sign in again with the same Google account. A short guide helps you add your first monitor.

## 2. Add what to watch

Pick the way that fits you. You can mix them: monitors from all of them live in the same list.

| You are | Use |
| --- | --- |
| Anyone, no code | [The dashboard](#one-tool-at-a-time-the-dashboard) |
| A directory, a marketplace, anyone with many tools | [A catalog](#your-whole-list-a-catalog) |
| A developer | [The API](#from-your-code-the-api) |
| Someone who works with Claude Code, Cursor or Codex | [MCP](#from-your-ai-agent-mcp) |

### One tool at a time: the dashboard

1. Open **Monitors** and press **Add a monitor**.
2. Choose what to check: a **website**, a **JSON health address**, or an **AI answer** (Pro).
3. Paste the address (`example.com` is enough) and press **Add monitor**.
4. The first check runs within one check period (a day for a website, unless you choose faster). Press
   **Check now** on the monitor's page to check at once.

### Your whole list: a catalog

1. Publish one file on your site with every tool: an **id**, a **name** and an **address**. JSON or CSV.
2. Open **Catalog** in the menu, paste the link to the file, and press **Save and check the list**.
3. That is all. Every day we read the list again: a new tool gets a monitor, a renamed or moved tool is updated,
   and a tool you removed is paused, with its history kept.

Example file (JSON):

```json
[
  { "id": "notion", "name": "Notion", "url": "https://www.notion.so" },
  { "id": "geondex", "name": "Geondex", "url": "https://geondex.com" }
]
```

We also read `slug` for the id, `title` for the name, and `website` or `link` for the address. A CSV needs a header
row. The details, including the safety rules that stop a broken list from pausing your monitors, are in
[Catalog in the API reference](api.md#catalog).

### From your code: the API

1. Open **API** in the menu and press **Make a new API key**. Copy it: it is shown only once.
2. Send it in the `Authorization` header of every request, as below.
3. Everything else is in the [API reference](api.md).

Add a monitor:

```bash
curl -X POST https://agentviva.dev/v1/monitors -H "Authorization: Bearer av_live_..." -H "content-type: application/json" -d '{"type":"page","name":"My tool","target":{"url":"https://example.com"}}'
```

Connect a catalog:

```bash
curl -X PUT https://agentviva.dev/v1/catalog -H "Authorization: Bearer av_live_..." -H "content-type: application/json" -d '{"url":"https://example.com/tools.json"}'
```

Read what changed:

```bash
curl "https://agentviva.dev/v1/changes?since=0" -H "Authorization: Bearer av_live_..."
```

List what does not work, problems first:

```bash
curl "https://agentviva.dev/v1/monitors?state=down,degraded&sort=problems" -H "Authorization: Bearer av_live_..."
```

### From your AI agent: MCP

1. Get your API key (see [the API](#from-your-code-the-api) above).
2. Connect AgentViva once. In Claude Code the easiest way is the plugin: it asks for your key once and brings both
   the MCP connection and the skill (our rules for the agent). In Cursor or Codex, add the address and header below
   in their MCP settings.
3. Then ask in plain words. For example: *"Watch <https://example.com> and tell me if it goes down."* ·
   *"Which of my catalog tools do not work?"* · *"Connect my catalog at <https://example.com/tools.json>."*

No key yet? The plugin still answers "is Notion up?" from the tools AgentViva already watches. Leave the key empty
and add it later.

Install the plugin (Claude Code):

```bash
claude plugin marketplace add AgentViva/docs
claude plugin install agentviva@agentviva
```

Installed it before version 1.2? Update it, then add your key in Claude Code with `/plugin` → AgentViva →
**Configure options**:

```bash
claude plugin marketplace update agentviva
claude plugin update agentviva@agentviva
```

Or connect MCP without the plugin (Claude Code):

```bash
claude mcp add --transport http agentviva https://agentviva.dev/mcp --header "Authorization: Bearer av_live_..."
```

claude.ai custom connectors do not work yet: they need a sign-in method (OAuth) we have not built. The tools are
listed in [MCP server in the API reference](api.md#mcp-server).

## 3. Get told when something breaks

1. Open **Alerts** in the menu.
2. **Webhook**: paste your address and press **Save**. We send a signed `POST` for every confirmed change. How to
   check the signature is in [Webhook deliveries](api.md#webhook-deliveries).
3. Press **Send a test** to see that it arrives. Each channel shows its status, and the page lists the last
   deliveries: what was sent, and whether it arrived.
4. **Telegram**: press **Connect Telegram**, then **Start** in the chat. Turn on the **daily summary** to get one
   message a day instead of one per change. (Until Telegram alerts are switched on for the service, the card says
   "Coming soon".)

An alert goes out when a tool stops working, works only partly, or works again. Never for `blocked`, and never for
a single bad check.

## 4. Show the status on your site

1. Open a monitor, press **Edit**, and tick **Public status page and badge**. Monitors from a catalog are public
   unless you chose otherwise.
2. On the monitor's page, under **Show this status on your site**, copy the code (HTML for a web page, Markdown for
   a README) and paste it where the status should show. It updates by itself.
3. Scripts can read the same status as JSON, with no key.

**Many tools from a catalog?** Put one line in the template of your tool pages and every tool gets its badge. The
**Catalog** page gives you the line with your team in it; your template puts each tool's id where it says
`TOOL_ID`:

```html
<a href="https://agentviva.dev/s/<team>/TOOL_ID"><img src="https://agentviva.dev/badge/<team>/TOOL_ID.svg" alt="Status" width="180" height="48"></a>
```

[![Geondex status](https://agentviva.dev/badge/aigregator/geondex.svg)](https://agentviva.dev/s/aigregator/geondex)

```html
<a href="https://agentviva.dev/s/<team>/<slug>"><img src="https://agentviva.dev/badge/<team>/<slug>.svg" alt="Status" width="180" height="48"></a>
```

```text
https://agentviva.dev/v1/public/<team>/<external_id>.json
```

## 5. See what is happening

- **Overview**: what needs attention now, and what changed in the last two days.
- **Monitors**: problems first. Each tool's address opens its site. Filter, search, and download a CSV.
- **Changes**: every confirmed change, grouped by day. New tools that worked on their first check are folded into
  one line.
- **Catalog**: how the tools in your list are doing, a CSV of the ones that do not work, and the badge line for your
  tool pages.
- **A monitor's page**: what is wrong and what to do, 30 days of history, every check with its response time, and
  the code to show its badge.
- **Get alerts for your tool** (in the account menu): is a tool you make listed on another site that uses
  AgentViva? Prove the domain is yours and get its alerts too.

## 6. What the states mean

A state changes only after several checks agree, so one bad check never raises an alarm.

| State | In words | Meaning |
| --- | --- | --- |
| `up` | Working | It answered, and correctly |
| `degraded` | Partly working | It answers, but something is off: slow, a redirect to another site, a parked domain |
| `down` | Not working | Confirmed gone: the server fails, the page is missing, or it does not answer |
| `blocked` | Blocked | The site refused our check. We cannot tell, so we never call it down and send no alert |
| `unknown` | Not checked yet | A new monitor before its first result |

## Help

[API reference](api.md) · [OpenAPI spec](openapi.yaml) · [How our checks behave](https://agentviva.dev/bot) ·
questions: <support@agentviva.dev>.
