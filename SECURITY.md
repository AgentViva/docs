# Security policy

## Reporting a vulnerability

Email **[support@agentviva.dev](mailto:support@agentviva.dev)** with `SECURITY` in the subject, or open a
private report through GitHub's **Report a vulnerability** button on this repository's Security tab.

Please do not open a public issue for a security problem.

Tell us what you found, how to reproduce it, and what an attacker could do with it. A proof of concept helps.
If you need to send something sensitive, say so and we will arrange an encrypted channel.

We aim to acknowledge a report within 3 working days and to tell you our assessment within 10. AgentViva is a
small service run by one person, so we will be honest about timelines rather than optimistic.

There is no bug bounty. We will credit you in [CHANGELOG.md](CHANGELOG.md) when a fix ships, unless you ask us
not to.

## Scope

In scope:

- `agentviva.dev` and everything under it, including the public status pages, badges and JSON endpoints
- The API described in [api.md](api.md) — authentication, key handling, tenant isolation, the claim
  verification flow, webhook signing
- This repository's contents

Out of scope:

- The sites and APIs that AgentViva monitors. They belong to other people. Report problems to them
- Reports produced only by an automated scanner, with no demonstrated impact
- Missing security headers, missing SPF/DMARC, or TLS configuration with no working attack behind it
- Denial of service, volumetric testing, and anything that degrades the service for other users
- Social engineering of anyone involved with AgentViva

## Testing rules

Test only against monitors and teams you own. Do not probe, claim or enumerate another team's data. If you
gain access to data that is not yours, stop, do not save it, and tell us what you saw.

Research that follows this policy is welcome and we will not pursue you for it.

## What we already do

- API keys are stored as hashes, never in plaintext, and are shown to you exactly once
- Webhook payloads are signed with HMAC-SHA256 over `<timestamp>.<raw body>`; verify before you parse, and
  reject timestamps older than five minutes
- Monitor targets and webhook URLs must be `http` or `https` on a public host — private address ranges are
  refused
- Every user-supplied value is escaped in Telegram messages and in SVG badges
