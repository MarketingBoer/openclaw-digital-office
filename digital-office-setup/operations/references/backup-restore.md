# Backup & Restore Reference

<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->

## CLI Commands (v2026.3.8+, PR #40163)

```bash
# Full backup
openclaw backup create

# Config only (no workspace data)
openclaw backup create --only-config

# Exclude workspace from backup
openclaw backup create --no-include-workspace

# Custom output path
openclaw backup create --output /backups/openclaw-$(date +%Y%m%d).tar.gz

# Verify archive integrity after creation
openclaw backup create --verify

# Preview (no actual backup)
openclaw backup create --dry-run --json

# Verify an existing archive
openclaw backup verify /backups/openclaw-20260320.tar.gz
```

## What's In the Backup

| Path | Contents | Critical? |
|------|----------|-----------|
| `openclaw.json` | Main config (agents, channels, hooks) | YES |
| `gateway.db` | SQLite: memory, conversations, state | YES |
| `credentials/` | API keys, channel tokens, WhatsApp creds | YES |
| `agents/*/sessions/` | Conversation history per agent | Medium |
| `workspace/` | SOUL.md, AGENTS.md, memory/, skills/ | YES |
| `extensions/` | Custom plugins (e.g., whatsapp-cloud-api) | If used |
| `exec-approvals.json` | Approved exec patterns | Low |

## SQLite WAL Safety

The `openclaw backup create` command does filesystem-level `tar`, not SQLite's
`.backup` API. On a live gateway, concurrent writes can corrupt the WAL.

**Safe backup approaches:**

```bash
# Option 1: Stop gateway first (cleanest)
docker compose down
openclaw backup create --verify
docker compose up -d

# Option 2: Separate SQLite backup (no downtime)
sqlite3 ~/.openclaw/gateway.db ".backup '/tmp/gateway-backup.db'"
# Then include /tmp/gateway-backup.db in your backup
```

## Restore (Manual — No CLI)

`openclaw backup restore` does NOT exist yet (Issue #13616, deferred).

```bash
# 1. Stop gateway
docker compose down

# 2. Unpack backup
tar -xzf /backups/openclaw-20260320.tar.gz -C ~/.openclaw/

# 3. Fix ownership (container runs as uid 1000)
sudo chown -R 1000:1000 ~/.openclaw/

# 4. Validate and fix
openclaw doctor --fix

# 5. Start gateway
docker compose up -d

# 6. Verify channels
docker compose logs openclaw-gateway --tail=50
# Check: Telegram connected, Discord socket, WhatsApp auth
```

**WhatsApp sessions are fragile:** After restore/migration, expect to re-scan QR.
Include `~/.openclaw/credentials/whatsapp/` in backup to minimize re-pairing.

## Scheduled Backup via Cron

```json
{
  "cron": {
    "jobs": {
      "backup-daily": {
        "schedule": "0 2 * * *",
        "agent": "main",
        "prompt": "Run openclaw backup create --verify --output /backups/openclaw-$(date +%Y%m%d).tar.gz"
      }
    }
  }
}
```

## Migration Procedure

1. Create verified backup on source machine
2. Transfer archive to target machine
3. Install Docker + pull OpenClaw image on target
4. Follow restore procedure above
5. Reconnect channels:
   - Telegram: automatic (token-based)
   - Discord: automatic (token-based)
   - WhatsApp Baileys: likely needs re-scan QR
   - Slack: automatic (app token-based)
