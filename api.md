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

MCP clients can also sign in in the browser instead of sending a key (OAuth 2.1, see
[Sign-in for MCP clients](#sign-in-for-mcp-clients-oauth-21)). Those tokens work only on `/mcp`, never on `/v1`.

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
| 503 | `unavailable` (`GET /v1/public/:team/:external_id.json` only: our database did not answer within 3 s and no recent copy was at hand; `Retry-After: 60`) |

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
  "source": null,
  "created_at": 1789100000000,
  "updated_at": 1789286400000
}
```

`source` is `"catalog"` when your [catalog](#catalog) owns the monitor, otherwise `null`.

Fields you send on create; everything else is set by the server:

| Field | Required | Rules |
| --- | --- | --- |
| `type` | yes | `page`, `json`, `canary`, `mcp` |
| `name` | yes | 1–120 characters |
| `external_id` | no | ≤ 128 characters; unique within your team; upsert key for bulk |
| `target.url` | yes | `http` or `https`; public hosts only (no 10/8, 172.16/12, 192.168/16, 127/8, link-local, `localhost`) |
| `target.method` | no | default `GET` for `page`, `POST` for `canary` and `mcp`; `mcp` takes only `POST` |
| `target.headers` | no | ≤ 10 entries, each value ≤ 1 KB; stored encrypted at rest **v1**, plain in v0 |
| `target.body` | no | ≤ 16 KB; string or JSON; must be left out for `mcp` (the check writes its own messages) |
| `expect.status` | no | `[min, max]`, default `[200, 299]`; not used by `mcp` (the protocol fixes the statuses) |
| `expect.max_ms` | no | default 10000; this bounds the HTTP fetch itself. For `page`/`json` checks that fetch is also the whole check, so 10 s is the worst-case total. A `canary` check adds a second, independent call to the judge model afterward (its own fixed ~8 s budget, separate from `max_ms`), so a canary's worst-case total latency is closer to 18 s; `latency_ms` in a probe result reflects only the first (fetch) phase. For `mcp`, it is compared with the one request that gave the good answer (`server/discover` or `initialize`), and the whole check has a 15 s budget |
| `expect.rules` | no | ≤ 10 of `{ "path": "choices[0].message.content", "op": "exists\|eq\|contains\|regex\|gte\|lte", "value": … }`; must be empty for `mcp` |
| `expect.canary` | canary only | `{ "question": "…", "expectation": "…", "reply_path": "choices[0].message.content" }`; the question is substituted into `target.body` where the string `{{question}}` appears; must be `null` for `mcp` |
| `expect.mcp` | mcp only | `{ "tools": ["search", "fetch"] }`: tool names the server must list, ≤ 50, each 1–128 characters of `A-Z a-z 0-9 _ - . /`; left out means `{ "tools": [] }`. Refused on the other types (`null` is accepted there and not stored) |
| `interval_sec` | no | `page` ≥ 3600 (default 86400); `json`, `canary` ≥ 300 (default 21600); `mcp` ≥ 300 (default 3600) |
| `confirmations` | no | 1–5; default 3 for `page`, 2 otherwise |
| `public` | no | default `false` |
| `public_slug` | no | required when `public`; `[a-z0-9-]{1,64}`; unique among your team's currently public monitors (a private monitor's `public_slug` is not checked for collisions and can be reused by another monitor) |

States: `unknown`, `up`, `degraded`, `down`, `blocked`. A state changes only after `confirmations` consecutive
probes agree, re-probed ten minutes apart; for a monitor checked hourly or less often, the probes after the second
wait an hour, so a short problem never commits. After a monitor becomes `down` or `degraded`, it is probed again
after 1, 4 and 12 hours, then at its normal interval (never later than the interval), so a recovery shows within
hours. `blocked` means the target refused the probe (401, 403, 429, 400, 405, 406, 451, a bot challenge), that an
MCP server asks AI apps to sign in (`auth_required`), or that our network reaches a different server than the public
does (`route_mismatch`), and is never treated as dead.

Reasons (`last_reason`, `state_changes.reason`): `net_error`, `timeout`, `http_404`, `http_5xx`, `http_other`,
`challenge`, `refused` (a `page` target answered 400, 405, 406 or 451: it refused our probe; state `blocked`),
`route_mismatch` (a `page` or `mcp` target gave a Cloudflare error, 520 to 527 or 530, while public DNS points the
host away from Cloudflare: our network reaches a stale Cloudflare setup, not the site; state `blocked`, evidence
`public_ips`), `parked`, `redirect_offsite`, `redirect_loop`, `redirect_refused`, `rule_failed`, `latency`,
`invalid_json` (the probe reason — a JSON/canary target's body did not parse; distinct from the `400`
request-validation error code of the same name above, which is about a malformed API *request*, not a
probe result), `body_truncated` (a JSON/canary response, or an MCP reply, was larger than our read cap and got cut
off before it could be parsed — `up`, not `degraded`, because this is our limitation, not evidence against
the target), `judge_failed`, `judge_unavailable`, `alarm_retry_exhausted` (the platform retried a
monitor's alarm invocation past our own retry limit; the probe row this reason is attached to never
carries a completed check result, and never feeds a state change). The `mcp` check adds `auth_required`,
`mcp_invalid`, `mcp_error`, `mcp_version`, `mcp_tools_failed` and `mcp_tool_missing` (see below).

### The `mcp` check

It answers "does this hosted MCP server work for an AI app right now?". One probe is a short client session over
Streamable HTTP. Every request is a `POST` to `target.url` with `content-type: application/json`,
`accept: application/json, text/event-stream`, our user agent, then your `target.headers` (for a server that needs
a key). A reply may be JSON or an event stream (SSE); both are read, each up to 1 MB.

1. `server/discover` with protocol `2026-07-28` (in `params._meta` and the `MCP-Protocol-Version` header). A server
   that lists `2026-07-28` is **modern**: we call `tools/list` with the same `_meta`.
2. Any other answer that shows an HTTP server (for example a `400` that names the versions it speaks, a `404`, a
   JSON-RPC error with any status, even a `5xx`) is **legacy**: we run `initialize` with the newest version both
   sides speak (`2025-11-25`, `2025-06-18`, `2025-03-26` or `2024-11-05`), send `notifications/initialized`, call
   `tools/list` with the `MCP-Protocol-Version` header and the `Mcp-Session-Id` the server gave, and end the session
   with a `DELETE`. So a legacy server gets one extra request per probe (the discover request), as the MCP
   specification recommends. When `initialize` is refused with an error that names the versions the server speaks
   (`data.supported`), we try once more with the newest of them that we speak.
3. `tools/list` runs only when the server says it has tools; we follow `nextCursor` for at most 10 pages. Every
   name in `expect.mcp.tools` must be listed. When our limits cut the list (10 pages, or a `nextCursor` longer than
   4,096 characters), a required name we did not see is not called missing: the result stays `up`, with
   `tools_truncated: true` and the names in `unchecked`.

A reply's `id` may be our number or the same number as a string.

The first request follows redirects like the other checks (private hosts refused); a `307` or `308` keeps the
`POST`, and a `301`, `302` or `303`, which would turn it into a `GET`, is `mcp_invalid`. Later requests go to the
final address and follow no redirect.

| Outcome | State | Reason |
| --- | --- | --- |
| Handshake good, tools listed (or no tools), required tools present | `up` | `null` |
| `401` (the server asks for sign-in), to any request | `blocked` | `auth_required` |
| A `403`, a `429` (to any request), a bot challenge | `blocked` | `challenge` |
| Network error / no answer in 15 s | `down` | `net_error` / `timeout` |
| `5xx` to discover with no JSON-RPC body, or to `initialize` | `down` | `http_5xx` (or `blocked` / `route_mismatch`) |
| `404` or `410` to both discover and `initialize`, with no JSON-RPC body | `down` | `http_404` |
| A `405` to `initialize`, with no JSON-RPC body (no MCP over `POST` here; evidence `hint: "old_sse_transport"` when the path ends in `/sse`) | `degraded` | `mcp_invalid` |
| Another non-2xx to `initialize`, with no JSON-RPC body | `down` | `http_other` |
| A 2xx that is not a valid JSON-RPC answer (a web page, a wrong shape), or a redirect that turns `POST` into `GET` | `degraded` | `mcp_invalid` |
| A JSON-RPC error to `initialize` | `degraded` | `mcp_error` |
| A modern server that lists no version we speak | `degraded` | `mcp_version` |
| `tools/list` fails (a `5xx` or another HTTP error, a JSON-RPC error, a wrong shape, no answer in time) | `degraded` | `mcp_tools_failed` |
| A required tool is not in the complete list | `degraded` | `mcp_tool_missing` |
| The good answer came slower than `expect.max_ms` | `degraded` | `latency` |

`latency_ms` is the time of the one request that gave the good answer (`server/discover` or `initialize`), not of
the whole probe, so a legacy server is not charged for the discover request it did not understand.

Evidence (every value capped): `check: "mcp"` (always), `era` (`modern` or `legacy`), `protocol_version`, `server`
(`name`, `version`, each ≤ 100 characters), `tool_count`, `tools` (the first 50 names, each ≤ 128 characters),
`missing` (required names not found), `tools_truncated` and `unchecked` (see step 3), `session` (`true` when the
server gave a session id; the id itself is never stored), `http_status`, `snippet` (≤ 200 characters) on failures,
`redirect` (the status) for a redirect that turns `POST` into `GET`, `hint` (see the table), `refused_url`
(≤ 500 characters) for `redirect_refused`, and `supported` (the versions the server named, up to 10) for
`mcp_version`. Your `target.headers` never appear in the evidence. The server's `instructions` are never
stored. For `auth_required`, when the `401` carries `WWW-Authenticate: Bearer resource_metadata="…"` on the server's
own origin, we fetch that document (no redirects, 1 MB) and, when it lists `authorization_servers` (RFC 9728), add
`auth: "oauth"` and the first 3 `authorization_servers` (each ≤ 200 characters). No other address is fetched. To
check such a server fully, put a key in `target.headers`.

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
`canary`, `mcp`), `external_id`, `public` (`true` or `false`), `source` (`catalog` or `manual`), `q` (text), `sort`
(`problems`, `changed` or `name`), `days` (integer 1 to 30), `limit` (integer ≥ 1, ≤ 500, default 100), `cursor`.
`q` matches the name or
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

`days` adds `daily` to every monitor: one entry per UTC day of the last `days` days (today counts as one) on which
it was checked, oldest first, each `{ "date": "2026-10-17", "worst": "down" }`. `worst` is the worst result of
that day, in the order `down`, `degraded`, `blocked`, `up`. A day with no check is not in the list. Results are
kept 30 days, so 30 is the most. A `days` value that is not an integer from 1 to 30 is `400 invalid_query`.

Returns `{ "ok": true, "monitors": [ … ], "next_cursor": "…" | null }`.

### `GET /v1/monitors/:id`

Returns the monitor plus `last_probes`: the last 30 results `{ at, verdict, http_status, latency_ms, reason }`,
and `daily`: the last 30 days in the same shape as `GET /v1/monitors?days=30`. An `mcp` monitor also has
`last_evidence`: the evidence of its newest check (server, protocol, tools; see [The `mcp` check](#the-mcp-check)),
or `null` before its first `mcp` check (also right after a change of type) and when it cannot be read at that
moment; the rest of the answer is not affected.

### `PATCH /v1/monitors/:id`

Body: any subset of the create fields, plus `paused` (boolean). Changing `interval_sec` or `target` reschedules
the next probe. Changing `type` away from `mcp` drops the stored `expect.mcp`; changing it to `mcp` needs
`target.method` `POST`, no `target.body` and no `expect.rules`, as on create (send them in the same patch when the
stored ones differ). `paused` pauses or resumes probing: a resumed monitor probes within about a minute, bulk upsert
never changes it, and it is allowed on a canary monitor even when the plan no longer includes canary.

### `DELETE /v1/monitors/:id`

Returns `204`. History rows are kept for 30 days and then pruned.

### `POST /v1/monitors/:id/probe`

Runs one probe now and returns `{ "ok": true, "result": { verdict, http_status, latency_ms, reason, evidence } }`.
Does not change the schedule. Counts toward the confirmation rule like a scheduled probe.

### `GET /v1/changes`

Query: `since` (ms, required), `limit` (integer ≥ 1, ≤ 1000, default 200), `cursor`, `monitor_id` (only the
changes of that monitor; a monitor of another team finds nothing), `order` (`asc`, the default, or `desc`). A
`limit` that is not a number, or is below 1, is `400 invalid_query`, same as a missing `since`; a `limit` above
1000 is clamped down to 1000 rather than rejected; an `order` other than `asc` or `desc` is `400 invalid_query`.
Returns state changes for your team, oldest first (newest first with `order=desc`, where `next_cursor` walks back
in time and `since` stays the lower bound):

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
Every field is optional; one you leave out stays as it is. `telegram_chat_id` is the chat to send to, up to 64
characters (spaces around it are removed): a number such as `123456789` (a group or channel has a minus sign first,
like `-1001234567890`) or the `@name` of a public channel. `telegram_digest` turns the daily Telegram summary on
or off (see [Telegram](#telegram)). `webhook_url` must
be `http` or `https` and a public host — the same rule a monitor's `target.url` follows (no `10/8`,
`172.16/12`, `192.168/16`, `127/8`, link-local, `localhost`); a disallowed URL is `400 invalid_target`. `https`
is recommended: the payload is signed, not encrypted, so a plain `http` endpoint exposes the change data (not
the secret) to anyone on the network path. Setting `webhook_url` then sends one test delivery and returns
`400 webhook_test_failed` if it does not answer 2xx within 5 s; a redirect is not followed and counts as a
failure like any other non-2xx.

### `POST /v1/team/webhook/test`

No body. Sends one test delivery to the webhook you saved, the same `{ "test": true, "team": "<slug>" }` body that
setting `webhook_url` sends, signed the same way. Returns `{ "ok": true }` when it answers 2xx within 5 s,
`400 webhook_test_failed` when it does not, and `400 invalid_query` when no webhook is set.

### `GET /v1/alerts`

Query: `limit` (integer ≥ 1, ≤ 100, default 20). Returns your team's newest alert deliveries, newest first, kept
90 days:

```json
{ "ok": true, "alerts": [ {
  "id": "…", "monitor_id": "…", "name": "Jellypod", "own": true, "change_id": "…", "from": "up", "to": "degraded",
  "channel": "webhook", "sent": false, "attempts": 1, "error": "webhook 500", "created_at": 1792238900000
} ] }
```

`channel` is `webhook` or `telegram`. `sent` says whether the last try arrived; `error` says why it did not;
`attempts` 0 means it was not tried yet. A failed delivery is tried again by the hourly job, 3 tries in all.
`own` is `false` for a monitor your team only has a claim on (and for a deleted monitor). `name` is `null` when
the monitor has been deleted; `from` and `to` stay, because changes are kept.

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

Query: `domain` (required), a domain name such as `geondex.com`. A pasted web address works too: the scheme, user
info, path, query, port and a leading `www.` are removed and the rest is lower-cased (`https://www.Geondex.com/pricing`
and `me@geondex.com` both mean `geondex.com`). The answer has `domain`, the registrable domain it looked up.
Returns the public monitors of other teams whose target is on that registrable domain, at most 20, sorted by name.
Your own monitors and private monitors are never in `monitors`. Use it to find the monitor to pass to
`POST /v1/claims`. `url` is the target's origin and path only: the query string, the
fragment and any user info are removed, because they can hold a token.

`own_monitors` lists your own monitors on that domain (public or not, at most 20, sorted by name, each `id`, `name`
and `url`). You cannot claim them, but it tells you the tool is already yours when `monitors` is empty.

```json
{
  "ok": true,
  "domain": "geondex.com",
  "monitors": [
    {
      "id": "0b2f1a2e-4c3d-4b8a-9e21-abcdef123456",
      "name": "Geondex",
      "team": "aigregator",
      "public_slug": "geondex",
      "url": "https://www.geondex.com/",
      "status_url": "https://agentviva.dev/s/aigregator/geondex"
    }
  ],
  "own_monitors": []
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
- A row without an id or without a URL is counted as invalid and skipped (`PUT /v1/catalog` names the first five).
  When an id appears twice, the first row is kept and the others are counted as invalid. Spaces around values are
  trimmed. A CSV with a quote that is never closed is refused, and so is a list with no valid tool at all (an empty
  array, a header-only CSV, or only invalid rows).

| Endpoint | What it does |
| --- | --- |
| `GET /v1/catalog` | `{ "ok": true, "catalog": { "url", "public", "last_sync_at", "next_sync_at", "last_result", "monitors_by_state" } }`, or `"catalog": null` when none is connected |
| `PUT /v1/catalog` | Body `{ "url": "https://…", "public": true }` (`public` defaults to `true`). Fetches and reads the list once, saves the link, schedules the first sync for now. Returns `catalog` and `"preview": { "found", "invalid", "sample", "skipped" }` where `sample` is the first 3 tools and `skipped` lists up to 5 of the invalid rows, each `{ "row", "name", "reason" }` (`row` counts the tools from 1, a CSV header is not counted; `name` is the name, id or address to recognise it by, up to 80 characters; `reason` is `no id`, `no address`, `id used twice` or `not an object`). `400` `invalid_target` for a disallowed link, `400` `catalog_unreadable` when it cannot be fetched or read, or holds no valid tool (`detail` says why); nothing is saved then |
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
| `GET /badge/:team/:public_slug.svg` | live SVG badge in one of seven forms (`?style=`, `?theme=`; see below), `max-age=300` |
| `GET /s/:team/:public_slug` | HTML status page, 90-day history, no JavaScript |
| `GET /v1/public/search?q=` | the public tools AgentViva already watches, by name or address; see below |
| `GET /bot` | what AgentVivaBot is and how to opt out |
| `GET /healthz` | `ok` |

The JSON, the badge and the status page answer fast even when our database is slow or down. They keep a copy in
Cloudflare's cache for 5 minutes, and when the database does not answer within 3 s they serve the last copy (up to 6
hours old, with the header `X-AV-Stale: 1`). With no copy at hand, the JSON answers `503 unavailable`, the badge
answers `200` with an "unknown" badge (`max-age=60`), and the status page answers `503` with a short "try again in a
minute" page. A monitor made private or deleted stops showing within 5 minutes, as before.

Team slug for aigregator is `aigregator`.

### `GET /badge/:team/:public_slug.svg`

No key. An SVG image for an `<img>` tag: no script, no cookie. It never shows the monitor's name. A private or
unknown monitor gets the same `404` image with the state `unknown`. Two optional query parameters:

| Parameter | Values | Default |
| --- | --- | --- |
| `style` | `dot`, `word`, `pill`, `card`, `seal`, `strip`, `panel` | `card` |
| `theme` | `dark`, `light`, `auto` | `dark` |

An unknown value is the default, so a link without parameters is the same 180×48 dark card as before. `auto`
follows the reader's light or dark system setting (`prefers-color-scheme` inside the SVG). GitHub uses its own
theme setting instead, so for a README use a `<picture>` with a `theme=dark` and a `theme=light` image.

| Form | Size | Shows |
| --- | --- | --- |
| `dot` | 14×14 | the state colour only; the words are in the image's accessible name |
| `word` | 116 to 172 wide, 22 high | the state word and "AgentViva", no background |
| `pill` | 147 to 178 wide, 20 high | README shield: "agentviva" and the state with how long (`alive · 26d`) |
| `card` | 180×48 | the state, how long, and when we last checked |
| `seal` | 96×96 | a round seal: the state and how long |
| `strip` | 320×68 | "alive for 26 days", the last check, and the last 30 days as bars |
| `panel` | 360×184 | the state, since when, 30 days, good days, the last response time, the last check, the check type |

"How long" shows only for `up` (alive), `degraded` and `down`; `blocked` shows "not checkable", and `unknown`
shows "unknown". The 30 days are UTC days, each the worst state of that day, as on the status page. The response
time shows only when the last check answered.

```text
https://agentviva.dev/badge/aigregator/notion.svg?style=pill&theme=auto
```

### `GET /v1/public/search`

No key. Query: `q`, a tool name (`Notion`) or an address (`notion.so`, `https://www.notion.so/x`), 2 to 100
characters. Returns up to 5 public monitors that are not paused: by registrable domain when `q` looks like an
address, else by name (an exact name first, then the shortest). It reads only what AgentViva already watches; it
sends no request to the address. `url` is the site's origin for a website check, `null` for an API check.

```json
{ "ok": true, "query": "notion", "tools": [ {
  "name": "Notion", "team": "aigregator", "state": "up", "since": 1789200000000, "last_probe_at": 1789286400000,
  "reason": null, "url": "https://www.notion.so",
  "status_url": "https://agentviva.dev/s/aigregator/notion",
  "badge_url": "https://agentviva.dev/badge/aigregator/notion.svg",
  "json_url": "https://agentviva.dev/v1/public/aigregator/notion.json"
} ] }
```

`Access-Control-Allow-Origin: *`, `Cache-Control: public, max-age=300`; the same query is answered from a cache for
5 minutes. A `q` that is too short, too long or holds a control character is `400 invalid_query`. Callers without a
key are limited to 30 requests a minute per IP address (shared with the keyless MCP mode); over it is
`429 rate_limited`. The address is used for the count only and is not stored.

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

`monitor_limit` is set per team, not by plan; `GET /v1/team` returns the team's own value. Page, JSON and MCP
checks are on both plans.

Rate limit: 600 requests per minute per key. Bulk calls count as one request.

## MCP server

An MCP (Model Context Protocol) server at `https://agentviva.dev/mcp`, so an AI agent can read monitor states and
manage monitors without writing REST calls. It calls the same handlers as the REST API, so every rule in this
document (plan gates, limits, address checks) applies unchanged. Delete is not offered; it stays REST-only.

### Connect

Three ways in:

- **Sign in in the browser (OAuth 2.1).** Add `https://agentviva.dev/mcp` with no header. The client gets `401`,
  finds our sign-in on its own, opens the browser; the user signs in with Google and presses **Allow**. All tools,
  for the user's own team. Works with Claude Code (the plugin, or `claude mcp add --transport http agentviva
  https://agentviva.dev/mcp`, then `/mcp` → agentviva → Authenticate), claude.ai custom connectors and other clients
  that support MCP sign-in. Details in [Sign-in for MCP clients](#sign-in-for-mcp-clients-oauth-21).
- **The API key in a header.** For clients that cannot open a browser, and for scripts:

  ```bash
  claude mcp add --transport http agentviva https://agentviva.dev/mcp --header "Authorization: Bearer av_live_..."
  ```

- **No account:** `https://agentviva.dev/mcp/public` answers without a key, with only `lookup_tool` (see below).

Browser-based inspectors are refused: they send a web `Origin` header (`403`, see Transport).

### Sign-in for MCP clients (OAuth 2.1)

Standard MCP authorization: OAuth 2.1 with PKCE, public clients only, discovered from the `401`.

| Endpoint | What it does |
| --- | --- |
| `GET /.well-known/oauth-protected-resource` (also `…/oauth-protected-resource/mcp`) | Resource metadata (RFC 9728): `resource` is `https://agentviva.dev/mcp`, `authorization_servers` is `["https://agentviva.dev"]`, scope `mcp` |
| `GET /.well-known/oauth-authorization-server` | Server metadata (RFC 8414): the endpoints below, `response_types_supported: ["code"]`, `grant_types_supported: ["authorization_code", "refresh_token"]`, `code_challenge_methods_supported: ["S256"]`, `token_endpoint_auth_methods_supported: ["none"]`, `authorization_response_iss_parameter_supported: true` |
| `POST /oauth/register` | Dynamic client registration (RFC 7591), JSON: `client_name` (optional, up to 100 characters; bidi and zero-width characters are removed) and `redirect_uris` (1 to 5, each up to 500 characters and at most 1,000 together, printable ASCII only). Allowed: `https:` addresses, `http:` only on `localhost`, `127.0.0.1` or `[::1]`, and app schemes that hold a dot (reverse-domain, RFC 8252 section 7.1, such as `com.example.app:`) or are one of `cursor`, `vscode`, `vscode-insiders`, `windsurf`, `claude`. Every other scheme is refused. Clients are public: a client that asks for a secret (`client_secret_post`, …) is registered as public anyway and the answer says `token_endpoint_auth_method: "none"`. Answers `201` with `client_id`; errors `400` `invalid_client_metadata` or `invalid_redirect_uri` |
| `GET /oauth/authorize` | `response_type=code`, `client_id`, an exact registered `redirect_uri`, `code_challenge` with `code_challenge_method=S256`, `state`; `resource` if sent must be `https://agentviva.dev/mcp`. A request that fails any check gets an error page and never goes back to the app. Otherwise it signs the user in if needed, then shows a consent page (the app's name, where it returns, the team, what it can do). **Allow** returns `code`, `state` and `iss`; **Cancel** returns `error=access_denied`, `state` and `iss`. `iss` is `https://agentviva.dev` (RFC 9207) |
| `POST /oauth/token` | Form or JSON. `grant_type=authorization_code` with `code`, `redirect_uri`, `client_id`, `code_verifier`; or `grant_type=refresh_token` with `refresh_token`, `client_id`. Answers `{ access_token, token_type: "Bearer", expires_in: 3600, refresh_token, scope: "mcp" }`. Errors (`400`): `invalid_request`, `invalid_grant`, `unsupported_grant_type`; an unknown `client_id` is `401` `invalid_client` |

- A code lives 10 minutes and works once. An access token (`av_oat_…`) lives 1 hour; a refresh token (`av_ort_…`)
  lives 30 days from its last use, and each refresh replaces both (the old refresh token stops working).
- A refresh token that was already replaced and is used again ends the whole grant: both tokens stop, the answer is
  `invalid_grant`, and the user signs in again. So does a refresh more than 90 days after the sign-in.
- The grant belongs to the signed-in user's own team. Each team sees its connected apps on the API page
  (`/app/api`), with **Disconnect**, which stops that app's tokens at once.
- `/oauth/register` is limited to 10 a minute per address (an IPv6 address counts as its /64) and 10 a minute for
  all callers together; `/oauth/token` to 60 a minute per `client_id`. Over it is `429` `rate_limited`. Both send
  CORS `*`. A client with no sign-in an hour after it registered is deleted; the app then registers again.

### Transport

Stateless Streamable HTTP with JSON responses (no SSE, no session id). Every message is one `POST /mcp` with one
JSON-RPC 2.0 request or notification in the body (a JSON array is `-32600`); the body is capped at 64 KB and the
rate limit is the REST one: 600 requests a minute per key, and with browser sign-in tokens 600 a minute for all of a
team's connected apps together.

- A request gets one JSON-RPC response as `application/json`. A notification gets `202 Accepted` with no body.
- `GET /mcp` and `DELETE /mcp` (and any method but `POST`) get `405` with `Allow: POST`.
- A request whose `Origin` header is an `http://` or `https://` web origin (a browser page) gets `403`, and no CORS
  headers are ever sent. No `Origin`, `Origin: null` or another scheme (a desktop app) is accepted.
- No `Authorization` header gets `401` with `WWW-Authenticate: Bearer
  resource_metadata="https://agentviva.dev/.well-known/oauth-protected-resource"`, which starts the sign-in. A key
  or token that is there but unknown or expired gets `401` with `error="invalid_token"` in the same header (the body
  is the REST error body); a client then refreshes or signs in again.
- `/mcp/public`, and `Bearer` with nothing after it (what plugin 1.3 sends without a key), are the keyless mode:
  only `lookup_tool`, limited to 30 requests a minute per IP address (`429` over it).
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
| `create_monitor` | `type`, `name`, `url`; optional `public`, `public_slug`, `interval_sec`, `external_id`, `method`, `body`, `headers` (up to 10, for an endpoint that needs a key), `rules` (json), `canary` (canary), `tools` (mcp: the names that must be listed, becomes `expect.mcp.tools`) | `POST /v1/monitors` |
| `probe_now` | `id` | `POST /v1/monitors/{id}/probe` |
| `get_changes` | `since` (ms, default 24 hours ago), `limit` (1 to 200, default 50), `cursor` | `GET /v1/changes` |
| `connect_catalog` | `url`; optional `public` (default `true`) | `PUT /v1/catalog` |
| `catalog_status` | none | `GET /v1/catalog` |
| `sync_catalog` | optional `force` | `POST /v1/catalog/sync` |
| `lookup_tool` | `query` (a tool name or an address, 2 to 100 characters); no key needed | `GET /v1/public/search` |

In the keyless mode, `tools/list` shows only `lookup_tool`, and calling another tool is a result with
`"isError": true` and a text that says how to sign in.

`create_monitor` takes the address as a flat `url`; `method`, `body` and `headers` are the `target` fields of the same
names, `rules` and `canary` are the `expect.rules` and `expect.canary` of the [monitor object](#monitor-object), and
`tools` is `expect.mcp.tools`.
Arguments a tool does not know are `-32602`, not ignored. `list_monitors` and `get_changes` return `next_cursor` in
`structuredContent` when more are available (the text also names it, and each monitor's `id` and `external_id`);
pass it back as `cursor`. Their text also shows each monitor's address.
