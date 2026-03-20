# Test Scenarios — models

TDD scenarios written BEFORE building the skill.

---

## Test: Budget overrun — no native spend cap

**Input:** "I set up my OpenClaw with Opus as the primary model. After a week my Anthropic bill is $400. Is there a spending limit I can set?"

**Expected WITHOUT skill:** Claude suggests checking Anthropic's billing dashboard or adding a hard cap in openclaw.json.

**Expected WITH skill:** Claude explains there is NO native hard spend cap in OpenClaw. Budget monitoring must be built yourself: cron job that aggregates session_status data, alerts at 50/75/90% thresholds. Recommends budget governance zones (Normal 0-70%, Warning 70-85%, Frugal 85-100%, Blocked >100%). Suggests switching to model tier routing: Opus only for architecture decisions with explicit approval, Sonnet for standard work, Haiku/DeepSeek for triage. Cross-references monitoring/ for dashboard setup.

**Verification:** SKILL.md states no native spend cap, lists budget governance zones with percentages, mentions cron-based monitoring.

---

## Test: Failover chain configuration

**Input:** "My Anthropic API key hit a rate limit. How do I set up a fallback to OpenAI?"

**Expected WITHOUT skill:** Claude provides a generic API key rotation setup.

**Expected WITH skill:** Claude explains the two-layer failover system. Layer 1: auth profile rotation WITHIN the provider (exponential backoff 60s→960s for 429, 5h for billing errors). Layer 2: model fallback BETWEEN providers (primary → fallbacks array). Provides the concept: configure primary + fallbacks in agent model config. Cross-references the failover-chains.md reference for exact JSON syntax.

**Verification:** Both layers explained, backoff times mentioned, failover-chains.md referenced for config syntax.

---

## Test: Heartbeat on expensive model

**Input:** "My heartbeat runs every 6 hours. I'm using Sonnet for everything including heartbeat."

**Expected WITHOUT skill:** Claude says Sonnet is fine for heartbeat.

**Expected WITH skill:** Claude warns this is a budget drain. Heartbeat should ALWAYS use the cheapest tier (gemini-flash, Haiku, or local Ollama). A Sonnet heartbeat every 6 hours can cost $15-30/month for what should be a $0-2 check. Recommends per-agent model override for the heartbeat/cron worker. Cross-references office-structure/ for cron worker role config.

**Verification:** SKILL.md states heartbeat ALWAYS cheapest tier, gives cost comparison.

---

## Test: Which model for which task

**Input:** "I have agents doing different things — email triage, content writing, architecture decisions. Which model for each?"

**Expected WITHOUT skill:** Claude suggests Opus for everything or makes generic recommendations.

**Expected WITH skill:** Claude applies the model tier routing decision tree: (1) deterministic/template → no AI call, (2) triage/classification/bulk → cheap (DeepSeek, Haiku), (3) standard execution → standard (Sonnet), (4) architecture/strategy → heavy (Opus, approval required), (5) budget >85% → force cheap. Maps: email triage → cheap tier, content writing → standard tier, architecture → heavy with approval.

**Verification:** Decision tree present in SKILL.md with 5 tiers.

---

## Test: Free model options

**Input:** "I want to run OpenClaw for free. What are my options?"

**Expected WITHOUT skill:** Claude suggests open source models vaguely.

**Expected WITH skill:** Claude lists concrete $0/month options: (1) Ollama local with Qwen 3.5 or Llama 3.3 — requires GPU 8GB+ VRAM, (2) OpenRouter :free models — rate limited but functional. Also mentions DeepSeek V3.2 at ~$15/month as the cheapest paid option proven stable in production. Warns about OpenRouter :free rate limits being aggressive.

**Verification:** SKILL.md lists concrete free tiers with hardware requirements.
