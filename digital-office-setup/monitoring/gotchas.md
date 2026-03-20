# Monitoring — Gotchas

Real pitfalls with dashboards, cost tracking, and context management.

---

## 1. No Native Spend Cap — Budget Overruns Are Silent

**What happens:** Monthly API costs exceed budget. No warning until you
check the provider's billing dashboard manually.

**Why:** OpenClaw has no built-in spending limit, alerting, or budget cap.
Budget management is intentionally delegated to the operator.

**Fix:** Build monitoring via cron + alerts (see workflows/). Set budget
governance zones in models/. Alert at 50%, 75%, 90% thresholds via Telegram.

---

## 2. Mission Control Requires Device Pairing

**What happens:** Dashboard deployed on same Docker network, network is fine,
but dashboard shows 403 or fails to connect.

**Why:** Connecting to the gateway requires device pairing — a security feature.
Network access alone is not sufficient. The dashboard must complete the pairing
flow in the gateway's control UI.

**Fix:** Open the gateway control UI (localhost:18789), approve the pending
device pairing request from the dashboard.

---

## 3. Compaction Can Cause Memory Loss

**What happens:** Agent forgets context from earlier in the conversation.
Behavior is inconsistent — sometimes remembers, sometimes doesn't.

**Why:** When the context window overflows, compaction (LLM-based summarization)
kicks in. It preserves recent messages but can lose details from earlier ones.
The agent doesn't know what was lost.

**Fix:** Monitor compaction events in logs. Increase `keepLastAssistants`
(default 3) if agents need more context. Enable `memoryFlush` to write
durable state to MEMORY.md before compaction starts.

---

## 4. Session Pruning Too Aggressive = Lost Context

**What happens:** Agent can't recall conversations from yesterday.
MEMORY.md wasn't updated before the session was pruned.

**Why:** `sessionRetention` (default 24h) deletes completed sessions.
If the agent didn't flush important context to MEMORY.md before the session
ended, that context is permanently lost.

**Fix:** Ensure `memoryFlush: true` in context pruning config. Review
`sessionRetention` — increase for agents that need multi-day context.
Also review `session.reset.idle` — 120 min idle may be too aggressive.

---

## 5. ClawMetry Cloud is Paid ($5/node/month)

**What happens:** User sets up ClawMetry expecting free remote monitoring,
then discovers the cloud dashboard requires a paid subscription.

**Why:** Local ClawMetry (localhost:8765) is free. The cloud option
(remote access, E2E encrypted) costs $5/node/month.

**Fix:** Use local ClawMetry for single-machine setups. Cloud is only
needed for remote monitoring across multiple gateways.
