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

> ⚠️ **Cron jobs are NOT configured in openclaw.json.** They are managed
> exclusively via the `openclaw cron` CLI. JSON config blocks for cron
> do NOT work — the schema rejects unknown keys like `"crons"` or `"cron.jobs"`.

### Create a cron job

```bash
openclaw cron add \
  --name "daily-hq-status" \
  --agent hq-mediadeboer \
  --cron "0 8 * * 1-5" \
  --message "Give a short daily status overview of active workspaces and open actions." \
  --announce \
  --channel telegram \
  --account hq \
  --description "Weekday morning status report"
```

**Key flags:**
- `--cron <expr>` — 5-field crontab expression (e.g., `"0 8 * * 1-5"` = weekdays 08:00)
- `--every <duration>` — alternative to `--cron` (e.g., `1h`, `30m`)
- `--at <when>` — one-shot run at ISO time or `+duration` (e.g., `+20m`)
- `--agent <id>` — target agent ID
- `--message <text>` — prompt sent to the agent
- `--announce` — deliver result summary to a channel (replaces deprecated `--deliver`)
- `--channel <name>` — delivery channel (`telegram`, `discord`, etc.)
- `--account <id>` — channel account ID for multi-bot setups
- `--session <target>` — `main` (shared) or `isolated` (fresh per run)
- `--timeout-seconds <n>` — max run time before timeout
- `--tz <iana>` — timezone for cron expression (e.g., `Europe/Amsterdam`)

### Manage cron jobs

```bash
openclaw cron list                    # list all jobs
openclaw cron run <id>                # run immediately (debug)
openclaw cron runs <id>               # show run history
openclaw cron disable <id>            # pause a job
openclaw cron enable <id>             # resume a job
openclaw cron edit <id> --cron "..."  # patch fields
openclaw cron rm <id>                 # delete permanently
openclaw cron status                  # scheduler status
```

**Report storage convention:** `reports/seo/{YYYY-MM-DD}-{project}.md` in the agent workspace.
