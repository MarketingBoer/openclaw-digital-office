# Channels — Gotchas

Real pitfalls when connecting messaging platforms to OpenClaw.

---

## 1. Telegram Privacy Mode is ON by Default

**What happens:** Bot added to a group chat only responds to /commands and
@mentions. Regular messages are invisible to the bot.

**Why:** Telegram's Privacy Mode is enabled by default for all bots. In this
mode, bots only receive messages that start with / or contain @botname.

**Fix:** @BotFather → /setprivacy → Disable. Then **re-add the bot to the
group** — the setting change only takes effect for new group joins.
Alternative: add the bot as group admin (sees all messages automatically).

---

## 2. Discord Message Content Intent Must Be Enabled

**What happens:** Bot receives message events but the content field is empty.
It looks like the messages have no text.

**Why:** Discord's Message Content Intent is a Privileged Gateway Intent that
must be explicitly enabled. Without it, Discord strips message content from
non-command events as a privacy measure.

**Fix:** Developer Portal → select your app → Bot tab → Privileged Gateway
Intents → enable "Message Content Intent". Also enable "Server Members Intent"
if you use role-based allowlists.

---

## 3. WhatsApp Baileys Ban is Permanent

**What happens:** WhatsApp account is banned. No warning, no appeal.
The phone number and WhatsApp account are permanently lost.

**Why:** Meta detects unofficial API usage (Baileys reverse-engineers the
WhatsApp Web protocol). Detection can happen at any time.

**Fix:** Prevention only — always use a separate, disposable phone number.
Never use your personal number. After a ban: get a new number, re-scan QR.
For zero ban risk: use Chakra (official Meta BSP) instead.

---

## 4. ChakraHQ "Chakra Webhooks (Beta)" is Incompatible

**What happens:** After enabling Chakra Webhooks Beta in the ChakraHQ
dashboard, messages stop arriving at OpenClaw. No errors in ChakraHQ logs.

**Why:** "Chakra Webhooks (Beta)" uses a different payload format than the
"Pass Through webhook". OpenClaw expects the raw Meta webhook payload —
the Beta format wraps it differently and is NOT compatible.

**Fix:** In ChakraHQ dashboard: ensure "Chakra Webhooks (Beta)" is OFF.
Only use "Pass Through webhook" for OpenClaw integration.

---

## 5. Facebook Messenger is NOT Supported

**What happens:** Users search for Facebook/Messenger setup docs and find
nothing. Community workarounds exist but require full custom bridge development.

**Why:** GitHub Issue #4375 was closed as "not planned" (Feb 2026). Two
community PRs (#6145, #17157) were also closed — maintainers prefer a
third-party plugin approach.

**Fix:** Accept that this requires a fully custom webhook bridge (e.g., via
n8n). There is no plug-and-play solution. Only attempt if Facebook Messenger
is a hard business requirement.
