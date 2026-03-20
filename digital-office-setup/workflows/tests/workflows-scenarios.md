# Test Scenarios — digital-office-workflows

## Test: configure-webhook-endpoint

**Input:** "I want n8n to trigger my OpenClaw agent when a form is submitted."

**Expected WITHOUT skill:**
Claude improvises vaguely: "You can set up a webhook endpoint in your OpenClaw config..." without knowing the exact `hooks` config block, the Bearer token auth requirement, or the specific payload fields. No mention of `allowRequestSessionKey` risk or `transformsDir`.

**Expected WITH skill:**
Claude provides the exact `hooks` config block (enabled, token from env var, path `/hooks`, defaultSessionKey, allowedSessionKeyPrefixes, allowedAgentIds), explains Bearer token auth (not query-string), lists required vs optional payload fields (message is required; agentId, sessionKey, deliver, channel, to are optional), and warns about `allowRequestSessionKey: false` security default.

**Verification:**
- Config block includes `OPENCLAW_HOOKS_TOKEN` env var reference
- Auth method identified as Bearer header, not query-string
- `message` field identified as required
- Security warning present for `allowRequestSessionKey`

---

## Test: setup-tdd-workflow

**Input:** "How do I set up TDD for my OpenClaw skill development?"

**Expected WITHOUT skill:**
Claude gives generic TDD advice (write tests, Red-Green-Refactor) but misses OpenClaw-specific workflow: OpenSpec spec-driven development, the Superpowers two-phase review, RFC 2119 keywords in specs, and the Iron Law enforcement. No mention of rationalization blocker.

**Expected WITH skill:**
Claude explains: (1) Iron Law — no production code without failing test, (2) Red-Green-Refactor cycle, (3) Superpowers two-phase review (conformity check + code quality check), (4) OpenSpec workflow (init → new change → plan → apply → verify → archive) with RFC 2119 (SHALL/MUST/SHOULD/MAY) and Given/When/Then scenarios, (5) the rationalization blocker: "skip TDD just this once" = STOP.

**Verification:**
- Iron Law stated
- Two-phase review described (both phases)
- OpenSpec init/archive cycle mentioned
- RFC 2119 keywords listed
- Anti-pattern (code before test → DELETE) described

---

## Test: connect-n8n-to-openclaw

**Input:** "How do I connect n8n to OpenClaw so my agents can trigger automations?"

**Expected WITHOUT skill:**
Claude describes generic API connection steps without knowing the Docker shared network pattern, credential isolation rule, the `openclaw-n8n-stack` reference, or the SOUL.md NEVER-store-API-keys rule.

**Expected WITH skill:**
Claude explains the architecture (OpenClaw reasoning ↔ webhook ↔ n8n execution ↔ External APIs), the Docker shared network allowing `http://n8n:5678/webhook/workflow` direct access, credential isolation (API keys ONLY in n8n encrypted store, never in OpenClaw env), SOUL.md rule, bidirectional patterns (n8n→OpenClaw, OpenClaw→n8n), and references the `openclaw-n8n-stack` pre-configured compose stack.

**Verification:**
- Architecture diagram described
- Docker network URL `http://n8n:5678/webhook/workflow` mentioned
- Credential isolation rule stated
- SOUL.md rule referenced
- Both integration patterns described

---

## Test: automate-seo-audit-with-cron

**Input:** "Set up an automated weekly SEO audit that runs every weekday morning."

**Expected WITHOUT skill:**
Claude gives generic cron advice, may suggest putting cron config in openclaw.json (which crashes the gateway), doesn't know the CLI-based workflow, or uses incorrect field names.

**Expected WITH skill:**
Claude provides the `openclaw cron add` CLI command with `--cron "0 6 * * 1-5"`, `--agent qa-auditor`, `--message` prompt, `--announce` (not deprecated `--deliver`), `--channel telegram`, and `--tz` for timezone. Warns cron jobs are NOT in openclaw.json. Mentions report convention `reports/seo/{date}-{project}.md`. Notes `gateway.mode=local` blocks CLI — workaround: write to `/data/.openclaw/cron/jobs.json` while gateway is stopped.

**Verification:**
- CLI command used (not JSON config)
- Warning: cron NOT in openclaw.json
- `--announce` used (not `--deliver`)
- Cron expression correct
- Report storage path follows convention
- gateway.mode=local workaround mentioned

---

## Test: webhook-security-review

**Input:** "Is my webhook config secure? I'm using query-string auth and allowRequestSessionKey: true."

**Expected WITHOUT skill:**
Claude gives generic "use HTTPS" advice without knowing OpenClaw-specific risks: that query-string auth is rejected by the gateway, that `allowRequestSessionKey: true` allows callers to hijack sessions, or that the hooks token must be in an env var not hardcoded.

**Expected WITH skill:**
Claude flags two critical issues: (1) query-string auth is rejected — must use Bearer header in Authorization header, (2) `allowRequestSessionKey: true` lets external callers specify any sessionKey and potentially hijack existing sessions. Also recommends token in env var (`OPENCLAW_HOOKS_TOKEN`), not hardcoded, and recommends restricting `allowedAgentIds` to only the agents that need webhook access.

**Verification:**
- Query-string auth rejection explained
- Session hijack risk for `allowRequestSessionKey: true` explained
- Env var recommendation for token
- `allowedAgentIds` restriction recommended
