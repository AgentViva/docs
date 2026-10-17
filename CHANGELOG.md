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
