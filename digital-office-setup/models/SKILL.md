---
name: digital-office-models
description: |
  Use when configuring LLM models, failover chains, cost optimization,
  or authentication setup. Triggers: "model setup", "failover", "reduce costs",
  "free model", "oauth setup", "api key", "which model".
---

# Digital Office Models

Tested against: OpenClaw v2026.3.x

This skill owns LLM model configuration, failover chains, cost optimization,
and budget governance. For per-agent model assignment, see **office-structure/**.
For cost dashboards, see **monitoring/**.

---

## Model Reference Format

All models use the format `provider/model`:
- `anthropic/claude-sonnet-4-5`
- `anthropic/claude-haiku-4-5`
- `google/gemini-2.0-flash`
- `openrouter/deepseek/deepseek-chat-v3`

---

## Failover Chain — Two Layers

Configure a primary model with fallbacks per agent. When the primary fails,
the gateway automatically tries the next option.

### Layer 1: Auth Profile Rotation (within provider)

On a 429 (rate limit): exponential backoff 60s → 120s → 240s → 480s → 960s max,
then try next auth profile for same provider.

On billing error: 5-hour initial backoff (not minutes — this is intentionally long).

Key rotation priority: `OPENCLAW_LIVE_<PROVIDER>_KEY` (highest) >
`<PROVIDER>_API_KEYS` (comma-separated list) > `<PROVIDER>_API_KEY` (single key).

### Layer 2: Model Fallback (between providers)

When all auth profiles for the current model are exhausted, the gateway moves
to the next model in the `fallbacks` array. This enables cross-provider resilience.

See `references/failover-chains.md` for exact JSON config syntax.

---

## Price/Quality Tiers

| Tier | Monthly Cost | Models | Use Case |
|------|-------------|--------|----------|
| Free (local) | $0 | Ollama: Qwen 3.5, Llama 3.3 | Requires GPU 8GB+ VRAM |
| Free (cloud) | $0 | OpenRouter `:free` models | Aggressive rate limits |
| Budget | ~$15 | DeepSeek V3.2 ($0.28/$0.42/MTok) | Proven stable in production |
| Professional | $50-100 | Sonnet primary + Haiku fallback | Standard office operations |
| On-demand | Variable | Opus | Complex strategy/architecture only |

**Heartbeat model:** ALWAYS use cheapest tier (gemini-flash, Haiku, or local).
Never primary model — heartbeat runs on every cycle and drains budget fast.
A Sonnet heartbeat every 6 hours can cost $15-30/month for a trivial check.

---

## Model Tier Routing — Decision Tree

For each task, choose the appropriate tier:

1. **Deterministic/template** → No AI call needed
2. **Triage/classification/bulk** → Cheap (DeepSeek, Haiku)
3. **Standard execution** → Standard (Sonnet)
4. **Architecture/strategy** → Heavy (Opus) — requires explicit approval
5. **Budget >85%** → Force cheap tier regardless of task type

---

## Budget Governance Zones

OpenClaw has **no native hard spend cap**. Budget monitoring must be built
yourself via cron jobs and alerts. See monitoring/ for dashboard setup.

| Zone | Budget Used | Action |
|------|-----------|--------|
| Normal | 0-70% | Standard tier routing |
| Warning | 70-85% | Notification, avoid duplicate calls |
| Frugal | 85-100% | Cheap-first, heavy tasks blocked |
| Blocked | >100% | Heavy only after explicit approval |

Example monthly budget: €120. Warning triggers at €84, frugal at €102.

---

## Model Allowlist

Prevent agents from picking arbitrary models by setting an allowlist:

Configure `agents.defaults.models` to restrict available models.
Without an allowlist, agents can select any model the provider supports,
leading to unexpected cost spikes.

---

## Authentication

Two approaches for provider authentication:

- **API key (recommended):** Simpler, more reliable. Store in `.env` file.
  Rotate periodically. One key per provider in env vars.
- **OAuth:** Can leverage existing subscriptions. Risk: token expiry can
  silently break the agent mid-session.

**OpenRouter as aggregator:** Single API key gives access to 100+ models
across providers. Useful for experimentation and fallback diversity.

Per-provider setup: Anthropic (ANTHROPIC_API_KEY), OpenAI (OPENAI_API_KEY),
OpenRouter (OPENROUTER_API_KEY). See foundation/ for env var details.

---

## Cross-References

| Topic | Skill |
|---|---|
| Per-agent model assignment, agent roles | office-structure/ |
| Cost dashboards, token tracking | monitoring/ |
| Environment variables for API keys | foundation/ |
| Budget alerting via cron | workflows/ |

---

## Sources

- docs.openclaw.ai/providers/anthropic
- docs.openclaw.ai/providers/openai
- docs.openclaw.ai/providers/ollama
- docs.openclaw.ai/providers/openrouter
