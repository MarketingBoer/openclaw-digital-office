# Mission Control Options — Detailed Reference

<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->

## Variant Comparison

| Feature | robsannaa | crshdn | TenacitOS | abhi1693 | builderz-labs |
|---------|-----------|--------|-----------|----------|---------------|
| Dashboard | Yes | Yes | Yes | Yes | Yes |
| Chat interface | Yes | Yes | No | Yes | No |
| Cost tracking | No | No | Yes | No | Yes |
| Kanban/planning | No | Yes | No | Yes | No |
| Security scanner | No | No | No | No | Yes |
| Auto-discovery | No | No | Yes | No | No |
| Approval workflows | No | No | No | Yes | No |
| Docker required | No | Yes | No | Yes | Yes |
| Tabs | Dashboard, Chat, Channels, Docs, Security, Permissions | Workspace, Planning, Live Feed, Agents | System Monitor, Agents, Costs, Cron, Memory, Files | Organizations, Boards, Tasks, Agents, Governance | Costs (Recharts), GitHub sync, Cron, Security |

## Connection Details

All variants connect via WebSocket to the gateway on port 18789.

```
ws://localhost:18789
```

For Docker deployments: join the `openclaw_default` network so the dashboard
can reach the gateway at `ws://openclaw-gateway:18789`.

**Device pairing required:** First connection triggers a pairing flow in the
gateway's control UI. The dashboard must be approved before it can access data.

## Quick Start: robsannaa (recommended first)

```bash
git clone https://github.com/robsannaa/openclaw-mission-control.git
cd openclaw-mission-control
npm install
npm start
# Open http://localhost:3000
# Complete device pairing in the gateway control UI
```

No Docker needed. Runs directly on the host.

## Quick Start: crshdn (Autensa)

```bash
git clone https://github.com/crshdn/mission-control.git
cd mission-control
cp .env.example .env.local
# Edit .env.local: set OPENCLAW_GATEWAY_URL and TOKEN
npm install && npm run build && npx next start -p 4000
# Open http://localhost:4000
```

## Quick Start: ClawMetry (real-time monitoring)

```bash
pip install clawmetry
clawmetry connect
# Open http://localhost:8765
```

Features: real-time flow visualization, token costs per model, sub-agent
activity tracking, cron job status, memory change monitoring.

Cloud option: $5/node/month (E2E encrypted, remote access).

## Context Pruning Configuration

```json
{
  "agents": {
    "defaults": {
      "context": {
        "pruning": {
          "mode": "safeguard",
          "cacheTtl": "6h",
          "keepLastAssistants": 3,
          "memoryFlush": true
        }
      }
    }
  }
}
```

- `mode: safeguard` — only compact when context overflow would block session
- `cacheTtl` — remove tool results older than this from context
- `keepLastAssistants` — preserve last N assistant messages during compaction
- `memoryFlush` — write MEMORY.md entries before compaction starts
