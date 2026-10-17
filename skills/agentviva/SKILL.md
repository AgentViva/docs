---
name: agentviva
description: >-
  Check whether an AI tool, agent, assistant or API is still alive, and set up monitoring that says so.
  AgentViva probes a target from the outside and reports one state — up, degraded, down, blocked or unknown —
  with the timestamp it last changed. Use when the user asks whether a tool is still up, still working, or
  dead; when a directory or catalogue needs to know which of its listed tools still exist; when a status badge,
  status page or uptime JSON is wanted for a site or README; or when the user mentions AgentViva, agentviva.dev
  or an `av_live_` key.
---

# AgentViva

AgentViva answers one question per target: is it alive, and since when.

**Read the reference before writing a request.** It has the exact field rules, error codes and limits, and
it is the only document an integration should rely on.

- If `../../api.md` exists next to this skill, read that file — you already have it locally.
- Otherwise fetch <https://raw.githubusercontent.com/AgentViva/docs/main/api.md>.
- The machine-readable spec is `../../openapi.yaml`, or
  <https://raw.githubusercontent.com/AgentViva/docs/main/openapi.yaml>.

## What to know before you call

- Base URL `https://agentviva.dev`. Auth is `Authorization: Bearer av_live_...`, one key per team, shown once.
  If the user has no key, tell them to email <support@agentviva.dev> — signup is not open.
- Three check types: `page` (does the site load), `json` (does a health endpoint answer correctly), `canary`
  (does a real prompt get a real answer, judged by a model). `canary` needs the `pro` plan.
- A state only changes after several probes agree, so `POST /v1/monitors/{id}/probe` returning `down` does not
  mean the monitor is `down` yet. Read `state` and `state_since` on the monitor, not the probe verdict.
- `blocked` means the target refused our probe. It is not evidence the tool is dead. Never report it as down.
- Poll `GET /v1/changes?since=<ms>` for updates; store the `at` of the last change you handled. Do not poll the
  monitor list on a loop.
- Many tools (a directory, a catalogue, a marketplace): prefer the catalog. Publish one JSON or CSV file with an
  id, a name and a URL per tool and connect its link once with `PUT /v1/catalog` (or the `connect_catalog` MCP
  tool). AgentViva reads it every day: new tools get a page monitor, changed ones are updated, tools that left the
  list are paused (never deleted). `GET /v1/catalog` shows the last sync and `monitors_by_state` for the catalog's
  monitors. A sync that would shrink the list by more than half, or pause more than half of the monitors, changes
  nothing until `POST /v1/catalog/sync` with `{ "force": true }`.
- Adding monitors from code without a catalog is `POST /v1/monitors/bulk`, up to 1,000 at a time, upserted by
  `external_id`.
- "Which of my tools are dead?": `GET /v1/monitors?state=down&source=catalog` (or `state=down,degraded`;
  `sort=problems` puts problems first). Each change in `GET /v1/changes` carries the tool's `url`.
- "Is it flapping, or dead for days?": add `days=14` to `GET /v1/monitors` (or read one monitor) for each day's worst
  result. One monitor's history of changes is `GET /v1/changes?since=0&monitor_id=<id>`.
- "Did our alerts arrive?": `GET /v1/alerts` lists the last deliveries with `sent` and `error`;
  `POST /v1/team/webhook/test` sends a test delivery to the saved webhook.
- "Is this tool up?" for a tool you do not watch yourself: `GET /v1/public/search?q=<name or address>` (no key) or the
  `lookup_tool` MCP tool, which works without a key too. It answers from tools AgentViva already watches and does
  not check the address now. Nothing found means nobody watches it yet: say so, do not guess.
- Public monitors need no key at all: `GET /v1/public/{team}/{external_id}.json`, a badge at
  `/badge/{team}/{slug}.svg`, and a status page at `/s/{team}/{slug}`. For a catalog tool the slug and the
  external id are the tool's id from the list, so one line in a tool-page template shows every tool's badge.

## MCP tools

If the user's agent has an `agentviva` MCP server connected, prefer its tools to hand-written REST calls:
`list_monitors`, `get_status`, `create_monitor`, `probe_now`, `get_changes`, `connect_catalog`, `catalog_status`
and `sync_catalog`. They apply the same rules and return the same JSON. For "which of my tools are dead", call
`list_monitors` with `states: ["down", "degraded"]`, `source: "catalog"` and `sort: "problems"`. This plugin
brings the server too and asks for the key when it is enabled (`/plugin` → AgentViva → Configure options sets it
later). Without the plugin (Claude Code):

```bash
claude mcp add --transport http agentviva https://agentviva.dev/mcp --header "Authorization: Bearer av_live_..."
```

The server needs a client that can send an `Authorization` header: Claude Code, Cursor and Codex can; claude.ai custom
connectors and browser-based inspectors cannot yet (no OAuth). Deleting a monitor is REST-only
(`DELETE /v1/monitors/{id}`).

## Times and limits

All times are Unix milliseconds. 600 requests per minute per key; a bulk call counts as one. Bodies over
64 KB are rejected.
