# Scaling — Gotchas

Real pitfalls when scaling OpenClaw deployments.

---

## 1. OpenClaw Cannot Horizontally Scale (Single-Instance)

**What happens:** Running replicas > 1 causes silent data corruption. Sessions
overwrite each other, workspace files get scrambled, SQLite WAL becomes inconsistent.

**Why:** The Gateway uses SQLite (single-writer) and filesystem-based workspaces.
Multiple instances writing concurrently to the same PVC corrupts both.

**Fix:** Never set replicas > 1. For capacity: add agents within one Gateway,
or deploy separate Gateways on separate machines with separate data volumes.

---

## 2. :latest Tag in Kubernetes = Unpredictable Rollouts

**What happens:** A pod restart pulls a new image version silently. The new
image may have a changed config schema, causing CrashLoopBackOff. Rollback
is difficult because the "previous" image tag is `:latest` (same tag, different content).

**Why:** `:latest` is a mutable tag. `imagePullPolicy: Always` + `:latest`
means every pod restart is a potential breaking upgrade.

**Fix:** Pin a specific semver tag. Use a CD pipeline (Flux/ArgoCD) to
automate upgrades with tested image versions.

---

## 3. No Native Resource Monitoring

**What happens:** CPU and memory exhaustion go unnoticed until the Gateway
crashes or sessions become unresponsive. OOM kills appear in system logs
but generate no alert.

**Why:** OpenClaw has no built-in metrics endpoint or alerting. Docker and
Kubernetes provide container-level metrics, but you must set up collection yourself.

**Fix:** Add Prometheus + Grafana. Monitor: CPU usage, memory, session count,
OOM events. Set alerts at 70% and 90% thresholds. See monitoring/ skill.

---

## 4. Recreate Strategy Causes Brief Downtime

**What happens:** Kubernetes Deployment with `strategy: Recreate` stops the
old pod before starting the new one. During this window (10-30s), the Gateway
is unavailable. Incoming messages are lost.

**Why:** Recreate is required because OpenClaw is single-instance — RollingUpdate
would briefly run two pods. But Recreate has a gap.

**Fix:** Accept the brief downtime, or use Path B (StatefulSet with operator)
which manages graceful transitions. For channels: Telegram long-poll recovers
automatically. Discord/Slack have internal retry. WhatsApp may need re-pairing
if the gap exceeds session timeout.
