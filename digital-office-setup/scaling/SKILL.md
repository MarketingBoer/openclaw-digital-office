---
name: digital-office-scaling
description: |
  Use when scaling to multiple gateways, Kubernetes deployment,
  or production hardening. Triggers: "scale up", "kubernetes", "multi-gateway",
  "production deployment", "high availability", "performance issues".
---

# Digital Office Scaling

Tested against: OpenClaw v2026.3.x

This skill covers when and how to scale an OpenClaw setup beyond a single Gateway.
For the base Docker setup, see **foundation/**. For security hardening, see **security/**.

---

## Critical Constraint: OpenClaw is Single-Instance

OpenClaw **cannot horizontally scale**. Running replicas > 1 causes data corruption
because multiple instances write to the same SQLite database and workspace files.

This means:
- **No load balancer** in front of multiple Gateway replicas
- **No `docker compose scale`** or Kubernetes `replicas: 3`
- **No shared-nothing database** — Gateway uses SQLite, not Postgres

### Scaling Strategy

| Capacity Need | Approach |
|---|---|
| Up to ~5-10 offices | Add more agents within one Gateway |
| Beyond 10 offices | Deploy separate Gateways on separate machines |
| High availability | Kubernetes Path B (StatefulSet, replicas=1, auto-rollback) |

---

## Scaling Signals — When to Act

Before scaling, verify you actually have a capacity problem. Slow responses
can be caused by model latency or tool timeouts, not infrastructure.

| Signal | Threshold | First Action |
|---|---|---|
| CPU usage | >80% sustained during concurrent sessions | Increase `cpus` limit in docker-compose |
| OOM kills | In `docker stats` or system logs | Add swap (see foundation/ gotchas), then increase `memory` limit |
| Concurrent sessions | >10 active simultaneously | Consider separate Gateway |
| Cron/heartbeat overlap | Heavy concurrent cron runs | Stagger schedules, set `maxConcurrentRuns` |

If none of these signals are present, the bottleneck is likely not infrastructure.
Diagnose session-level slowness (model choice, context size, tool latency) first.

---

## Docker Compose Production

For single-Gateway production deployments beyond the base setup in foundation/:

- **Resource limits:** Increase `cpus` and `memory` in `deploy.resources.limits`
- **Healthcheck:** Node fetch every 30s (already in foundation/ compose reference)
- **Nginx reverse proxy:** TLS termination on port 443 → upstream 127.0.0.1:18789
- **Named volumes:** Survive `docker compose down` — data persists across restarts
- **Monitoring:** Add Prometheus endpoint + Grafana dashboard (see monitoring/)

---

## Kubernetes — Two Paths

See `references/kubernetes-helm.md` for detailed configuration of both paths.

### Path A: Community Helm Charts (quick setup)

- **Charts:** serhanekicii/openclaw-helm, Chrisbattarbee/openclaw-helm
- **Workload:** Deployment + strategy: Recreate (NOT RollingUpdate)
- **Replicas:** Hardcoded to 1 — single-instance constraint
- **Best for:** Personal use, small teams, quick setup

### Path B: Official Operator (production)

- **Repository:** openclaw-rocks/k8s-operator
- **Workload:** StatefulSet + RollingUpdate
- **CRDs:** OpenClawInstance + OpenClawSelfConfig
- **Features:** HPA-based scaling (CPU/memory), PodDisruptionBudgets, auto-rollback
- **OCI registry:** `oci://ghcr.io/openclaw-rocks/charts/openclaw-operator`
- **Best for:** Production, enterprise, managed upgrades

### Shared Requirements (both paths)

- PVC for `~/.openclaw` (persistent storage)
- NetworkPolicy: allow DNS + public internet, block RFC1918 private ranges
- Secrets: `kubectl create secret generic`
- Resources: requests 200m CPU / 512Mi, limits 2000m / 2Gi
- Security context: runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities

---

## Multi-Gateway Access — Tailscale

For managing multiple Gateways across machines:

1. Install Tailscale on each machine
2. Each Gateway binds to 127.0.0.1:18789 (loopback only — see security/)
3. Tailscale creates a private mesh network — Gateways are reachable via tailnet IPs
4. Control plane (dashboard) accessible only within the tailnet
5. Monitoring: Prometheus endpoint per Gateway, Grafana federation across tailnet

This avoids opening ports publicly or setting up complex VPN configurations.

---

## Monitoring at Scale

- **Prometheus:** Expose metrics endpoint per Gateway
- **Grafana:** Dashboards for Gateway health, session count, token throughput
- **Alerts:** CPU/memory thresholds, OOM events, session count spikes
- See **monitoring/** for full dashboard and cost tracking setup

---

## Cross-References

| Topic | Skill |
|---|---|
| Base Docker setup, compose file | foundation/ |
| Security hardening, loopback binding | security/ |
| Monitoring dashboards, cost tracking | monitoring/ |
| Agent configuration within a Gateway | office-structure/ |

---

## Sources

- docs.openclaw.ai/install/docker
- github.com/serhanekicii/openclaw-helm
- github.com/openclaw-rocks/k8s-operator
