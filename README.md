# OpenClaw Digital Office — Free WhatsApp Read-Only + Complete Office Setup

Production-tested Claude Code skills for building a complete digital office with OpenClaw. From Docker to multi-agent routing to budget governance — everything you need, documented from real production experience.

**The only guide that sets up WhatsApp via the official Meta API (ChakraHQ) — free, read-only, zero ban risk.**

## Why This Exists

There are dozens of OpenClaw setup guides. They all use Baileys for WhatsApp (unofficial, ban risk, supply chain attacks). None of them document the official route.

This skill-set is different:

- **WhatsApp via official Meta API** — ChakraHQ pass-through, free tier (1K conversations/month), no ban
- **Production gotchas you won't find elsewhere** — plugin ownership cascading failures, `:latest` tag crashes, WAL corruption during backup, config validation blocking all CLI commands
- **Security hardening from real incidents** — CVE-2026-25253, 1,184+ malicious ClawHub skills, credential theft via infostealers
- **5 Mission Control variants compared** — with a proven pure CSS approach that doesn't need PixiJS
- **Budget governance** — because OpenClaw has no native spend cap and nobody tells you that

## Quick Start

```bash
# Copy skills to Claude Code
cp -r digital-office-setup/ ~/.claude/skills/digital-office-setup/

# Then just ask Claude Code:
# "Set up OpenClaw with Docker"          → foundation/
# "Configure WhatsApp officially"         → channels/
# "Harden my OpenClaw setup"             → security/
# "Set up model failover under $15/mo"   → models/
# "Show me Mission Control options"       → monitoring/
```

## What's Included

| Skill | What You Get | Key File |
|-------|-------------|----------|
| `foundation/` | Docker setup, gateway config, pre-flight checklist | `docker-compose-prod.yml` |
| `security/` | Hardening, tool deny-lists, real incident database | `incidents-lessons.md` |
| `channels/` | Telegram, Discord, WhatsApp (official + Baileys warning), Slack | `whatsapp-chakra-official.md` |
| `office-structure/` | Workspaces, agent roles, routing, multi-office | `agent-roles.md` |
| `models/` | Failover chains, free/budget tiers, OAuth vs API keys | `failover-chains.md` |
| `workflows/` | TDD, webhooks, n8n integration, cron automation | `webhook-config.md` |
| `operations/` | Backup/restore (with SQLite WAL warning), upgrades, daily ops | `backup-restore.md` |
| `monitoring/` | 5 Mission Control variants, token cost tracking, budget zones | `mission-control-options.md` |
| `scaling/` | Docker production, Kubernetes (community + official operator) | `kubernetes-helm.md` |

## Free WhatsApp Setup (The USP)

Every other guide uses Baileys. Here's why you shouldn't:

| | Baileys | ChakraHQ (this guide) |
|---|---------|----------------------|
| API | Unofficial (reverse-engineered) | Official Meta BSP |
| Ban risk | **Permanent** | None |
| Cost | Free | Free (1K/month) |
| Supply chain | [Malicious packages exist](https://github.com/nicepkg/openclaw-skills/issues/1) | Verified provider |
| Phone needed | Must stay online | Not needed |

**Setup in 4 steps:**
1. Create ChakraHQ account (free)
2. Connect WhatsApp Business number
3. Point pass-through webhook at OpenClaw
4. Messages flow in — read-only, no outbound

Full guide with gotchas: `channels/references/whatsapp-chakra-official.md`

## Recommended Setup Order

1. `foundation/` — Docker + gateway (start here)
2. `security/` — Harden before anything goes live
3. `channels/` — Connect messaging platforms
4. `office-structure/` — Create agents and workspaces
5. `models/` — Configure LLMs and failover
6. `workflows/` — Set up automation
7. `monitoring/` — Dashboards and cost tracking
8. `operations/` — Backup and maintenance routines
9. `scaling/` — Only when you actually need it

## How This Was Built

Not generated from docs — built from production experience, then verified:

| Step | What | Tool |
|------|------|------|
| Research | 2 deep search reports (docs, GitHub issues, security incidents) | Web research |
| Specification | 1162-line master prompt with exact requirements per skill | Claude Opus 4.6 |
| Risk analysis | 8 risks identified, mitigations designed, reviewed externally | Grok (xAI) |
| Build | Sequential TDD per skill (RED → GREEN → SELF-CHECK) | Claude Opus 4.6 |
| Cross-review | Consistency verification across all 9 skills | Ralph autonomous agent |
| External audit | Independent review at 4 checkpoints | Grok (xAI) |
| Validation | Tested against live OpenClaw production environment | Real-world usage |

### Quality gates per skill
- 5 TDD test scenarios written before building
- SKILL.md < 500 lines, concept-level
- Total < 15,000 characters per skill
- Shared ontology for consistent terminology
- Verified date tag on every reference file
- Reviewed by separate agent (not the builder)

## Tested Against

- OpenClaw v2026.3.x
- Docker 27+
- Ubuntu 22.04 / 24.04

## Contributing

Found something wrong? Open an issue:
1. Which skill
2. What you expected
3. What happened
4. Your OpenClaw version

PRs welcome — especially production gotchas.

## License

MIT
