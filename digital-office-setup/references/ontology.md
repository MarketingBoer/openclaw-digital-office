# Ontology — Digital Office Setup

Canonical terminology for all skills in the digital-office-setup skill-set.
Every skill MUST use these definitions consistently. When in doubt, this file is the source of truth.

---

## Terms

### Gateway
- **Definition:** The core OpenClaw server process that manages agents, channels, sessions, and LLM routing.
- **Example:** A single Docker container running `openclaw-gateway` on port 18789.
- **OpenClaw context:** One gateway = one OpenClaw instance. It cannot horizontally scale — multiple offices run as multiple agents within one gateway. For more capacity, deploy separate gateways on separate machines.

### Workspace
- **Definition:** The file-based configuration directory for a single agent, containing persona, instructions, memory, and skills.
- **Example:** `~/.openclaw/workspace-hq-mediadeboer/` with SOUL.md, AGENTS.md, USER.md, IDENTITY.md, MEMORY.md, and skills/.
- **OpenClaw context:** Each agent has exactly one workspace. Workspace files are loaded at session start. The workspace is the agent's persistent identity and operational context.

### Agent
- **Definition:** A configured AI persona within a gateway that handles conversations, executes tools, and follows workspace instructions.
- **Example:** `hq-mediadeboer` — an operations agent for a specific client office, with its own SOUL.md personality and tool permissions.
- **OpenClaw context:** Agents are defined in `openclaw.json` under `agents.list`. Each agent has a unique ID, a workspace, model settings, and routing rules. Multiple agents share one gateway.

### Session
- **Definition:** A single conversation thread between a user and an agent, with its own context window, memory scope, and lifetime.
- **Example:** A Telegram DM conversation that resets daily at 04:00 or after 120 minutes idle.
- **OpenClaw context:** Sessions are scoped by `dmScope` (per-channel-peer recommended). Reset policies (`daily`, `idle`) control lifetime. Session data includes conversation history, tool results, and temporary state. Completed sessions are pruned by `sessionRetention`.

### Channel
- **Definition:** A messaging platform connector that bridges external communication services to the gateway.
- **Example:** Telegram Bot API, Discord Bot, WhatsApp via Baileys, Slack via Socket Mode.
- **OpenClaw context:** Channels are configured in `openclaw.json` under `channels.<type>`. Each channel has its own authentication (botToken, appToken), routing rules, and security policies (dmPolicy, allowlist). One gateway can serve multiple channels simultaneously.

### Binding
- **Definition:** A routing rule that maps an incoming message (from a specific channel, server, or thread) to a specific agent.
- **Example:** Discord server `#seo-channel` bound to agent `qa-auditor`, so all messages in that channel go to the SEO specialist.
- **OpenClaw context:** Bindings follow a hierarchy: peer > parentPeer > guildId > accountId > channel > fallback. Thread bindings (/focus, /unfocus) allow temporary agent switching within a conversation.

### Entrypoint
- **Definition:** The network endpoint where external systems can reach the gateway — for webhooks, API calls, or channel connections.
- **Example:** `http://localhost:18789/hooks` for webhook ingress, or `http://localhost:18789` for the gateway API.
- **OpenClaw context:** The gateway exposes a single HTTP server on OPENCLAW_GATEWAY_PORT (default 18789). Bind to 127.0.0.1 for security. External access requires a reverse proxy (Nginx) or SSH tunnel. The `/hooks` path handles webhook integrations.

### Skill
- **Definition:** A reusable instruction document (SKILL.md + references) that gives an agent specialized knowledge or procedures for a specific domain.
- **Example:** `seo-audit` skill in `~/.openclaw/skills/seo-audit/SKILL.md` that guides an agent through a complete SEO analysis workflow.
- **OpenClaw context:** Skills have three precedence levels: workspace/skills/ (highest, per-agent) > ~/.openclaw/skills/ (global) > bundled (lowest). Name conflicts resolve by precedence. Skills are documentation, not executable code — they guide agent behavior through instructions.

### Heartbeat
- **Definition:** A periodic scheduled run where an agent executes a lightweight checklist without external trigger.
- **Example:** Every 6 hours, agent checks for unread messages, pending reports, and system health.
- **OpenClaw context:** Configured per-agent via HEARTBEAT.md in the workspace. Should use the cheapest model tier (e.g., gemini-flash). Keep HEARTBEAT.md short — it runs frequently and consumes tokens each time.

### Cron Job
- **Definition:** A time-scheduled task that triggers an agent to perform a specific action on a fixed schedule (crontab syntax).
- **Example:** `"0 6 * * 1-5"` — run SEO audit every weekday at 06:00.
- **OpenClaw context:** Configured in `openclaw.json` under `cron.jobs`. Each job specifies: schedule, agent, prompt, and optional model/channel. Use `maxConcurrentRuns` to prevent overlap. Session retention controls cleanup. Cron is separate from heartbeat — cron is for specific tasks, heartbeat is for general check-ins.

---

## Usage Rules

1. **Use these terms exactly** — do not invent synonyms (e.g., "instance" for gateway, "bot" for agent, "conversation" for session).
2. **Capitalize consistently** — lowercase in running text, Title Case when referring to the concept as a proper noun (e.g., "the Gateway" vs "a gateway process").
3. **Cross-reference** — when a skill mentions a term owned by another skill, link to that skill rather than redefining.
4. **Scope boundaries:**
   - `foundation/` owns: Gateway, Entrypoint (Docker/infra level)
   - `office-structure/` owns: Workspace, Agent, Binding (configuration level)
   - `channels/` owns: Channel (per-platform details)
   - `models/` owns: model configuration, failover chains
   - `security/` owns: hardening, permissions, tool restrictions
   - `workflows/` owns: Cron Job, webhook integrations
   - `operations/` owns: backup, restore, upgrades, Skill management
   - `scaling/` owns: multi-gateway, Kubernetes
   - `monitoring/` owns: Heartbeat monitoring, dashboards, cost tracking
