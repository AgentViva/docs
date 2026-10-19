# How to use AgentViva

AgentViva checks your AI tools, sites and APIs from the outside and tells you when one stops working. This guide
shows how to start, step by step. The same guide is on the website: <https://agentviva.dev/docs>.

1. [Get access](#1-get-access)
2. [Add what to watch](#2-add-what-to-watch)
3. [Connect your AI agent (MCP)](#3-connect-your-ai-agent-mcp)
4. [Get told when something breaks](#4-get-told-when-something-breaks)
5. [Show the status on your site](#5-show-the-status-on-your-site)
6. [See what is happening](#6-see-what-is-happening)
7. [What the states mean](#7-what-the-states-mean)

## 1. Get access

1. Go to <https://agentviva.dev/request-access> and sign in with your Google account. We record your request.
2. We read every request and write to you at that email when your invite is ready.
3. Sign in again with the same Google account. A short guide helps you add your first monitor.

## 2. Add what to watch

Pick the way that fits you. You can mix them: monitors from all of them live in the same list.

| You are | Use |
| --- | --- |
| Anyone, no code | [The dashboard](#one-tool-at-a-time-the-dashboard) |
| A directory, a marketplace, anyone with many tools | [A catalog](#your-whole-list-a-catalog) |
| A developer | [The API](#from-your-code-the-api) |
| Someone who works with Claude Code, Cursor or Codex | [MCP](#3-connect-your-ai-agent-mcp) (section 3) |

### One tool at a time: the dashboard

1. Open **Monitors** and press **Add a monitor**.
2. Choose what to check: a **website**, a **JSON health address**, an **AI answer** (Pro), or an **MCP server** (we
   connect like an AI app and list its tools).
3. Paste the address (`example.com` is enough) and press **Add monitor**.
4. The first check runs within one check period (a day for a website, unless you choose faster). Press
   **Check now** on the monitor's page to check at once.

### Your whole list: a catalog

1. Publish one file on your site with every tool: an **id**, a **name** and an **address**. JSON or CSV.
2. Open **Catalog** in the menu, paste the link to the file, and press **Save and check the list**.
3. That is all. Every day we read the list again: a new tool gets a monitor, a renamed or moved tool is updated,
   and a tool you removed is paused, with its history kept.

The free plan watches up to 25 tools. For a longer list, write to <support@agentviva.dev>.

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

## 3. Connect your AI agent (MCP)

MCP lets an AI agent (Claude Code, claude.ai, Cursor, Codex or another MCP client) use AgentViva for you. Then you
ask in plain words: *"Watch <https://example.com> and tell me if it goes down."* · *"Which of my catalog tools do
not work?"* · *"Connect my catalog at <https://example.com/tools.json>."*

You sign in once in your browser and press **Allow**. There is no key to copy.

### Claude Code: the plugin (easiest)

It brings the MCP connection and the skill (our rules for the agent).

1. Install it from your terminal:

   ```bash
   claude plugin marketplace add AgentViva/docs
   claude plugin install agentviva@agentviva
   ```

2. Restart Claude Code, then type `/mcp`.
3. Choose **agentviva**, then **Authenticate**. Your browser opens: sign in with Google and press **Allow**.
4. Back in Claude Code, **agentviva** shows **9 tools**. Ask in plain words, as above.

Installed it before? Update it first, then restart Claude Code:

```bash
claude plugin marketplace update agentviva
claude plugin update agentviva@agentviva
```

### claude.ai

A custom connector, in your claude.ai settings.

1. Open **Settings**, then **Connectors**, then **Add custom connector**.
2. Paste this address: `https://agentviva.dev/mcp`
3. Sign in and press **Allow**.

### Other MCP clients

Claude Code without the plugin, one line:

```bash
claude mcp add --transport http agentviva https://agentviva.dev/mcp
```

Then type `/mcp`, choose **agentviva**, then **Authenticate**.

Any other MCP client (Cursor, Codex and others): add a remote (HTTP) server with this address. It opens your browser
to sign in.

```text
Address: https://agentviva.dev/mcp
```

No account yet? Add `https://agentviva.dev/mcp/public` instead. It has one tool, `lookup_tool`: it answers "is
Notion up?" from the tools AgentViva already watches.

### With your API key instead

For your code, and for a client that cannot open a browser.

- **Make it once.** Open **API** in the menu and press **Make a new API key**.
- **Shown once.** We show the full key only when you make it, and keep only a hash of it. Save it in a password
  manager.
- **One key for the team.** Your code, your scripts and an agent that sends a header all use this same key.
- **Lost it?** Make a new one. The old key stops at once, so put the new key everywhere the old one was.
- **Keep it secret.** Anyone with the key can add, change and delete your monitors.

Claude Code, one line:

```bash
claude mcp add --transport http agentviva https://agentviva.dev/mcp --header "Authorization: Bearer av_live_..."
```

Any other MCP client:

```text
Address: https://agentviva.dev/mcp
Header:  Authorization: Bearer av_live_...
```

The tools are listed in [MCP server in the API reference](api.md#mcp-server).

## 4. Get told when something breaks

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

## 5. Show the status on your site

1. Open a monitor, press **Edit**, and tick **Public status page and badge**. Monitors from a catalog are public
   unless you chose otherwise.
2. On the monitor's page, under **Show this status on your site**, pick a form and a theme, copy the code (HTML for
   a web page, Markdown for a README) and paste it where the status should show. It updates by itself.
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

Seven forms. Add `?style=` to the badge link: `dot` (colour only), `word`, `pill` (for a README), `card` (the
default), `seal`, `strip` (with the last 30 days) or `panel` (everything). Add `&theme=light` or `&theme=auto`
(follows the reader's light or dark setting); `dark` is the default.

On the badge, the state words are shorter: working shows as alive, partly working as degraded, and not working as
down.

```text
https://agentviva.dev/v1/public/<team>/<external_id>.json
```

## 6. See what is happening

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

## 7. What the states mean

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
