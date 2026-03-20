# Changelog — OpenClaw Digital Office Setup

This changelog documents real-world discoveries made while running the skills
live in production on **mediadeboer.nl** (OpenClaw local install on Ubuntu desktop).
Each entry reflects something found in the field that improved the skill.

---

## [v0.1.3] — 2026-03-20

### Live discovery: WhatsApp tunnel & webhook stability

Running the channels skill in production revealed gaps in the tunnel documentation.

**Found in production:**
- Named cloudflared tunnel requires a domain managed in Cloudflare DNS — this was
  not documented. Users without a domain cannot use option B.
- Existing domain owners can move DNS to Cloudflare for free (~10 min) without
  affecting their host — no new purchase needed.
- ChakraHQ has a REST API (Settings → API Keys). Webhook URL can be updated
  automatically via script after tunnel restarts — no new account or domain required.
- "Could not auto-approve pending device" warning in logs is non-fatal when using
  ChakraHQ Cloud API in read-only mode.

**Skills updated:**
- `channels/gotchas.md` — Gotcha #6 expanded with 3 permanent fix options
  (A: ChakraHQ API auto-update, B: named tunnel + domain clarification, C: Tailscale Funnel)

---

## [v0.1.2] — 2026-03-20

### Live discovery: cron jobs and gateway mode

Running the workflows skill revealed that the documented cron setup method was wrong.

**Found in production:**
- `openclaw cron add` CLI fails with WebSocket error when `gateway.mode: local` is set.
  This is the required mode for local installs without a reverse proxy.
- Workaround: write cron jobs directly to `/data/.openclaw/cron/jobs.json` while
  container is stopped. Jobs load correctly on next start.
- `gateway.mode: remote` crashes the gateway entirely ("Gateway start blocked").
  Do not use.

**Skills updated:**
- `workflows/SKILL.md` — cron setup corrected from JSON config to CLI commands
- `workflows/gotchas.md` — Gotcha added: gateway.mode=local blocks CLI cron commands
- `workflows/tests/` — test scenario updated to expect CLI command, not JSON config

---

## [v0.1.1] — 2026-03-20

### Live discovery: production gotchas from first real deployment

First live run of the full 9-step skill on a real OpenClaw instance exposed
multiple undocumented behaviors.

**Found in production:**
- Docker image pinning: `:latest` tag causes uncontrolled upgrades — always pin to digest
- Discord dmPolicy must be set to `allowlist` — default `open` accepts messages from anyone
- WhatsApp dmPolicy same issue — `open` by default
- Telegram groupPolicy default is `open` — changed to `allowlist` for all bots
- `gateway.mode: local` is required for local installs (no reverse proxy) but blocks CLI
- Backup script needs WAL checkpoint before SQLite copy — otherwise data loss risk
- cloudflared quick tunnel URL changes on every restart — manual update required

**Skills updated:**
- `foundation/gotchas.md` — 5 new gotchas added (#5–#9)
- `channels/gotchas.md` — WhatsApp tunnel gotcha added (#6)
- `security/` — dmPolicy and groupPolicy defaults documented

---

## [v0.1.0] — 2026-03-20

### Initial release

First published version of the OpenClaw Digital Office Setup skill-set.

**Included:**
- 9 sub-skills: foundation, security, office-setup, channels, models, workflows,
  monitoring, scaling, handover
- 2 standalone skills: digital-office-setup (orchestrator), health-check
- TDD test scenarios for each skill (9/9 passing)
- Grok audit completed
- GitHub repository live: github.com/MarketingBoer/openclaw-digital-office
