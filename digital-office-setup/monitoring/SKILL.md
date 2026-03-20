---
name: digital-office-monitoring
description: |
  Use when setting up monitoring, Mission Control dashboard,
  cost tracking, or token usage analysis. Triggers: "dashboard setup",
  "mission control", "monitor costs", "token usage", "monitoring",
  "how much does it cost", "budget".
---

# Digital Office Monitoring

Tested against: OpenClaw v2026.3.x

This skill owns dashboards, cost tracking, and context management.
For budget governance zones, see **models/**. For cron scheduling, see **workflows/**.

---

## Mission Control — Dashboard Options

All variants connect via WebSocket on port 18789. All require device pairing.

| Variant | Strengths | Docker | Best For |
|---------|-----------|--------|----------|
| robsannaa | Zero deps, terminal in browser | No (on host) | Starting point |
| crshdn (Autensa) | Kanban + Live Feed | Docker Compose | Project management |
| TenacitOS | Cost tracking, auto-discovery | Next.js | Cost-focused teams |
| abhi1693 | Enterprise, approval workflows | Docker | Organizations |
| builderz-labs | Cost dashboards, security scanners | Docker Compose | Security-focused |

**Recommendation:** Start with robsannaa (simplest). Upgrade to crshdn or TenacitOS
when you need project management or cost analytics.

**Alternative:** Pure CSS office visualization (proven in production — Next.js 15 +
CSS animations, no PixiJS needed for MVP).

---

## No Native Spend Cap

OpenClaw has **no built-in spending limit**. Budget overruns are silent until
you check your provider dashboard. You must build monitoring yourself.

### Budget Monitoring via Cron

Set up a Cron Job (see workflows/) that:
1. Aggregates session token usage from gateway.db
2. Compares against monthly budget
3. Sends alerts at 50%, 75%, 90% thresholds via Telegram/Discord
4. Downgrades model tier when frugal zone is reached

Cross-reference: **models/** for budget governance zones (Normal/Warning/Frugal/Blocked).

---

## Token & Cost Tracking Tools

### ClawMetry (real-time)

```bash
pip install clawmetry
clawmetry connect
# Open http://localhost:8765
```

Features: real-time flow visualization, token costs, sub-agent activity,
cron jobs, memory changes. Cloud option: $5/node/month (E2E encrypted).

### Token Dashboard (custom)

Express + SQLite (openclaw-tokens.db) + Prometheus.
Endpoints: /api/overview, /api/model-status, /api/cost-analysis.
Per-model cost breakdown and rate limit tracking.

### TenacitOS (built-in analytics)

Built-in cost analytics from OpenClaw session data (SQLite).
Install via Docker, auto-discovers gateway on shared network.

### Prometheus + Grafana (infrastructure)

Custom dashboards for gateway health, session count, token throughput.
Alert rules for CPU, memory, OOM events, budget thresholds.

---

## Context Pruning — Control Token Consumption

Long sessions consume more tokens per request. Configure pruning to manage costs:

### cache-ttl Mode
Trim expired tool results before LLM calls. Example: `ttl: 6h` removes
tool results older than 6 hours from context.

### Compaction (safeguard mode)
When context overflows, LLM-based summarization kicks in. This preserves
the conversation but can lose details from earlier messages.

Key settings:
- `keepLastAssistants: 3` — keep last 3 assistant messages intact
- `memoryFlush` — write durable state before compaction starts
- `mode: safeguard` — only compact when overflow would block the session

### Session Reset

`sessionRetention` and `session.reset` control session lifetime.
Shorter sessions = less token accumulation, but more context loss.
Balance based on your use case.

---

## Gotcha Awareness

- No spend cap → budget overruns are silent (see gotchas.md)
- Mission Control requires device pairing, not just network access
- Compaction can cause memory loss — monitor compaction events
- ClawMetry cloud is paid ($5/node/month)

---

## Cross-References

| Topic | Skill |
|---|---|
| Budget governance zones, model tiers | models/ |
| Cron job scheduling for monitoring | workflows/ |
| Gateway Docker setup | foundation/ |
| Session management, archiveAfterMinutes | office-structure/ |

---

## Sources

- github.com/robsannaa/openclaw-mission-control
- github.com/crshdn/mission-control
- github.com/carlosazaustre/tenacitOS
- producthunt.com/products/clawmetry
