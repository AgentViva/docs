<!-- Source of truth lives in the private service repo at docs/api.md. Edit it there, then copy here. -->

# AgentViva public API — v1 (v0 build)

This is the only document a customer integration may rely on, aigregator's included. Everything here is
implemented in v0 unless marked **v1**.

Base URL: `https://agentviva.dev`. All request and response bodies are JSON. Times are Unix milliseconds.

## Authentication

```text
Authorization: Bearer av_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Keys are issued per team and shown once. During early access, invited users sign in with Google at
`https://agentviva.dev/login` and make or replace their key at `https://agentviva.dev/app/api`; making a new key
stops the old one at once. Missing or unknown key → `401 { "ok": false, "error": "unauthorized" }`.

## Errors

```json
{ "ok": false, "error": "invalid_target", "detail": "private address ranges are not allowed" }
```

`detail` starts with the field path when one field is at fault, e.g. `target.url: Invalid url`.

| Status | `error` |
| --- | --- |
| 400 | `invalid_json`, `invalid_monitor`, `invalid_target` (a disallowed scheme or private host in either a monitor's `target.url` or `PATCH /v1/team`'s `webhook_url` — the same rule applies to both), `invalid_query` (a malformed query string, or a request body that fails validation when no more specific code applies, e.g. `PATCH /v1/team`), `domain_not_claimable` (`POST /v1/claims` on a monitor, or `GET /v1/claimable` for a domain, that is a shared or public suffix nobody can prove they own), `webhook_test_failed` (`PATCH /v1/team` set `webhook_url` and the one test delivery did not answer 2xx within 5 s), `catalog_unreadable` (`PUT /v1/catalog`: the link could not be fetched or is not a JSON or CSV list with an id and a URL per tool, or holds no valid tool; `detail` says why) |
| 401 | `unauthorized` |
| 402 | `plan_limit` (monitor limit for the plan reached), `plan_required` (the request needs a plan this team does not have — today: a canary check on `free`) |
| 404 | `not_found` |
| 413 | `body_too_large` (> 64 KB) |
| 429 | `rate_limited` (600 requests per minute per key) |
| 500 | `internal_error` (unexpected server error; logged our side, safe to retry) |

## Monitor object

```json
{
  "id": "0b2f…",
  "external_id": "notion",
  "type": "page",
  "name": "Notion",
  "target": { "url": "https://www.notion.so", "method": "GET", "headers": {}, "body": null },
  "expect": { "status": [200, 299], "max_ms": 10000, "rules": [], "canary": null },
  "interval_sec": 86400,
  "confirmations": 3,
  "public": true,
  "public_slug": "notion",
  "paused": false,
  "state": "up",
  "state_since": 1789200000000,
  "last_probe_at": 1789286400000,
  "last_ok_at": 1789286400000,
  "last_latency_ms": 412,
  "last_reason": null,
  "created_at": 1789100000000,
  "updated_at": 1789286400000
}
```

Fields you send on create; everything else is set by the server:

| Field | Required | Rules |
| --- | --- | --- |
| `type` | yes | `page`, `json`, `canary` |
| `name` | yes | 1–120 characters |
| `external_id` | no | ≤ 128 characters; unique within your team; upsert key for bulk |
| `target.url` | yes | `http` or `https`; public hosts only (no 10/8, 172.16/12, 192.168/16, 127/8, link-local, `localhost`) |
| `target.method` | no | default `GET` for `page`, `POST` for `canary` |
| `target.headers` | no | ≤ 10 entries, each value ≤ 1 KB; stored encrypted at rest **v1**, plain in v0 |
| `target.body` | no | ≤ 16 KB; string or JSON |
| `expect.status` | no | `[min, max]`, default `[200, 299]` |
| `expect.max_ms` | no | default 10000; this bounds the HTTP fetch itself. For `page`/`json` checks that fetch is also the whole check, so 10 s is the worst-case total. A `canary` check adds a second, independent call to the judge model afterward (its own fixed ~8 s budget, separate from `max_ms`), so a canary's worst-case total latency is closer to 18 s; `latency_ms` in a probe result reflects only the first (fetch) phase |
| `expect.rules` | no | ≤ 10 of `{ "path": "choices[0].message.content", "op": "exists\|eq\|contains\|regex\|gte\|lte", "value": … }` |
| `expect.canary` | canary only | `{ "question": "…", "expectation": "…", "reply_path": "choices[0].message.content" }`; the question is substituted into `target.body` where the string `{{question}}` appears |
| `interval_sec` | no | `page` ≥ 3600 (default 86400); `json`, `canary` ≥ 300 (default 21600) |
| `confirmations` | no | 1–5; default 3 for `page`, 2 otherwise |
| `public` | no | default `false` |
| `public_slug` | no | required when `public`; `[a-z0-9-]{1,64}`; unique among your team's currently public monitors (a private monitor's `public_slug` is not checked for collisions and can be reused by another monitor) |

States: `unknown`, `up`, `degraded`, `down`, `blocked`. A state changes only after `confirmations` consecutive
probes agree, re-probed ten minutes apart. `blocked` means the target refused the probe (403, 429, bot challenge)
and is never treated as dead.

Reasons (`last_reason`, `state_changes.reason`): `net_error`, `timeout`, `http_404`, `http_5xx`, `http_other`,
`challenge`, `parked`, `redirect_offsite`, `redirect_loop`, `redirect_refused`, `rule_failed`, `latency`,
`invalid_json` (the probe reason — a JSON/canary target's body did not parse; distinct from the `400`
request-validation error code of the same name above, which is about a malformed API *request*, not a
probe result), `body_truncated` (a JSON/canary response was larger than our read cap and got cut off
before it could be parsed — `up`, not `degraded`, because this is our limitation, not evidence against
the target), `judge_failed`, `judge_unavailable`, `alarm_retry_exhausted` (the platform retried a
monitor's alarm invocation past our own retry limit; the probe row this reason is attached to never
carries a completed check result, and never feeds a state change).

## Endpoints

### `POST /v1/monitors`

Body: one monitor object (create fields). Returns `201 { "ok": true, "monitor": { … } }`. The first probe runs
within `interval_sec` (randomised so bulk creations spread out).

### `POST /v1/monitors/bulk`

```json
{ "monitors": [
  { "external_id": "notion", "type": "page", "name": "Notion",
    "target": { "url": "https://www.notion.so" }, "public": true, "public_slug": "notion" }
] }
```

Up to 1,000 items. Upsert by `external_id`: an existing monitor with the same `external_id` is updated (fields
you send replace the stored ones; state and history are kept), otherwise created. Returns
`200 { "ok": true, "results": [ { "external_id": "notion", "id": "…", "outcome": "created" } ] }` where
`outcome` is `created`, `updated` or `error` (with `error` and `detail`). Items are processed independently; one
bad item does not fail the batch. A per-item `error` uses the same codes as `POST /v1/monitors`
(`invalid_monitor`, `invalid_target`), plus `plan_required` for a `canary` item when the team's plan does not
include canary checks, `plan_limit` when the team's monitor limit is reached partway through the batch, or
`internal_error` for an unexpected failure processing that one item (logged our side, safe to retry that
item).

### `GET /v1/monitors`

Query: `state` (one of the states above, or a comma list such as `down,degraded`), `type` (`page`, `json`,
`canary`), `external_id`, `public` (`true` or `false`), `source` (`catalog` or `manual`), `q` (text), `sort`
(`problems`, `changed` or `name`), `limit` (integer ≥ 1, ≤ 500, default 100), `cursor`. `q` matches the name or
the target, ignoring case, as a plain substring: `%` and `_` mean themselves. It must be 1 to 100 characters after
trimming, otherwise `400 invalid_query`. It combines with the other filters. A `limit` that is not a number, or is
below 1, is `400 invalid_query`; a `limit` above 500 is clamped down to 500 rather than rejected. A `state`
(every item of a comma list must be a state), `type`, `public`, `source` or `sort` value outside the ones listed
here is `400 invalid_query`, not a silently empty or default page. A comma list matches a monitor in any of its
states; a state that appears twice does no harm.

`source=catalog` returns the monitors your [catalog](#catalog) owns. `source=manual` returns the monitors made by
hand or through the API. A monitor whose tool left the catalog list and whose sync has not paused it yet matches
neither.

`sort` sets the order:

| `sort` | Order |
| --- | --- |
| (none) | By monitor `id`, which is the order of creation. This is the default and does not change |
| `problems` | `down`, `degraded`, `blocked`, `unknown`, `up`. Inside a state, the newest `state_since` first (a monitor that was never checked last), then `id` |
| `changed` | The newest `state_since` first (a monitor that was never checked last), then `id` |
| `name` | The name, ignoring case, A to Z, then `id` |

With no `sort`, `next_cursor` is the `id` of the last monitor on the page. With a `sort`, `next_cursor` is
positional: it says how many monitors came before the next page, so treat it as an opaque string and pass it back
unchanged. A cursor that is not one `next_cursor` gave you is `400 invalid_query` on a sorted list. Because the
cursor is a position, a monitor can be skipped or shown twice when monitors change state, are added or are deleted
between two pages of a sorted list. Leave out `sort` when you need an exact copy.

Returns `{ "ok": true, "monitors": [ … ], "next_cursor": "…" | null }`.

### `GET /v1/monitors/:id`

Returns the monitor plus `last_probes`: the last 30 results `{ at, verdict, http_status, latency_ms, reason }`.

### `PATCH /v1/monitors/:id`

Body: any subset of the create fields, plus `paused` (boolean). Changing `interval_sec` or `target` reschedules
the next probe. `paused` pauses or resumes probing: a resumed monitor probes within about a minute, bulk upsert
never changes it, and it is allowed on a canary monitor even when the plan no longer includes canary.

### `DELETE /v1/monitors/:id`

Returns `204`. History rows are kept for 30 days and then pruned.

### `POST /v1/monitors/:id/probe`

Runs one probe now and returns `{ "ok": true, "result": { verdict, http_status, latency_ms, reason, evidence } }`.
Does not change the schedule. Counts toward the confirmation rule like a scheduled probe.

### `GET /v1/changes`

Query: `since` (ms, required), `limit` (integer ≥ 1, ≤ 1000, default 200), `cursor`. A `limit` that is not a
number, or is below 1, is `400 invalid_query`, same as a missing `since`; a `limit` above 1000 is clamped down
to 1000 rather than rejected. Returns state changes for your team, oldest first:

```json
{ "ok": true, "changes": [ {
  "id": "…", "monitor_id": "…", "external_id": "notion", "name": "Notion",
  "from": "up", "to": "down", "at": 1789286400000, "reason": "http_404",
  "evidence": { "http_status": 404, "final_url": "https://www.notion.so/", "snippet": null, "judge_note": null },
  "url": "https://www.notion.so/",
  "status_url": "https://agentviva.dev/s/aigregator/notion"
} ], "next_cursor": null }
```

`url` is the address the monitor checks (`target.url`). It is `null` when the monitor has been deleted.

Store the `at` of the last change you processed and pass it as `since` next time. Changes are never edited or
deleted inside 90 days, so this feed is safe to replay.

### `GET /v1/team`

Returns `{ "ok": true, "team": { "name", "slug", "plan", "monitor_limit", "monitors_by_state": { … },
"telegram_linked": bool, "telegram_digest": bool, "webhook_configured": bool } }`.

### `PATCH /v1/team`

Body: `{ "webhook_url": "http://…" | "https://…" | null, "telegram_chat_id": "…" | null, "telegram_digest": bool }`.
Every field is optional; one you leave out stays as it is. `telegram_digest` turns the daily Telegram summary on
or off (see [Telegram](#telegram)). `webhook_url` must
be `http` or `https` and a public host — the same rule a monitor's `target.url` follows (no `10/8`,
`172.16/12`, `192.168/16`, `127/8`, link-local, `localhost`); a disallowed URL is `400 invalid_target`. `https`
is recommended: the payload is signed, not encrypted, so a plain `http` endpoint exposes the change data (not
the secret) to anyone on the network path. Setting `webhook_url` then sends one test delivery and returns
`400 webhook_test_failed` if it does not answer 2xx within 5 s; a redirect is not followed and counts as a
failure like any other non-2xx.

## Claims

A claim gives your team rights over a public monitor owned by someone else — typically the monitor an AI
directory created for your tool. Ownership does not move; the original team keeps the monitor.

**What a verified claim buys:** every state change on that monitor is delivered to your team's channels too,
alongside the owner's. That is all it buys today. It does not let you edit, pause or delete the monitor, and
it does not give you a canary check — changing a monitor's type is the owner's to do. A *pending* claim buys
nothing at all: anyone may create one against any public monitor, so only verification routes anything.

| Endpoint | What it does |
| --- | --- |
| `POST /v1/claims` | `{ "monitor_id": "..." }` → a claim with a token and both verification methods. Calling it twice returns the same token. `404` for a private or unknown monitor; `400` `invalid_query` if your team already owns it; `400` `domain_not_claimable` if its domain cannot be proved (see below); `402` `plan_required` if your plan does not include claims — both current plans do, so this cannot happen today |
| `POST /v1/claims/:id/verify` | checks the proof. `200` verified, `422` `verification_failed`, `409` `already_claimed` |
| `GET /v1/claimable` | public monitors of other teams on a domain; see below |
| `GET /v1/claims` | your team's claims |
| `DELETE /v1/claims/:id` | `204`, no body. Drops a claim, freeing the monitor for someone else |

### `GET /v1/claimable`

Query: `domain` (required), a domain name such as `geondex.com`. Returns the public monitors of other teams whose
target is on that registrable domain, at most 20, sorted by name. Your own monitors and private monitors are never
listed. Use it to find the monitor to pass to `POST /v1/claims`. `url` is the target's origin and path only: the
query string, the fragment and any user info are removed, because they can hold a token.

```json
{
  "ok": true,
  "monitors": [
    {
      "id": "0b2f1a2e-4c3d-4b8a-9e21-abcdef123456",
      "name": "Geondex",
      "team": "aigregator",
      "public_slug": "geondex",
      "url": "https://www.geondex.com/",
      "status_url": "https://agentviva.dev/s/aigregator/geondex"
    }
  ]
}
```

A malformed or missing `domain` is `400 invalid_query`. A shared or public suffix such as `foo.vercel.app` is
`400 domain_not_claimable`, the same refusal `POST /v1/claims` gives.

Prove ownership of the registrable domain either way:

- **DNS**, preferred: a `TXT` record at `_agentviva.<domain>` whose value is the token.
- **File**: the token as the entire body of `https://<domain>/.well-known/agentviva-challenge`.

We check DNS first, then the file. Verifying `example.com` covers a monitor pointing at
`https://www.example.com/product`.

A monitor hosted on a shared suffix cannot be claimed: `*.vercel.app`, `*.hf.space`, `*.streamlit.app`,
`*.github.io`, `*.pages.dev`, `*.replit.app` and the like, plus country second levels such as `com.ua`,
`co.id` and `com.pl`. For those the domain we would ask you to prove is the platform's, not yours, so
`POST /v1/claims` refuses with `400` `domain_not_claimable` instead of issuing a token you could never
satisfy. Point the monitor at a domain you control, or ask the monitor's owner to.

A claim is verified once and is never re-checked. If the owner later points the monitor at a different
domain, or makes it private, an existing verified claim stays verified and keeps routing alerts to you
until one of you deletes it.

## Catalog

Instead of creating monitors one by one, give AgentViva one link to a list of your tools. A team has at most one
catalog. These endpoints store the link and schedule a sync; they never create, change or delete a monitor
themselves. The sync (creating, updating, pausing and resuming monitors from the list) runs in the background.

**The list.** A public `http` or `https` link to a JSON or CSV file, at most 10 MB. The fetch uses
`AgentVivaBot`, waits at most 15 s, follows at most 5 redirects, and applies the same host rule as a monitor's
`target.url` to the link and to every redirect (no private ranges, no `localhost`, `http` and `https` only).

- JSON: an array of objects, or an object whose `tools`, `items`, `monitors` or `data` field is that array.
- CSV: a header row, then one row per tool. Quoted fields, doubled quotes and newlines inside quotes work.
- Field names (case-insensitive, the first one found wins): the id is `id`, `external_id`, `slug` or
  `name_slug`; the name is `name` or `title` (the id is used when there is none); the URL is `url`, `website`,
  `homepage`, `link` or `ai_tool_url`.
- A row without an id or without a URL is counted as invalid and skipped. When an id appears twice, the first
  row is kept and the others are counted as invalid. Spaces around values are trimmed. A CSV with a quote that is
  never closed is refused, and so is a list with no valid tool at all (an empty array, a header-only CSV, or only
  invalid rows).

| Endpoint | What it does |
| --- | --- |
| `GET /v1/catalog` | `{ "ok": true, "catalog": { "url", "public", "last_sync_at", "next_sync_at", "last_result", "monitors_by_state" } }`, or `"catalog": null` when none is connected |
| `PUT /v1/catalog` | Body `{ "url": "https://…", "public": true }` (`public` defaults to `true`). Fetches and reads the list once, saves the link, schedules the first sync for now. Returns `catalog` and `"preview": { "found", "invalid", "sample" }` where `sample` is the first 3 tools. `400` `invalid_target` for a disallowed link, `400` `catalog_unreadable` when it cannot be fetched or read, or holds no valid tool (`detail` says why); nothing is saved then |
| `POST /v1/catalog/sync` | Schedules a sync now (it starts within 10 minutes) and returns `catalog`. Optional body `{ "force": true }` ("Apply anyway"): the next sync skips the two safety rules below once. `400` `invalid_query` when `force` is not a boolean, `404` `not_found` when no catalog is connected |
| `DELETE /v1/catalog` | `204`, no body. Removes the link. Every monitor stays as it is; delete them yourself if you want them gone. `404` `not_found` when no catalog is connected |

Times are Unix milliseconds; `last_sync_at` and `last_result` are `null` until the first sync has run, and
`last_result` is an object once it has. `public` says whether the monitors the sync creates are public.
`monitors_by_state` counts the monitors the catalog owns (`source=catalog`) by state, like `monitors_by_state` in
`GET /v1/team`; a state with no monitor is left out. `PUT /v1/catalog` and `POST /v1/catalog/sync` return it too.

**`last_result`** has these fields:

| Field | Meaning |
| --- | --- |
| `found` | Tools with a valid id and URL in the list |
| `invalid` | Rows skipped: no id, no URL, or an id that appeared twice |
| `created` | Monitors added |
| `updated` | Monitors whose name or URL changed |
| `unchanged` | Monitors that already matched the list |
| `paused` | Monitors paused because their tool left the list |
| `resumed` | Paused monitors resumed because their tool came back |
| `partial` | `true` when the run stopped with work left (the batch limit of 1,000 creates and updates, or the 8-minute time limit); the next run, within 10 minutes, continues |
| `remaining` | Creates and updates still waiting. It is `0` when the time limit stopped a run before its resumes or pauses were done |
| `errors` | The first 20 tools that could not be created, updated, paused or resumed: `[{ "external_id", "detail" }]` |
| `fatal` | Only when the sync stopped before changing anything: a sentence that says what to check |
| `fatal_kind` | With `fatal`: `unreadable`, `empty`, `shrank`, `limit`, `mass_pause` or `internal`. `shrank` and `mass_pause` are the two stops `force` lifts |
| `fatal_numbers` | With `shrank`: `[monitors the catalog has, tools in the list]`. With `mass_pause`: `[monitors to pause, running catalog monitors]` |

A monitor you resume by hand after its tool left the list is paused again by the next sync; to keep watching a
tool, keep it in the list.

**Safety rules.** A sync stops before it changes any monitor, and `last_result.fatal` says why, when the link
cannot be read, when the list is empty, when the list has more tools than your monitor limit allows, and in two
more cases that protect you from a broken or cut-short list:

- the list has fewer than half as many tools as the catalog has monitors (it shrank by more than half);
- the sync would pause more than half of the catalog's running monitors (for example, the same number of tools
  but other ids).

When such a change is real, send `POST /v1/catalog/sync` with `{ "force": true }`. The next sync then skips these
two rules once. The force waits for a sync that gets past the other checks (an unreadable link does not use it
up), and a forced run that stops at the time limit keeps it for the next run. A `PUT` or `DELETE` of the catalog
drops it.

## Public endpoints (no key)

| Path | Returns |
| --- | --- |
| `GET /v1/public/:team/:external_id.json` | `{ state, since, last_probe_at, latency_ms, reason }` for public monitors; `Access-Control-Allow-Origin: *`; `Cache-Control: public, max-age=300`; 404 for private or unknown |
| `GET /badge/:team/:public_slug.svg` | live SVG badge, 180×48, `max-age=300` |
| `GET /s/:team/:public_slug` | HTML status page, 30-day history, no JavaScript |
| `GET /bot` | what AgentVivaBot is and how to opt out |
| `GET /healthz` | `ok` |

Team slug for aigregator is `aigregator`.

## Webhook deliveries

When configured, every committed state change into `down`, `degraded` or back to `up` is delivered once:

```text
POST <webhook_url>
Content-Type: application/json
X-AgentViva-Timestamp: 1789286400
X-AgentViva-Signature: sha256=<hex HMAC-SHA256(webhook_secret, "<timestamp>.<raw body>")>
```

Body: the same object as one item of `GET /v1/changes`. Any 2xx within 5 s is success; a redirect is not
followed and counts as a failure. Failures are retried by the hourly job up to 3 times. Verify the signature
over the raw body before parsing; reject timestamps older than five minutes. `blocked` and `unknown → up` are
never delivered.

## Telegram

Link a chat by setting `telegram_chat_id` through `PATCH /v1/team`. In the dashboard there is a shorter way: Alerts,
then Connect Telegram, opens the AgentViva bot with a one-time link; press Start in the chat and the chat id is saved
for you. The link works once and expires after 15 minutes. The button appears only when the bot is set up. The API
is the same either way. Messages are sent for the same events as webhooks, one per change, in HTML mode with all
values escaped.

Set `telegram_digest` to `true` to get one summary a day instead. Telegram then gets no message per change; at
07:00 UTC it gets one message with the confirmed changes of the previous 24 hours, in three groups (Stopped working,
Partly working, Working again) with up to 15 names each, a status link for every public monitor, "and N more" and a
link to the dashboard. A long summary is shortened (shorter names, then fewer names per group) to fit Telegram's
message limit; the counts stay exact. A day with no changes sends nothing. Changes to `blocked` and
`unknown → up` are left out, as they are from single messages. If Telegram does not accept the message, it is not
sent again; the next day's summary covers its own 24 hours. Webhooks are not affected: they still get one delivery
per change.

## Plans and limits

| Plan | Canary checks | Public monitors | History | Claims | Channels |
| --- | --- | --- | --- | --- | --- |
| `free` | no | yes | 30 days | yes | webhook, Telegram |
| `pro` | yes | yes | 30 days (365 planned) | yes | webhook, Telegram (Slack, email **v1**) |

`monitor_limit` is set per team, not by plan; `GET /v1/team` returns the team's own value.

Rate limit: 600 requests per minute per key. Bulk calls count as one request.

## MCP server

An MCP (Model Context Protocol) server at `https://agentviva.dev/mcp`, so an AI agent can read monitor states and
manage monitors without writing REST calls. It calls the same handlers as the REST API, so every rule in this
document (plan gates, limits, address checks) applies unchanged. Delete is not offered; it stays REST-only.

### Connect

Claude Code, one line:

```bash
claude mcp add --transport http agentviva https://agentviva.dev/mcp --header "Authorization: Bearer av_live_..."
```

The server needs the key in an `Authorization` header, as the REST API does. So it works with clients that can send
a custom header: Claude Code (`--header`), Cursor (`headers` in `mcp.json`) and Codex (`http_headers` in
`config.toml`). It does **not** yet work with claude.ai custom connectors, nor with browser-based inspectors: those
ask for OAuth or send a web `Origin` header, which this endpoint refuses (`403`). OAuth sign-in is not built.

### Transport

Stateless Streamable HTTP with JSON responses (no SSE, no session id). Every message is one `POST /mcp` with one
JSON-RPC 2.0 request or notification in the body (a JSON array is `-32600`); the body is capped at 64 KB and the
rate limit is the REST one.

- A request gets one JSON-RPC response as `application/json`. A notification gets `202 Accepted` with no body.
- `GET /mcp` and `DELETE /mcp` (and any method but `POST`) get `405` with `Allow: POST`.
- A request whose `Origin` header is an `http://` or `https://` web origin (a browser page) gets `403`, and no CORS
  headers are ever sent. No `Origin`, `Origin: null` or another scheme (a desktop app) is accepted. A missing or
  unknown key gets `401` (the REST error body).
- `Accept`, `Mcp-Method` and `Mcp-Name` headers are not required.
- Errors use the JSON-RPC codes: `-32700` bad JSON, `-32600` bad message, `-32601` unknown method, `-32602` unknown
  tool or bad arguments (the message says which argument), `-32022` unsupported protocol version.

Two protocol eras are served from the same URL:

| Era | Versions | How a client starts |
| --- | --- | --- |
| Modern | `2026-07-28` | No handshake. Call `server/discover` to learn the server, then send every request with `params._meta["io.modelcontextprotocol/protocolVersion"]` (and the `MCP-Protocol-Version` header; if both are sent they must match, else `400`). Results carry `"resultType": "complete"`. |
| Legacy | `2025-11-25`, `2025-06-18`, `2025-03-26` | `initialize`, then `notifications/initialized` (`202`), then requests with the `MCP-Protocol-Version` header (it may be absent for `2025-03-26`). `ping` answers `{}`. |

An unsupported version is `400` with error `-32022` and `data: { "supported": [...], "requested": "..." }`.
`server/discover` answers whatever version the client names.

### Tools

`tools/list` returns each tool's name, title, description and JSON Schema. `tools/call` returns a short text summary in
`content` and the REST JSON body in `structuredContent`. A failed REST call (a `4xx`) is a result with
`"isError": true`, the text `<error>: <detail>` (for example `invalid_target: private address ranges are not
allowed`) and the REST error body in `structuredContent`; the error codes are the ones in [Errors](#errors).

| Tool | Arguments | Same as |
| --- | --- | --- |
| `list_monitors` | `state` or `states` (a list, for example `["down", "degraded"]`; not both), `type`, `source` (`catalog` or `manual`), `sort` (`problems`, `changed` or `name`), `q`, `limit` (1 to 100, default 25), `cursor` | `GET /v1/monitors` |
| `get_status` | exactly one of `id` or `external_id` | `GET /v1/monitors/{id}` |
| `create_monitor` | `type`, `name`, `url`; optional `public`, `public_slug`, `interval_sec`, `external_id`, `method`, `body`, `headers` (up to 10, for an endpoint that needs a key), `rules` (json), `canary` (canary) | `POST /v1/monitors` |
| `probe_now` | `id` | `POST /v1/monitors/{id}/probe` |
| `get_changes` | `since` (ms, default 24 hours ago), `limit` (1 to 200, default 50), `cursor` | `GET /v1/changes` |
| `connect_catalog` | `url`; optional `public` (default `true`) | `PUT /v1/catalog` |
| `catalog_status` | none | `GET /v1/catalog` |
| `sync_catalog` | optional `force` | `POST /v1/catalog/sync` |

`create_monitor` takes the address as a flat `url`; `method`, `body` and `headers` are the `target` fields of the same
names, and `rules` and `canary` are the `expect.rules` and `expect.canary` of the [monitor object](#monitor-object).
Arguments a tool does not know are `-32602`, not ignored. `list_monitors` and `get_changes` return `next_cursor` in
`structuredContent` when more are available (the text also names it, and each monitor's `id` and `external_id`);
pass it back as `cursor`. Their text also shows each monitor's address.
