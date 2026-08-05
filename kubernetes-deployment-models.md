# Common Kubernetes Deployment Models

A practical guide to how workloads get released onto a cluster — native Deployment strategies, traffic-shifting patterns, progressive delivery, and how they fit with GitOps.

---

## Table of Contents

1. [Mental Model](#1-mental-model)
2. [Recreate](#2-recreate)
3. [Rolling Update](#3-rolling-update)
4. [Blue-Green](#4-blue-green)
5. [Canary](#5-canary)
6. [A/B & Header-Based Routing](#6-ab--header-based-routing)
7. [Shadow / Traffic Mirroring](#7-shadow--traffic-mirroring)
8. [Progressive Delivery Controllers](#8-progressive-delivery-controllers)
9. [Batch & Job-Style Deploys](#9-batch--job-style-deploys)
10. [GitOps Promotion Models](#10-gitops-promotion-models)
11. [Choosing a Model](#11-choosing-a-model)
12. [Command Sheet](#12-command-sheet)
13. [Anti-Patterns](#13-anti-patterns)

---

## 1. Mental Model

A **deployment model** answers:

> How do we move from version **N** to **N+1** while controlling risk, downtime, and traffic?

| Layer | What it controls |
|-------|------------------|
| Pod / ReplicaSet strategy | How new pods replace old ones (`Recreate` vs `RollingUpdate`) |
| Service / Ingress / mesh | Which pods receive user traffic |
| Release controller | Automation, analysis, automatic rollback (Argo Rollouts, Flaggeer, …) |
| Git / CI | What image tag is desired and when it is promoted |

```
Git / CI  →  desired image & config
                │
                ▼
        Workload controller          ← rolling / recreate / Rollout CR
                │
                ▼
        Traffic routing              ← Service, Ingress, mesh, weights
                │
                ▼
             Users
```

Native `Deployment` only covers **Recreate** and **RollingUpdate**. Blue-green, canary, and A/B need **two revisions + traffic control** (or a progressive delivery tool).

---

## 2. Recreate

**All old pods terminate, then new pods start.** Brief downtime is expected.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: migrator-ui
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: migrator-ui
  template:
    metadata:
      labels:
        app: migrator-ui
    spec:
      containers:
        - name: app
          image: ghcr.io/org/migrator-ui:1.2.0
```

| Fit | Avoid when |
|-----|------------|
| Dev clusters, jobs with exclusive locks, apps that cannot run two versions together (hard schema break) | User-facing APIs that need zero downtime |

```bash
kubectl set image deploy/migrator-ui app=ghcr.io/org/migrator-ui:1.2.0
kubectl rollout status deploy/migrator-ui
```

---

## 3. Rolling Update

**Default Deployment strategy.** New pods come up gradually; old pods drain. Traffic stays on one Service; readiness gates who gets requests.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1          # or 25%
      maxSurge: 1                # or 25%
  minReadySeconds: 10
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: ghcr.io/org/api:1.4.2
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
```

| Knob | Effect |
|------|--------|
| `maxSurge` | Extra pods allowed above desired count during rollout |
| `maxUnavailable` | Pods that may be down during rollout |
| `minReadySeconds` | Delay before a pod counts as available |
| readinessProbe | **Required** so bad pods don’t take traffic |

```bash
kubectl set image deploy/api api=ghcr.io/org/api:1.4.2
kubectl rollout status deploy/api
kubectl rollout pause deploy/api
kubectl rollout resume deploy/api
kubectl rollout undo deploy/api
kubectl rollout history deploy/api
```

| Fit | Avoid when |
|-----|------------|
| Most services; compatible N and N+1; simple ops | Breaking API/schema between versions; need % traffic experiments; need instant cutover |

**GPU note:** rolling updates with `maxSurge: 1` need spare GPU capacity. If the pool is full, use `maxSurge: 0` and tolerate brief capacity loss, or use blue-green on a second node pool.

---

## 4. Blue-Green

Run **two full stacks** (blue = live, green = new). Switch traffic all at once when green is healthy. Instant rollback = switch back.

```
          ┌──────── blue (v1) ────────┐
Users ──► │ Service / Ingress        │
          └──────── green (v2) ───────┘
                    ▲
              flip selector / weight 100%
```

### Pattern with two Deployments + one Service

```yaml
# blue and green Deployments differ by label version: blue|green
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
    version: blue          # flip to green for cutover
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Deploy green alongside blue
kubectl apply -f api-green.yaml
kubectl -n prod rollout status deploy/api-green

# Cut over
kubectl patch svc api -p '{"spec":{"selector":{"app":"api","version":"green"}}}'

# Rollback
kubectl patch svc api -p '{"spec":{"selector":{"app":"api","version":"blue"}}}'
```

With Ingress / Gateway API / service mesh, cutover is often a **weight 0→100** or **route backend** change instead of a Service selector flip.

| Fit | Avoid when |
|-----|------------|
| Need fast cutover & rollback; incompatible versions; DB migrations gated behind switch | Cluster cannot afford 2× replicas/GPUs during the overlap |

---

## 5. Canary

Send a **small % of traffic** to the new version, watch metrics/errors, then ramp up (or roll back).

```
Users ──► 90% → stable (v1)
       └─ 10% → canary (v2)
```

### Lightweight: two Deployments + weighted Ingress (nginx example)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-canary
                port:
                  number: 80
# Primary Ingress points at api-stable with no canary annotations
```

### Replica-ratio canary (no mesh)

Scale canary replicas as a fraction of stable (blunt: Service picks pods evenly by readiness, not true % if counts differ by node locality — prefer Ingress/mesh weights).

```bash
kubectl scale deploy/api-stable --replicas=9
kubectl scale deploy/api-canary --replicas=1    # ~10% if both behind one Service
```

| Fit | Avoid when |
|-----|------------|
| Reduce blast radius; validate in production; automate with metrics | No metrics/alerts; versions incompatible for mixed traffic |

Promote: increase weight (10 → 25 → 50 → 100) then retire stable, or flip blue-green style at the end.

---

## 6. A/B & Header-Based Routing

Route by **identity or request attributes**, not only by percentage — e.g. sticky cohort, `X-Canary: true`, cookie, mobile app version.

```yaml
# Contour / Istio / Gateway API style (conceptual)
# Match header X-Experiment: B → backend v2
# Everyone else → backend v1
```

Gateway API example sketch:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api
spec:
  parentRefs:
    - name: prod-gateway
  rules:
    - matches:
        - headers:
            - name: x-canary
              value: "true"
      backendRefs:
        - name: api-v2
          port: 80
    - backendRefs:
        - name: api-v1
          port: 80
```

| Fit | Avoid when |
|-----|------------|
| Experiments, internal dogfood, beta users | You only need gradual % ramp with no targeting |

---

## 7. Shadow / Traffic Mirroring

**Copy** live traffic to a new version **without serving responses** from it. Good for soak tests and catching errors before a real canary.

```
Users ──► v1 (primary responses)
       └─► v2 (mirrored; responses discarded)
```

Typically requires a **service mesh** or proxy that supports mirror (Istio `mirror`, Linkerd, some Gateways).

| Fit | Avoid when |
|-----|------------|
| High-risk changes; nondestructive read APIs; perf comparison | Requests are not idempotent / cause duplicate side effects on v2 |

Ensure the shadow stack **cannot** write to prod databases (separate credentials, dry-run mode, or read replicas).

---

## 8. Progressive Delivery Controllers

Tools that encode canary/blue-green as CRDs with **analysis and auto-rollback**.

### Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 10m }
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: ghcr.io/org/api:1.5.0
```

```bash
kubectl argo rollouts get rollout api -n prod
kubectl argo rollouts promote api -n prod
kubectl argo rollouts abort api -n prod
kubectl argo rollouts undo api -n prod
```

### Flagger

Watches a Deployment / Service and drives canary via mesh/Ingress metrics (Prometheus). Promote or rollback from analysis — similar outcome, different CRDs (`Canary`).

| Tool | Notes |
|------|-------|
| Argo Rollouts | Fits Argo CD GitOps well; CLI for promote/abort |
| Flagger | Strong mesh/Ingress integrations; MetricTemplate + Canary |
| Native Deployment | Rolling only — enough for many apps |

---

## 9. Batch & Job-Style Deploys

Not every release is a long-lived Deployment.

| Model | Use |
|-------|-----|
| `Job` | One-shot migration, training run, batch inference |
| `CronJob` | Scheduled ETL / retrain |
| JobSet / MPIJob / Volcano | Multi-node distributed training |
| “Deploy” = new Job with new image tag | Prefer immutable Jobs over mutating a live Deployment for batch |

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: schema-migrate-1-5-0
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: ghcr.io/org/api:1.5.0
          command: ["migrate", "up"]
```

**Ordering with apps:** run migrate Job (sync wave / PreSync hook) → then roll the API. See Argo CD hooks in the [integration guide](./argocd-integration-guide.md).

---

## 10. GitOps Promotion Models

How environments advance when Argo CD (or similar) is the deployer:

| Model | How it works |
|-------|--------------|
| **Path-per-env** | `overlays/dev` → `staging` → `prod`; promote = PR updating the next overlay’s image tag |
| **Branch-per-env** | `main`→dev, `staging`, `prod` branches; merge forward to promote |
| **App-of-Apps** | Root app spawns env apps; same overlay pattern underneath |
| **Progressive in-cluster** | Git jumps to new tag once; Rollouts/Flagger ramps traffic inside the env |

```
CI builds image @sha256:…  →  PR updates gitops overlay  →  Argo CD syncs
                                                         →  Rollout/canary steps (optional)
```

Promotion should update **Git** (digest-pinned image), not only `kubectl set image` on the cluster.

---

## 11. Choosing a Model

```
Need zero-ish downtime?
├─ no  → Recreate (or Job replace)
└─ yes
   ├─ Can N and N+1 run together?
   │  ├─ no  → Blue-green (full flip) + careful migrations
   │  └─ yes
   │     ├─ Want automatic metric gates?
   │     │  └─ yes → Argo Rollouts / Flagger canary
   │     ├─ Need user/header targeting?
   │     │  └─ yes → A/B / header routes
   │     ├─ Want soak without user impact?
   │     │  └─ yes → Shadow mirror, then canary
   │     └─ default → RollingUpdate (tune surge / probes)
   └─ Batch / train / migrate → Job (optionally gated before Deployment)
```

| Situation | Typical choice |
|-----------|----------------|
| Stateless API, compatible versions | RollingUpdate |
| Mobile app backend with breaking change | Blue-green |
| High-revenue path, good metrics | Canary (Rollouts/Flagger) |
| Internal dogfood cohort | Header A/B |
| GPU inference, tight capacity | Rolling with `maxSurge: 0` or blue-green on spare pool |
| Schema migrate + API | Job then Rolling / blue-green |
| Multi-env release | GitOps path/branch promote + one of the above in prod |

---

## 12. Command Sheet

```bash
# --- Rolling (Deployment) ---
kubectl set image deploy/<name> <container>=<image>
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>
kubectl rollout undo deploy/<name>
kubectl rollout undo deploy/<name> --to-revision=2
kubectl rollout pause deploy/<name>
kubectl rollout resume deploy/<name>
kubectl rollout restart deploy/<name>

# Inspect strategy
kubectl get deploy/<name> -o jsonpath='{.spec.strategy}{"\n"}'

# --- Blue-green cutover via Service selector ---
kubectl patch svc <svc> -p '{"spec":{"selector":{"app":"<app>","version":"green"}}}'

# --- Scale-based crude canary ---
kubectl scale deploy/<stable> --replicas=9
kubectl scale deploy/<canary> --replicas=1

# --- Argo Rollouts ---
kubectl argo rollouts list rollouts -n <ns>
kubectl argo rollouts get rollout <name> -n <ns> -w
kubectl argo rollouts promote <name> -n <ns>
kubectl argo rollouts abort <name> -n <ns>

# --- Verify who gets traffic ---
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl get pods -l app=<app> -o wide --show-labels
```

---

## 13. Anti-Patterns

| Avoid | Prefer |
|-------|--------|
| Rolling deploy with no readiness probe | Probe on a real dependency path (`/readyz`) |
| Canary with no metrics or abort criteria | Error rate / latency analysis + alerts |
| Blue-green that shares a breaking DB schema without expand/contract | Expand → deploy → contract migrations |
| `:latest` tags for any model | Digest-pinned images in Git |
| Manual `kubectl set image` as the system of record | GitOps promote (see [Argo CD guide](./argocd-integration-guide.md)) |
| Shadow traffic that writes to prod | Isolated credentials / dry-run consumers |
| `maxSurge` that oversubscribes GPUs | Plan capacity or use surge 0 / separate pool |

---

## Further Reading

- [Deployments — Kubernetes docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Argo Rollouts](https://argoproj.github.io/rollouts/)
- [Flagger](https://docs.flagger.app/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [argocd-integration-guide.md](./argocd-integration-guide.md) — GitOps sync & promotion
- [argocd-cheat-sheet.md](./argocd-cheat-sheet.md) — sync / rollback commands

---

*Companion to `kubernetes-cheat-sheet.md`. Adjust Ingress annotations, mesh CRDs, and Rollout analysis templates for your platform.*
