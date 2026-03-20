# Reference: Complete OpenClaw Digital Office Config Template

<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->
<!-- sources: production setup + github.com/JarvisDeLaAri/OpenClawIT, jeffs365/myagents, -->
<!--           gensecaihq/Wazuh-Openclaw-Autopilot, 0xNagato/openclaw-mcporter-agency -->
<!-- schema: https://docs.openclaw.ai/schema/openclaw.json -->

Full `openclaw.json` template for a **scalable multi-agent digital office**.
Designed to start small (Phase 1: 4 agents) and grow without refactoring (Phase 3: 15+ agents).

Replace all `{PLACEHOLDER}` values before deployment.

---

## Architectural Philosophy (Grok-validated)

```
WhatsApp  → intake only (read-only, never auto-reply)
Telegram  → one bot per agent (control + per-client)
Discord   → internal nervous system (owner + team)

Agent flow:
  WhatsApp inbound → whatsapp-review (seal, no dispatch)
  Telegram/Discord → hq-agent (orchestrator)
  hq-agent → delegates to specialist agents
  specialist → guardian (quality gate before any delivery)
```

**Never share entrypoints between agents. One bot = one agent.**

---

## Phase 1 — Start (4 agents, 1 person, 3-5 clients)

Minimum viable office. You are the human governor.

| Agent | Role | Model | Telegram Bot | Tools |
|---|---|---|---|---|
| `hq-{office}` | Orchestrator / HQ | Sonnet | `@hq_bot` | read/write/cron |
| `client-{name}` | Delivery per client | Sonnet | `@client_bot` | read/write/web_fetch |
| `qa-{office}` | QA auditor (read-only) | Haiku | `@qa_bot` | read-only (deny-list) |
| `whatsapp-review` | WhatsApp intake | DeepSeek | _(no Telegram)_ | sealed |

Add **Guardian** in Phase 2 when you have 3+ active delivery agents.

---

## Full Template

```json
{
  "$schema": "https://docs.openclaw.ai/schema/openclaw.json",

  "meta": {
    "lastTouchedVersion": "2026.3.x"
  },

  "plugins": {
    "allow": [
      "telegram",
      "discord",
      "openclaw-whatsapp-cloud-api"
    ],
    "entries": {
      "telegram": { "enabled": true },
      "discord":  { "enabled": true },
      "openclaw-whatsapp-cloud-api": { "enabled": true }
    }
  },

  "channels": {

    "telegram": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "groupPolicy": "allowlist",
      "streaming": "partial",
      "botToken": "{TELEGRAM_DEFAULT_BOT_TOKEN}",
      "accounts": {
        "default": {
          "dmPolicy": "allowlist",
          "botToken": "{TELEGRAM_DEFAULT_BOT_TOKEN}",
          "allowFrom": ["{OWNER_TELEGRAM_USER_ID}"],
          "groupPolicy": "allowlist",
          "streaming": "partial"
        },
        "hq": {
          "name": "{HQ_BOT_NAME}",
          "enabled": true,
          "dmPolicy": "allowlist",
          "botToken": "{TELEGRAM_HQ_BOT_TOKEN}",
          "allowFrom": ["{OWNER_TELEGRAM_USER_ID}"],
          "groupPolicy": "allowlist",
          "streaming": "partial"
        },
        "qa": {
          "name": "{QA_BOT_NAME}",
          "enabled": true,
          "dmPolicy": "allowlist",
          "botToken": "{TELEGRAM_QA_BOT_TOKEN}",
          "allowFrom": ["{OWNER_TELEGRAM_USER_ID}"],
          "groupPolicy": "allowlist",
          "streaming": "partial"
        },
        "client-{name}": {
          "name": "{Client Bot Name}",
          "enabled": true,
          "dmPolicy": "allowlist",
          "botToken": "{TELEGRAM_CLIENT_BOT_TOKEN}",
          "allowFrom": ["{OWNER_TELEGRAM_USER_ID}"],
          "groupPolicy": "allowlist",
          "streaming": "partial"
        }
      }
    },

    "discord": {
      "enabled": true,
      "token": "{DISCORD_DEFAULT_BOT_TOKEN}",
      "dmPolicy": "allowlist",
      "groupPolicy": "allowlist",
      "streaming": "off",
      "allowFrom": ["{OWNER_DISCORD_USER_ID}"],
      "guilds": {
        "{DISCORD_SERVER_ID}": {
          "requireMention": false,
          "users": ["{OWNER_DISCORD_USER_ID}"]
        }
      },
      "accounts": {
        "hq": {
          "name": "{HQ_DISCORD_BOT_NAME}",
          "token": "{DISCORD_HQ_BOT_TOKEN}",
          "groupPolicy": "allowlist",
          "streaming": "off",
          "guilds": {
            "{DISCORD_SERVER_ID}": {
              "users": ["{OWNER_DISCORD_USER_ID}"],
              "channels": {
                "{DISCORD_HQ_CHANNEL_ID}": {
                  "allow": true,
                  "requireMention": false
                }
              }
            }
          }
        }
      }
    },

    "whatsapp-cloud": {
      "phoneNumberId": "{WHATSAPP_PHONE_NUMBER_ID}",
      "businessAccountId": "{WHATSAPP_BUSINESS_ACCOUNT_ID}",
      "appId": "{WHATSAPP_APP_ID}",
      "accessToken": "DISABLED_READ_ONLY_MODE",
      "verifyToken": "{WHATSAPP_VERIFY_TOKEN}",
      "webhookUrl": "{CLOUDFLARED_TUNNEL_URL}/webhook/whatsapp-cloud",
      "webhookPort": 3100,
      "dmPolicy": "allowlist",
      "allowFrom": ["*"],
      "sendReadReceipts": true,
      "enabled": true
    }

  },

  "agents": {
    "defaults": {
      "model": {
        "primary": "deepseek/deepseek-chat",
        "fallbacks": [
          "google/gemini-2.5-flash",
          "anthropic/claude-sonnet-4-6"
        ]
      },
      "contextTokens": 500000,
      "maxConcurrent": 100,
      "memorySearch": {
        "sources": ["memory", "sessions"],
        "experimental": { "sessionMemory": true }
      },
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "6h",
        "keepLastAssistants": 3
      },
      "compaction": {
        "mode": "safeguard",
        "reserveTokensFloor": 20000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000
        }
      },
      "sandbox": { "mode": "off" }
    },

    "list": [

      {
        "id": "hq-{office}",
        "name": "HQ {Office}",
        "workspace": "/data/.openclaw/workspace-hq-{office}",
        "model": {
          "primary": "anthropic/claude-sonnet-4-6",
          "fallbacks": ["deepseek/deepseek-chat"]
        },
        "skills": ["healthcheck", "weather", "clawhub"],
        "tools": {
          "allow": ["read", "write", "edit", "web_fetch", "nodes", "cron"]
        }
      },

      {
        "id": "client-{name}",
        "name": "{Client Name} Delivery",
        "workspace": "/data/.openclaw/workspace-client-{name}",
        "model": {
          "primary": "anthropic/claude-sonnet-4-6",
          "fallbacks": ["deepseek/deepseek-chat"]
        },
        "skills": ["healthcheck", "weather"],
        "tools": {
          "allow": ["read", "write", "edit", "web_fetch"]
        }
      },

      {
        "id": "qa-{office}",
        "name": "{Office} QA Auditor",
        "workspace": "/data/.openclaw/workspace-qa-{office}",
        "model": {
          "primary": "anthropic/claude-haiku-4-5",
          "fallbacks": ["anthropic/claude-sonnet-4-6"]
        },
        "skills": ["healthcheck"],
        "tools": {
          "deny": [
            "write", "edit", "apply_patch", "exec",
            "process", "browser", "canvas", "nodes",
            "cron", "gateway", "image"
          ]
        }
      },

      {
        "id": "whatsapp-review",
        "name": "WhatsApp Review",
        "model": {
          "primary": "deepseek/deepseek-chat",
          "fallbacks": [
            "google/gemini-2.5-flash",
            "anthropic/claude-sonnet-4-6"
          ]
        },
        "skills": ["healthcheck", "clawhub", "weather"]
      }

    ]
  },

  "tools": {
    "exec": { "security": "allowlist" },
    "agentToAgent": {
      "enabled": true
    },
    "sessions": {
      "visibility": "all"
    }
  },

  "bindings": [

    {
      "type": "route",
      "agentId": "whatsapp-review",
      "comment": "All WhatsApp Cloud inbound → whatsapp-review (sealed read-only)",
      "match": { "channel": "whatsapp-cloud" }
    },

    {
      "type": "route",
      "agentId": "hq-{office}",
      "comment": "HQ Telegram bot → HQ agent",
      "match": {
        "channel": "telegram",
        "accountId": "hq"
      }
    },

    {
      "type": "route",
      "agentId": "qa-{office}",
      "comment": "QA Telegram bot → QA agent",
      "match": {
        "channel": "telegram",
        "accountId": "qa"
      }
    },

    {
      "type": "route",
      "agentId": "client-{name}",
      "comment": "Client Telegram bot → client delivery agent",
      "match": {
        "channel": "telegram",
        "accountId": "client-{name}"
      }
    },

    {
      "type": "route",
      "agentId": "hq-{office}",
      "comment": "Discord owner DM → HQ agent",
      "match": {
        "channel": "discord",
        "peer": {
          "kind": "direct",
          "id": "{OWNER_DISCORD_USER_ID}"
        }
      }
    },

    {
      "type": "route",
      "agentId": "hq-{office}",
      "comment": "Discord HQ channel → HQ agent",
      "match": {
        "channel": "discord",
        "accountId": "hq"
      }
    }

  ],

  "cron": {
    "enabled": true,
    "maxConcurrentRuns": 3
  },

  "session": {
    "dmScope": "per-channel-peer"
  },

  "gateway": {
    "mode": "local",
    "controlUi": {
      "dangerouslyAllowHostHeaderOriginFallback": true,
      "allowInsecureAuth": true,
      "dangerouslyDisableDeviceAuth": true
    },
    "auth": {
      "mode": "token",
      "token": "{GATEWAY_TOKEN}"
    },
    "trustedProxies": ["127.0.0.1/32"]
  },

  "update": {
    "channel": "stable",
    "checkOnStart": false
  },

  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "bash": true,
    "restart": true
  },

  "logging": {
    "redactSensitive": true
  }

}
```

---

## Cron Jobs Template (jobs.json)

> **Critical:** Always include `chatId` in delivery. Without it OpenClaw throws:
> `"Delivering to Telegram requires target <chatId>"`

```json
{
  "version": 1,
  "jobs": [
    {
      "id": "daily-hq-status",
      "name": "Daily HQ Status",
      "description": "Weekday morning status report",
      "agentId": "hq-{office}",
      "schedule": {
        "kind": "cron",
        "expr": "0 8 * * 1-5"
      },
      "tz": "Europe/Amsterdam",
      "accountId": "hq",
      "enabled": true,
      "wakeMode": "now",
      "payload": {
        "kind": "agentTurn",
        "message": "Geef een kort dagelijks statusoverzicht: actieve workspaces, openstaande acties, en of er iets aandacht nodig heeft.",
        "timeoutSeconds": 120
      },
      "sessionTarget": "isolated",
      "delivery": {
        "mode": "announce",
        "channel": "telegram",
        "accountId": "hq",
        "chatId": "{OWNER_TELEGRAM_USER_ID}"
      }
    },
    {
      "id": "weekly-budget-check",
      "name": "Weekly Budget Check",
      "description": "Monday morning cost overview",
      "agentId": "hq-{office}",
      "schedule": {
        "kind": "cron",
        "expr": "0 9 * * 1"
      },
      "tz": "Europe/Amsterdam",
      "accountId": "hq",
      "enabled": true,
      "wakeMode": "now",
      "payload": {
        "kind": "agentTurn",
        "message": "Wekelijks budget check: modelbesparing afgelopen week, token-gebruik per agent, advies: normaal / let op / actie vereist.",
        "timeoutSeconds": 120
      },
      "sessionTarget": "isolated",
      "delivery": {
        "mode": "announce",
        "channel": "telegram",
        "accountId": "hq",
        "chatId": "{OWNER_TELEGRAM_USER_ID}"
      }
    }
  ]
}
```

---

## Growth Roadmap

### Phase 1 — Start (1 person, 3-5 clients)
**4 agents:** hq + 1-2 client delivery + qa + whatsapp-review
You are the human governor. All output reviewed manually.

### Phase 2 — Growth (2-4 people, 10-25 clients)
**Add:** guardian agent (quality gate before client delivery)
**Add:** specialist agents (seo, ads, content, dev) per discipline
**Add:** `tools.agentToAgent.enabled: true` for agent orchestration
**Signal to add agent:** HQ utilization >65% over 2 weeks, or new domain consistently requested

### Phase 3 — Scale (8+ people, 40+ clients)
**Add:** workspace orchestrators per client
**Add:** department orchestrators (Creative, Strategy, Tech)
**Add:** autonomous account agents
**Structure:** hierarchical routing (client WS → department → specialist → guardian)

---

## Platform Roles

| Platform | Purpose | Who talks here |
|---|---|---|
| WhatsApp | Client intake (read-only) | Clients → whatsapp-review only |
| Telegram | Per-agent control plane | Owner ↔ each agent via own bot |
| Discord | Internal nervous system | Owner + team + agent-to-agent |

**Rule:** No agent replies to clients via any channel other than the designated intake channel.

---

## WhatsApp Read-Only SEAL — 3 Layers

```
1. Token lock:   accessToken = "DISABLED_READ_ONLY_MODE" → Meta rejects all outbound
2. Code lock:    onMessage handler has early return; before any dispatch
3. Prompt lock:  system prompt forbids replies and autonomous action
```

Never remove any layer without an explicit architectural decision. See security/SKILL.md.

---

## Placeholders Reference

| Placeholder | Source |
|---|---|
| `{TELEGRAM_*_BOT_TOKEN}` | @BotFather → /newbot |
| `{OWNER_TELEGRAM_USER_ID}` | @userinfobot — numeric ID only |
| `{DISCORD_*_BOT_TOKEN}` | discord.com/developers → Bot tab |
| `{DISCORD_SERVER_ID}` | Right-click server → Copy Server ID |
| `{DISCORD_CHANNEL_ID}` | Right-click channel → Copy Channel ID |
| `{OWNER_DISCORD_USER_ID}` | Discord Developer Mode → Copy User ID |
| `{WHATSAPP_PHONE_NUMBER_ID}` | Meta Business Suite → WhatsApp |
| `{WHATSAPP_VERIFY_TOKEN}` | Any random string (e.g. `myoffice_verify_2026`) |
| `{CLOUDFLARED_TUNNEL_URL}` | `cloudflared tunnel --url http://localhost:3100` |
| `{GATEWAY_TOKEN}` | `openssl rand -hex 32` |
| `{office}` | Your office identifier (lowercase, e.g. `acme`) |

---

## Discord: Enable Message Content Intent

**Critical:** Without this, Discord sends events with empty message bodies.

1. discord.com/developers → Your Application → Bot tab
2. Enable **Message Content Intent** (Privileged Gateway Intents)
3. Also enable: **Server Members Intent** (for role-based allowlists)

---

## Cross-References

| Topic | Skill |
|---|---|
| Docker setup, gateway | foundation/ |
| Per-channel setup (BotFather steps, Chakra) | channels/ |
| Workspace files (SOUL.md, AGENTS.md) | office-structure/ |
| Security hardening, SEAL pattern | security/ |
| Cron job scheduling | workflows/ |
| Model failover + budget governance | models/ |
| Backup and restore | operations/ |
| Mission Control dashboards | monitoring/ |
