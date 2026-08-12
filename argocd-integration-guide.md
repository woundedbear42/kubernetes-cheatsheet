# Argo CD Integration Guide

A practical guide to integrating Kubernetes with Argo CD — install options, GitOps layout, Application manifests, day-to-day workflow, and troubleshooting.

For a command-focused quick reference, see [`argocd-cheat-sheet.md`](./argocd-cheat-sheet.md).

---

## Table of Contents

1. [Mental Model](#1-mental-model)
2. [Install Argo CD](#2-install-argo-cd)
3. [Access the UI & CLI](#3-access-the-ui--cli)
4. [Git Repository Layout](#4-git-repository-layout)
5. [Register a Repo & Create an Application](#5-register-a-repo--create-an-application)
6. [End-to-End Workflow](#6-end-to-end-workflow)
7. [Sync Policies & Health](#7-sync-policies--health)
8. [App of Apps & ApplicationSets](#8-app-of-apps--applicationsets)
9. [Multi-Cluster & Environments](#9-multi-cluster--environments)
10. [Secrets & External Config](#10-secrets--external-config)
11. [RBAC for Argo CD](#11-rbac-for-argo-cd)
12. [CI Integration Pattern](#12-ci-integration-pattern)
13. [Useful Commands](#13-useful-commands)
14. [Troubleshooting](#14-troubleshooting)
15. [Anti-Patterns](#15-anti-patterns)

---

## 1. Mental Model

Argo CD is a **GitOps continuous delivery** controller for Kubernetes.

> **Git** holds desired state. Argo CD **watches** that state and **reconciles** the cluster until live matches Git.

| Piece | Role |
|-------|------|
| Git repo (manifests / Helm / Kustomize) | Source of truth for desired state |
| Argo CD Application | Declares *what* to sync (repo, path, revision) and *where* (cluster, namespace) |
| Argo CD controller | Detects drift, syncs, reports health |
| Cluster | Live state that Argo CD makes match Git |

```
Developer / CI
      │  commit / PR merge
      ▼
   Git repo  ──────── watches ────────►  Argo CD
   (desired)                              │
                                          │ apply / prune
                                          ▼
                                    Kubernetes API
                                      (live state)
```

**Important distinctions**

- Argo CD does **not** build images. CI builds and pushes images; Git (image tag / digest) is what Argo CD deploys.
- `kubectl apply` against a managed resource will be **overwritten** on the next sync (or show as OutOfSync until sync).
- Prefer **declarative Applications** in Git over one-off `argocd app create` in production.

---

## 2. Install Argo CD

### Quick install (upstream manifests)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd -w
```

### Helm (common for platforms)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --set server.service.type=ClusterIP
```

Pin chart / app versions in your own values file for reproducible installs. Many teams manage Argo CD itself with Argo CD (self-managed bootstrap) after the first install.

### Verify CRDs

```bash
kubectl api-resources | grep argoproj.io
# applications, applicationsets, appprojects, ...
```

---

## 3. Access the UI & CLI

### Initial admin password

```bash
# Username: admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Change or disable the initial secret after first login; integrate SSO (OIDC / Dex) for real users.

### Port-forward the UI

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# open https://localhost:8080
```

### CLI login

```bash
# Install: https://argo-cd.readthedocs.io/en/stable/cli_installation/
argocd login localhost:8080 --insecure   # with port-forward
# or: argocd login argocd.example.com

argocd account get-user-info
argocd version
```

---

## 4. Git Repository Layout

Pick a layout and stick to it. Two common patterns:

### Pattern A — App repo + config repo (recommended)

```
# app source (CI builds images)
my-service/
  Dockerfile
  src/
  .github/workflows/ci.yml   # build → push image → update config repo tag

# desired state (Argo CD watches this)
gitops/
  apps/
    my-service/
      base/                  # Kustomize base or raw YAML
      overlays/
        dev/
        staging/
        prod/
  argocd/
    projects/
      platform.yaml
    applications/
      my-service-dev.yaml
      my-service-prod.yaml
```

### Pattern B — Monorepo

```
platform-gitops/
  apps/
    frontend/
    api/
  infrastructure/
    ingress/
    monitoring/
  root/                      # App-of-Apps or ApplicationSet
    root-app.yaml
```

| Source type | When to use |
|-------------|-------------|
| Directory of YAML | Simple apps, learning |
| Kustomize | Overlays per env without templating |
| Helm chart (path or repo) | Chart-native apps; pass `helm.values` / `valueFiles` |
| Jsonnet / custom plugin | Advanced / org-specific tooling |

---

## 5. Register a Repo & Create an Application

### Connect a Git repository

```bash
# HTTPS + token / password
argocd repo add https://github.com/org/gitops.git \
  --username git \
  --password <token>

# Or SSH
argocd repo add git@github.com:org/gitops.git --ssh-private-key-path ~/.ssh/argocd_deploy

# Declarative (preferred): create a Secret labeled for Argo CD
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitops-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/org/gitops.git
  password: <token>
  username: git
```

### Declarative Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # cascade delete on app delete (optional)
spec:
  project: default                            # or a custom AppProject
  source:
    repoURL: https://github.com/org/gitops.git
    targetRevision: main
    path: apps/my-service/overlays/dev
  destination:
    server: https://kubernetes.default.svc    # in-cluster
    namespace: my-service-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd/applications/my-service-dev.yaml
# or
argocd app create -f argocd/applications/my-service-dev.yaml
```

### Helm source example

```yaml
spec:
  source:
    repoURL: https://github.com/org/gitops.git
    targetRevision: main
    path: charts/my-service
    helm:
      valueFiles:
        - values-dev.yaml
      parameters:
        - name: image.tag
          value: "1.4.2"
```

---

## 6. End-to-End Workflow

This is the typical day-to-day loop once Argo CD is installed and Applications exist.

```
1. Developer opens PR (app code and/or gitops manifests)
2. CI builds image, runs tests
3. On merge (or promote job): CI updates image tag/digest in the gitops repo
4. Argo CD detects Git change (webhook or poll)
5. Sync applies manifests → cluster converges
6. Argo CD health checks report Healthy / Degraded
7. Rollback = revert Git commit (or pin targetRevision) → sync again
```

### Detailed steps

**A. Change application code**

```bash
# In app repo
git checkout -b feat/faster-api
# ... code changes ...
git commit -m "Improve API latency"
git push origin feat/faster-api
# open PR → CI builds image ghcr.io/org/my-service:<sha>
```

**B. Promote via Git (config / gitops repo)**

```bash
# After image is published, update the desired tag in gitops
# e.g. apps/my-service/overlays/dev/kustomization.yaml
images:
  - name: ghcr.io/org/my-service
    newTag: "a1b2c3d"
```

Or with a helper:

```bash
cd gitops
kustomize edit set image ghcr.io/org/my-service=ghcr.io/org/my-service:a1b2c3d
git commit -am "deploy my-service a1b2c3d to dev"
git push
```

**C. Argo CD syncs**

- With **automated sync**: merge to the watched branch → Argo CD syncs within poll interval (or immediately via webhook).
- With **manual sync**:

```bash
argocd app sync my-service-dev
argocd app wait my-service-dev --health
argocd app get my-service-dev
```

**D. Verify**

```bash
kubectl -n my-service-dev get deploy,pods,svc
argocd app resources my-service-dev
```

**E. Rollback**

```bash
# Preferred: revert the gitops commit
git revert <commit> && git push

# Or sync to a previous Git revision without changing default branch tip long-term
argocd app sync my-service-dev --revision <git-sha>

# History from Argo CD
argocd app history my-service-dev
argocd app rollback my-service-dev <history-id>
```

Git revert is clearer for audit trails; `rollback` is useful for emergencies.

### Webhook (faster than polling)

Point GitHub/GitLab webhook at `https://argocd.example.com/api/webhook` so Argo CD refreshes on push instead of waiting for the default poll (~3 minutes).

---

## 7. Sync Policies & Health

| Setting | Meaning |
|---------|---------|
| `automated.prune` | Delete cluster resources removed from Git |
| `automated.selfHeal` | Revert manual kubectl drift |
| `syncOptions: CreateNamespace=true` | Create destination namespace if missing |
| `syncOptions: ServerSideApply=true` | Prefer SSA for large CRDs / field ownership |
| `retry` | Retry failed syncs with backoff |

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
  syncOptions:
    - CreateNamespace=true
    - PruneLast=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

**Health** — Argo CD understands Deployments, StatefulSets, Ingress, and many CRDs. Custom resources may need [custom health Lua](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/) in `argocd-cm`.

**Sync waves / hooks** — Order multi-resource deploys:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"          # lower waves first
    argocd.argoproj.io/hook: PreSync            # PreSync | Sync | PostSync | ...
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

---

## 8. App of Apps & ApplicationSets

### App of Apps — mental model

**App of Apps** means one **root** Application whose Git path contains other `Application` (and often `AppProject`) manifests. Syncing the root creates/updates the child Applications; those children then sync the real workloads.

```
Git: argocd/applications/*.yaml     (Application CRs)
              │
              ▼
        Root Application  ──sync──►  creates child Applications in the argocd namespace
              │
              ├──────────────►  my-service-dev   ──sync──►  apps/my-service/overlays/dev
              ├──────────────►  my-service-prod  ──sync──►  apps/my-service/overlays/prod
              └──────────────►  ingress-nginx    ──sync──►  infra/ingress
```

Use it to **bootstrap a platform from a single entrypoint**: install Argo CD once, apply (or declare) the root app, and the rest of the estate comes from Git.

### Repo layout for App of Apps

```
gitops/
  apps/                              # workload desired state
    my-service/
      base/
      overlays/
        dev/
        staging/
        prod/
    other-service/
      ...
  infrastructure/
    ingress/
    cert-manager/
  argocd/
    projects/
      platform.yaml                  # AppProject(s)
      team-a.yaml
    applications/                    # <-- root Application points HERE
      my-service-dev.yaml
      my-service-staging.yaml
      my-service-prod.yaml
      ingress.yaml
    root-app.yaml                    # the root Application itself (optional in-repo)
```

**Convention:** children live under something like `argocd/applications/`; workloads stay under `apps/` or `infrastructure/`. Don’t mix Application CRs and Deployments in the same path the root syncs — the root should mostly render `Application` / `AppProject` kinds.

### Root Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  # finalizer optional: cascade-delete children when root is deleted
  # finalizers:
  #   - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops.git
    targetRevision: main
    path: argocd/applications          # directory of child Application manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd                   # Applications must live in the Argo CD namespace
  syncPolicy:
    automated:
      prune: true                      # delete child Apps removed from Git
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Bootstrap options:

```bash
# One-time imperative bootstrap (then manage root from Git too, if you want)
argocd app create root \
  --repo https://github.com/org/gitops.git \
  --path argocd/applications \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated \
  --auto-prune --self-heal

# Or apply the root Application YAML once
kubectl apply -n argocd -f argocd/root-app.yaml
```

After that, **adding a new service = commit a new child Application YAML**; the root auto-sync picks it up.

### Child Application examples

**Workload (Kustomize overlay):**

```yaml
# argocd/applications/my-service-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-dev
  namespace: argocd
  labels:
    app.kubernetes.io/name: my-service
    env: dev
spec:
  project: platform
  source:
    repoURL: https://github.com/org/gitops.git
    targetRevision: main
    path: apps/my-service/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: my-service-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Prod child (often manual sync):**

```yaml
# argocd/applications/my-service-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-prod
  namespace: argocd
  labels:
    app.kubernetes.io/name: my-service
    env: prod
spec:
  project: platform
  source:
    repoURL: https://github.com/org/gitops.git
    targetRevision: main
    path: apps/my-service/overlays/prod
  destination:
    name: prod-cluster                 # registered cluster name, or server URL
    namespace: my-service
  # No automated sync — gated / manual for change control
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

**Infra child (Helm chart):**

```yaml
# argocd/applications/ingress.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/org/gitops.git
    targetRevision: main
    path: infrastructure/ingress       # chart or values wrapper in Git
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Sync cascade (what happens in order)

1. **You commit** a new/changed file under `argocd/applications/` (or change a workload overlay).
2. **Root Application** refreshes (webhook or poll), shows OutOfSync if child Application CRs changed.
3. **Root syncs** → Argo CD applies/updates `Application` objects in the `argocd` namespace.
4. **Each child Application** independently refreshes its own `spec.source` path.
5. **Child syncs** (auto or manual) → Deployments/Services/etc. converge on the target cluster/namespace.
6. **Health rolls up:** root is Healthy when its resources (the child Apps) exist; **workload health is on the children** — check those for pod failures.

```
Commit child YAML  →  root OutOfSync  →  root sync  →  child App exists
Commit overlay     →  child OutOfSync →  child sync →  pods/services update
```

Useful commands:

```bash
argocd app get root
argocd app sync root                  # create/update child Applications only
argocd app list                       # should show root + children
argocd app get my-service-dev
argocd app sync my-service-dev        # sync the workload
argocd app wait my-service-dev --health --sync
```

### Ordering with sync waves (optional)

When children must appear in order (CRDs/operators before workloads), set waves on the **Application manifests** the root syncs:

```yaml
metadata:
  name: cert-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---
metadata:
  name: my-service-dev
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

Lower waves sync first on the root. Within a child, use waves/hooks on the workload resources as usual.

### App of Apps — common pitfalls

| Pitfall | What goes wrong | Fix |
|---------|-----------------|-----|
| Root `destination.namespace` not `argocd` | Child Applications land in the wrong namespace; controller ignores them | Destination namespace = Argo CD install namespace |
| Root path includes workload YAML | Root tries to sync Deployments into `argocd` | Root path = Application/AppProject manifests only |
| No auto-prune on root | Deleted child YAML leaves orphan Applications | `automated.prune: true` on root (with care) |
| Expecting root health = pod health | Root shows Healthy while an app is Broken | Monitor **child** Apps for workload health |
| Giant single root with hundreds of hand-written children | YAML fatigue; drift between envs | Prefer **ApplicationSets** for generation; keep App of Apps for bootstrap / sparse roots |
| `kubectl apply` children outside Git | Drift from the root’s desired set; next prune may delete them | Children only via Git + root sync |
| Finalizer + delete root casually | Cascade deletes all children and their resources | Understand finalizers before enabling; prefer prune via Git removes |
| Same project `destination`/`source` too open | Any child can point anywhere | Tighten AppProjects; separate projects per team |

### App of Apps vs ApplicationSets

| | **App of Apps** | **ApplicationSets** |
|--|-----------------|---------------------|
| How children appear | Explicit YAML files synced by a root Application | Generator + template creates Applications |
| Best for | Bootstrap, small/medium estates, infra apps you want fully explicit | Multi-env, multi-cluster, PR previews, identical app shapes at scale |
| Change to add N clusters | N new YAML files (or copy/paste) | Often one generator element / cluster secret |
| SE soundbite | “One app installs the platform.” | “One template stamps out the fleet.” |

They compose: a root App of Apps can include an ApplicationSet manifest, or you bootstrap with a root app and let ApplicationSets own the high-cardinality children.

### ApplicationSet (generate many apps)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-service-envs
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: my-service-dev
          - env: staging
            namespace: my-service-staging
          - env: prod
            namespace: my-service-prod
  template:
    metadata:
      name: 'my-service-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops.git
        targetRevision: main
        path: 'apps/my-service/overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

Use ApplicationSets for multi-env, multi-cluster, or pull-request preview environments (SCM generator).

---

## 9. Multi-Cluster & Environments

### Register an external cluster

```bash
argocd cluster add <kube-context-name>
argocd cluster list
```

Destination in the Application:

```yaml
destination:
  name: prod-cluster          # or server: https://<api>
  namespace: my-service
```

### Environment promotion pattern

| Env | Git source | Sync |
|-----|------------|------|
| dev | `overlays/dev` on `main` | Auto-sync |
| staging | `overlays/staging` | Auto or manual after smoke tests |
| prod | `overlays/prod` | Manual sync or gated PR into `prod` branch / path |

Promotion = **merge or commit that updates the next env’s image tag / overlay**, not a direct push to the cluster.

---

## 10. Secrets & External Config

Do **not** commit plaintext Secrets to Git.

| Approach | Notes |
|----------|--------|
| [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) | Encrypt for a cluster; commit SealedSecret |
| [External Secrets Operator](https://external-secrets.io/) | Sync from AWS/GCP/Azure/Vault into Secret objects |
| [SOPS + KSOPS / helm-secrets](https://github.com/getsops/sops) | Encrypt values in Git; decrypt at sync |
| Argo CD + Vault plugin | Inject at render time (operationally heavier) |

Argo CD repo credentials and cluster credentials live as Secrets in the `argocd` namespace — protect that namespace with tight RBAC.

---

## 11. RBAC for Argo CD

Argo CD has its **own** RBAC (in `argocd-rbac-cm`), separate from Kubernetes RBAC.

```yaml
# ConfigMap argocd-rbac-cm
policy.csv: |
  p, role:developer, applications, get, default/*, allow
  p, role:developer, applications, sync, default/*, allow
  p, role:developer, applications, action/*, default/*, deny
  g, alice@example.com, role:developer
```

Use **AppProjects** to scope which repos, clusters, and namespaces a team may deploy to:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-ai
  namespace: argocd
spec:
  description: AI team applications
  sourceRepos:
    - 'https://github.com/org/gitops.git'
  destinations:
    - namespace: 'ai-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

Map SSO groups into Argo CD roles; avoid sharing the local `admin` account.

---

## 12. CI Integration Pattern

Argo CD is CD, not CI. Keep the boundary clean:

```
┌──────────── CI ────────────┐     ┌────────── GitOps / CD ──────────┐
│ test → build → push image  │ ──► │ update tag in gitops repo       │
│                            │     │ Argo CD syncs cluster from Git  │
└────────────────────────────┘     └─────────────────────────────────┘
```

**Minimal CI job after a successful image push:**

```bash
# Example: update digests/tags and open or push to gitops
git clone https://github.com/org/gitops.git
cd gitops
cd apps/my-service/overlays/dev
kustomize edit set image ghcr.io/org/my-service=ghcr.io/org/my-service@sha256:...
git config user.name "ci-bot"
git config user.email "ci-bot@example.com"
git commit -am "deploy my-service ${GIT_SHA}"
git push
```

Prefer **immutable digests** (`@sha256:...`) over moving tags like `latest`.

Optional: CI can call `argocd app sync` / `argocd app wait` for environments that are manual or to fail the pipeline if sync/health fails — still keep Git as the desired-state record.

---

## 13. Useful Commands

```bash
# Apps
argocd app list
argocd app get my-service-dev
argocd app sync my-service-dev
argocd app sync my-service-dev --prune
argocd app wait my-service-dev --health --timeout 300
argocd app diff my-service-dev
argocd app history my-service-dev
argocd app rollback my-service-dev <id>
argocd app delete my-service-dev

# Refresh from Git without syncing
argocd app get my-service-dev --refresh
argocd app get my-service-dev --hard-refresh

# Repos / clusters / projects
argocd repo list
argocd cluster list
argocd proj list

# kubectl equivalents for Application CRDs
kubectl -n argocd get applications
kubectl -n argocd get applicationsets
kubectl -n argocd describe application my-service-dev
kubectl -n argocd get appprojects
```

---

## 14. Troubleshooting

| Symptom | Checks |
|---------|--------|
| Unknown / ComparisonError | Repo credentials, path, `targetRevision`, Helm/Kustomize render errors: `argocd app get` / controller logs |
| OutOfSync that won’t clear | Diff: `argocd app diff`; ignore differences (`ignoreDifferences`) for controller-owned fields; confirm selfHeal/prune |
| Sync failed | `kubectl -n argocd logs -l app.kubernetes.io/name=argocd-application-controller`; Events on the Application |
| Progressing forever | Pods Pending / CrashLoop; image pull secrets; readiness probes |
| Healthy but wrong version | Wrong overlay/path or tag not committed; confirm Git SHA Argo CD resolved |
| Permission denied on sync | AppProject destination/source whitelist; Argo CD RBAC; destination cluster RBAC for the Argo CD SA |

```bash
# Controller & server logs
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-application-controller --tail=100
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-server --tail=100

# What Git revision is live?
argocd app get my-service-dev -o json | jq '{sync:.status.sync, health:.status.health, revision:.status.sync.revision}'
```

---

## 15. Anti-Patterns

| Avoid | Prefer |
|-------|--------|
| `kubectl apply` as the release process for Argo-managed apps | Commit to Git and sync |
| Storing plaintext Secrets in the gitops repo | Sealed Secrets / ESO / SOPS |
| One giant Application for the whole cluster | App of Apps or ApplicationSets with clear boundaries |
| Deploying `:latest` | Digest-pinned images |
| Sharing `admin` + initial password long-term | SSO + least-privilege Argo CD roles |
| CI talking only to the cluster, never updating Git | CI updates Git; Argo CD updates the cluster |
| Auto-sync + prune on prod with no review | PRs into prod overlays; manual sync or protected branches |

---

## Quick Decision Tree

```
New service to deploy with Argo CD?
├─ Manifests in Git? ── no ──► Add Kustomize/Helm under gitops/apps/<svc>
├─ Application exists?
│  ├─ no ──► Add Application (or ApplicationSet element) under argocd/
│  └─ yes ──► Point path/revision at the right overlay
├─ Env promotion?
│  └─ Update image tag in that env’s overlay via PR
└─ Incident / bad release?
   └─ Revert gitops commit (or argocd app rollback) → sync → verify health
```

---

## Further Reading

- [Argo CD documentation](https://argo-cd.readthedocs.io/)
- [Core concepts](https://argo-cd.readthedocs.io/en/stable/core_concepts/)
- [Application specification](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [Best practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Declarative setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)

---

*Companion to `kubernetes-cheat-sheet.md` and `kubernetes-rbac-advanced-guide.md`. Adjust repo URLs, namespaces, and SSO group names for your platform.*
