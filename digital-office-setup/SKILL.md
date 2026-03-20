---
name: digital-office-setup
description: |
  Use when setting up or managing a complete digital office with OpenClaw.
  Routes to the right sub-skill. Triggers: "digital office", "office setup",
  "complete infra", "set up new business", "openclaw setup".
---

# Digital Office Setup — Router

Tested against: OpenClaw v2026.3.x

Architecture: 1 Gateway → N offices → agents per office.
USP: Free WhatsApp integration via ChakraHQ (official Meta BSP).

## Setup Order (new installation)

| Step | Skill | What |
|------|-------|------|
| 1 | foundation/ | Docker + Gateway setup |
| 2 | security/ | Hardening BEFORE anything else |
| 3 | office-structure/ | First office + agents |
| 4 | channels/ | Connect messaging platforms |
| 5 | models/ | LLM configuration + failover |
| 6 | workflows/ | TDD + webhooks + n8n |
| 7 | monitoring/ | Dashboard + cost tracking |
| 8 | operations/ | Backup + daily management |
| 9 | scaling/ | Only when proven necessary |

## Routing Table

| Question about... | Go to |
|-------------------|-------|
| Docker, install, gateway, env vars | foundation/ |
| Hardening, permissions, incidents, SEAL | security/ |
| Workspace files, agents, routing, roles | office-structure/ |
| Telegram, Discord, WhatsApp, Slack | channels/ |
| LLM models, failover, costs, budget | models/ |
| TDD, webhooks, n8n, cron, automation | workflows/ |
| Backup, restore, upgrade, daily ops | operations/ |
| Kubernetes, multi-gateway, scaling | scaling/ |
| Dashboard, Mission Control, token costs | monitoring/ |

## Terminology

See `references/ontology.md` for canonical definitions of:
Gateway, Workspace, Agent, Session, Channel, Binding, Entrypoint, Skill, Heartbeat, Cron Job.

## Optional Standalone Skills

- `skill-execution-method/` — choosing between parallel, teams, or hybrid approach
- `skill-distribution/` — publishing skills via GitHub or ClawHub
