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

**The reference is [api.md](../../api.md) in this repository, and the machine-readable spec is
[openapi.yaml](../../openapi.yaml). Read api.md before writing a request** — it has the exact field rules,
error codes and limits, and it is the only document an integration should rely on.

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
- Adding many monitors is `POST /v1/monitors/bulk`, up to 1,000 at a time, upserted by `external_id`.
- Public monitors need no key at all: `GET /v1/public/{team}/{external_id}.json`, a badge at
  `/badge/{team}/{slug}.svg`, and a status page at `/s/{team}/{slug}`.

## Times and limits

All times are Unix milliseconds. 600 requests per minute per key; a bulk call counts as one. Bodies over
64 KB are rejected.
