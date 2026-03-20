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
