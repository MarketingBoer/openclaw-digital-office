# Workflows — Gotchas

Real pitfalls with webhook, cron, and n8n integrations in OpenClaw.

---

## 1. Hooks Token is a Password — Don't Reuse

**What happens:** Attacker gains webhook access and can trigger agent runs,
inject prompts, or exfiltrate workspace data via deliver+channel.

**Why:** The hooks token grants full access to trigger any allowed agent.
Reusing it across environments or sharing it in config files exposes all.

**Fix:** Generate unique token per environment: `openssl rand -hex 32`.
Store in `OPENCLAW_HOOKS_TOKEN` env var only. Rotate after any leak.

---

## 2. Query-String Auth is Rejected

**What happens:** Webhook calls with `?token=...` in the URL silently fail.
No error message — the gateway just returns 401.

**Why:** OpenClaw only accepts Bearer token in the Authorization header.
Query-string auth is explicitly rejected for security (tokens in URLs
get logged in server access logs, proxy logs, and browser history).

**Fix:** Use `Authorization: Bearer {token}` header. Update n8n/external
systems that default to query-string auth.

---

## 3. allowRequestSessionKey=true Enables Session Hijack

**What happens:** External callers can specify any sessionKey in the webhook
payload, targeting and reading existing agent sessions.

**Why:** With `allowRequestSessionKey: true`, the gateway trusts the caller's
sessionKey without validation. A malicious caller can impersonate sessions.

**Fix:** Keep `allowRequestSessionKey: false` (the default). If you need
custom session keys, restrict via `allowedSessionKeyPrefixes` to prevent
targeting of non-hook sessions.

---

## 4. Cron Jobs Are NOT Configured in openclaw.json

**What happens:** Adding `"crons"` or `"cron": { "jobs": {...} }` to
`openclaw.json` causes a config validation error on startup:
`Unrecognized key: "crons"` or `Unrecognized key: "jobs"`. OpenClaw
refuses to load — all plugins fail, Telegram/Discord stop responding.

**Why:** Cron jobs in OpenClaw are NOT schema-driven config — they are
managed as runtime state via the `openclaw cron` CLI. The config schema
rejects any unknown top-level keys.

**Fix:** Never add cron config to `openclaw.json`. Use the CLI:
```bash
openclaw cron add \
  --name "daily-status" \
  --agent hq-mediadeboer \
  --cron "0 8 * * 1-5" \
  --message "Give a short daily status overview." \
  --announce \
  --channel telegram
```
Run `openclaw cron --help` for all options.

---

## 5. Cron --deliver Flag is Deprecated

**What happens:** `--deliver` flag is accepted but marked deprecated. The
job may not deliver results as expected in newer OpenClaw versions.

**Fix:** Use `--announce` instead of `--deliver`. Same behavior, current API.

---

## 6. n8n Credentials Must Stay in n8n — Never in OpenClaw

## 6. n8n Credentials Must Stay in n8n — Never in OpenClaw

**What happens:** API keys stored in OpenClaw env vars or workspace files
are accessible to agents, logged in sessions, and potentially exfiltrable
via prompt injection or context theft.

**Why:** n8n has an encrypted credential store designed for API key security.
OpenClaw's workspace files are loaded into the context window.

**Fix:** SOUL.md rule: "NEVER store API keys in my environment or skill files."
Agent only knows the n8n webhook URL. All credentials in n8n encrypted store.
