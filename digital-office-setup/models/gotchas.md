# Models — Gotchas

Real pitfalls with LLM model configuration in OpenClaw.

---

## 1. No Native Spend Cap — Budget Overruns Are Silent

**What happens:** Monthly API costs exceed budget with no warning. OpenClaw
has no built-in spending limit or alert system.

**Why:** Budget governance is intentionally left to the operator. OpenClaw
manages model routing, not billing.

**Fix:** Build budget monitoring yourself: cron job aggregating session data,
alerts at 50/75/90% thresholds, automatic tier downgrade at 85%.
See monitoring/ for dashboard options.

---

## 2. Heartbeat on Expensive Model Drains Budget Fast

**What happens:** Monthly costs spike unexpectedly. Investigation shows
heartbeat runs consuming a large portion of the budget.

**Why:** Heartbeat runs on every cycle (e.g., every 6 hours). If the heartbeat
agent uses the same model as the operations agent (Sonnet/Opus), each
heartbeat costs $0.10-0.50. Over a month: $15-30+ for what should be a $0-2 check.

**Fix:** Set heartbeat/cron worker model to cheapest tier via per-agent override:
`google/gemini-2.0-flash` or `anthropic/claude-haiku-4-5`. Never use the
primary model for heartbeat.

---

## 3. Billing Error Backoff is 5 HOURS

**What happens:** After a billing error (expired card, quota exceeded), the
provider is unavailable for 5 hours even after fixing the billing issue.

**Why:** OpenClaw uses a 5-hour initial backoff for billing errors to prevent
hammering the provider API. This is intentionally conservative.

**Fix:** Know that the 5h backoff exists. When fixing billing: either wait
for the backoff to expire, or restart the gateway to clear the backoff timer.
Always have a fallback model from a different provider.

---

## 4. OpenRouter :free Models Have Aggressive Rate Limits

**What happens:** Agent gets frequent 429 errors and falls back constantly.
Response quality degrades because the fallback is a weaker model.

**Why:** OpenRouter's free-tier models have much lower rate limits than paid
models. During peak hours, even basic conversations hit the limit.

**Fix:** Use :free models only for testing or very low-volume agents.
For production: DeepSeek V3.2 at ~$15/month is the cheapest proven-stable option.

---

## 5. No Model Allowlist = Agents Pick Any Model

**What happens:** Token costs spike because an agent started using Opus
for routine tasks. No one configured it — the agent "decided" Opus was better.

**Why:** Without `agents.defaults.models` allowlist, agents can select any
model the provider supports. Model selection drift happens naturally as
agents optimize for response quality.

**Fix:** Always set a model allowlist in agent defaults to restrict available
models. This prevents both cost drift and unexpected provider usage.
