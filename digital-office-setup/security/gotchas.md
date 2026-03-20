# Security — Gotchas

Real security pitfalls for OpenClaw hardening.
Each entry: what happens, why, and the fix.

---

## 1. dmPolicy "open" in Production = Instant Exposure

**What happens:** Any user on the messaging platform can send messages to
your agent. Within hours, bots and scanners discover the endpoint.

**Why:** "open" means no authentication check on inbound messages.
This was the primary vector in the 220,000+ exposed instances crisis.

**Fix:** Set dmPolicy to "allowlist" with explicit numeric user IDs.
Add requireMention: true in all group contexts.
Never use "open" outside isolated local testing.

---

## 2. :latest Image Breaks Config Schema on Update

**What happens:** After `docker compose pull`, the gateway crashes because
the new image changed the config schema. `agents.entries` or other
config keys are no longer recognized.

**Why:** Using `:latest` means any upstream release can silently change
the config format. No migration warning is given.

**Fix:** Pin a specific image tag in docker-compose.yml. Read release
notes before upgrading. Test on staging first. See foundation/ gotchas
for the full incident description.

---

## 3. Config Validation Cascade — One Plugin Blocks Everything

**What happens:** After adding a plugin or extension, ALL `openclaw`
CLI commands fail — not just the plugin. Unrelated channels break.
Error message: config validation failure.

**Why:** OpenClaw validates the entire openclaw.json on every CLI
invocation. One invalid entry (bad plugin, broken permission, uid
mismatch) blocks the entire config parse.

**Fix:** Remove the problematic entry from openclaw.json manually.
Or fix ownership: `docker exec -u root <container> chown -R 1000:1000
/home/node/.openclaw/extensions/`. Then restart the gateway.

---

## 4. Plugin uid Ownership Cascade Failure

**What happens:** Extensions installed as root (uid 0) inside a
container running as uid 1000 fail config validation. This cascades
to block all agents and channels.

**Why:** The gateway process (uid 1000) cannot read files owned by
root. Config validation fails before any agent code runs.

**Fix:** See foundation/ gotchas #4 for detailed fix options.
Prevent by always installing plugins through the gateway process,
never via a root shell.
