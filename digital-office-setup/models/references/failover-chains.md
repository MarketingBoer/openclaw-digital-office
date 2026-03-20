# Failover Chain Configuration

<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->

## Basic Failover Config

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5",
        "fallbacks": ["anthropic/claude-haiku-4-5", "openrouter/deepseek/deepseek-chat-v3"]
      }
    }
  }
}
```

Per-agent override (in `agents.list`):
```json
{
  "id": "{office}-ops",
  "model": {
    "primary": "anthropic/claude-sonnet-4-5",
    "fallbacks": ["anthropic/claude-opus-4-5"]
  }
}
```

## Layer 1: Auth Profile Rotation

When a provider returns an error, the gateway rotates through auth profiles
before falling back to another model.

**Rate limit (429):**
- Exponential backoff: 60s → 120s → 240s → 480s → 960s (max)
- After max backoff: try next auth profile for same provider
- When all profiles exhausted: move to Layer 2

**Billing error:**
- Initial backoff: 5 hours (not minutes — intentionally long to prevent hammering)
- After backoff: retry once, then move to next profile/model

**Key rotation priority:**
```
OPENCLAW_LIVE_ANTHROPIC_KEY     (highest — hot-swappable without restart)
ANTHROPIC_API_KEYS              (comma-separated list — rotated in order)
ANTHROPIC_API_KEY               (single key — lowest priority)
```

Same pattern for OPENAI_, OPENROUTER_, etc.

## Layer 2: Model Fallback

When all auth profiles for the current model are exhausted:

1. Gateway picks next model from `fallbacks[]` array
2. Applies Layer 1 rotation for that model's provider
3. If all fallbacks exhausted: session fails with provider error

**Cross-provider example:**
```json
{
  "model": {
    "primary": "anthropic/claude-sonnet-4-5",
    "fallbacks": [
      "anthropic/claude-haiku-4-5",
      "openrouter/deepseek/deepseek-chat-v3",
      "ollama/qwen3.5"
    ]
  }
}
```

Fallback order: Sonnet (Anthropic) → Haiku (Anthropic, different model) →
DeepSeek (OpenRouter, different provider) → Qwen (local Ollama, no API needed).

## Budget Governance Zones

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5",
        "fallbacks": ["anthropic/claude-haiku-4-5"]
      }
    }
  }
}
```

Budget enforcement is NOT automatic. Build via cron + alerts:

| Zone | Threshold (€120/mo) | Model Policy |
|------|---------------------|-------------|
| Normal | €0-84 (0-70%) | Standard tier routing |
| Warning | €84-102 (70-85%) | Alert, reduce non-essential calls |
| Frugal | €102-120 (85-100%) | Force cheap tier for all tasks |
| Blocked | >€120 (>100%) | Heavy tasks only with approval |

## Provider Auth Quick Reference

**Anthropic:**
```
ANTHROPIC_API_KEY=sk-ant-api03-...
```

**OpenAI:**
```
OPENAI_API_KEY=sk-...
```

**OpenRouter:**
```
OPENROUTER_API_KEY=sk-or-v1-...
```

**Ollama (local):**
No API key needed. Requires Ollama running on the host or accessible network.
```
OLLAMA_HOST=http://host.docker.internal:11434
```
