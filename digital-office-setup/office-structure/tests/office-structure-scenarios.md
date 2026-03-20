# Test Scenarios — office-structure

TDD scenarios written BEFORE building the skill. Each scenario defines expected behavior with and without the skill loaded.

---

## Test: New office from scratch

**Input:** "I want to create a new office for client Acme. What workspace files do I need?"

**Expected WITHOUT skill:** Claude lists some generic files, possibly invents names, misses BOOTSTRAP.md lifecycle rule, misses HEARTBEAT.md, confuses AGENTS.md with TOOLS.md purposes, gets bootstrap size limits wrong or omits them.

**Expected WITH skill:** Claude lists all 11 canonical workspace files (SOUL.md, AGENTS.md, USER.md, IDENTITY.md, TOOLS.md, HEARTBEAT.md, BOOT.md, BOOTSTRAP.md, MEMORY.md, memory/YYYY-MM-DD.md, skills/), explains purpose of each, notes BOOTSTRAP.md must be deleted after first run, gives the 20K-per-file / 150K-total limits, and suggests `openclaw agents add {name}` to start.

**Verification:** Response includes all 11 file names, mentions "delete after" for BOOTSTRAP.md, states bootstrapMaxChars=20000 and bootstrapTotalMaxChars=150000.

---

## Test: Agent roles — which model for which job

**Input:** "I have three agents: one for day-to-day client work, one for SEO audits, and one that runs nightly reports. Which model should each use?"

**Expected WITHOUT skill:** Claude may suggest Opus for everything, or guess random model assignments without reasoning about read-only auditor or cost implications for cron workers.

**Expected WITH skill:** Claude maps Operations Agent → Sonnet primary + Opus fallback, QA Auditor → Haiku primary + Sonnet fallback with read-only tool restriction, Cron Worker → cheapest tier (Haiku or local). Explains why QA Auditor must not have exec tools. Cross-references models/ skill for failover config details.

**Verification:** Three roles named correctly, Haiku for QA, cheapest for cron, read-only tools mentioned for auditor.

---

## Test: Routing — which agent handles which message

**Input:** "I have two agents: one for sales (hq-sales) and one for support (hq-support). A Telegram message comes in from user 12345 — how does the gateway decide which agent answers?"

**Expected WITHOUT skill:** Claude describes a vague "routing system" without hierarchy specifics, may confuse peer vs channel routing, misses guildId for Discord vs Telegram difference.

**Expected WITH skill:** Claude explains the full routing hierarchy: peer → parentPeer → guildId → accountId → channel → fallback. Explains that for Telegram user 12345, the gateway checks peer binding first, then falls back to channel default, then global fallback. Notes Telegram shows 1 bot to the user regardless. Mentions dmScope per-channel-peer for multi-user inboxes.

**Verification:** All 6 routing tiers named in order, dmScope per-channel-peer mentioned, Telegram 1-bot-per-agent-behind-scenes explained.

---

## Test: BOOTSTRAP.md — what happens if you forget to delete it

**Input:** "My agent keeps running its initial setup steps every time it restarts. Why?"

**Expected WITHOUT skill:** Claude guesses it might be a BOOT.md issue or cron job, does not identify BOOTSTRAP.md as the root cause.

**Expected WITH skill:** Claude immediately identifies that BOOTSTRAP.md is the one-time first-run ritual file and it re-executes on every gateway restart if not deleted. Instructs user to delete BOOTSTRAP.md from the workspace after the first run is confirmed complete.

**Verification:** Response names BOOTSTRAP.md as the cause, states "delete after completion", distinguishes from BOOT.md (which is intentionally persistent).

---

## Test: Sub-agent config — prevent runaway spawning

**Input:** "I want my main agent to spawn sub-agents for research tasks, but I'm worried about runaway costs. What settings control this?"

**Expected WITHOUT skill:** Claude suggests setting a timeout or vaguely mentions resource limits, does not name the exact config keys.

**Expected WITH skill:** Claude names all 5 sub-agent config keys: maxConcurrent, maxSpawnDepth, maxChildrenPerAgent, runTimeoutSeconds, archiveAfterMinutes. Explains maxSpawnDepth=1 prevents sub-sub-agents. Notes archiveAfterMinutes=60 default can lose context for long-running tasks. Cross-references models/ for per-agent model override to keep sub-agents on cheap tier.

**Verification:** All 5 keys named, maxSpawnDepth explained, archiveAfterMinutes gotcha mentioned.
