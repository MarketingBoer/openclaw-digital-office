# Test Scenarios — monitoring

---

## Test: Choose a Mission Control dashboard

**Input:** "I want a dashboard to monitor my OpenClaw agents. What are my options?"

**Expected WITHOUT skill:** Claude suggests building a custom dashboard or vaguely mentions Grafana.

**Expected WITH skill:** Claude presents the 5 Mission Control variants (robsannaa, crshdn/Autensa, TenacitOS, abhi1693, builderz-labs) with their strengths. Recommends starting with robsannaa (simplest, zero deps). Notes all connect via WebSocket on port 18789 and require device pairing.

**Verification:** 5 variants listed, robsannaa recommended as starting point, WebSocket 18789 mentioned.

---

## Test: No native spend cap

**Input:** "How do I set a monthly spending limit on my OpenClaw API costs?"

**Expected WITHOUT skill:** Claude looks for a config setting or suggests a third-party billing tool.

**Expected WITH skill:** Claude states there is NO native hard spend cap in OpenClaw. Budget monitoring must be built: cron job aggregating session data, alerts at 50/75/90% thresholds. Points to budget governance zones in models/ skill. Suggests context pruning config (cache-ttl, compaction, keepLastAssistants) to control token consumption.

**Verification:** "no native spend cap" stated, cron-based monitoring recommended, models/ cross-referenced for governance zones.

---

## Test: Compaction causing memory loss

**Input:** "My agent seems to forget things from earlier in the conversation. It was working fine yesterday."

**Expected WITHOUT skill:** Claude suggests checking MEMORY.md or restarting the gateway.

**Expected WITH skill:** Claude identifies compaction as a likely cause: when context overflow triggers LLM-based summarization (compaction mode: safeguard), details from earlier messages can be lost. Recommends: check `keepLastAssistants` setting (default 3), increase if needed. Also check `sessionRetention` — aggressive pruning loses entire sessions. Monitor compaction events in logs.

**Verification:** Compaction identified as cause, keepLastAssistants mentioned, sessionRetention link.

---

## Test: Mission Control requires device pairing

**Input:** "I deployed Mission Control in Docker but it can't connect to the gateway. I get a 403 error."

**Expected WITHOUT skill:** Claude suggests checking network configuration or firewall rules.

**Expected WITH skill:** Claude identifies that Mission Control requires device pairing — just being on the same Docker network is not enough. The dashboard must complete the pairing flow via the gateway's control UI. This is a security feature, not a bug.

**Verification:** Device pairing requirement explained, not just network access.

---

## Test: ClawMetry setup

**Input:** "I want real-time visibility into what my agents are doing."

**Expected WITHOUT skill:** Claude suggests tailing Docker logs.

**Expected WITH skill:** Claude recommends ClawMetry for real-time flow visualization: `pip install clawmetry && clawmetry connect`, then open localhost:8765. Shows token costs, sub-agent activity, cron jobs, memory changes in real-time. Notes: cloud option exists ($5/node/month, E2E encrypted) for remote access.

**Verification:** ClawMetry install command given, localhost:8765, cloud pricing mentioned.
