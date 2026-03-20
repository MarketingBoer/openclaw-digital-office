---
name: digital-office-security
description: |
  Use when hardening OpenClaw, configuring permissions, tool restrictions,
  or security review. Triggers: "security", "hardening", "permissions", "read-only",
  "security check", "tool restrictions", "secure setup".
---

# Digital Office Security

Tested against: OpenClaw v2026.3.x

This skill covers hardening an OpenClaw Gateway at every layer: Docker, network,
channels, tools, credentials, and third-party skills. For the base Docker setup,
see **foundation/** — this skill owns the security rationale behind each flag.

For known incidents and CVEs, see `references/incidents-lessons.md`.

---

## Docker Hardening

The `docker-compose-prod.yml` in **foundation/** implements these flags.
This section explains WHY each one exists.

| Flag | Purpose |
|---|---|
| `security_opt: no-new-privileges` | Prevents privilege escalation inside the container |
| `read_only: true` | Immutable root filesystem — malware cannot persist |
| `cap_drop: ALL` | No Linux capabilities — blocks mount, network admin, ptrace |
| `user: "1000:1000"` | Non-root — limits blast radius of container escape |
| `tmpfs: /tmp` | Writable scratch without persistent disk access |
| `shm_size: '2g'` | Required for Chromium; default 64MB causes crashes |

### Docker Socket — Never Mount

Mounting the Docker socket (`/var/run/docker.sock`) gives the container full control
over the host Docker daemon. An agent with exec permissions could create privileged
containers, access host filesystems, or install rootkits. There is no safe way to
mount the Docker socket in a production OpenClaw setup.

### Port Binding — Loopback Only

`OPENCLAW_GATEWAY_BIND` must be `127.0.0.1`. Binding to `0.0.0.0` exposes the
Gateway Entrypoint to the network. For remote access:
- SSH tunnel: `ssh -L 18789:localhost:18789 user@{HOST}`
- Reverse proxy: Nginx with TLS on port 443 → upstream 127.0.0.1:18789

---

## Gateway Security

- **Token strength:** Generate with `openssl rand -hex 32`. Rotate quarterly.
  See CVE-2026-25253 in incidents-lessons.md — weak tokens enable 1-click RCE.
- **Token storage:** In `.env` file only, never in docker-compose.yml or workspace files.
- **Control UI:** Access via localhost or Tailscale only. Never expose publicly.
- **`openclaw doctor --fix`:** Detects overly permissive file modes and token issues.

---

## Channel Security

### dmPolicy — The Most Critical Setting

**NEVER set dmPolicy to "open" in production.**

The 220,000+ exposed Gateway crisis (SecurityScorecard/Penligent AI, early 2026) was
primarily caused by misconfigured dmPolicy settings. With "open", any platform user
can send messages to your agent, extract workspace data, and inject prompts.

| Setting | When to Use |
|---|---|
| `allowlist` | Production — only listed numeric user IDs can interact |
| `pairing` | Setup phase — requires device confirmation per new user |
| `open` | **NEVER in production** — use only for isolated local testing |

### Additional Channel Rules

- **Allowlists:** Always use numeric platform IDs, not @usernames (usernames change).
- **Groups:** Set `requireMention: true` so agents only respond to direct @mentions.
- **WhatsApp:** Always use a separate phone number (eSIM or second phone).
  Never the owner's personal number.

---

## Tool Deny Lists — Read-Only Configuration

For agents that should observe but never modify (e.g., WhatsApp review, QA auditor),
configure a tool deny list in the agent's config:

```json
{
  "tools": {
    "deny": ["write", "edit", "apply_patch", "exec", "process", "browser", "canvas", "nodes", "cron", "gateway", "image"]
  }
}
```

Additional restrictions:
- `workspaceOnly: true` for `fs` and `apply_patch` — limits file access to workspace
- `sessions.visibility: "tree"` — agent can only see its own sessions and sub-agents

Cross-ref **office-structure/** for per-agent tool configuration within workspace files.

---

## Credentials Management

- **Directory permissions:** 700 for dirs, 600 for files under `~/.openclaw`
- **NEVER store secrets in workspace files** (SOUL.md, AGENTS.md, MEMORY.md).
  These files are loaded into the agent's context window, logged in sessions,
  and can be exfiltrated by infostealers. See Vidar variant incident in
  incidents-lessons.md.
- **Store keys in:** `.env` file (Docker) or environment variables (systemd)
- **Run:** `openclaw doctor --fix` to detect overly permissive file modes

---

## WhatsApp Read-Only SEAL Pattern

For agents that must read WhatsApp messages but never send replies, use three
independent locks. A single lock is insufficient — config drift from restarts
and manual interventions has re-enabled sending in single-lock setups.

### Three-Lock SEAL

1. **Code-lock:** Early return in `onMessage` handler before any LLM call
2. **Token-lock:** Use a dummy/revoked `accessToken` so send API calls fail at auth
3. **Prompt-lock:** SOUL.md system prompt explicitly forbids outbound messages

### Drift Detection

Create `SEAL.md` with SHA-256 checksums of all three sealed files.
Store `system-prompt.md.canonical` and `CANONICAL-BINDINGS.json` as restore sources.
Verify checksums on each Gateway restart.

---

## Third-Party Skills Security

The ClawHub ecosystem has significant security risks. Before installing ANY skill:

1. **Check VirusTotal report** on the ClawHub skill page
2. **Review raw source code** — look for prompt injection, exfiltration patterns,
   external URL calls, or suspicious exec commands
3. **Pin to a specific version** — never install from a floating reference
4. **Treat as untrusted code** — skills run in your agent's context window

Background: 1,184+ malicious skills found on ClawHub; 36% of audited skills
contain security flaws. See incidents-lessons.md for details.

---

## Cross-References

| Topic | Skill |
|---|---|
| Docker compose setup (base config) | foundation/ |
| Per-agent tool configuration, workspace files | office-structure/ |
| Channel-specific setup details | channels/ |
| Backup and restore procedures | operations/ |
| Budget monitoring and cost alerts | monitoring/ |

---

## Sources

- docs.openclaw.ai/gateway/security
- docs.openclaw.ai/tools/skills (security section)
- nebius.com/blog/posts/openclaw-security
