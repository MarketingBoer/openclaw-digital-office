# Test Scenarios — channels

TDD scenarios written BEFORE building the skill. Each scenario defines expected behavior with and without the skill loaded.

---

## Test: Channel selection — which platform to start with

**Input:** "I want to connect my OpenClaw agent to messaging. Should I use Telegram, WhatsApp, or Discord?"

**Expected WITHOUT skill:** Claude gives generic advice about messaging platforms, may recommend WhatsApp first because it's popular, does not mention ban risk, does not mention Facebook's unsupported status, gives no comparison matrix.

**Expected WITH skill:** Claude presents the channel comparison matrix showing Telegram as "always first" (no ban risk, low complexity, native), WhatsApp Baileys as "HIGH ban risk — only with separate number", WhatsApp Chakra as official but requiring a custom bridge, Discord as "team/community", Facebook as "NOT supported natively". Recommends Telegram as the starting point.

**Verification:** Matrix is shown or summarized, Telegram recommended first, WhatsApp ban risk explicitly called HIGH, Facebook marked as NOT supported.

---

## Test: Telegram Privacy Mode — bot doesn't see group messages

**Input:** "I added my Telegram bot to a group chat but it's not responding to messages. It only responds when I use /commands."

**Expected WITHOUT skill:** Claude suggests checking the bot token, checking if the bot is banned, or checking internet connectivity. Does not identify Privacy Mode as the cause.

**Expected WITH skill:** Claude immediately identifies Telegram Privacy Mode — it is ON by default and causes the bot to only see @mentions and /commands in groups. Provides two fixes: (1) @BotFather → /setprivacy → Disable, then re-add bot to group, or (2) add the bot as group admin. Notes the bot must be re-added after disabling Privacy Mode.

**Verification:** Response names "Privacy Mode", states it is ON by default, gives the @BotFather fix path, mentions bot must be re-added.

---

## Test: Discord — bot sees empty messages

**Input:** "My Discord bot receives events but the message content is always empty. I can see the message was sent but the body is blank."

**Expected WITHOUT skill:** Claude suggests a bug in the code, a webhook misconfiguration, or a permissions problem. Does not identify Message Content Intent as the cause.

**Expected WITH skill:** Claude immediately identifies that Discord's Message Content Intent (a Privileged Intent) must be enabled in the Developer Portal for the bot to receive message content. Without it, Discord sends events with empty message bodies for non-command messages. Gives the exact fix: Developer Portal → Bot tab → Privileged Gateway Intents → Message Content Intent → Enable. Notes Server Members Intent is also recommended for allowlists.

**Verification:** Response names "Message Content Intent", "Privileged Intent", points to Developer Portal Bot tab, states messages are empty without it.

---

## Test: WhatsApp Baileys — ban risk and supply chain

**Input:** "I found a npm package called 'lotusbail' that claims to be a lightweight WhatsApp library. Can I use it instead of Baileys for my OpenClaw setup?"

**Expected WITHOUT skill:** Claude may say "check the documentation" or analyze the package at face value.

**Expected WITH skill:** Claude immediately warns that `lotusbail` is a confirmed malicious package — a Baileys lookalike with 56,000 downloads that steals auth tokens, messages, contacts, and media via a persistent backdoor. States the only safe package is `@whiskeysockets/baileys` from the official source. Also reinforces that Baileys itself carries a permanent ban risk from Meta and requires a separate phone number.

**Verification:** Response names `lotusbail` as malicious, states 56K downloads, names `@whiskeysockets/baileys` as the correct package, mentions permanent ban risk.

---

## Test: WhatsApp Chakra — tunnel restart breaks webhook

**Input:** "My WhatsApp Chakra setup was working, but after I restarted my cloudflared tunnel it stopped receiving messages."

**Expected WITHOUT skill:** Claude suggests checking ChakraHQ configuration vaguely, or suggests restarting the OpenClaw container.

**Expected WITH skill:** Claude explains that every cloudflared quick tunnel restart generates a new URL. The ChakraHQ "Pass Through webhook" URL must be manually updated in the ChakraHQ dashboard (app.chakrahq.com → WhatsApp Setup → Pass-through webhook url) after every tunnel restart. Also warns that "Chakra Webhooks Beta" is a different, incompatible format — it must be OFF. Recommends setting up a named tunnel to get a stable URL.

**Verification:** Response identifies tunnel URL change as root cause, mentions manual dashboard update, warns about "Chakra Webhooks Beta" incompatibility, suggests named tunnel.
