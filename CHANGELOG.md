# Changelog

Every change to the public AgentViva API, newest first. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## How we version

The API is `v1`. A breaking change gets a new path prefix, not a silent edit — `/v1` will keep behaving the way
this repository describes for as long as anyone is calling it. Additive changes (a new field, a new endpoint,
a new `reason` value) ship without notice, so parse defensively: ignore fields you do not know, and treat an
unrecognised `state` or `reason` as "something we have not seen yet" rather than an error.

Anything marked **v1** in [api.md](api.md) is planned, not built. Do not depend on it.

## [Unreleased]

### Added

- Discovery files for agents: `GET /openapi.json` and `/openapi.yaml` on the site (the home page links them with
  `Link: rel="service-desc"`), `GET /.well-known/ard.json` and `/.well-known/ai-catalog.json`, the MCP server card
  `GET /mcp/server-card`, JSON-LD on the home page, and WebMCP tools (`check_url`, `lookup_tool`, `report_problem`)
  for agents in the browser on the home page and `/check`. The MCP server is `dev.agentviva/agentviva` in the MCP
  Registry (`server.json` in this repo).

- `GET /v1/public/tools.json`: the liveness feed for directories, every tool on the public list with its domain,
  state, since when and its status, badge and JSON links; no key, updated hourly. `GET /for-directories` explains what a
  directory gets.
- `GET /research` and `GET /research.json`: "The state of AI tools", the public tools by state, why the rest fail,
  how many stop working each month, and how long dead tools stay dead.

- `GET /v1/public/check?url=&type=`: one check of any public website (`page`) or MCP server (`mcp`), right now, no
  key, with `evidence`; the same check is a form on the home page and at `/check`. MCP: `check_url` (keyless).
- `POST /v1/public/report`: tell AgentViva that a watched tool does not work (`{ "query", "note" }`, no key); it checks
  the tool again (at most once an hour) and never changes a state by itself. MCP: `report_problem` (keyless). The
  status page has a "Report a problem" form, and `GET /v1/monitors/:id` has `reports_7d`.
- The keyless MCP mode (`/mcp/public`) lists `lookup_tool`, `check_url` and `report_problem`; with a key, 16 tools.

- AgentVivaBot signs every request with Web Bot Auth (`Signature`, `Signature-Input`,
  `Signature-Agent: "https://agentviva.dev"`). Its public keys: `GET /.well-known/http-message-signatures-directory`.
  See [AgentVivaBot](api.md#agentvivabot).
- The reason `robots` (state `blocked`): a `page` check reads the site's `robots.txt` and does not open a page that
  `AgentVivaBot` (or `*`) is asked not to open. Never `down`, no alert.

- `evidence` on `GET /v1/monitors/:id` and on the public JSON (`GET /v1/public/:team/:external_id.json`): what the
  newest check saw, public-safe fields only (`http_status`, `latency_ms`, and per type `final_host`, `redirects`,
  `moved_to`, `title`, `tool_count`, `tools`, `missing`, `confidence`). The status page shows the same as "What our
  last check saw". See [Evidence](api.md#evidence).
- The reason `moved`: an address that redirects to a domain with the same name (`v0.dev` to `v0.app`) and works is
  now `up` with reason `moved` and `evidence.moved_to`, not `degraded / redirect_offsite`. It is the one reason an `up`
  monitor carries; `GET /v1/public/search` keeps it too. A redirect to a site with a different name stays
  `degraded / redirect_offsite`.

- `GET /status` and `GET /status/:letter`: a public list of the tools we watch, by first letter, with the tools not
  working right now. A public monitor joins it after 14 days, under the same rule as search and the sitemap. Make the
  monitor private to take it off.
- `GET /robots.txt` and `GET /llms.txt`.
- "Needs a decision": `GET /v1/decisions` lists the tools down (or on a parked domain) for at least
  `decide_after_days` days (default 7, set with `PATCH /v1/team`), longest first; `status=remove_requested` lists
  the ones already sent for removal. `POST /v1/decisions` answers for up to 500 at once: `keep`, `false_alarm`,
  `stop` or `remove`. See [Needs a decision](api.md#needs-a-decision).
- The `remove_requested` webhook: `remove` sends your webhook a signed `{ "event": "remove_requested", … }` body, one
  per tool, so your own site can hide the tool. AgentViva never changes your site. Its deliveries show in
  `GET /v1/alerts` with channel `action`. Only a tool that is down, or on a parked domain, can be sent
  (`not_failing` otherwise), and a retry is never sent for a tool that came back.
- `POST /v1/monitors/actions`: `pause`, `resume` or `delete` for up to 500 monitors at once.
- `GET /v1/monitors` filters `for_days` (the state began at least N days ago), `tag` and `ids`.
- Tags: monitors have `tags` (up to 10, 1–32 of `a-z 0-9 - _`), set on create and patch or from a `tags` field in
  your catalog list. `PATCH /v1/team` takes `instant_tags`: tools with one of them still get their own Telegram
  message when the daily summary is on, and come first in the summary.
- Deleting a catalog monitor sticks: the tool goes on the catalog's not-watched list and the sync leaves it out.
  `GET /v1/catalog/not-watched` lists them; `DELETE /v1/catalog/not-watched` (body `{ "external_id" }`) is "Watch
  again". The catalog gains `not_watched` and `down_30_days`; `last_result` gains `not_watched`.
- Changes and state-change webhooks carry `event: "state_change"`, `checks` (how many agreeing checks committed
  it) and `first_seen_at`.
- `GET /v1/team` returns `decide_after_days` and `instant_tags`; monitors return `tags` and `decision`.
- MCP: `pause_monitors`, `resume_monitors`, `delete_monitors` (marked destructive), `list_decisions` and `decide`;
  `list_monitors` takes `for_days` and `tag`. Tools carry MCP annotations (`readOnlyHint`, `destructiveHint`).

- `GET /v1/claimable` accepts a pasted web address as `domain` (`https://www.Geondex.com/pricing` means
  `geondex.com`) and returns `domain`, the registrable domain it looked up.
- `PUT /v1/catalog` returns `preview.skipped`: up to five invalid rows, each `{ row, name, reason }`.
- The public JSON, the badge and the status page answer fast when our database is slow: a 5-minute cached copy,
  then the last copy up to 6 hours old (`X-AV-Stale: 1`), else `503 unavailable` with `Retry-After: 60` for the
  JSON, an "unknown" badge (`200`, `max-age=60`) and a short `503` page.
- `PATCH /v1/team` trims spaces around `telegram_chat_id`.
- An unknown path under `/v1/*` answers the documented `{ "ok": false, "error": "not_found" }` instead of plain
  text.
- The `mcp` check type: it checks a hosted MCP server the way an AI app connects (Streamable HTTP). It tries the
  modern `server/discover` first and falls back to the legacy `initialize` handshake, then lists the tools. Create
  one with `type: "mcp"`; `target.method` is `POST` only and `target.body` and `expect.rules` are left out.
  `expect.mcp.tools` (up to 50 names, each 1 to 128 characters of `A-Z a-z 0-9 _ - . /`) lists the tools the
  server must have. The default `interval_sec` is 3600 (minimum 300) and the default `confirmations` is 2.
  `GET /v1/monitors?type=mcp` filters by it. Both plans include it. See [The `mcp` check](api.md#the-mcp-check).
- New probe reasons: `auth_required` (state `blocked`: the MCP server asks for sign-in), and `mcp_invalid`,
  `mcp_error`, `mcp_version`, `mcp_tools_failed`, `mcp_tool_missing` (state `degraded`). `route_mismatch` and
  `body_truncated` also apply to `mcp` monitors.
- New evidence fields for `mcp` checks: `check`, `era`, `protocol_version`, `server`, `tool_count`, `tools`,
  `missing`, `tools_truncated`, `unchecked`, `session`, `redirect`, `hint`, `supported`, and for `auth_required`
  `auth` and `authorization_servers`.
- `GET /v1/monitors/{id}` returns `last_evidence` for an `mcp` monitor: the evidence of its newest check, or `null`.
- The `create_monitor` MCP tool takes `tools` (for `mcp`: the names that must be listed; it becomes
  `expect.mcp.tools`).
- `PATCH /v1/monitors/{id}`: changing `type` away from `mcp` drops the stored `expect.mcp`.
- `GET /badge/{team}/{public_slug}.svg` takes two optional query parameters: `style` (`dot`, `word`, `pill`, `card`,
  `seal`, `strip`, `panel`; default `card`) and `theme` (`dark`, `light`, `auto`; default `dark`). Unknown values fall
  back to the default, so existing links do not change (still the 180x48 dark card), except that the card now also
  shows how long the state has held (for example "alive 26 days"). `theme=auto` follows the reader's light or dark
  setting.
- Sign-in for MCP clients (OAuth 2.1 with PKCE): `/.well-known/oauth-protected-resource`,
  `/.well-known/oauth-authorization-server`, `POST /oauth/register`, `/oauth/authorize`, `POST /oauth/token`. An MCP
  client adds `https://agentviva.dev/mcp` with no header, the user signs in in the browser and presses **Allow**.
  Tokens work only on `/mcp`. Works with claude.ai custom connectors too. Access tokens live 1 hour; refresh
  tokens 30 days from last use, rotated on each refresh (reusing an old one ends the sign-in), and a sign-in must be
  renewed after 90 days. Registration: 10 a minute per network and overall; token: 60 a minute per client.
- `/mcp/public`: the keyless MCP mode (only `lookup_tool`) at its own address.
- Probe reasons `refused` (a `page` target answered 400, 405, 406 or 451; state `blocked`) and `route_mismatch` (a
  Cloudflare error while public DNS points the host away from Cloudflare: our network reaches a stale setup, not the
  site; state `blocked`, evidence `public_ips`). Monitors that showed `down` for these reasons move to `blocked` at
  their next probe, without an alert.

- `GET /v1/public/search?q=` (no key): up to 5 public tools AgentViva already watches, by name or address. Cached for
  5 minutes; 30 requests a minute per IP address without a key.
- MCP without a key: `/mcp` with no `Authorization` header (or `Bearer` with nothing after it) lists only the new
  `lookup_tool` (the same lookup); a key that is there but wrong is still `401`. With a key, `lookup_tool` is added
  to the eight tools.
- `GET /v1/claimable` returns `own_monitors`: your own monitors on that domain (public or not). You cannot claim
  them, but it tells you the tool is already yours when `monitors` is empty.
- `GET /v1/monitors?days=N` (1 to 30) adds `daily` to every monitor: one `{ date, worst }` per UTC day with a check,
  oldest first. `GET /v1/monitors/:id` always carries `daily` for the last 30 days.
- `GET /v1/changes` takes `monitor_id` (only the changes of that monitor) and `order=desc` (newest first).
- Every monitor object has `source`: `"catalog"` when your catalog owns it, otherwise `null`.
- `GET /v1/alerts`: your team's newest alert deliveries (tool, change, channel, whether it arrived, the error).
- `POST /v1/team/webhook/test`: one test delivery to the webhook you saved.
- MCP `list_monitors` takes `states` (a list, for example `["down", "degraded"]`), `source` (`catalog` or
  `manual`) and `sort` (`problems`, `changed` or `name`), the same filters as `GET /v1/monitors`. The text of
  `list_monitors` and `get_changes` shows each monitor's address.
- `GET /v1/monitors` takes a comma list in `state` (`state=down,degraded`), `source=catalog|manual`, and
  `sort=problems|changed|name`. Without `sort` the order and the cursor are unchanged; with a `sort` the cursor is
  positional and stays opaque.
- `GET /v1/changes`: every change has `url`, the address the monitor checks (`null` for a deleted monitor).
- `GET /v1/catalog` (and `PUT /v1/catalog`, `POST /v1/catalog/sync`): `catalog.monitors_by_state` counts the
  monitors the catalog owns, by state.
- MCP server at `https://agentviva.dev/mcp`: stateless Streamable HTTP (JSON responses, no SSE), the same Bearer key
  and rate limit as `/v1`, protocol versions `2026-07-28` (`server/discover`) and `2025-11-25`, `2025-06-18`,
  `2025-03-26` (`initialize`). Eight tools: `list_monitors`, `get_status`, `create_monitor`, `probe_now`,
  `get_changes`, `connect_catalog`, `catalog_status`, `sync_catalog`; each returns a short text and the REST JSON as
  `structuredContent`, and a REST error as `isError: true`; `create_monitor` takes `headers` for a canary on an
  endpoint that needs a key. Delete stays REST-only. It needs a client that can send an `Authorization` header
  (Claude Code, Cursor, Codex); a request whose `Origin` is an `http(s)` web origin is `403` and no CORS headers are
  sent, so claude.ai custom connectors and browser inspectors do not work until OAuth exists. No REST endpoint changed.
- Daily Telegram summary: `PATCH /v1/team` accepts `telegram_digest` (boolean) and `GET /v1/team` returns it. When
  `true`, Telegram gets no message per change; at 07:00 UTC it gets one summary of the last 24 hours (Stopped
  working, Partly working, Working again; up to 15 names each, then "and N more"). Webhooks still get one delivery
  per change.
- Connect catalog: give AgentViva one link to a JSON or CSV list of your tools instead of creating monitors one by
  one. New endpoints `GET /v1/catalog`, `PUT /v1/catalog` (reads the list once and returns a preview, saves nothing
  on failure), `POST /v1/catalog/sync` and `DELETE /v1/catalog`. They store the link and schedule a sync; none of
  them changes a monitor. New error `catalog_unreadable` (`400`), also returned when the list has no valid tool or
  a CSV quote is never closed.
- Catalog safety rules: a sync changes nothing when the list shrank by more than half, or when it would pause more
  than half of the catalog's running monitors (`last_result.fatal` says why). `POST /v1/catalog/sync` accepts an
  optional body `{ "force": true }` that lets the next sync skip these two rules once.
- Catalog `last_result` is now documented field by field and has a schema in `openapi.yaml` (`found`, `invalid`,
  `created`, `updated`, `unchanged`, `paused`, `resumed`, `partial`, `remaining`, `errors`). New optional fields on a
  stopped sync: `fatal_kind` (`unreadable`, `empty`, `shrank`, `limit`, `mass_pause`, `internal`) and `fatal_numbers`.
  The `fatal` texts for the two forceable stops now say to run the sync again with `force: true`. A sync run stops
  new work after 8 minutes (`partial` is `true`, the next run continues). A monitor resumed by hand whose tool left
  the list is paused again by the next sync.
- `POST /v1/catalog/sync` changes only `next_sync_at`, so a sync that finishes at the same moment keeps its
  `last_result`.

### Changed

- `/mcp` with no `Authorization` header now answers `401` with `WWW-Authenticate: Bearer resource_metadata="…"` (it
  starts the sign-in) instead of the keyless mode. The keyless mode is `/mcp/public`, or `Bearer` with nothing after
  it. A wrong key or token also carries `WWW-Authenticate` with `error="invalid_token"`.
- For monitors checked hourly or less often, the confirming probes after the second wait an hour instead of ten
  minutes. After `down` or `degraded`, a monitor is probed again after 1, 4 and 12 hours, then at its interval.

## 2026-10-16

### Added

- Telegram can be linked from the dashboard: Alerts, then Connect Telegram, opens the bot with a one-time link
  (15 minutes, single use). Setting `telegram_chat_id` through `PATCH /v1/team` works as before. No endpoint changed.
- The key page is now `https://agentviva.dev/app/api` (it was `/app/settings`). No endpoint changed.
- `GET /v1/monitors` accepts `q`: a case-insensitive substring of the monitor's name or target, 1 to 100
  characters. `%` and `_` match themselves. It combines with the other filters.
- `GET /v1/claimable?domain=…` lists the public monitors of other teams on a domain, at most 20, each with its
  `status_url`. Use it to find the monitor to pass to `POST /v1/claims`. A shared suffix such as `vercel.app` is
  `400 domain_not_claimable`. Each `url` is the target's origin and path only; the query string, fragment and user
  info are removed.

## 2026-10-11

### Added

- During early access, invited users sign in with Google at `https://agentviva.dev/login` and make or replace
  their API key at `https://agentviva.dev/app/settings`. Making a new key stops the old one at once. No endpoint
  changed.
- `PATCH /v1/monitors/:id` accepts `paused`. A pause-only patch is also allowed on a canary monitor after a
  downgrade from pro.

### Fixed

- Editing a monitor created without `external_id` or `public_slug` no longer fails.
- Error `detail` names the field when one field is at fault, for example `target.url: Invalid url`.

## [1.0.1] - 2026-09-22

### Fixed

- The skill pointed at `../../api.md` and `../../openapi.yaml` as plain relative paths. That resolves when
  the whole repository is installed, but not when only the skill directory is copied, which is what
  `npx skills add` does. The skill now says to read the local file if it is there and otherwise fetch the
  raw URL.

### Added

- `skills.sh.json`, so the skill is grouped correctly on skills.sh
- The README documents `npx skills add AgentViva/docs` alongside the Claude Code plugin install

## [1.0.0] - 2026-09-22

First public release of the reference. The service has been live at `agentviva.dev` since 2026-09-21.

### Added

- `api.md` — the full v1 contract: monitors, bulk upsert, probes, the change feed, team settings, claims, and
  the public status, badge and JSON surfaces
- `openapi.yaml` — the same contract as OpenAPI 3.1
- A Claude Code skill at `skills/agentviva`
- `SECURITY.md` and this changelog
