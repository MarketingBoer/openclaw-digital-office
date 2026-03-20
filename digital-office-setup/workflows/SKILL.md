---
name: digital-office-workflows
description: |
  Use when setting up TDD workflows, webhook integrations, n8n connections,
  or automated audit pipelines. Triggers: "tdd setup", "configure webhook",
  "connect n8n", "automate audit", "seo audit automatic", "build workflow".
---

# Skill: digital-office-workflows

Tested against: OpenClaw v2026.3.x

This skill covers the automation and workflow layer of a digital office: how to enforce quality through TDD, how to receive external triggers via webhooks, how to connect n8n for execution, and how to schedule recurring audits with Cron Jobs.

---

## Ownership

This skill owns: **Cron Job** configuration, **webhook** integrations, **n8n** integration patterns.
Cross-references: `channels/` for channel-specific webhook delivery, `operations/` for Cron Job scheduling in daily management, `monitoring/` for audit report review.

---

## 1. TDD Workflow

### Iron Law
No production code without a failing test. This is non-negotiable. If you encounter the rationalization "skip TDD just this once" — STOP. Do not proceed.

### Red-Green-Refactor Cycle
1. **RED** — Write a failing test that describes the desired behavior
2. **GREEN** — Write the minimal code/skill content to make it pass
3. **REFACTOR** — Improve without breaking the test
4. Anti-pattern: writing skill content before the test → DELETE and start over

### Superpowers Two-Phase Review
When reviewing skill output, use two distinct phases:
- **Phase 1 — Conformity:** Does the output match the plan/spec?
- **Phase 2 — Quality:** Is it readable, performant, and secure?
Both phases are required. Combining them misses quality issues.

### OpenSpec: Spec-Driven Development
OpenSpec enforces formal specifications before implementation:

```
openspec init
openspec new change "{feature-name}"
# → writes RFC with Given/When/Then scenarios
openspec plan
openspec apply
openspec verify
openspec archive   # merges delta spec into master spec
```

**RFC 2119 keywords in specs:**
- `SHALL` / `MUST` — absolute requirement
- `SHOULD` — recommended, exceptions require justification
- `MAY` — optional

**Delta spec format:**
- `ADDED:` — new behavior
- `MODIFIED:` — changed behavior
- `REMOVED:` — deleted behavior

---

## 2. Webhook Integrations

### Concept
The Gateway exposes an Entrypoint at `/hooks` for receiving external events. External systems (n8n, CI/CD, forms, monitoring) POST to this endpoint to trigger an agent run.

### Quick Setup
See `references/webhook-config.md` for the exact config block, payload fields, and auth details.

### Key Principles
- **Token is a password** — store in `OPENCLAW_HOOKS_TOKEN` env var, never hardcode
- **Auth method** — Bearer token in Authorization header; query-string is rejected
- **Session safety** — keep `allowRequestSessionKey: false` (default) to prevent external callers from targeting arbitrary sessions
- **`allowedAgentIds`** — restrict to only the agents that need webhook access

### Endpoint Decision
| Need | Use |
|------|-----|
| Trigger a full agent run with a prompt | `POST /hooks` |
| Nudge agent on its next heartbeat | `POST /hooks/wake` with `wakeMode: "next-heartbeat"` |
| Immediate nudge without full prompt | `POST /hooks/wake` with `wakeMode: "now"` |

### Payload Quick Reference
Required: `message` (the prompt string)
Optional: `agentId`, `sessionKey` (if allowed), `deliver`, `channel`, `to`, `model`, `thinking`, `timeoutSeconds`

### Advanced Routing
- **Presets** — named payload templates for common sources (e.g., `"gmail"`)
- **Mappings** — conditional routing based on payload field values
- **TransformsDir** — JS/TS modules for complex payload transformation before routing

---

## 3. n8n Integration

### Architecture
```
OpenClaw (reasoning)
    ↕ webhook
n8n (execution)
    ↕ API calls
External Services (email, CRM, databases, etc.)
```

**Principle:** OpenClaw decides what to do. n8n does the doing. This separation keeps agent config clean and API keys out of OpenClaw.

### Docker Shared Network
When both run on the same Docker network, n8n is reachable at `http://n8n:5678/webhook/workflow` without exposing it externally. Use the pre-configured `openclaw-n8n-stack` (github.com/caprihan/openclaw-n8n-stack) for a ready-made setup.

### Credential Isolation
**SOUL.md rule:** "NEVER store API keys in my environment or skill files."

API keys and credentials belong exclusively in n8n's encrypted credential store. The agent only knows the n8n webhook URL — never the downstream API key.

### Integration Patterns

**Pattern A — n8n triggers OpenClaw:**
n8n schedule or external event → HTTP POST to Gateway `/hooks` → agent processes and responds

**Pattern B — OpenClaw triggers n8n:**
Agent decides an action is needed → calls n8n webhook URL → n8n executes (send email, update CRM, etc.)

**Pattern C — Bidirectional:**
Example: email triage pipeline — n8n receives email → triggers OpenClaw to classify → OpenClaw responds → n8n routes based on classification

### Error Handling
- n8n: configure retry mechanism for failed webhook calls
- OpenClaw: configure fallback notification via Telegram channel on timeout

---

## 4. Audit Automation

### Workflow
```
Client request
  → Create project directory in agent workspace
  → Configure Cron Job targeting qa-auditor agent
  → Cron fires on schedule
  → Agent runs seo-audit skill
  → Report saved to workspace reports/
  → Notification delivered via channel
```

### Cron Job Concept
A **Cron Job** is a time-scheduled task managed via the `openclaw cron` CLI.
It triggers an agent with a fixed prompt on a crontab schedule.

> ⚠️ Cron jobs are **not** configured in `openclaw.json` — the schema
> rejects any cron-related keys at the root level. Use the CLI only.

```bash
# Add a weekday morning status job
openclaw cron add \
  --name "daily-status" \
  --agent hq-mediadeboer \
  --cron "0 8 * * 1-5" \
  --message "Give a short daily status overview." \
  --announce \
  --channel telegram
```

See `references/webhook-config.md` for all flags and management commands.

### Key Flags
- `--cron <expr>` — crontab expression (e.g., `"0 8 * * 1-5"` = weekdays 08:00)
- `--every <duration>` — interval-based alternative (e.g., `1h`, `30m`)
- `--announce` — deliver result to channel after completion (replaces deprecated `--deliver`)
- `--session isolated` — fresh session per run (prevents context bleed between runs)

### Report Convention
Store audit reports at: `reports/seo/{YYYY-MM-DD}-{project-name}.md` in the agent workspace.

### seo-audit Skill
The `seo-audit` skill (from playbooks.com/skills/openclaw/skills/seo-audit) provides:
- URL fetch and HTML analysis
- Score per category (0–100)
- Markdown report output

---

## Cross-References

| Topic | Skill |
|---|---|
| Channel-specific webhook delivery | channels/ |
| Cron worker agent config | office-structure/ |
| Backup scheduling | operations/ |
| Budget alerts via cron | models/ |
| Dashboard for audit reports | monitoring/ |

---

## Sources

- docs.openclaw.ai/automation/webhook
- docs.openclaw.ai/automation/cron-jobs
- github.com/caprihan/openclaw-n8n-stack
- lobehub.com/skills/oabdelmaksoud-openclaw-skills-test-driven-development
- playbooks.com/skills/openclaw/skills/seo-audit
