---
name: digital-office-foundation
description: |
  Use when setting up a new OpenClaw Docker environment, gateway configuration,
  or first installation. Triggers: "docker setup", "gateway start", "new installation",
  "digital office setup", "infra setup".
---

# Digital Office Foundation

Tested against: OpenClaw v2026.3.x

This skill covers the infrastructure layer for a new OpenClaw digital office:
Docker setup, Gateway configuration, environment variables, and the pre-flight checklist.

For security hardening of this infrastructure, see **security/** skill.
For scaling beyond one Gateway, see **scaling/** skill.

---

## Architecture Overview

An OpenClaw installation runs **two Docker services** sharing a network:

| Service | Role |
|---|---|
| `openclaw-gateway` | Core Gateway process — manages agents, channels, sessions, LLM routing. Listens on port 18789. |
| `openclaw-cli` | Companion CLI container for running `openclaw` commands against the Gateway. Runs on demand via Docker profiles. |

One Gateway = one OpenClaw instance. It cannot horizontally scale.
Multiple offices = multiple agents within one Gateway (up to ~5-10).
Beyond that, deploy separate Gateways on separate machines. See **scaling/** for details.

---

## Setup Paths

### Path A: Official Setup (recommended for first install)

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
bash docker-setup.sh
```

The setup script runs an onboarding wizard that creates the initial `openclaw.json`,
generates a Gateway token, and writes a production-ready docker-compose.yml.

### Path B: Custom Setup (existing infrastructure)

Use your own Dockerfile and pin the official image:

```
OPENCLAW_IMAGE=ghcr.io/openclaw/openclaw:2026.3.x
```

Reference the `docker-compose-prod.yml` in this skill's references directory.
Copy it to your server and create a `.env` file alongside it.

---

## Key Environment Variables

All variables are set in the `.env` file alongside docker-compose.yml.
Never hardcode values directly in docker-compose.yml.

| Variable | Required | Description |
|---|---|---|
| `OPENCLAW_IMAGE` | yes | Pinned image reference. Never `:latest` in production. |
| `OPENCLAW_GATEWAY_BIND` | yes | Always `127.0.0.1` (loopback). Never `0.0.0.0`. |
| `OPENCLAW_GATEWAY_PORT` | yes | Default `18789`. |
| `OPENCLAW_GATEWAY_TOKEN` | yes | Strong random token. Generate: `openssl rand -hex 32`. |
| `OPENCLAW_SANDBOX` | recommended | Set `1` to enable sandbox mode. |
| `OPENCLAW_HOME_VOLUME` | optional | Named Docker volume for `/home/node` persistence. |
| `OPENCLAW_WORKSPACE_DIR` | optional | Override default workspace directory path. |
| `OPENCLAW_CONFIG_PATH` | optional | Alternate path to `openclaw.json` — useful for multi-Gateway setups. |
| `OPENCLAW_EXTRA_MOUNTS` | optional | Comma-separated bind mounts: `source:target[:opts]`. |
| `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT` | optional | Chromium renderer limit. Default `2`. Set `1` on constrained VPS. |
| `OPENCLAW_DOCKER_APT_PACKAGES` | optional | Extra apt packages to install in the image. |
| `OPENCLAW_EXTENSIONS` | optional | Space-separated extension names to load. |
| `OPENCLAW_DOCKER_SOCKET` | optional | Override Docker socket path (use with extreme caution — see security/). |
| `ANTHROPIC_API_KEY` | if using Claude | `sk-ant-...` |
| `OPENAI_API_KEY` | if using OpenAI | `sk-...` |
| `OPENROUTER_API_KEY` | if using OpenRouter | `sk-or-...` |

---

## Image Pinning — Non-Negotiable Rule

**Never use `:latest` in production.**

A Hostinger image update changed the agent config schema (`agents.entries` was
restructured). Every Gateway running `:latest` crashed simultaneously on the next
restart. No warning, no migration guide.

Always pin a specific tag:
```
OPENCLAW_IMAGE=ghcr.io/openclaw/openclaw:2026.3.x
```

Before upgrading: read the release notes, test on staging, update config schema
if needed, then update the pinned tag in your `.env` file.

---

## Port Binding — Security Rule

`OPENCLAW_GATEWAY_BIND` must always be `127.0.0.1` (loopback).

The Gateway Entrypoint (`http://127.0.0.1:18789`) should never be exposed directly
on a public interface. For remote access:

- **SSH tunnel:** `ssh -L 18789:localhost:18789 user@{HOST}`
- **Reverse proxy:** Nginx on port 443 → upstream `127.0.0.1:18789`

See **security/** for full Gateway security configuration, token management, and Nginx setup.

---

## Pre-Flight Checklist

Run these checks on the target machine BEFORE `docker compose up`:

### 1. Docker Installed
```bash
docker --version && docker compose version
```

### 2. Port 18789 Free
```bash
ss -tlnp | grep 18789
# Should return nothing. If occupied: find and stop the conflicting process.
```

### 3. Swap — Critical on 1GB VPS
```bash
free -h
# Swap must be >= 2GB. If not, add swap before proceeding.
# See gotchas.md #1 for swap setup commands.
```

### 4. Disk Space
```bash
df -h /var/lib/docker
# Need at least 5GB free. OpenClaw image is ~2GB; workspace data grows over time.
```

### 5. uid 1000 Ownership (if using bind mounts)
```bash
ls -lan /path/to/config
# Owner must be 1000. Fix: sudo chown -R 1000:1000 /path/to/config
```

### 6. Network Connectivity
```bash
docker pull ghcr.io/openclaw/openclaw:2026.3.x
# Must succeed. If not: check firewall rules for ghcr.io access.
```

---

## Starting the Gateway

```bash
# First run — creates volumes, starts Gateway
docker compose up -d

# Check Gateway health
docker compose ps
docker compose logs openclaw-gateway --tail=50

# Run CLI command (uses profile)
docker compose --profile cli run openclaw-cli models list

# Stop Gateway
docker compose down
```

---

## docker-compose-prod.yml Summary

The reference file (`references/docker-compose-prod.yml`) implements:

- `user: "1000:1000"` — non-root execution
- `security_opt: no-new-privileges:true` — no privilege escalation
- `read_only: true` — immutable root filesystem
- `cap_drop: ALL` — no Linux capabilities
- `tmpfs: /tmp` — writable scratch space without persistent disk access
- `shm_size: '2g'` — Chromium shared memory (required for browser tool)
- `ports: 127.0.0.1:18789:18789` — loopback-only Gateway Entrypoint
- No Docker socket mount — prevents container escape
- Named volumes for config and workspaces — Docker manages ownership
- Healthcheck via node fetch every 30s
- Resource limits: 2 CPU / 2GB memory (adjust for your VPS)

For the security rationale behind each flag, see **security/** skill.

---

## Cross-References

| Topic | Skill |
|---|---|
| Docker security hardening (flags, token rotation) | security/ |
| Multiple offices, Kubernetes, resource limits | scaling/ |
| Agent and workspace configuration | office-structure/ |
| Channel connections (Telegram, Discord, WhatsApp) | channels/ |
| Monitoring and dashboards | monitoring/ |

---

## Sources

- docs.openclaw.ai/install/docker
- github.com/openclaw/openclaw/blob/main/docker-compose.yml
- docs.openclaw.ai/gateway/security (Docker section)
