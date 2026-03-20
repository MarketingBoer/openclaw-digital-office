<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->

# Reference: Agent Roles

Canonical role definitions with config examples. Owner skill: office-structure/.

---

## Operations Agent

**Purpose:** Day-to-day project work, content creation, client communication, general assistant tasks.

**Model:** Sonnet primary → Opus fallback (for complex strategy)

**Tool profile:** Full workspace access, browser, web_fetch

```json
{
  "id": "{office}-ops",
  "workspace": "workspace-{office}/",
  "model": {
    "primary": "anthropic/claude-sonnet-4-5",
    "fallbacks": ["anthropic/claude-opus-4-5"]
  },
  "tools": {
    "allow": ["read", "write", "edit", "browser", "web_fetch", "nodes", "cron"]
  },
  "sessions": {
    "dmScope": "per-channel-peer"
  }
}
```

**Workspace files focus:**
- SOUL.md: define communication style, tone, domain expertise
- AGENTS.md: memory rules, escalation paths, safety boundaries
- USER.md: owner name, timezone, preferences, key contacts

---

## QA Auditor

**Purpose:** SEO/SEA audits, quality reviews, reporting. Observe and analyze only — no writes.

**Model:** Haiku primary → Sonnet fallback

**Tool profile:** Read-only workspace, web_fetch — NO exec, NO write, NO edit

```json
{
  "id": "{office}-qa",
  "workspace": "workspace-{office}-qa/",
  "model": {
    "primary": "anthropic/claude-haiku-4-5",
    "fallbacks": ["anthropic/claude-sonnet-4-5"]
  },
  "tools": {
    "deny": ["write", "edit", "apply_patch", "exec", "process", "browser", "canvas", "nodes", "cron", "gateway", "image"]
  },
  "sessions": {
    "dmScope": "per-channel-peer"
  }
}
```

**Workspace files focus:**
- SOUL.md: analytical persona, precision-focused, no action-taking
- AGENTS.md: read-only mandate, escalation to ops agent for any write action
- TOOLS.md: reinforces read-only behavior at instruction level

**Critical:** The deny-list in `tools.deny` is the security control. TOOLS.md guidance alone is insufficient — an agent can ignore instructions. See security/ for hardening.

---

## Cron Worker

**Purpose:** Scheduled background tasks — nightly reports, bulk processing, data aggregation.

**Model:** Cheapest available (Haiku, google/gemini-flash, or local Ollama)

**Tool profile:** Task-specific, minimal. Typically read + write reports, web_fetch for data.

```json
{
  "id": "{office}-cron",
  "workspace": "workspace-{office}-cron/",
  "model": {
    "primary": "google/gemini-2.0-flash",
    "fallbacks": ["anthropic/claude-haiku-4-5"]
  },
  "tools": {
    "allow": ["read", "write", "web_fetch"]
  },
  "sessions": {
    "dmScope": "per-channel-peer",
    "reset": {
      "idle": { "idleMinutes": 30 }
    }
  }
}
```

**Workspace files focus:**
- HEARTBEAT.md: the primary operating instruction for scheduled runs (keep < 100 lines)
- AGENTS.md: define what output files to write, naming conventions
- BOOT.md: verify output directories exist on startup

**Cost note:** Heartbeat runs on every cycle. A single misplaced Opus call here can drain a monthly budget in days. Always verify the cron worker model is on the cheapest tier. See models/ for budget governance.

---

## Multi-Agent Office Config Example

Full `openclaw.json` agents section for a typical two-agent office (ops + qa):

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxConcurrent": 3,
        "maxSpawnDepth": 2,
        "maxChildrenPerAgent": 5,
        "runTimeoutSeconds": 300,
        "archiveAfterMinutes": 60
      },
      "sessions": {
        "dmScope": "per-channel-peer",
        "reset": {
          "daily": { "atHour": 4 },
          "idle": { "idleMinutes": 120 }
        }
      }
    },
    "list": [
      {
        "id": "{office}-ops",
        "workspace": "workspace-{office}/",
        "model": {
          "primary": "anthropic/claude-sonnet-4-5",
          "fallbacks": ["anthropic/claude-opus-4-5"]
        },
        "tools": {
          "allow": ["read", "write", "edit", "browser", "web_fetch", "nodes"]
        }
      },
      {
        "id": "{office}-qa",
        "workspace": "workspace-{office}-qa/",
        "model": {
          "primary": "anthropic/claude-haiku-4-5",
          "fallbacks": ["anthropic/claude-sonnet-4-5"]
        },
        "tools": {
          "deny": ["write", "edit", "apply_patch", "exec", "process", "browser", "canvas", "nodes", "cron", "gateway", "image"]
        }
      }
    ]
  }
}
```

Replace `{office}` with your office identifier (e.g., `acme`, `mediadeboer`).
