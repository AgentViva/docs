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

Nothing yet.

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
