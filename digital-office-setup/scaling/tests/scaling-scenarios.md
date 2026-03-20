# Scaling Skill — Test Scenarios

These scenarios were written BEFORE the skill was built (TDD: RED phase).
Each scenario defines what the skill must produce to pass.

---

## Test: Horizontal Scaling Request

**Input:** "I want to run 3 OpenClaw gateway replicas behind a load balancer for high availability."

**Expected WITHOUT skill:** Claude provides a docker-compose scale or k8s replicas: 3 config.

**Expected WITH skill:**
Claude blocks horizontal scaling. OpenClaw is SINGLE-INSTANCE — it cannot be replicated.
Running replicas > 1 causes data corruption because multiple instances write to the same
SQLite database and workspace files concurrently. Correct approach for capacity:
- Up to ~5-10 offices: add more agents within one gateway.
- Beyond that: deploy separate gateways on separate machines, each with its own data volume.
For true HA: Kubernetes Path B (official operator) with a StatefulSet and replicas=1
plus PodDisruptionBudget and auto-rollback on health check failure.

**Verification:** SKILL.md explicitly states single-instance constraint with "replicas > 1 = data corruption" warning.
kubernetes-helm.md documents replicas: 1 as mandatory.

---

## Test: When to Scale — Signal Detection

**Input:** "My OpenClaw setup feels slow. Should I scale up?"

**Expected WITHOUT skill:** Claude suggests adding more agents or containers immediately.

**Expected WITH skill:**
Claude asks for specific signals before recommending scaling:
- CPU >80% sustained during concurrent sessions? → resource limit issue, increase cpus/memory first.
- OOM kills in `docker stats` or system logs? → add swap, then increase memory limit.
- >10 concurrent sessions? → consider separate gateway.
- Heavy cron/heartbeat concurrency? → stagger schedules first, then consider separate gateway.
If signals are absent: diagnose slow sessions (model latency, tool timeouts) before
assuming infrastructure capacity is the bottleneck.

**Verification:** SKILL.md contains scaling-signals section with specific thresholds (CPU >80%, >10 sessions, OOM).

---

## Test: Kubernetes Path Selection

**Input:** "We want to deploy OpenClaw on our company Kubernetes cluster. Which Helm chart should we use?"

**Expected WITHOUT skill:** Claude picks one Helm chart without explaining the tradeoffs.

**Expected WITH skill:**
Claude presents both paths clearly:

Path A (Community Helm): serhanekicii or Chrisbattarbee charts.
- Workload: Deployment + strategy: Recreate (NOT RollingUpdate — single-instance).
- Replicas: 1 (hardcoded).
- Good for: personal use, small teams, quick setup.

Path B (Official Operator): openclaw-rocks/k8s-operator.
- Workload: StatefulSet + RollingUpdate.
- CRDs: OpenClawInstance + OpenClawSelfConfig.
- HPA-based scaling (CPU/memory metrics), PodDisruptionBudgets, auto-rollback.
- OCI registry: oci://ghcr.io/openclaw-rocks/charts/openclaw-operator.
- Good for: production, enterprise, managed upgrades.

Both paths require: PVC for ~/.openclaw, NetworkPolicy blocking RFC1918, Secrets via kubectl,
non-root security context, readOnlyRootFilesystem.

**Verification:** kubernetes-helm.md documents both paths with Deployment vs StatefulSet distinction.

---

## Test: :latest Tag in Kubernetes

**Input:** "I'll use imagePullPolicy: Always with :latest so my k8s cluster auto-updates."

**Expected WITHOUT skill:** Claude approves this as a valid auto-update strategy.

**Expected WITH skill:**
Claude blocks :latest in Kubernetes. Reasons:
1. :latest is unpredictable — a breaking schema change can crash all pods simultaneously
   (the same incident that crashed the Hostinger gateway).
2. imagePullPolicy: Always with :latest makes rollback nearly impossible because the
   "previous" image tag is gone.
3. In k8s, a bad :latest image can cause a CrashLoopBackOff that blocks all traffic.
Correct approach: pin to a specific semver tag. Use a CD pipeline (Flux/ArgoCD) to
automate upgrades with a tested image tag.

**Verification:** gotchas.md contains ":latest in k8s = unpredictable" with CrashLoopBackOff consequence.

---

## Test: Tailscale Multi-Gateway Access

**Input:** "I have two OpenClaw gateways on different VPS machines. How do I manage them securely?"

**Expected WITHOUT skill:** Claude suggests opening ports on each server or using a VPN with complex setup.

**Expected WITH skill:**
Claude recommends Tailscale as the secure remote access layer between gateways:
1. Install Tailscale on each machine.
2. Each gateway binds to 127.0.0.1:18789 (loopback only).
3. Tailscale creates a private mesh network — gateways communicate over tailnet IPs.
4. Control plane access (dashboard) is reachable only within the tailnet.
5. For monitoring across gateways: Prometheus endpoint per gateway, Grafana federation.
Cross-reference: foundation/ for single-gateway Docker setup, security/ for loopback binding rule.

**Verification:** SKILL.md mentions Tailscale for multi-gateway access with loopback-only binding prerequisite.
