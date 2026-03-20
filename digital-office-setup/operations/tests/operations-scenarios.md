# Test Scenarios — operations

---

## Test: Backup procedure — what to include

**Input:** "How do I back up my OpenClaw instance?"

**Expected WITHOUT skill:** Claude suggests copying the workspace folder, misses credentials/, gateway.db, sessions, and extensions.

**Expected WITH skill:** Claude explains the official CLI (`openclaw backup create` with flags), lists everything under ~/.openclaw that must be backed up (openclaw.json, gateway.db, credentials/, sessions/, workspace/, extensions/, exec-approvals.json). Warns about SQLite WAL corruption on live backup. Recommends stopping the gateway first or using `sqlite3 .backup` separately.

**Verification:** Full backup list present, WAL warning included, CLI flags documented.

---

## Test: Restore is manual — no CLI command

**Input:** "How do I restore from an OpenClaw backup?"

**Expected WITHOUT skill:** Claude suggests `openclaw backup restore` or a generic Docker restore command.

**Expected WITH skill:** Claude states that `openclaw backup restore` DOES NOT EXIST yet (Issue #13616 still open). Restore is manual: unpack backup archive to ~/.openclaw/, fix ownership (`chown -R 1000:1000`), run `openclaw doctor --fix`. Warns that WhatsApp sessions are fragile on migration — likely need re-scan QR.

**Verification:** "restore does not exist" explicitly stated, manual procedure given, WhatsApp re-pair warning included.

---

## Test: Docker upgrade gone wrong

**Input:** "I ran docker compose pull and now my gateway won't start. The logs say config validation error."

**Expected WITHOUT skill:** Claude suggests reinstalling OpenClaw or checking Docker volumes.

**Expected WITH skill:** Claude identifies the :latest tag crash pattern: upstream image changed config schema (agents.entries renamed). Fix: change OPENCLAW_IMAGE back to the previous pinned tag in .env, then `docker compose up -d`. Long-term: always pin specific version tags, read release notes before upgrading. Cross-references foundation/ gotchas for the full incident.

**Verification:** :latest incident identified, rollback procedure given, image pinning rule stated.

---

## Test: Skills security before install

**Input:** "I want to install a skill from ClawHub. What should I check?"

**Expected WITHOUT skill:** Claude says `clawhub install author/skill-name` without security review.

**Expected WITH skill:** Claude enforces the security protocol: (1) check VirusTotal report on ClawHub page, (2) review raw source on GitHub, (3) pin to specific version, (4) treat as untrusted code. Mentions 1,184+ malicious skills and 36% flaw rate. Cross-references security/ for full incident details.

**Verification:** All 4 steps listed, malicious skill stats referenced, security/ cross-referenced.

---

## Test: Gateway restart procedure

**Input:** "My gateway is acting weird after a config change. How do I properly restart it?"

**Expected WITHOUT skill:** Claude suggests `docker compose restart`.

**Expected WITH skill:** Claude gives the full restart procedure: (1) rm -f ~/.openclaw/cron/jobs.json (prevents cron conflict), (2) stop gateway, (3) start gateway, (4) verify: check Telegram bot startup in logs, Slack socket, WhatsApp auth state, (5) send test message per channel. Mentions `openclaw doctor` for post-restart validation.

**Verification:** jobs.json removal step included, per-channel verification checklist present, doctor command mentioned.
