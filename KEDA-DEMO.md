# ⚡ KEDA Scaling Demo — AMP + Dynatrace

> **Kubernetes Event-Driven Autoscaling** using two independent metric sources:  
> AWS AMP (Prometheus) and Dynatrace — with Cron as a night floor.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   [nginx pod]                                                               │
│      │                                                                      │
│      ├──── port 9113 ──▶  [nginx-prometheus-exporter]                      │
│      │                          │                    │                      │
│      │                          │                    │                      │
│      │              [DT OneAgent scrapes]   [AWS managed scraper]           │
│      │                          │            (every 60s)                   │
│      │                          ▼                    ▼                      │
│      │               [Dynatrace Metrics v2]    [AMP workspace]              │
│      │                          │                    │                      │
│      └──────────────────────────┴────────────────────┘                     │
│                                          │                                  │
│                                   [KEDA Operator]                           │
│                                   polls every 15s                           │
│                                          │                                  │
│                              [KEDA Metrics Adapter]                         │
│                           exposes ExternalMetric to HPA                     │
│                                          │                                  │
│                                 [Kubernetes HPA]                            │
│                                          │                                  │
│                              scales [nginx Deployment]                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🎯 ScaledObject — Annotated

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-keda-sample
  namespace: keda-testing
spec:
  scaleTargetRef:
    name: nginx-keda-sample      # must match Deployment name exactly

  minReplicaCount: 1             # never scale to zero — no cold start
  maxReplicaCount: 10            # HPA ceiling

  pollingInterval: 15            # KEDA fetches metrics every 15s
  cooldownPeriod: 60             # wait 60s after triggers deactivate before scaling down

  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
          # Kubernetes HPA default is 300s — it holds the highest replica
          # count from the last 5 minutes before allowing scale-down.
          # We override to 60s to match cooldownPeriod, so scale-down
          # happens within ~60s of load dropping, not 5 minutes later.

  triggers:

    # ── TRIGGER 1: Dynatrace ──────────────────────────────────────────────
    - type: dynatrace
      authenticationRef:
        name: keda-trigger-auth-dynatrace
        kind: ClusterTriggerAuthentication
        # → reads dt-keda-token SealedSecret from keda namespace
        # → token + host decrypted at runtime by Bitnami SealedSecrets
      metadata:
        metricSelector: "nginx_connections_active\
          :filter(and(eq(k8s.namespace.name,keda-testing)))\
          :sum\
          :fold"
        #   :filter  — scope to keda-testing namespace
        #   :sum     — aggregate across all pods
        #   :fold    — collapse time window into single average value

        from: "now-2m"
        # KEDA Dynatrace scaler defaults to from=now-2h if not set.
        # With a 2h window, :fold averages 2h of idle + short load spike
        # → metric barely moves. Setting now-2m makes it respond to
        # real-time load within a single polling cycle.

        threshold: "10"
        # desiredReplicas = ceil( metricValue / threshold )
        # e.g. metric=100, threshold=10 → ceil(100/10) = 10 replicas

        activationThreshold: "7"
        # metric must EXCEED this before trigger becomes active at all.
        # idle nginx_connections_active ≈ 3-6 (keep-alive connections).
        # >7 means real load is present. Prevents false activations.

    # ── TRIGGER 2: AMP (Amazon Managed Prometheus) ───────────────────────
    - type: prometheus
      authenticationRef:
        name: aws-amp
        kind: ClusterTriggerAuthentication
        # → IRSA (pod identity) — no static credentials
        # → KEDA operator pod IAM role has aps:QueryMetrics permission
      metadata:
        serverAddress: "${AMP_WORKSPACE_URL}"
        metricName: nginx_connections_active
        query: "sum(nginx_connections_active)"
        # No namespace filter — AWS managed scraper (pod_exporter job)
        # does not attach a namespace label to scraped series.
        # All nginx_connections_active series in AMP are from keda-testing
        # only, so unfiltered sum is safe.

        threshold: "20"
        activationThreshold: "10"
        # AMP scrapes every 60s (point-in-time gauge).
        # Load pods hold persistent TCP connections open for 20s each,
        # so the 60s scraper always captures elevated values.
        # 10 pods × ~20 conns/pod = ~200 active connections → well above 20.

        awsRegion: "eu-north-1"

    # ── TRIGGER 3: Cron ──────────────────────────────────────────────────
    - type: cron
      metadata:
        timezone: "Europe/Stockholm"
        start: "0 18 * * *"     # activates at 18:00
        end: "0 6 * * *"        # deactivates at 06:00
        desiredReplicas: "1"    # night floor — holds 1 replica overnight
```

---

## 🔑 Key Concepts

### `activationThreshold` vs `threshold`

| Field | Purpose | Formula |
|---|---|---|
| `activationThreshold` | Gate: trigger is **inactive** below this. Prevents false activations from idle noise. | `if metric <= activationThreshold → isActive: false` |
| `threshold` | Scale target **per replica** once active. | `desiredReplicas = ceil( metric / threshold )` |

**Example:**
```
metric = 100,  threshold = 10
desiredReplicas = ceil(100 / 10) = 10 replicas

metric = 6,  activationThreshold = 7
→ trigger inactive → replicas stay at minReplicaCount
```

### OR Semantics (multiple triggers)

KEDA calculates `desiredReplicas` for each **active** trigger independently.  
The HPA takes the **maximum** of all desired values.

```
DT  desired = ceil(150 / 10) = 15  →  capped at maxReplicaCount = 10
AMP desired = ceil(200 / 20) = 10
HPA desired = max(10, 10)   = 10 replicas
```

Both triggers must go **inactive** before cooldown begins.  
Either trigger alone keeps replicas scaled up.

### `from: "now-2m"` — Why It Matters

KEDA's Dynatrace scaler defaults to `from=now-2h` when `from` is not set.  
The `:fold` operator averages all data points in that window:

```
2h window: 117 min idle (metric=3) + 3 min load (metric=100)
average = (117×3 + 3×100) / 120 ≈ 5.4  →  never exceeds threshold=10
```

With `from=now-2m`:
```
2m window: all under load
average = ~100  →  triggers immediately
```

---

## 🔐 Authentication

### Dynatrace — ClusterTriggerAuthentication + SealedSecret

```
ClusterTriggerAuthentication (keda-trigger-auth-dynatrace)
  └── secretTargetRef → dt-keda-token (Secret in keda namespace)
                            ├── token: <DT API token>
                            └── host:  https://jzw21825.live.dynatrace.com
```

Secret is stored as a **SealedSecret** (Bitnami) — encrypted with the cluster's public key, safe to commit to git. Decrypted at runtime by the sealed-secrets controller.

> ⚠️ DT host must be `live.dynatrace.com` — Metrics v2 API is **not** available on `apps.dynatrace.com`.

### AMP — IRSA (no static credentials)

```
ClusterTriggerAuthentication (aws-amp)
  └── podIdentity.provider: aws
        └── KEDA operator ServiceAccount → IAM Role (via EKS IRSA)
              └── aps:QueryMetrics on AMP workspace ARN
```

---

## 🎬 Demo Flow

```
PHASE 0 — Baseline
  nginx_connections_active ≈ 3-6 (idle keep-alive)
  Both triggers: inactive
  Replicas: 1

       │
       ▼  launch 5 load pods
       │  each pod: 1 persistent TCP conn/sec, held open 20s
       │  after warmup: 5 × 20 = ~100 active connections

PHASE 1 — Dynatrace fires first
  DT 2m fold: ~100  >  activationThreshold=7  ✔
  desiredReplicas = ceil(100/10) = 10
  Replicas scale up ↑

       │
       ▼  launch 5 more pods (10 total)
       │  ~200 active connections
       │  AMP 60s scraper captures the spike

PHASE 2 — AMP fires
  AMP value: ~200  >  activationThreshold=10  ✔
  desiredReplicas = ceil(200/20) = 10
  Both DT + AMP active → HPA = max(10, 10) = 10

       │
       ▼  remove all load pods
       │  nc connections drain within 20s
       │  metric drops below activationThreshold

PHASE 3 — Cooldown
  Both triggers: inactive
  cooldownPeriod (60s) elapses
  stabilizationWindowSeconds (60s) satisfied
  Replicas scale down to 1 ↓
```

---

## 🐛 Lessons Learned

| Problem | Root Cause | Fix |
|---|---|---|
| DT trigger never fired at `threshold=7` | KEDA defaults `from=now-2h`; 2h `:fold` average barely moves during short load test | Add `from: "now-2m"` to trigger metadata |
| AMP metric always `0` despite pods running | `pod_exporter` scraper job doesn't attach `namespace` label; `{namespace="keda-testing"}` filter matched nothing | Remove namespace filter → `sum(nginx_connections_active)` |
| nginx stayed at 2 replicas at idle | `activationThreshold` too close to idle baseline (~3-6 conns); triggers permanently active | Raise `activationThreshold` above idle: DT=7, AMP=10 |
| Replicas took 5 min to scale down after load removed | Kubernetes HPA default `stabilizationWindowSeconds=300` | Set `stabilizationWindowSeconds: 60` in ScaledObject `advanced` |
| `host` field → `unsupported protocol scheme ""` | SealedSecret sealed without `https://` prefix | Reseal with full URL including scheme |
| `404` on Dynatrace API | Used `apps.dynatrace.com` instead of `live.dynatrace.com` | Metrics v2 API only available on `live.dynatrace.com` |

---

## 📁 Files

| File | Location |
|---|---|
| ScaledObject | `dataplatform-devtest-eks/releases/platform-engineering/keda-testing/scaledobject.yaml` |
| ClusterTriggerAuthentication (DT) | `dataplatform-clusters/devtest/releases/keda/dt.yaml` |
| SealedSecret (DT token) | `dataplatform-clusters/devtest/releases/keda/dt-keda-token.yaml` |
| Demo script | `~/Desktop/keda/demo-scaling.sh` |
