# Kubernetes Deployment — Two Paths

<!-- verified: 2026-03-20 against OpenClaw v2026.3.x -->

## Path A: Community Helm Charts

Maintained by community contributors. Quick to deploy, minimal configuration.

**Charts:**
- serhanekicii/openclaw-helm
- Chrisbattarbee/openclaw-helm

**Workload type:** Deployment (not StatefulSet)

**Strategy:** Recreate — never RollingUpdate. OpenClaw is single-instance;
RollingUpdate would briefly run two pods writing to the same PVC.

**Replicas:** Hardcoded to 1. Do not change this value.

**Example values.yaml:**
```yaml
replicaCount: 1  # NEVER increase — single-instance constraint

image:
  repository: ghcr.io/openclaw/openclaw
  tag: "2026.3.x"  # pin specific tag, never :latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 18789

persistence:
  enabled: true
  storageClass: ""  # use cluster default
  size: 10Gi
  accessModes:
    - ReadWriteOnce

resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 2Gi

securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false

env:
  OPENCLAW_GATEWAY_BIND: "0.0.0.0"  # in k8s, bind to pod IP (service handles routing)
  OPENCLAW_GATEWAY_PORT: "18789"
  OPENCLAW_SANDBOX: "1"
```

**Note on GATEWAY_BIND:** In Kubernetes, the pod binds to 0.0.0.0 because
the Service handles network isolation. External access is controlled by
NetworkPolicy and Ingress, not by the bind address.

---

## Path B: Official Operator

Production-grade operator with custom resource definitions and lifecycle management.

**Repository:** openclaw-rocks/k8s-operator
**OCI registry:** `oci://ghcr.io/openclaw-rocks/charts/openclaw-operator`

**Workload type:** StatefulSet

**Strategy:** RollingUpdate — the operator manages graceful transitions
with health checks and automatic rollback on failure.

**CRDs:**
- `OpenClawInstance` — defines a Gateway deployment
- `OpenClawSelfConfig` — configures auto-update and self-management

**Features:**
- HPA-based scaling (CPU/memory metrics) — scales support pods, not the Gateway itself
- PodDisruptionBudgets — prevents involuntary eviction during maintenance
- Auto-rollback on failed health checks
- Managed upgrades via CRD version field

**Install:**
```bash
helm install openclaw-operator \
  oci://ghcr.io/openclaw-rocks/charts/openclaw-operator \
  --namespace openclaw-system \
  --create-namespace
```

**Example OpenClawInstance:**
```yaml
apiVersion: openclaw.rocks/v1alpha1
kind: OpenClawInstance
metadata:
  name: production
  namespace: openclaw
spec:
  image: ghcr.io/openclaw/openclaw:2026.3.x
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: "2"
      memory: 2Gi
  persistence:
    size: 10Gi
  gateway:
    token:
      secretName: openclaw-gateway-token
      key: token
```

---

## Shared Configuration (both paths)

### Persistent Volume
```yaml
persistence:
  enabled: true
  accessModes: [ReadWriteOnce]  # single-instance, no shared access
  size: 10Gi
```

### NetworkPolicy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: openclaw-egress
spec:
  podSelector:
    matchLabels:
      app: openclaw
  policyTypes: [Egress]
  egress:
    - to: []  # allow public internet (LLM APIs)
      ports:
        - port: 443
        - port: 53
          protocol: UDP
```

Block RFC1918 ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) except
for DNS and the Kubernetes API server.

### Secrets
```bash
kubectl create secret generic openclaw-gateway-token \
  --from-literal=token="$(openssl rand -hex 32)" \
  --namespace openclaw

kubectl create secret generic openclaw-api-keys \
  --from-literal=ANTHROPIC_API_KEY="sk-ant-..." \
  --namespace openclaw
```

### Security Context
Both paths should enforce:
- `runAsNonRoot: true`
- `runAsUser: 1000`
- `readOnlyRootFilesystem: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: [ALL]`
