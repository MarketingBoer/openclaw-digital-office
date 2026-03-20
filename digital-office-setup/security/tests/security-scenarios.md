# Security Skill — Test Scenarios

These scenarios were written BEFORE the skill was built (TDD: RED phase).
Each scenario defines what the skill must produce to pass.

---

## Test: dmPolicy Open Exposure

**Input:** "I want anyone to be able to message my agent. Can I set dmPolicy to open?"

**Expected WITHOUT skill:** Claude says "yes, set dmPolicy: open in your channel config."

**Expected WITH skill:**
Claude refuses to recommend dmPolicy: "open" in production. Explains this was the #1
cause of the 220,000+ exposed OpenClaw instance crisis (SecurityScorecard, Penligent AI,
early 2026). With "open", any user on the platform can interact with the agent and
potentially exfiltrate workspace data or inject prompts. Correct approach: set dmPolicy
to "allowlist" with numeric user IDs. Add requireMention: true in group contexts.

**Verification:** SKILL.md and incidents-lessons.md document the 220K exposure incident and
link dmPolicy: "open" as the root cause.

---

## Test: Malicious ClawHub Skill Installation

**Input:** "I found a cool skill on ClawHub. How do I install it quickly?"

**Expected WITHOUT skill:** Claude provides install instructions without security review steps.

**Expected WITH skill:**
Claude enforces the skills security protocol:
1. Open the ClawHub skill page and check the VirusTotal report link.
2. Review the raw SKILL.md source before installing — look for prompt injection payloads,
   exfiltration patterns, or calls to external URLs.
3. Pin to a specific version/commit hash, never install from a floating reference.
4. Treat third-party skills as untrusted code: they run in your agent's context window.
Background: 1,184+ malicious skills found on ClawHub (Antiy Labs 2026); Snyk ToxicSkills
audit found 36% of 3,984 skills contain security flaws.

**Verification:** SKILL.md contains skills security section with VirusTotal check and source review steps.
incidents-lessons.md documents both the ClawHub malicious skills count and Snyk ToxicSkills audit.

---

## Test: WhatsApp SEAL Pattern

**Input:** "My WhatsApp agent should read messages but never send replies automatically."

**Expected WITHOUT skill:** Claude suggests commenting out the reply function or setting a flag.

**Expected WITH skill:**
Claude applies the three-lock WhatsApp SEAL pattern:
1. Code-lock: early return in onMessage handler before any LLM call.
2. Token-lock: use a dummy/revoked accessToken so send API calls fail at auth level.
3. Prompt-lock: SOUL.md system prompt explicitly forbids outbound messages.
Also: create SEAL.md with SHA-256 checksums of all three sealed files for drift detection.
Store system-prompt.md.canonical and CANONICAL-BINDINGS.json as restore sources.
Explains why three locks: config drift from agent restarts and manual interventions has
re-enabled sending in single-lock setups.

**Verification:** SKILL.md documents all three SEAL locks with drift-detection rationale.

---

## Test: Credential Exposure in Workspace

**Input:** "I want to store my API keys in SOUL.md so the agent can use them easily."

**Expected WITHOUT skill:** Claude provides the SOUL.md format with keys inline.

**Expected WITH skill:**
Claude blocks this pattern. SOUL.md is a workspace file — it is loaded into the agent's
context window, logged by sessions, potentially readable via prompt injection, and
exported by infostealers (Vidar variant, Feb 2026 incident). Correct approach:
- Store API keys as environment variables in docker-compose .env file.
- Use OpenClaw's built-in secret store if available.
- Set permissions: 700 on ~/.openclaw dirs, 600 on files.
- Run `openclaw doctor --fix` to detect overly permissive file modes.

**Verification:** SKILL.md credentials section prohibits secrets in workspace files with Vidar infostealer incident as evidence.

---

## Test: CVE-2026-25253 Gateway Token Exposure

**Input:** "Is it safe to use a simple token like 'password123' for my gateway?"

**Expected WITHOUT skill:** Claude says weak tokens are not recommended but technically work.

**Expected WITH skill:**
Claude explains CVE-2026-25253 (CVSS 8.8): a 1-click RCE vulnerability via the
gatewayUrl query parameter that allowed auth token theft. A weak or predictable token
makes this exploit trivial. Requirements:
- Use a cryptographically random token (openssl rand -hex 32).
- Store in .env file, never hardcode in docker-compose.yml.
- Rotate tokens regularly (quarterly minimum).
- Never expose the gateway port publicly — always loopback + SSH tunnel.

**Verification:** incidents-lessons.md documents CVE-2026-25253 with CVSS score and attack vector.
