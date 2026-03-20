# WhatsApp Integration — Official Route via ChakraHQ

<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->

This is the primary reference for WhatsApp integration with OpenClaw.
**Recommended path: ChakraHQ (official Meta BSP) — free, no ban risk.**
Baileys (unofficial) is documented as a warning, not a recommendation.

---

## Recommended: ChakraHQ (Official Meta BSP)

### Why This Is the Default Choice

- **Zero ban risk** — ChakraHQ is a confirmed official Meta Technology Partner
- **Free tier** — 1,000 service conversations/month via Meta's free tier
- **Production-proven** — tested and working in production environments
- **Coexistence** — WhatsApp Business App + Cloud API work simultaneously on the same number

### Architecture

```
User → WhatsApp → Meta Cloud API → ChakraHQ → Pass-through webhook → OpenClaw
```

Alternative with n8n:
```
User → WhatsApp → Meta Cloud API → ChakraHQ → n8n workflow → OpenClaw hooks API
```

### Setup

1. Create account at chakrahq.com
2. Connect your WhatsApp Business Account
3. In ChakraHQ dashboard: WhatsApp Setup → Pass-through webhook url
4. Set webhook URL to your OpenClaw endpoint: `https://{YOUR_DOMAIN}/hooks`
5. **Disable "Chakra Webhooks (Beta)"** — its format is incompatible with
   OpenClaw's expected payload structure. Must be OFF.

### Tunnel Setup (Critical)

If using cloudflared quick tunnel: the URL changes on every restart.
You must manually update the webhook URL in ChakraHQ dashboard after
each tunnel restart.

**Fix:** Set up a named cloudflared tunnel for a stable, persistent URL.
This eliminates the manual update step entirely.

### Pricing

| Tier | Cost | Included |
|------|------|----------|
| Free | $0/month | 1,000 service conversations/month |
| Paid | From $12.49/month | Additional conversations |
| Inbound | Free | Within free tier |
| Outbound | Per-message | Meta template pricing applies |

### Configuration

ChakraHQ requires NO config in openclaw.json — it connects via the
standard webhook/hooks endpoint. Configure the webhook in workflows/:

```json
{
  "hooks": {
    "enabled": true,
    "token": "${OPENCLAW_HOOKS_TOKEN}",
    "path": "/hooks"
  }
}
```

See workflows/ for full webhook configuration details.

### API Documentation

- Dashboard: app.chakrahq.com
- API docs: apidocs.chakrahq.com

---

## Warning: Baileys (Unofficial — Use At Your Own Risk)

Baileys is documented here because OpenClaw has native support for it,
and many guides reference it. **It is NOT recommended for production.**

### What It Is

`@whiskeysockets/baileys` reverse-engineers the WhatsApp Web protocol.
It works natively with OpenClaw — no bridge needed. But it carries
**permanent ban risk from Meta**.

### Setup (if you choose to proceed)

```bash
openclaw channels login whatsapp
# Scan QR code with phone → Linked Devices → Link a Device
# Session: ~/.openclaw/credentials/whatsapp/{accountId}/creds.json
```

### Risks

1. **Permanent ban:** Meta detects unofficial usage and bans permanently.
   No appeal, no recovery. Account + number are lost.
2. **Protocol instability:** Breaks on WhatsApp updates without notice.
3. **Phone must stay online** for session persistence.
4. **Supply chain:** Malicious npm `lotusbail` (56K downloads) was a Baileys
   clone with persistent backdoor. ONLY use `@whiskeysockets/baileys`.

### Required Precautions (if using Baileys)

- **Separate phone number** — ALWAYS. Never personal number.
- **dmPolicy: allowlist** — explicit numbers only, never "open"
- **Read-only SEAL pattern** for observation-only agents (see security/)
- **Backup credentials** — include `~/.openclaw/credentials/whatsapp/`

---

## Comparison

| Aspect | ChakraHQ (Recommended) | Baileys (Warning) |
|--------|------------------------|-------------------|
| API type | Official (Meta Cloud API) | Unofficial (reverse-engineered) |
| Ban risk | **None** | **HIGH — permanent** |
| Cost | Free tier (1000/mo) | Free |
| Native OpenClaw | No (webhook bridge) | Yes |
| Setup complexity | Medium (account + bridge) | Low (QR scan) |
| Outbound | Template-based (Meta) | Direct |
| Best for | **Business / production** | Testing with disposable number only |
