---
name: digital-office-structure
description: |
  Use when creating a new office, workspace, or agent configuration.
  Triggers: "add office", "new workspace", "create agent",
  "project structure", "routing setup", "agent roles".
---

# Skill: Digital Office Structure

Tested against: OpenClaw v2026.3.x

This skill owns: Workspace files, Agent configuration, Binding hierarchy, Session management, sub-agent controls. For Channel per-platform details → see channels/. For model failover → see models/.

---

## 1. Workspace Files

Every agent has exactly one workspace directory. These files are loaded at session start (each session = one conversation thread). All files are optional except SOUL.md and AGENTS.md.

| File | Loaded | Purpose |
|------|--------|---------|
| SOUL.md | Every session | Persona, tone, personality, behavioral boundaries |
| AGENTS.md | Every session | Operational instructions: memory rules, safety, priorities |
| USER.md | Every session | Owner context: name, timezone, preferences |
| IDENTITY.md | Every session | Agent name, theme, emoji, avatar |
| TOOLS.md | Every session | Tool usage guidance (does NOT control availability — see security/) |
| HEARTBEAT.md | Heartbeat runs only | Lightweight periodic checklist — keep this SHORT |
| BOOT.md | Gateway restart | Optional startup checklist — runs every restart |
| BOOTSTRAP.md | First run only | One-time setup ritual — **DELETE after completion** |
| MEMORY.md | Main/private sessions | Curated long-term memory |
| memory/YYYY-MM-DD.md | Session start | Daily memory logs (auto-created by compaction) |
| skills/ | Session start | Workspace-specific skills (highest precedence) |

### Bootstrap Size Limits

- `bootstrapMaxChars`: 20,000 characters per file
- `bootstrapTotalMaxChars`: 150,000 characters total across all workspace files

Files exceeding the total limit are silently truncated. See gotchas.md for implications.

### Skills Precedence (3 levels)

1. `{workspace}/skills/` — highest priority, workspace-specific
2. `~/.openclaw/skills/` — global/managed
3. Bundled skills — shipped with OpenClaw (lowest)

Name conflicts resolve by precedence. A workspace skill overrides a global skill with the same name.

### To add a new agent

```
openclaw agents add {name}
```

The wizard creates the workspace directory and scaffolds the standard files.

---

## 2. Agent Roles

See references/agent-roles.md for full config examples. Three canonical roles per office:

### Operations Agent
- Purpose: day-to-day project work, content, client communication
- Model: Sonnet primary, Opus fallback (see models/ for failover config)
- Tools: read/write workspace, browser, web_fetch

### QA Auditor
- Purpose: SEO audits, SEA audits, reporting — observe and analyze only
- Model: Haiku primary, Sonnet fallback
- Tools: **read-only workspace, web_fetch — NO exec, NO write**
- Critical: must have exec tools in deny-list (see security/)

### Cron Worker (optional)
- Purpose: scheduled tasks, nightly jobs, bulk processing
- Model: cheapest available tier (Haiku or local Ollama)
- Tools: task-specific, minimal permissions
- See workflows/ for cron job configuration

---

## 3. Routing Hierarchy

When a message arrives, the gateway resolves the target agent by this hierarchy (first match wins):

```
peer → parentPeer → guildId → accountId → channel → fallback
```

- **peer**: direct match on sender ID (most specific)
- **parentPeer**: parent thread or conversation ID
- **guildId**: Discord server, Slack workspace
- **accountId**: bot account (for multi-bot setups)
- **channel**: channel-wide default (e.g., all Telegram DMs to this bot)
- **fallback**: gateway-level default agent

### Agent Visibility in Chat (user perspective)

The user sees one interface. Routing happens behind the scenes.

| Platform | User Experience | Backend Routing |
|----------|----------------|-----------------|
| Telegram | 1 bot per accountId | Routing selects agent; user sees same bot handle |
| Discord | /agents shows available agents, /focus {agent} switches | Thread bindings for per-channel isolation |
| WhatsApp | 1 number = 1 bot | allowlist + group config determines agent |
| Slack | 1 app | Binding per channel or workspace |

For per-channel routing details → see channels/.

---

## 4. Session Management

Sessions are scoped conversation threads. Config lives in `agents.defaults.sessions`:

```json
{
  "sessions": {
    "dmScope": "per-channel-peer",
    "reset": {
      "daily": { "atHour": 4 },
      "idle": { "idleMinutes": 120 }
    }
  }
}
```

- `dmScope: "per-channel-peer"` — recommended for multi-user inboxes. Each (channel, user) pair gets its own session. Without this, all users share one session.
- Reset policy: daily (at 04:00) + idle (120 min) — whichever triggers first resets the session.
- `resetByChannel` / `resetByType`: per-channel or per-type overrides.

---

## 5. Sub-Agent Configuration

Controls how agents spawn child agents. Configure under `agents.defaults.subagents`:

| Key | What it controls |
|-----|-----------------|
| `maxConcurrent` | Max simultaneous sub-agent sessions at one time |
| `maxSpawnDepth` | Nesting depth: 1 = no sub-sub-agents, 2 = one level deep |
| `maxChildrenPerAgent` | Max total children any single agent can spawn |
| `runTimeoutSeconds` | Max runtime per sub-agent run before forced termination |
| `archiveAfterMinutes` | Auto-archive after inactivity (default: 60) |

**Cost control**: set `maxConcurrent` and `maxChildrenPerAgent` low. Set sub-agent model to cheap tier via per-agent model override (see models/).

**Context safety**: `archiveAfterMinutes=60` can archive a long-running sub-agent mid-task. Increase for research tasks, decrease for quick classifiers.

---

## 6. Additional Workspace Directories

- `projects/` — project-specific data (not loaded at session start)
- `reports/` — output files from audits and cron jobs

---

## Cross-References

- Channel routing per platform → channels/SKILL.md
- Model config and failover → models/SKILL.md
- Tool deny-lists and read-only config → security/SKILL.md
- Cron job scheduling → workflows/SKILL.md
- Backup of workspace files → operations/SKILL.md
