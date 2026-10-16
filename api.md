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
| 400 | `invalid_json`, `invalid_monitor`, `invalid_target` (a disallowed scheme or private host in either a monitor's `target.url` or `PATCH /v1/team`'s `webhook_url` — the same rule applies to both), `invalid_query` (a malformed query string, or a request body that fails validation when no more specific code applies, e.g. `PATCH /v1/team`), `domain_not_claimable` (`POST /v1/claims` on a monitor, or `GET /v1/claimable` for a domain, that is a shared or public suffix nobody can prove they own), `webhook_test_failed` (`PATCH /v1/team` set `webhook_url` and the one test delivery did not answer 2xx within 5 s) |
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

Query: `state` (one of the states above), `type` (`page`, `json`, `canary`), `external_id`, `public` (`true`
or `false`), `q` (text), `limit` (integer ≥ 1, ≤ 500, default 100), `cursor`. `q` matches the name or the target,
ignoring case, as a plain substring: `%` and `_` mean themselves. It must be 1 to 100 characters after trimming,
otherwise `400 invalid_query`. It combines with the other filters. A `limit` that is not a number, or is below
1, is `400 invalid_query`; a `limit` above 500 is clamped down to 500 rather than rejected. A `state`/`type`/
`public` value outside the ones listed here is `400 invalid_query`, not a silently empty or default page.
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
  "status_url": "https://agentviva.dev/s/aigregator/notion"
} ], "next_cursor": null }
```

Store the `at` of the last change you processed and pass it as `since` next time. Changes are never edited or
deleted inside 90 days, so this feed is safe to replay.

### `GET /v1/team`

Returns `{ "ok": true, "team": { "name", "slug", "plan", "monitor_limit", "monitors_by_state": { … },
"telegram_linked": bool, "webhook_configured": bool } }`.

### `PATCH /v1/team`

Body: `{ "webhook_url": "http://…" | "https://…" | null, "telegram_chat_id": "…" | null }`. `webhook_url` must
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

## Plans and limits

| Plan | Canary checks | Public monitors | History | Claims | Channels |
| --- | --- | --- | --- | --- | --- |
| `free` | no | yes | 30 days | yes | webhook, Telegram |
| `pro` | yes | yes | 30 days (365 planned) | yes | webhook, Telegram (Slack, email **v1**) |

`monitor_limit` is set per team, not by plan; `GET /v1/team` returns the team's own value.

Rate limit: 600 requests per minute per key. Bulk calls count as one request.

## MCP (v1)

A Streamable HTTP endpoint at `/mcp` with the same key in the `Authorization` header and five tools:
`create_monitor`, `list_monitors`, `get_status`, `get_changes`, `probe_now`. Delete is API-only.
