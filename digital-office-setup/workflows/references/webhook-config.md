<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->
# Reference: Webhook Configuration

Source: docs.openclaw.ai/automation/webhook

---

## Hooks Config Block (openclaw.json)

```json
{
  "hooks": {
    "enabled": true,
    "token": "${OPENCLAW_HOOKS_TOKEN}",
    "path": "/hooks",
    "defaultSessionKey": "hook:ingress",
    "allowRequestSessionKey": false,
    "allowedSessionKeyPrefixes": ["hook:"],
    "allowedAgentIds": ["hooks", "main"]
  }
}
```

**Field explanations:**
- `token` — must reference an env var, never hardcode
- `defaultSessionKey` — session used when no sessionKey is specified in payload
- `allowRequestSessionKey` — when `false` (default), callers cannot override the sessionKey (prevents session hijack)
- `allowedSessionKeyPrefixes` — limits which session keys callers may request (only active when `allowRequestSessionKey: true`)
- `allowedAgentIds` — restricts which agents may be targeted via webhook

---

## Authentication

**Method:** Bearer token in Authorization header

```http
POST /hooks HTTP/1.1
Authorization: Bearer {OPENCLAW_HOOKS_TOKEN}
Content-Type: application/json
```

**Rejected:** query-string auth (`?token=...`) — the gateway does not accept this form.

---

## Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /hooks` | Send message to agent (full prompt + delivery) |
| `POST /hooks/wake` | Nudge agent without full prompt (wakeMode: now \| next-heartbeat) |

---

## Payload Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `message` | YES | string | The prompt/instruction to send to the agent |
| `name` | no | string | Display name for the incoming message |
| `agentId` | no | string | Target agent ID (overrides defaultAgent) |
| `sessionKey` | no | string | Session key (only if allowRequestSessionKey: true) |
| `wakeMode` | no | `"now"` \| `"next-heartbeat"` | Used for /hooks/wake endpoint |
| `deliver` | no | boolean | Whether to deliver response to a channel |
| `channel` | no | string | Channel type for delivery (e.g., `"telegram"`) |
| `to` | no | string | Recipient ID in the target channel |
| `model` | no | string | Override the default model for this request |
| `thinking` | no | boolean | Enable extended thinking for this request |
| `timeoutSeconds` | no | number | Max time to wait for agent response |

---

## Mappings, Presets, and Transforms

**Presets** — named payload templates for common sources:

```json
{
  "hooks": {
    "presets": {
      "gmail": {
        "agentId": "mail-agent",
        "sessionKey": "hook:gmail"
      }
    }
  }
}
```

**Mappings** — conditional routing based on payload content:

```json
{
  "hooks": {
    "mappings": [
      {
        "match": { "name": "github" },
        "action": { "agentId": "dev-agent", "sessionKey": "hook:github" }
      }
    ]
  }
}
```

**TransformsDir** — directory of JS/TS modules for complex payload transformations:

```json
{
  "hooks": {
    "transformsDir": "./hooks-transforms/"
  }
}
```

---

## Cron Job Config (Audit Automation)

```json
{
  "cron": {
    "enabled": true,
    "maxConcurrentRuns": 2,
    "sessionRetention": "24h",
    "jobs": {
      "daily-seo-scan": {
        "schedule": "0 6 * * 1-5",
        "agent": "qa-auditor",
        "prompt": "Run SEO audit for all active projects, save reports to reports/seo/",
        "deliver": true,
        "channel": "telegram"
      }
    }
  }
}
```

**Fields:**
- `schedule` — crontab expression (e.g., `"0 6 * * 1-5"` = weekdays at 06:00)
- `maxConcurrentRuns` — max simultaneous runs of this job (prevents overlap)
- `sessionRetention` — how long completed cron sessions are retained before pruning
- `deliver` + `channel` — send result to a messaging channel after completion

**Report storage convention:** `reports/seo/{YYYY-MM-DD}-{project}.md` in the agent workspace.
