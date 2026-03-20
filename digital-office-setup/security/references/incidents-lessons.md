# Known Security Incidents — OpenClaw Ecosystem

<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->

Real incidents and CVEs that inform the security guidance in this skill-set.
Each entry includes: what happened, impact, and the lesson.

---

## 1. Mass Exposure: 220,000+ Instances (early 2026)

**What:** SecurityScorecard initially found 40,214 exposed OpenClaw instances.
Penligent AI expanded this to 135,000+, later growing to 220,000+.
Most had default configurations with open ports and no authentication.

**Impact:** Agent workspace data, conversation history, and API keys accessible
to anyone with the Gateway URL.

**Lesson:** Always bind to loopback (127.0.0.1), set a strong Gateway token,
and never use dmPolicy: "open" in production.

---

## 2. CVE-2026-25253 — 1-Click RCE via gatewayUrl (CVSS 8.8)

**What:** A vulnerability in the Gateway's web interface allowed auth token theft
via the `gatewayUrl` query parameter. An attacker could craft a URL that, when
clicked by an authenticated user, would leak the Gateway token to an
attacker-controlled server.

**Impact:** With the stolen token, the attacker gains full Gateway access:
read/write workspace, execute commands, access all channels.

**Lesson:** Use strong random tokens (`openssl rand -hex 32`), rotate quarterly,
never expose the Gateway port publicly.

---

## 3. Malicious ClawHub Skills (1,184+ detected)

**What:** Koi Security initially found 341 malicious skills in the ClawHub
registry (Jan 2026). Antiy Labs expanded this to 1,184+ by Mar 2026.
Payloads included: prompt injection, credential exfiltration, reverse shells,
and persistent backdoors.

**Snyk ToxicSkills audit:** 36% of 3,984 audited skills contained security
flaws. 1,467 contained clearly malicious payloads.

**Impact:** Skills run in the agent's context window with full access to
workspace files, conversation history, and tool permissions.

**Lesson:** Always review source code before installing. Check VirusTotal.
Pin versions. Treat third-party skills as untrusted code.

---

## 4. Moltbook Database Exposure (1.5M tokens + 35K emails)

**What:** A misconfigured Supabase instance exposed 1.5 million API tokens and
35,000 email addresses from a major OpenClaw hosting provider. Discovered by
Wiz researcher Gal Nagli.

**Impact:** API tokens could be used to access user Gateways. Email addresses
enabled targeted phishing.

**Lesson:** Never store credentials in shared databases without proper access
controls. Rotate any tokens that may have been exposed.

---

## 5. Cognitive Context Theft — Vidar Infostealer Variant (Feb 2026)

**What:** A Vidar infostealer variant (detected 13 Feb 2026) specifically
targets OpenClaw workspace files: SOUL.md, AGENTS.md, MEMORY.md.
These files contain the agent's personality, instructions, and long-term memory.

**Impact:** Stolen workspace files create a psychological profile usable for
social engineering. MEMORY.md may contain sensitive business context.

**Lesson:** Never store secrets in workspace files. Set strict file permissions
(700 dirs, 600 files). Monitor for unauthorized access to ~/.openclaw.

---

## 6. Supply Chain: lotusbail npm Package (56K downloads)

**What:** A malicious npm package named `lotusbail` mimicked the legitimate
`@whiskeysockets/baileys` WhatsApp library. It was a functional Baileys clone
that also exfiltrated auth tokens, messages, contacts, and media through a
persistent backdoor.

**Impact:** 56,000 downloads before discovery. Complete WhatsApp account
compromise for affected users.

**Lesson:** Always install `@whiskeysockets/baileys` from the official source.
Never use alternative Baileys packages. Verify package authenticity.

---

## 7. Prompt Injection via Inbound Messages

**What:** Attackers craft messages containing prompt injection payloads sent
through channels (email forwarding, web content, chat messages). The agent
processes these as regular input, potentially following injected instructions.

**Impact:** Agent may execute unintended actions, leak workspace data, or
bypass tool restrictions if the injection overrides system prompt boundaries.

**Lesson:** SOUL.md must include explicit security boundaries. Use tool deny
lists to limit blast radius. Never grant exec permissions to public-facing agents.
