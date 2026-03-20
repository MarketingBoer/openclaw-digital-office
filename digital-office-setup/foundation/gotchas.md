# Foundation — Gotchas

Real incidents and pitfalls for OpenClaw Docker setup.
Each entry includes: what happens, why, and the fix.

---

## 1. OOM Crash During First Install (1GB VPS)

**What happens:** `docker compose up` starts, then the container exits silently.
Logs show pnpm or node OOM-killed. Gateway never comes up.

**Why:** `pnpm install` inside the image requires at least 2GB RAM to run without swap.
A 1GB VPS has no swap by default.

**Fix — add 2GB swap BEFORE starting:**
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
# Make permanent:
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**Pre-flight check:** `free -h` — confirm swap is present before `docker compose up`.

---

## 2. EACCES Permission Errors on Bind Mounts

**What happens:** Gateway starts but writes fail. Logs show `EACCES: permission denied`.
The gateway container runs as uid 1000, but the host directories are owned by root.

**Why:** Docker bind mounts inherit host file ownership. If the host directory is owned
by root (uid 0), the container's uid 1000 process cannot write to it.

**Fix — set ownership before starting:**
```bash
sudo chown -R 1000:1000 /path/to/openclaw-config
sudo chown -R 1000:1000 /path/to/openclaw-workspaces
```

**Named volumes (preferred):** If using named Docker volumes (as in docker-compose-prod.yml),
Docker handles ownership automatically. Prefer named volumes over host bind mounts.

---

## 3. Chromium Crash: shm_size Too Small

**What happens:** The browser tool fails. Gateway logs show Chromium renderer crashes
or `--disable-dev-shm-usage` warnings. Screenshots return blank or never complete.

**Why:** Chromium's default shared memory allocation is 64MB in Docker.
Chromium requires significantly more for rendering real pages.

**Fix:** Add to the gateway service in docker-compose.yml:
```yaml
shm_size: '2g'
```

Also consider: `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT: "1"` on constrained VPS
to limit concurrent Chromium processes.

---

## 4. Plugin uid=1000 Ownership Cascade Failure

**What happens:** After adding a plugin (extension), ALL `openclaw` CLI commands fail —
not just the new plugin. Error: config validation failure. Unrelated channels break.

**Why:** Plugins installed as root (uid 0) inside a container that runs as uid 1000
fail the config validation check. OpenClaw's config validation is all-or-nothing:
one bad entry blocks the entire config parse, making ALL agents and channels inaccessible.

**Fix options:**
1. Re-install the plugin via the gateway running as uid 1000 (not via a root shell).
2. Add a docker-entrypoint-wrapper.sh that runs `chown root:root` on extension files
   at container startup before the gateway process starts.
3. Manually fix ownership: `docker exec -u root <container> chown -R 1000:1000 /home/node/.openclaw/extensions/`

**Pattern:** This is a cascading failure. Symptoms look like a complete gateway failure,
but the root cause is a single plugin ownership mismatch.

---

## 5. :latest Image Breaks Production on Gateway Restart

**What happens:** After a `docker compose pull && docker compose up`, the gateway crashes
or agents are no longer recognized. Config format appears corrupt. No data loss, but
gateway is down.

**Why:** OpenClaw image updates can change the config schema (e.g., `agents.entries`
was renamed/restructured). Using `:latest` means any `docker compose pull` can
silently upgrade to a breaking schema version.

**Real incident:** A Hostinger image update changed the agent config structure.
All instances using `:latest` crashed simultaneously on next restart.

**Fix:** Always pin a specific image tag:
```yaml
image: ghcr.io/openclaw/openclaw:2026.3.x  # NEVER :latest
```

Before upgrading: read the release notes, test on a staging instance, update config
schema if needed, THEN update the pinned tag.

---

## 6. Local Docker Without Reverse Proxy Requires Dangerous Flags

**What happens:** Running OpenClaw locally (not on a server) requires three flags in
openclaw.json that are flagged as dangerous:

```json
{
  "dangerouslyAllowHostHeaderOriginFallback": true,
  "allowInsecureAuth": true,
  "dangerouslyDisableDeviceAuth": true
}
```

**Why:** The gateway's security model assumes HTTPS and proper host headers.
Without a reverse proxy providing these, auth and origin checks fail.

**Fix for local dev only:** Use the flags above. They are safe on localhost
(only accessible to your local machine).

**NEVER use these flags on a public-facing server.** Use Nginx + TLS + loopback instead.
See security/ skill for reverse proxy setup guidance.

---

## 7. Gateway Lock File Stale After Crash

**What happens:** After a container crash or OOM kill, the gateway refuses to
start. Logs show `gateway already running` or lock acquisition failure.
Restarting the container does not help — the same error repeats.

**Why:** The gateway writes a lock file at `/tmp/openclaw-{uid}/gateway.*.lock`
on startup. A clean shutdown removes it. A crash (OOM, SIGKILL, power loss)
leaves the lock file behind. On restart, the gateway sees the stale lock and
refuses to start a second instance.

**Fix:**
```bash
docker exec <container> openclaw doctor --fix
# Or manually:
docker exec <container> find /tmp -name 'gateway.*.lock' -delete
docker restart <container>
```

**Prevention:** Add a lock cleanup step to your entrypoint wrapper:
```bash
find /tmp -name 'gateway.*.lock' -delete 2>/dev/null || true
```
This is safe — the lock is only meaningful while the process is running.

---

## 8. Wrapper Entrypoint Pattern for Extension Ownership

**What happens:** Foundation gotcha #4 describes the uid ownership cascade
failure. The most reliable fix is a wrapper entrypoint that runs ownership
corrections at every container start — before the gateway process launches.

**Why:** Manual `docker exec chown` fixes the immediate problem but doesn't
survive container recreation. Plugin reinstalls or image updates can reintroduce
the ownership mismatch. A wrapper entrypoint is permanent.

**Proven pattern:**
```bash
#!/bin/bash
set -e

# Fix data directory ownership for gateway process
chown -R node:node /data

# Clean stale lock files from previous crashes
find /data -name '*.lock' -delete 2>/dev/null || true

# Extensions must be root-owned (OpenClaw trust policy: uid=0 = trusted plugin code)
chown -R root:root /data/.openclaw/extensions/ 2>/dev/null || true

# Start the gateway as the non-root user
cd /hostinger  # or /app depending on image
exec runuser -u node -- node server.mjs
```

Mount it read-only in docker-compose.yml:
```yaml
entrypoint: ["/docker-entrypoint-wrapper.sh"]
volumes:
  - ./docker-entrypoint-wrapper.sh:/docker-entrypoint-wrapper.sh:ro
```

**Note:** The `chown root:root` on extensions is counterintuitive — you'd expect
uid 1000 ownership. But OpenClaw's plugin trust model requires root ownership
as proof that the plugin was installed by a privileged user, not injected by
the gateway process itself.

---

## 8. gateway.mode=local Blocks CLI Cron Commands

**What happens:** `openclaw cron add` fails with `gateway closed (1000)` or
`gateway closed (1006)`. The CLI connects but the gateway immediately closes
the WebSocket connection. `openclaw cron status` works (reads local file),
but any write command fails.

**Why:** In `gateway.mode: "local"`, the gateway does not expose a persistent
WebSocket endpoint for CLI write operations. The CLI needs `mode: "remote"`
with a configured `remote.url` to connect. But setting `mode: "remote"` without
proper configuration causes `Gateway start blocked` and crashes the gateway.

**Fix — workaround:** Write cron jobs directly to the jobs file while the
gateway is stopped:
1. `docker compose stop openclaw`
2. Edit `/data/.openclaw/cron/jobs.json` (format: `{"version":1,"jobs":[...]}`)
3. `docker compose start openclaw`

The gateway reads `jobs.json` at startup and schedules all enabled jobs.
Manual edits are only safe when the gateway is stopped.

**Fix — proper (if CLI access needed):** Configure remote mode correctly:
```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token"
    }
  }
}
```
Note: this requires the gateway to accept the connection, which may need
additional network configuration in Docker environments.
