---
name: digital-office-channels
description: |
  Use when connecting messaging platforms to OpenClaw.
  Triggers: "connect telegram", "discord setup", "whatsapp setup",
  "chakra whatsapp", "connect facebook", "slack setup", "add channel".
---

# Digital Office Channels

Tested against: OpenClaw v2026.3.x

This skill owns per-channel setup details. For agent routing and bindings,
see **office-structure/**. For webhook integrations, see **workflows/**.

---

## Channel Comparison Matrix

| Channel | API Type | Ban Risk | Complexity | Native OpenClaw | Recommendation |
|---------|----------|----------|------------|-----------------|----------------|
| Telegram | Official (Bot API) | None | Low | YES | Always first |
| Discord | Official (Bot API) | None | Low-Medium | YES | Team/community |
| WhatsApp Baileys | Unofficial | **HIGH** | Medium | YES | Only with separate number |
| WhatsApp Chakra | Official (Meta BSP) | None | Medium-High | **NO** (custom bridge) | Business, no ban risk |
| Facebook | **NOT supported** | — | Very High | **NO** | Only if you can bridge yourself |
| Slack | Official (Bolt) | None | Medium | YES | Internal teams |

**Start with Telegram.** It has zero ban risk, official API, lowest complexity,
and native OpenClaw support. Add other channels after Telegram is working.

> **Facebook Messenger:** NO native OpenClaw support (GitHub Issue #4375,
> closed "not planned", Feb 2026). Two community PRs also closed. Only viable
> via a fully custom webhook bridge — high effort, no plug-and-play solution.

---

## Telegram

- **Setup:** @BotFather → /newbot → copy token
- **Config:** `channels.telegram.enabled`, `botToken`
- **dmPolicy:** `pairing` (setup), `allowlist` (production), `open` (**NEVER** in production)
- **Groups:** `requireMention: true` + `mentionPatterns`
- **Privacy Mode:** ON by default — bot only sees @mentions and /commands in groups
  - Fix: @BotFather → /setprivacy → Disable, then **re-add bot to group**
  - Alternative: add bot as group admin → sees all messages
- **Connection:** Long-poll (default, no public URL, 2-5s latency) vs webhook (HTTPS, instant)
- **Allowlist:** Numeric Telegram user IDs, not @usernames
- **Multiple bots:** `channels.telegram.accounts.<accountId>` with separate botToken per account
- **Proven pattern:** One bot per agent (e.g., default → whatsapp-review, hq → hq-mediadeboer)

---

## Discord

- **Setup:** Developer Portal → New Application → Bot tab → copy token
- **Message Content Intent:** **MUST be enabled** (Privileged Gateway Intents)
  - Without it: Discord sends events with empty message bodies. Looks like a bug but is a permissions issue.
  - Also enable: Server Members Intent (for role-based allowlists)
- **Config:** `channels.discord.enabled`, `botToken`
- **Routing:** Server/channel bindings — /focus, /unfocus, /agents, /session
- **Thread bindings:** Per-channel agent isolation via /focus {agent}
- **Roles:** Control which users see which agents in which channels

---

## WhatsApp — Chakra (Official Meta BSP) — Recommended

**Full details in references/whatsapp-chakra-official.md — this is the project USP.**

- **Provider:** ChakraHQ — confirmed official Meta Technology Partner
- **Ban risk:** None — official Cloud API
- **Cost:** FREE tier (1000 conversations/month via Meta)
- **Native OpenClaw support:** **NO** — requires webhook bridge (proven in production)
- **Architecture:** ChakraHQ "Pass Through webhook" → OpenClaw webhook endpoint
  - Alternative: ChakraHQ → n8n workflow → OpenClaw hooks API
- **Inbound messages:** FREE within free tier
- **"Chakra Webhooks (Beta)":** Different format, **NOT compatible** with OpenClaw → must be OFF
- **Tunnel restart:** Quick tunnel URL changes → must manually update webhook URL in
  ChakraHQ dashboard (app.chakrahq.com → WhatsApp Setup → Pass-through webhook url)
  - Fix: Set up a named tunnel for a stable URL
- **Coexistence:** WhatsApp Business App + Cloud API work simultaneously on same number (confirmed)
- **Pricing:** Free tier (1000 conversations), paid from $12.49/month
- **API docs:** apidocs.chakrahq.com

---

## WhatsApp — Baileys (Unofficial — Warning)

- **Status:** Works but carries **permanent ban risk**. Use Chakra instead for production.
- **Protocol:** Reverse-engineers WhatsApp Web (@whiskeysockets/baileys)
- **Setup:** `openclaw channels login whatsapp` → QR scan
- **Ban:** Meta detects and bans permanently. No appeal.
- **Supply chain:** `lotusbail` (56K downloads) was malicious. Only use official package.
- **Required:** Separate disposable number, allowlist, read-only SEAL pattern (see security/)
- Full details in references/whatsapp-chakra-official.md (Baileys warning section)

---

## Slack

- **Setup:** api.slack.com/apps → Create New App
- **OAuth scopes:** chat:write, channels:history, users:read
- **Socket Mode (recommended):** No public URL needed, requires `appToken`
- **Events API (alternative):** Webhook URL required, HTTPS mandatory
- **Config:** `channels.slack.enabled`, `botToken`, `appToken` (Socket Mode)

---

## Cross-References

| Topic | Skill |
|---|---|
| Agent routing, bindings, visibility | office-structure/ |
| Webhook config, n8n bridge | workflows/ |
| dmPolicy hardening, tool deny-lists, SEAL pattern | security/ |
| Backup of channel credentials | operations/ |

---

## Sources

- docs.openclaw.ai/channels/telegram
- docs.openclaw.ai/channels/discord
- docs.openclaw.ai/channels/whatsapp
- docs.openclaw.ai/channels/slack
- chakrahq.com + apidocs.chakrahq.com
