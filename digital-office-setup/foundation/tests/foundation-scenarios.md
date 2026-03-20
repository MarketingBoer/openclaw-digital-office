# Foundation Skill — Test Scenarios

These scenarios were written BEFORE the skill was built (TDD: RED phase).
Each scenario defines what the skill must produce to pass.

---

## Test: New VPS Docker Setup

**Input:** "I have a fresh 1GB VPS. How do I install OpenClaw?"

**Expected WITHOUT skill:** Claude improvises a generic docker run command,
binds port 0.0.0.0:18789, uses :latest, forgets swap, forgets uid 1000 ownership.

**Expected WITH skill:**
1. Run pre-flight checklist: Docker installed, port 18789 free, swap >= 2GB (critical on 1GB VPS), disk >= 5GB, uid 1000 ownership on config dir.
2. Provide the two-service docker-compose-prod.yml (gateway + cli) with loopback binding 127.0.0.1:18789:18789, pinned image tag (never :latest), user 1000:1000.
3. Warn: add 2GB swap before running docker compose up — pnpm install OOMs without it.
4. Remind: chown -R 1000:1000 on all bind-mount directories before starting.

**Verification:** docker-compose-prod.yml reference file exists with correct loopback port and user 1000:1000.

---

## Test: Image Pinning Warning

**Input:** "Can I just use openclaw:latest for my production setup?"

**Expected WITHOUT skill:** Claude says "sure, latest is fine for testing."

**Expected WITH skill:**
Claude refuses to recommend :latest. Explains the concrete risk: a Hostinger update
broke production because the new image had a changed config schema (`agents.entries`
was not recognized), causing a gateway crash. Instructs to always pin a specific tag,
e.g. `ghcr.io/openclaw/openclaw:2026.3.x`.

**Verification:** gotchas.md contains the :latest incident with concrete consequence documented.

---

## Test: Port Binding Security

**Input:** "My gateway needs to be accessible from the internet. Should I bind to 0.0.0.0?"

**Expected WITHOUT skill:** Claude suggests 0.0.0.0 with a note to add firewall rules.

**Expected WITH skill:**
Claude explains that OPENCLAW_GATEWAY_BIND must ALWAYS be 127.0.0.1 (loopback) in production.
For remote access, use SSH tunneling (`ssh -L 18789:localhost:18789 user@host`) or
a reverse proxy (Nginx). Direct 0.0.0.0 exposure risks unauthorized gateway access.
References security/ skill for full hardening details.

**Verification:** SKILL.md contains loopback-binding rule with cross-reference to security/.

---

## Test: Chromium / Playwright OOM Crash

**Input:** "My OpenClaw gateway crashes when I use the browser tool."

**Expected WITHOUT skill:** Claude suggests increasing Docker memory limits generically.

**Expected WITH skill:**
Claude diagnoses: the default shared memory (shm_size) is 64MB, which is too small for
Chromium. Fix: add `shm_size: '2g'` to the gateway service in docker-compose.yml.
Also checks if OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT is set (default 2 is reasonable;
reduce to 1 on constrained VPS).

**Verification:** docker-compose-prod.yml reference includes `shm_size: '2g'`.

---

## Test: Plugin EACCES Cascade Failure

**Input:** "After adding a plugin, all openclaw CLI commands fail with permission errors."

**Expected WITHOUT skill:** Claude suggests re-installing openclaw or checking file permissions generically.

**Expected WITH skill:**
Claude identifies the plugin uid ownership cascade: plugins installed as root (uid 0) in
a container running as uid 1000 cause config validation failure, which blocks ALL CLI
commands — not just the plugin command. Fix options:
1. Use a docker-entrypoint-wrapper.sh that runs `chown root:root` on extensions at startup.
2. Re-install the plugin with the correct uid.
Warns: this is a cascading failure — one broken plugin blocks unrelated channels.

**Verification:** gotchas.md contains the plugin uid cascade with "cascading failure" label.
