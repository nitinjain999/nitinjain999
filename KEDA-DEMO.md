<div align="center">

# ⚡ KEDA Scaling Demo

![KEDA](https://img.shields.io/badge/KEDA-2.x-blue?style=for-the-badge&logo=kubernetes&logoColor=white)
![Dynatrace](https://img.shields.io/badge/Dynatrace-Metrics_v2-b4a0e5?style=for-the-badge&logo=dynatrace&logoColor=white)
![AMP](https://img.shields.io/badge/AWS_AMP-Prometheus-ff9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![GitOps](https://img.shields.io/badge/GitOps-FluxCD-5468ff?style=for-the-badge&logo=flux&logoColor=white)

**Kubernetes Event-Driven Autoscaling** with two independent metric sources —  
AWS AMP (Prometheus) + Dynatrace — plus Cron as a night floor.

</div>

---

## 🏗️ Live Architecture

<div align="center">
<img src="assets/keda-architecture.svg" alt="KEDA Architecture — animated" width="900"/>
</div>

> Dots flowing along each path show live data moving: metrics scraped → KEDA polling → HPA scaling.

---

## 📊 Scaling Phases — Animated

<div align="center">
<img src="assets/keda-scaling-phases.svg" alt="KEDA Scaling Phases — animated" width="900"/>
</div>

---

## 🤔 What Problem Does KEDA Solve?

**Without KEDA**, Kubernetes can only scale your app based on CPU and memory.

```
❌ "Scale up when CPU > 80%"   ← only option without KEDA
```

**With KEDA**, you can scale on *anything* — queue depth, active connections, custom business metrics, time of day.

```
✅ "Scale up when there are > 100 messages in an SQS queue"
✅ "Scale up when nginx has > 10 active connections"
✅ "Scale up at 09:00 and back down at 18:00"
✅ "Scale to zero at night, back up when traffic arrives"
```

KEDA sits between your app and the Kubernetes HPA (Horizontal Pod Autoscaler). It reads external metrics and feeds them into the HPA so it can make scaling decisions.

---

## 📐 How Kubernetes Scaling Works (The Basics)

Before diving into KEDA, here is how Kubernetes scaling works at its core.

### The HPA (Horizontal Pod Autoscaler)

The HPA watches a metric and adjusts how many pod replicas are running.

```
┌──────────────┐    watches metric    ┌──────────────────┐    adjusts    ┌─────────────┐
│     HPA      │ ──────────────────▶  │  metric value    │               │  Deployment │
│              │                      │  e.g. 47 conns   │ ────────────▶ │  replicas   │
└──────────────┘                      └──────────────────┘               └─────────────┘
```

### The Replica Formula

This is the core formula the HPA uses every time it checks the metric:

```
desiredReplicas = ceil( currentMetricValue / threshold )
```

> **`ceil`** means "round up to the nearest whole number"  
> because you can't have 2.3 pods — it becomes 3.

**Step-by-step example:**

```
Scenario: nginx has 47 active connections. threshold = 10.

Step 1:  currentMetricValue / threshold
         = 47 / 10
         = 4.7

Step 2:  ceil(4.7)
         = 5

Result:  HPA sets nginx to 5 replicas.

Why 5?  Because with 5 replicas, each replica handles 47/5 = 9.4 connections
        which is below the threshold of 10. The load is balanced.
```

**More examples to make it click:**

| metric value | threshold | calculation | replicas |
|---|---|---|---|
| 10 | 10 | ceil(10/10) = ceil(1.0) | **1** |
| 11 | 10 | ceil(11/10) = ceil(1.1) | **2** |
| 50 | 10 | ceil(50/10) = ceil(5.0) | **5** |
| 95 | 10 | ceil(95/10) = ceil(9.5) | **10** |
| 200 | 20 | ceil(200/20) = ceil(10.0) | **10** |
| 3 | 10 | ceil(3/10) = ceil(0.3) | **1** (min floor) |

---

## 🎯 ScaledObject — Every Field Explained

A `ScaledObject` is the KEDA resource you create to tell KEDA:
- **what** to scale (your Deployment)
- **when** to scale (your triggers and thresholds)
- **how fast** to scale up and down

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-keda-sample
  namespace: keda-testing
spec:

  scaleTargetRef:
    name: nginx-keda-sample
    # ↑ The name of the Deployment you want to scale.
    #   Must match exactly. KEDA will fail to reconcile if this is wrong.

  minReplicaCount: 1
    # ↑ The minimum number of replicas — even when all triggers are inactive.
    #   Set to 0 if you want scale-to-zero (no traffic = no pods).
    #   Set to 1 (like here) if your app needs to always be ready.
    #   Cold start problem: if set to 0, the first request after idle
    #   has to wait for a pod to start (~10-30s). Not ideal for web apps.

  maxReplicaCount: 10
    # ↑ The maximum number of replicas — HPA will never go above this.
    #   Protects against runaway scaling due to a buggy metric.
    #   Set this based on your cluster capacity and cost budget.

  pollingInterval: 15
    # ↑ How often (in seconds) KEDA fetches the metric from each source.
    #   Every 15s: KEDA calls Dynatrace API and AMP to get current values.
    #   Lower = faster reaction, more API calls.
    #   Higher = slower reaction, fewer API calls (cheaper for pay-per-query sources).

  cooldownPeriod: 60
    # ↑ How long (in seconds) KEDA waits AFTER all triggers deactivate
    #   before it allows the HPA to scale DOWN.
    #
    #   Why?  Prevents "thrashing" — imagine load spikes every 30s:
    #         without cooldown: scale up → scale down → scale up → scale down
    #         with cooldown=60: scale up → wait 60s → only scale down if still quiet
    #
    #   Think of it like a thermostat: you don't want your heating to turn
    #   on and off every 10 seconds. You wait until it's been cold for a while.

  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
          # ↑ This is a Kubernetes HPA setting (not KEDA-specific).
          #
          #   The HPA keeps a rolling window of "what was the highest
          #   desired replica count in the last N seconds?" and uses
          #   THAT as the current desired count when scaling down.
          #
          #   Default is 300 seconds (5 minutes).
          #   That means even after all metrics drop to zero, the HPA
          #   waits 5 minutes before scaling down. Very conservative.
          #
          #   We set it to 60s to match cooldownPeriod, so the full
          #   scale-down cycle is ~60-90s after load disappears.
          #
          #   Without this setting (real problem we hit):
          #   Load removed at T=0 → triggers deactivate → cooldown passes
          #   → HPA still sees "5 min ago I wanted 10 replicas" → stays at 10
          #   → 5 minutes of unnecessary pods running = wasted cost

  triggers:
    # Triggers are the heart of KEDA. Each trigger watches one metric source.
    # KEDA evaluates ALL triggers every pollingInterval seconds.
    # If ANY trigger is active → KEDA scales up.
    # ALL triggers must be inactive before cooldown starts.

    # ── TRIGGER 1: Dynatrace ────────────────────────────────────────────────
    - type: dynatrace
      authenticationRef:
        name: keda-trigger-auth-dynatrace
        kind: ClusterTriggerAuthentication
        # ↑ ClusterTriggerAuthentication (cluster-wide) vs TriggerAuthentication (namespace-scoped)
        #   Cluster-wide: one auth resource shared across all namespaces.
        #   Namespace-scoped: each namespace needs its own copy.
        #   We use cluster-wide because KEDA reads secrets from the keda namespace,
        #   not the application namespace.
      metadata:
        metricSelector: "nginx_connections_active:filter(and(eq(k8s.namespace.name,keda-testing))):sum:fold"
        # ↑ This is a Dynatrace Metrics v2 selector. Read it left to right:
        #
        #   nginx_connections_active
        #     → the base metric name (scraped by DT OneAgent from nginx exporter)
        #
        #   :filter(and(eq(k8s.namespace.name,keda-testing)))
        #     → only include data points where the Kubernetes namespace label = keda-testing
        #       (without this, you'd get connections from ALL nginx pods cluster-wide)
        #
        #   :sum
        #     → add up values across all matching pods
        #       e.g. pod-A=30, pod-B=25, pod-C=40 → sum = 95
        #
        #   :fold
        #     → collapse all data points in the time window into ONE average number
        #       e.g. [10, 15, 95, 100, 98] over 2 minutes → fold = 63.6
        #       KEDA needs a single number to compare against threshold.

        from: "now-2m"
        # ↑ The time window for the query. KEDA sends this to Dynatrace API.
        #
        #   DEFAULT (if you don't set this): now-2h  ← 2 hour window!
        #
        #   THE BUG WE HIT:
        #   With from=now-2h and :fold:
        #     - 117 minutes of idle traffic (metric=3 per minute)
        #     - 3 minutes of load (metric=100 per minute)
        #     - fold average = (117×3 + 3×100) / 120 = 5.4
        #     - threshold=10 → 5.4 never exceeds 10 → trigger never fires!
        #
        #   With from=now-2m:
        #     - 2 minutes of load (metric=100)
        #     - fold average ≈ 100
        #     - threshold=10 → 100 > 10 → trigger fires immediately ✔

        threshold: "10"
        # ↑ The scaling target per replica.
        #   desiredReplicas = ceil( metricValue / threshold )
        #
        #   With metric=100 and threshold=10:
        #   ceil(100/10) = 10 replicas
        #
        #   This means: "I want each replica to handle at most 10 connections"

        activationThreshold: "7"
        # ↑ The minimum value the metric must EXCEED before this trigger
        #   is considered "active" at all.
        #
        #   Think of it as a noise filter or a minimum signal level.
        #
        #   WHY IS THIS NEEDED?
        #   nginx always has some idle connections (~3-6) from keep-alive.
        #   These are connections that browsers/clients open and hold open
        #   "just in case" — even when no requests are happening.
        #
        #   If activationThreshold=0:
        #     idle metric=4 → trigger active → ceil(4/10)=1 → no scale-up
        #     but trigger IS active, which means cooldown starts running
        #     even at idle. This causes oscillation and confusion.
        #
        #   With activationThreshold=7:
        #     idle metric=4 → 4 ≤ 7 → trigger INACTIVE → system fully quiet
        #     load metric=100 → 100 > 7 → trigger ACTIVE → scaling begins

    # ── TRIGGER 2: AMP (Amazon Managed Prometheus) ──────────────────────────
    - type: prometheus
      authenticationRef:
        name: aws-amp
        kind: ClusterTriggerAuthentication
        # ↑ This uses IRSA — IAM Roles for Service Accounts.
        #   Instead of storing an AWS access key/secret in a Kubernetes Secret,
        #   the KEDA operator pod itself has an IAM role attached to it.
        #   AWS automatically provides temporary credentials via the pod's
        #   service account token. No static credentials anywhere.
      metadata:
        serverAddress: "${AMP_WORKSPACE_URL}"
        # ↑ The AMP workspace HTTP endpoint.
        #   ${AMP_WORKSPACE_URL} is a variable — Flux substitutes the real
        #   URL at deploy time from a ConfigMap. This keeps the URL out of git.

        metricName: nginx_connections_active
        # ↑ A name KEDA uses internally to identify this metric in the HPA.
        #   Must be unique within the ScaledObject.

        query: "sum(nginx_connections_active)"
        # ↑ PromQL query sent to AMP every pollingInterval seconds.
        #
        #   sum() adds up values across all pods/instances.
        #   e.g. pod-A=60, pod-B=70, pod-C=80 → sum = 210
        #
        #   WHY NO NAMESPACE FILTER?
        #   You might expect: sum(nginx_connections_active{namespace="keda-testing"})
        #   But the AWS managed scraper job does NOT add a "namespace" label
        #   to the metrics it collects. So that filter returns zero results.
        #   Since all nginx exporters in this cluster are in keda-testing,
        #   the plain sum() is correct and safe.

        threshold: "20"
        activationThreshold: "10"
        # ↑ Same concepts as DT above but different values.
        #   AMP scrapes every 60 seconds (point-in-time snapshot).
        #   We use persistent TCP connections (nc) that stay open 20s,
        #   so the 60s scraper always catches them.
        #   10 pods × 20 conns/pod = ~200 connections → ceil(200/20) = 10 replicas.

        awsRegion: "eu-north-1"
        # ↑ Required for KEDA to sign requests to AMP with AWS SigV4.

    # ── TRIGGER 3: Cron ─────────────────────────────────────────────────────
    - type: cron
      metadata:
        timezone: "Europe/Stockholm"
        # ↑ Always specify a timezone — cron without timezone uses UTC
        #   which can cause confusion when your team works in local time.

        start: "0 18 * * *"
        # ↑ Standard 5-field cron: minute hour day month weekday
        #   0 18 * * * = "at minute 0 of hour 18, every day" = 18:00 daily

        end: "0 6 * * *"
        # ↑ 06:00 daily — trigger deactivates

        desiredReplicas: "1"
        # ↑ While this trigger is active (18:00-06:00), KEDA holds
        #   at least this many replicas regardless of other triggers.
        #   This is the "night floor" — prevents scale-to-zero at night
        #   so the app is always ready for early morning traffic.
```

---

## 🧮 The Scaling Formula — Step by Step

### Step 1 — Is the trigger active?

```
if metric > activationThreshold:
    trigger is ACTIVE   → proceed to step 2
else:
    trigger is INACTIVE → replicas stay at minReplicaCount
```

### Step 2 — How many replicas?

```
desiredReplicas = ceil( metric / threshold )
```

**Real numbers from this demo:**

```
┌─────────────────────────────────────────────────────────────────┐
│  IDLE (no load)                                                 │
│                                                                 │
│  DT metric  = 4   activationThreshold = 7                       │
│  4 ≤ 7  →  DT trigger INACTIVE                                  │
│                                                                 │
│  AMP metric = 5   activationThreshold = 10                      │
│  5 ≤ 10 →  AMP trigger INACTIVE                                 │
│                                                                 │
│  Both inactive → nginx stays at minReplicaCount = 1 replica     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1 (5 load pods = ~100 connections)                       │
│                                                                 │
│  DT metric = 100   activationThreshold = 7                      │
│  100 > 7  →  DT trigger ACTIVE                                  │
│  desiredReplicas = ceil(100 / 10) = ceil(10.0) = 10             │
│                                                                 │
│  AMP metric = 5   (AMP hasn't scraped yet — 60s interval)       │
│  5 ≤ 10 →  AMP trigger still INACTIVE                           │
│                                                                 │
│  HPA desired = max(DT=10, AMP=inactive) = 10                    │
│  nginx scales to 10 replicas (capped at maxReplicaCount)        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2 (10 load pods = ~200 connections, AMP scraped)         │
│                                                                 │
│  DT metric  = 200   → ceil(200/10) = 20 → capped at max=10      │
│  AMP metric = 200   → ceil(200/20) = 10                         │
│                                                                 │
│  HPA desired = max(DT=10, AMP=10) = 10 replicas                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3 (load removed, connections drain)                      │
│                                                                 │
│  DT metric drops to 4  →  4 ≤ 7  →  DT INACTIVE                │
│  AMP metric drops to 5 →  5 ≤ 10 →  AMP INACTIVE               │
│                                                                 │
│  Both inactive → cooldownPeriod=60s starts                      │
│  After 60s → stabilizationWindowSeconds=60s passes              │
│  HPA scales nginx back to minReplicaCount = 1 replica           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔀 OR Semantics — Multiple Triggers Explained

When you have multiple triggers, KEDA uses **OR logic** — any one trigger is enough to scale up.

```
Think of it like a light switch with two switches wired in parallel:
Either switch ON  →  light ON
Both switches OFF →  light OFF
```

For replica counts, KEDA calculates independently and takes the **maximum**:

```
Trigger A says: "I need 5 replicas"
Trigger B says: "I need 8 replicas"

HPA takes: max(5, 8) = 8 replicas
```

**Why maximum and not sum?**

Because both triggers are watching the same application. If DT says "you need 5 pods to handle the load" and AMP says "you need 8 pods", you only need to run 8 (not 13). Running 8 satisfies both conditions simultaneously.

---

## 🔑 Authentication — How Credentials Stay Secure

### Dynatrace — SealedSecret chain

A common mistake is putting API tokens directly in your YAML files and committing them to git. **Never do this.**

Instead we use Bitnami SealedSecrets:

```
Your DT API token (plaintext)
        │
        │  kubeseal --controller-namespace flux-system
        ▼
SealedSecret YAML (encrypted with cluster public key)
        │
        │  safe to commit to git — only this cluster can decrypt it
        ▼
Flux deploys SealedSecret to cluster
        │
        │  sealed-secrets controller decrypts it
        ▼
Kubernetes Secret (plaintext, inside cluster only)
        │
        ▼
ClusterTriggerAuthentication reads the Secret
        │
        ▼
KEDA uses token to call Dynatrace API
```

> ⚠️ **Important:** Dynatrace host must be `live.dynatrace.com`  
> The Metrics v2 API (`/api/v2/metrics/query`) is **not** available on `apps.dynatrace.com`.  
> This causes a `404` error that looks like an auth problem.

### AMP — IRSA (IAM Roles for Service Accounts)

Instead of an API key, AMP uses AWS IAM.

```
The old way (bad):
  AWS_ACCESS_KEY_ID=AKIA...     ← static key stored in a Secret
  AWS_SECRET_ACCESS_KEY=xxx     ← rotates manually, leak risk

The IRSA way (good):
  KEDA pod has a Kubernetes ServiceAccount
  That ServiceAccount is annotated with an IAM role ARN
  AWS automatically provides short-lived credentials (15 min expiry)
  No key to store, no key to rotate, no key to leak
```

---

## 🎬 Demo Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ PHASE 0 — Baseline (no load)                                        │
│                                                                     │
│  What you see:                                                      │
│    nginx_connections_active ≈ 3-6 (idle keep-alive connections)     │
│    DT trigger:  INACTIVE  (4 ≤ activationThreshold 7)               │
│    AMP trigger: INACTIVE  (5 ≤ activationThreshold 10)              │
│    nginx replicas: 1                                                │
│                                                                     │
│  What "keep-alive connections" are:                                 │
│    Browsers and clients open a TCP connection and KEEP it open      │
│    even when idle. This avoids the overhead of reconnecting for     │
│    every request. nginx counts these as "active connections" even   │
│    though no requests are in-flight.                                │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           │  5 load pods launched
                           │  Each pod: opens 1 TCP connection per second
                           │            holds it open for 20 seconds
                           │  After 20s: 5 pods × 20 conns = ~100 open connections
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ PHASE 1 — Dynatrace trigger fires first                             │
│                                                                     │
│  Why DT fires first:                                                │
│    DT OneAgent scrapes nginx exporter continuously.                 │
│    KEDA polls DT every 15s.                                         │
│    As soon as connections > 7, DT trigger activates.                │
│                                                                     │
│  Math:                                                              │
│    DT metric = 100 (2m fold of ~100 connections)                    │
│    100 > activationThreshold(7) → ACTIVE                           │
│    desiredReplicas = ceil(100 / 10) = 10                            │
│    nginx scales up ↑                                               │
│                                                                     │
│  Why AMP hasn't fired yet:                                          │
│    AMP scrapes every 60 seconds.                                    │
│    It might not have sampled since load started.                    │
│    Still waiting for the next 60s scrape tick.                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           │  5 more pods (10 total)
                           │  ~200 open connections
                           │  AMP 60s scraper fires and captures ~200
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ PHASE 2 — AMP trigger fires                                         │
│                                                                     │
│  Math:                                                              │
│    AMP metric = 200                                                 │
│    200 > activationThreshold(10) → ACTIVE                          │
│    desiredReplicas = ceil(200 / 20) = 10                            │
│                                                                     │
│  Both triggers now active:                                          │
│    DT  desired = 10                                                 │
│    AMP desired = 10                                                 │
│    HPA desired = max(10, 10) = 10 replicas                          │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           │  all load pods deleted
                           │  TCP connections close within 20s
                           │  metric drops back to 3-6
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ PHASE 3 — Cooldown and scale-down                                   │
│                                                                     │
│  Timeline:                                                          │
│    T+0s:  load pods deleted                                         │
│    T+20s: nc connections drain, metric drops below thresholds       │
│    T+20s: DT trigger → INACTIVE (next 15s poll)                    │
│    T+60s: AMP trigger → INACTIVE (next 60s scrape)                 │
│    T+60s: cooldownPeriod=60s starts counting                        │
│    T+120s: cooldown expires + stabilizationWindow satisfied         │
│    T+120s: HPA scales nginx back to 1 replica ↓                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🐛 Lessons Learned

Every one of these bugs was real — hit during this exact setup.

| # | What went wrong | Why it happened | How to fix it |
|---|---|---|---|
| 1 | DT trigger never fired even under heavy load | KEDA defaults `from=now-2h`. With `:fold`, a 3-min load spike in a 2h window barely moves the average (117 min idle × 3 + 3 min load × 100 = avg 5.4). Threshold was 7. | Set `from: "now-2m"` explicitly in trigger metadata |
| 2 | AMP metric always returned `0` despite nginx running | AWS managed scraper (`pod_exporter` job) doesn't add a `namespace` label. Query `{namespace="keda-testing"}` matched nothing. | Remove namespace filter — use `sum(nginx_connections_active)` |
| 3 | nginx oscillated between 1 and 2 replicas at idle with no load | `activationThreshold=3` was below the idle baseline of 3-6 connections. Triggers were permanently "active" even at idle. KEDA was always calculating 1-2 desired replicas. | Raise activationThreshold above idle: DT=7, AMP=10 |
| 4 | Replicas stayed at 2-3 for 5 minutes after load was removed | Kubernetes HPA default `stabilizationWindowSeconds=300` — it holds the highest replica recommendation for 5 min before scaling down | Set `stabilizationWindowSeconds: 60` in ScaledObject `advanced` section |
| 5 | `unsupported protocol scheme ""` error from KEDA | SealedSecret was sealed with `host: <your-env-id>.live.dynatrace.com` (no `https://`). KEDA tried to connect with empty scheme. | Reseal with full URL: `https://<your-env-id>.live.dynatrace.com` |
| 6 | `404 Not Found` on every Dynatrace API call | Used `apps.dynatrace.com` as the host. Metrics v2 API (`/api/v2/metrics/query`) only exists on `live.dynatrace.com` | Change host to `live.dynatrace.com` |
| 7 | `400 Bad Request: Illegal transform operator :sum` | Tried using `.count` suffix metric with `:sum` transform. Counter metrics don't support `:sum`. | Remove `:sum` from the metricSelector, or switch to a gauge metric |
| 8 | AMP metric stuck at 3 even with 10 load pods sending requests | `wget` requests complete in under 1 millisecond. AMP scrapes every 60s. By the time AMP samples, all connections are already closed. | Switch to `nc` (netcat) which holds a TCP connection open for 20s. AMP always catches them. |
