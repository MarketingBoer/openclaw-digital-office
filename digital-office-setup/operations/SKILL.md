---
name: digital-office-operations
description: |
  Use when doing backup, restore, upgrades, daily management, or skills management.
  Triggers: "backup", "restore", "upgrade", "update", "maintenance",
  "doctor", "install skills", "daily ops".
---

# Digital Office Operations

Tested against: OpenClaw v2026.3.x

This skill owns backup/restore, upgrades, daily management, and skills management.
For Docker compose setup, see **foundation/**. For security hardening, see **security/**.

---

## Backup

### Official CLI (v2026.3.8+)

```
openclaw backup create                    # full backup
openclaw backup create --only-config      # config only
openclaw backup create --no-include-workspace
openclaw backup create --output /path/to/backup.tar.gz
openclaw backup create --verify           # validate after creation
openclaw backup create --dry-run --json   # preview what would be backed up
openclaw backup verify {archive}          # validate an existing archive
```

### What to Back Up (everything under ~/.openclaw)

- `openclaw.json` — main configuration
- `gateway.db` — SQLite database (memory, conversations, state)
- `credentials/` — API keys, channel tokens, WhatsApp sessions
- `agents/{agentId}/sessions/` — conversation history
- `workspace/` — SOUL.md, AGENTS.md, memory, skills
- `extensions/` — custom plugins (e.g., whatsapp-cloud-api)
- `exec-approvals.json` — approved exec command patterns

**Only backing up workspace/ = losing credentials + sessions = re-pairing all channels.**

### SQLite WAL Corruption Risk

`openclaw backup create` does a filesystem-level tar, NOT a SQLite `.backup` API call.
On a live gateway, WAL (Write-Ahead Log) corruption can occur.

**Safe options:**
1. Stop the gateway before backup
2. Or separately: `sqlite3 ~/.openclaw/gateway.db ".backup '/tmp/gateway-backup.db'"`

### Scheduled Backup

Use a Cron Job (see workflows/): `openclaw cron create "backup:daily" --schedule "0 2 * * *"`

---

## Restore — Manual Only

**`openclaw backup restore` DOES NOT EXIST YET** (Issue #13616, still open).

Manual restore procedure:
1. Stop the gateway
2. Unpack backup archive to `~/.openclaw/`
3. Fix ownership: `sudo chown -R 1000:1000 ~/.openclaw/`
4. Run `openclaw doctor --fix` to validate config and fix file modes
5. Start the gateway
6. Verify: test each channel (Telegram, Discord, WhatsApp)

**WhatsApp sessions are fragile on migration** — expect to re-scan QR after restore.

---

## Upgrade Procedure

### When to Upgrade

| Type | Action |
|------|--------|
| Security patches | Immediately — tool execution, OAuth, webhook auth vulnerabilities |
| Bugfixes that affect you | Carefully — backup first, test on staging |
| New features | Wait a few days — let others find breaking changes |

### Procedure

1. Read CHANGELOG.md for breaking changes
2. `openclaw backup create --verify`
3. Stop the gateway
4. Upgrade: update pinned image tag in `.env`, then `docker compose pull && docker compose up -d --force-recreate`
5. Start gateway, verify: channels connected, logs clean, send test message

### Rollback (Docker)

Change `OPENCLAW_IMAGE` in `.env` back to previous tag, then `docker compose up -d`.
Named volumes persist across recreates — no data loss.

### `openclaw doctor --fix`

Automatically creates backup, validates config, detects and fixes:
- Overly permissive file modes
- Missing or invalid tokens
- Broken config entries
- OAuth token expiry

---

## Daily Management

### Gateway Restart Procedure

1. `rm -f ~/.openclaw/cron/jobs.json` — prevents cron scheduling conflict
2. Stop gateway (`docker compose down` or `systemctl stop openclaw-gateway`)
3. Start gateway
4. Verify per channel:
   - Telegram: bot startup message in logs
   - Discord: socket connected
   - WhatsApp: auth state valid (may need re-pair)
   - Slack: socket mode connected
5. Send test message on each active channel

### Log Monitoring

- Docker: `docker compose logs -f openclaw-gateway`
- Systemd: `journalctl -u openclaw-gateway -f`

### Session Pruning

`cron.sessionRetention` (default 24h) automatically cleans completed sessions.
Monitor: longer sessions = more tokens per request.

### Memory Maintenance

Review MEMORY.md periodically. Trim stale entries. Ensure no sensitive data persists.

---

## Skills Management

### Precedence (3 levels)

1. `{workspace}/skills/` — highest, per-agent
2. `~/.openclaw/skills/` — global/managed
3. Bundled — shipped with OpenClaw (lowest)

### Install from ClawHub

**Security first** — before ANY install:
1. Check VirusTotal report on ClawHub skill page
2. Review raw source code on GitHub
3. Pin to specific version/commit
4. Treat as untrusted code (see security/ for 1,184+ malicious skills stats)

```
clawhub install author/skill-name
clawhub update skill-name
```

### Config Validation Cascade

One broken skill or plugin blocks ALL `openclaw` CLI commands.
Fix: remove problematic entry from openclaw.json, restart gateway.

---

## Cross-References

| Topic | Skill |
|---|---|
| Docker compose, env vars | foundation/ |
| Security hardening, incidents | security/ |
| Cron job scheduling | workflows/ |
| Cost monitoring, dashboards | monitoring/ |
| Agent workspace files | office-structure/ |

---

## Sources

- docs.openclaw.ai/cli/backup
- docs.openclaw.ai/tools/skills
- docs.openclaw.ai/help/testing (doctor)
