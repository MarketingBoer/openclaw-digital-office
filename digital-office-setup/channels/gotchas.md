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

---

## 6. Quick Tunnel URL Changes on Every Restart

**What happens:** After restarting a cloudflared quick tunnel, the WhatsApp
webhook stops receiving messages. ChakraHQ still shows the old URL. No error
in OpenClaw logs — messages simply never arrive.

**Why:** Cloudflared quick tunnels (`cloudflared tunnel --url`) generate a
random subdomain on every start (e.g., `food-therapeutic-bottom-valid.trycloudflare.com`).
When the tunnel restarts, the URL changes silently. ChakraHQ and openclaw.json
still point to the old URL.

**Fix — immediate (after each restart):**
1. Get the new URL: `curl -s http://127.0.0.1:20242/metrics | grep userHostname`
2. Update ChakraHQ: app.chakrahq.com → WhatsApp Setup → Pass-through webhook URL
3. Update `webhookUrl` in openclaw.json → restart gateway

**Fix — permanent:** Set up a named cloudflared tunnel with a stable subdomain:
```bash
cloudflared tunnel login          # browser auth — one-time
cloudflared tunnel create openclaw
cloudflared tunnel route dns openclaw webhook.yourdomain.com
```
Then configure the fixed URL in ChakraHQ once. No more manual updates.

---

## 7. Telegram Multi-Bot Polling Conflict

**What happens:** After migrating OpenClaw to a new machine, Telegram messages
arrive intermittently or not at all. Some messages reach the old instance,
some reach the new one, some are lost entirely.

**Why:** Telegram long-polling is exclusive — only one client can poll a bot
token at a time. If two containers (e.g., VPS and local) poll the same bot
token simultaneously, Telegram alternates between them unpredictably.

**Fix:** Before starting OpenClaw on the new machine, STOP the container on the
old machine:
```bash
ssh user@old-server 'docker compose -f /path/to/docker-compose.yml stop openclaw'
```
Only then start the local container. Never run two instances polling the same
bot token. This applies per bot — if you have multiple bot accounts (default + hq),
each token must be polled by exactly one instance.
