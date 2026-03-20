# Operations — Gotchas

Real pitfalls with backup, restore, upgrades, and daily management.

---

## 1. openclaw backup restore DOES NOT EXIST

**What happens:** You run `openclaw backup restore` and get a "command not found" error.

**Why:** The restore CLI is deferred (Issue #13616, still open as of v2026.3.x).
Only `backup create` and `backup verify` are implemented.

**Fix:** Restore is manual: unpack tar.gz → chown 1000:1000 → `openclaw doctor --fix`.

---

## 2. SQLite WAL Corruption on Live Backup

**What happens:** After restoring from a backup taken while the gateway was running,
the database is corrupt. Sessions, memory, or conversation state is lost.

**Why:** `openclaw backup create` does filesystem-level tar, not SQLite's `.backup` API.
Concurrent writes during tar can leave the WAL in an inconsistent state.

**Fix:** Either stop the gateway before backup, or separately:
`sqlite3 ~/.openclaw/gateway.db ".backup '/tmp/gateway-backup.db'"`

---

## 3. :latest Tag Crash on Upgrade

**What happens:** After `docker compose pull && docker compose up`, the gateway crashes.
Config validation errors in logs. Agents not recognized.

**Why:** Upstream image changed config schema (e.g., `agents.entries` restructured).
Using `:latest` means any pull can silently break the config format.

**Fix:** Rollback: change OPENCLAW_IMAGE to previous pinned tag, `docker compose up -d`.
Prevention: always pin specific version tags. Read release notes before upgrading.

---

## 4. Only Backing Up Workspace = Losing Everything Else

**What happens:** After restore, all channels are disconnected. No conversations,
no credentials, no WhatsApp session. Only workspace files survived.

**Why:** A common mistake is backing up only `~/.openclaw/workspace/`. But credentials/,
gateway.db, sessions/, and extensions/ are equally critical.

**Fix:** Always back up the entire `~/.openclaw/` directory. Use `openclaw backup create`
which includes everything by default.

---

## 5. WhatsApp Re-Pair After Migration

**What happens:** After restoring on a new machine, WhatsApp channel doesn't connect.
Other channels (Telegram, Discord, Slack) work fine.

**Why:** WhatsApp Baileys sessions are tied to the device cryptographic state.
Migration often invalidates the session even if credentials/ is backed up.

**Fix:** After migration: `openclaw channels login whatsapp` → re-scan QR.
This is expected behavior, not a bug. Include credentials/ in backup to minimize
re-pairing — it sometimes works, but not guaranteed.

---

## 6. Autoheal Container for Gateway Resilience

**What happens:** Gateway crashes (OOM, upstream schema change, lock file) and
stays down until someone manually restarts it. On a local machine this is
noticed quickly, but overnight or on a VPS it can mean hours of downtime.

**Why:** Docker's `restart: unless-stopped` restarts the container, but the
gateway process inside may crash-loop without the container exiting (e.g., stale
lock file, config validation failure). Docker sees the container as "running"
and does nothing.

**Fix — autoheal sidecar:**
```yaml
services:
  autoheal:
    image: willfarrell/autoheal
    restart: unless-stopped
    environment:
      AUTOHEAL_CONTAINER_LABEL: all
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Pair with a proper healthcheck on the gateway service:
```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -sf http://127.0.0.1:18789/ --max-time 5 -o /dev/null || exit 1"]
  interval: 30s
  timeout: 10s
  retries: 5
  start_period: 120s
```

When the healthcheck fails 5 consecutive times, autoheal restarts the container
automatically. Recovery time: ~3-5 minutes.

**Note:** The autoheal container requires the Docker socket mount. This is safe
for the autoheal sidecar (it only calls `docker restart`) but NEVER mount the
Docker socket inside the OpenClaw container itself — see security/ for why.
