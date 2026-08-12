# Argo CD Cheat Sheet

Quick reference for everyday `argocd` / `kubectl` commands when operating Argo CD Applications.

For install, GitOps layout, and promotion workflows, see [`argocd-integration-guide.md`](./argocd-integration-guide.md).  
For core Kubernetes object definitions (Pod, ConfigMap, CRD, …), see [`kubernetes-cheat-sheet.md`](./kubernetes-cheat-sheet.md#key-concepts-definitions).

---

## Table of Contents

1. [Key Concepts (Definitions)](#key-concepts-definitions)
2. [Install CLI & Login](#install-cli--login)
3. [Apps — List / Get / Status](#apps--list--get--status)
4. [Sync, Refresh & Wait](#sync-refresh--wait)
5. [Diff, History & Rollback](#diff-history--rollback)
6. [Create / Update / Delete Apps](#create--update--delete-apps)
7. [Repos & Clusters](#repos--clusters)
8. [Projects](#projects)
9. [ApplicationSets](#applicationsets)
10. [kubectl Shortcuts (CRDs)](#kubectl-shortcuts-crds)
11. [Logs & Debugging](#logs--debugging)
12. [Sync Status & Health Legend](#sync-status--health-legend)
13. [Useful One-Liners](#useful-one-liners)

---

## Key Concepts (Definitions)

Argo CD terms you’ll see in the UI, CLI, and CRDs. These are themselves Kubernetes **Custom Resources** once Argo CD is installed.

### Core GitOps objects

| Term | Definition |
|------|------------|
| **GitOps** | Desired cluster state lives in Git; a controller continuously reconciles the live cluster to match. |
| **Application** | Argo CD CR that declares *what* to deploy (repo, path/chart, revision) and *where* (cluster, namespace). Primary unit you sync and monitor. |
| **AppProject (Project)** | Tenancy boundary for Applications: which repos, clusters, and namespaces are allowed, plus RBAC roles. |
| **ApplicationSet** | CR that **generates** many Applications from a template + generators (list, git, cluster, SCM/PR, …). Scale-out alternative to hand-writing every Application. |
| **App of Apps** | Pattern: one root Application whose path contains other Application manifests; syncing the root bootstraps child apps. |
| **CRD** | CustomResourceDefinition — how Kubernetes learns new kinds. Argo CD installs CRDs for Application, ApplicationSet, AppProject. |
| **Custom Resource (CR)** | An instance of a CRD kind (e.g. one `Application` named `my-service-dev`). |

### Sync & health

| Term | Definition |
|------|------------|
| **Sync** | Apply the desired Git/Helm/Kustomize state to the target cluster so live matches desired. |
| **Refresh** | Re-read Git (and rediscover live state) without necessarily applying; updates OutOfSync detection. |
| **Hard refresh** | Invalidate caches and re-render manifests from source — use when diffs look stale. |
| **Sync status** | Whether live matches Git: `Synced` / `OutOfSync` / `Unknown`. |
| **Health status** | Whether resources are operational: `Healthy` / `Progressing` / `Degraded` / `Missing` / … |
| **OutOfSync (drift)** | Live objects differ from desired (manual `kubectl` edits, failed apply, or new Git commits not yet synced). |
| **Auto-sync** | Application `syncPolicy.automated` — Argo CD syncs when Git changes (optionally prune + self-heal). |
| **Prune** | Delete cluster resources that exist live but were removed from Git. |
| **Self-heal** | Automatically re-sync when someone mutates live state away from Git. |
| **Sync wave** | Ordering hint (`argocd.argoproj.io/sync-wave`) so resource groups apply in sequence (e.g. CRDs before workloads). |
| **Hook** | Lifecycle Job/resource run around sync (`PreSync`, `Sync`, `PostSync`, …) — migrations, smoke tests. |
| **Revision** | Git SHA, Helm chart version, or other source version currently desired/synced. |

### Sources & destinations

| Term | Definition |
|------|------------|
| **Source** | Where manifests come from: Git path, Helm chart repo, or multi-source combo. |
| **Destination** | Target cluster API + namespace for the Application’s resources. |
| **Repo credential** | Secret Argo CD uses to clone private Git or pull Helm charts. |
| **Cluster secret** | Credential/config so Argo CD can manage an external cluster (hub-and-spoke). |
| **Kustomize** | Native manifest composition (base + overlays). Argo CD runs `kustomize build` on the path. |
| **Helm** | Templated charts + values; Argo CD renders and applies the chart as the Application source. |

### Control plane pieces

| Term | Definition |
|------|------------|
| **argocd-server** | API + UI you log into (`argocd login`, browser). |
| **application-controller** | Reconciles Applications: compare, sync, health. |
| **repo-server** | Clones repos / renders Helm & Kustomize manifests for the controller. |
| **applicationset-controller** | Watches ApplicationSets and creates/updates/deletes generated Applications. |
| **Redis** | Cache/session store used by Argo CD components. |
| **Notifications / Image Updater** (optional) | Adjacent tools: alert on sync/health; propose image tag updates in Git. |

### Related ecosystem (often asked together)

| Term | Definition |
|------|------------|
| **Argo Rollouts** | Progressive delivery CRDs (canary/blue-green) *inside* a cluster; complements Argo CD sync. |
| **Kargo** | Continuous *promotion* across stages (dev→staging→prod); updates Git for Argo CD to sync. |
| **ApplicationSet generator** | Strategy that supplies parameters to the Application template (e.g. one Application per cluster). |

### Quick “which Argo object do I want?”

| You need to… | Start with |
|--------------|------------|
| Deploy one service/env from Git | **Application** |
| Stamp out many apps (multi-env/cluster/PR) | **ApplicationSet** |
| Bootstrap a whole platform folder of apps | **App of Apps** (root Application) |
| Limit which teams can deploy where | **AppProject** |
| Force live to match Git now | **Sync** (optionally prune) |
| Detect manual cluster edits | Sync status + **self-heal** |

---

## Install CLI & Login

```bash
# macOS
brew install argocd

# Linux (example)
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Initial admin password (in-cluster install)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Port-forward UI / API
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Login
argocd login localhost:8080 --insecure          # port-forward
argocd login argocd.example.com                 # ingress / LB
argocd login argocd.example.com --sso           # SSO

argocd account get-user-info
argocd version
argocd context                                  # saved contexts
argocd logout argocd.example.com
```

---

## Apps — List / Get / Status

```bash
argocd app list
argocd app list -o wide
argocd app list -p <project>
argocd app list -l env=prod

argocd app get <app>
argocd app get <app> -o yaml
argocd app get <app> -o json

# Resources managed by the app
argocd app resources <app>
argocd app manifests <app>                # rendered desired manifests

# Watch until Healthy + Synced
argocd app wait <app> --health --sync --timeout 300
```

| Field (from `app get`) | Meaning |
|------------------------|---------|
| Sync Status | Does live match Git? (`Synced` / `OutOfSync`) |
| Health | Kubernetes health (`Healthy` / `Progressing` / `Degraded` / …) |
| Revision | Git SHA (or chart version) currently synced |
| Destination | Target cluster + namespace |

---

## Sync, Refresh & Wait

```bash
# Sync (apply Git → cluster)
argocd app sync <app>
argocd app sync <app> --prune                 # delete orphaned resources
argocd app sync <app> --force                 # replace / force apply
argocd app sync <app> --dry-run
argocd app sync <app> --revision <git-sha>
argocd app sync <app> --resource :Service:<name>   # sync one resource
argocd app sync <app> --label app.kubernetes.io/name=<name>

# Refresh desired state from Git (no apply)
argocd app get <app> --refresh
argocd app get <app> --hard-refresh           # clear caches / re-render

# Wait helpers
argocd app wait <app> --sync
argocd app wait <app> --health
argocd app wait <app> --operation
argocd app wait <app> --suspended=false
```

**Automated sync** is configured on the Application (`syncPolicy.automated`); CLI sync is for manual / gated envs.

---

## Diff, History & Rollback

```bash
# Live vs desired
argocd app diff <app>
argocd app diff <app> --exit-code             # exit 1 if differences

# Sync history
argocd app history <app>
argocd app history <app> -o wide

# Rollback to a history id from `history`
argocd app rollback <app> <history-id>
argocd app rollback <app> <history-id> --prune

# Prefer Git revert for audited prod rollbacks:
#   git revert <sha> && git push
# then sync (or wait for auto-sync)
```

---

## Create / Update / Delete Apps

```bash
# From flags
argocd app create <app> \
  --repo https://github.com/org/gitops.git \
  --path apps/my-service/overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-service-dev \
  --revision main \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# From manifest (preferred in GitOps)
argocd app create -f application.yaml
argocd app create -f application.yaml --upsert

# Patch / set
argocd app set <app> --revision main
argocd app set <app> --path apps/my-service/overlays/prod
argocd app set <app> --helm-set image.tag=1.2.3
argocd app set <app> --kustomize-image ghcr.io/org/svc=ghcr.io/org/svc:abc123
argocd app set <app> --sync-policy none       # disable auto-sync
argocd app set <app> --sync-policy automated --auto-prune --self-heal

# Delete
argocd app delete <app>
argocd app delete <app> --cascade             # also delete cluster resources
argocd app delete <app> --yes
```

### Minimal Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-dev
  namespace: argocd
spec:
  project: default
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

```bash
kubectl apply -f application.yaml
```

---

## Repos & Clusters

```bash
# Repositories
argocd repo list
argocd repo add https://github.com/org/gitops.git \
  --username git --password <token>
argocd repo add git@github.com:org/gitops.git \
  --ssh-private-key-path ~/.ssh/argocd_deploy
argocd repo rm https://github.com/org/gitops.git
argocd repo get https://github.com/org/gitops.git

# Helm OCI / chart repos (as needed)
argocd repo add https://charts.example.com --type helm --name mycharts

# Clusters
argocd cluster list
argocd cluster add <kube-context>             # registers from local kubeconfig
argocd cluster get <server-or-name>
argocd cluster rm <server-or-name>
```

Declarative repo Secret:

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
  username: git
  password: <token>
```

---

## Projects

```bash
argocd proj list
argocd proj get <project>
argocd proj create <project> \
  --src https://github.com/org/gitops.git \
  --dest https://kubernetes.default.svc,*

argocd proj add-source <project> https://github.com/org/other.git
argocd proj add-destination <project> https://kubernetes.default.svc ai-*
argocd proj windows list <project>            # sync windows
```

```bash
kubectl -n argocd get appprojects
kubectl -n argocd edit appproject <project>
```

---

## ApplicationSets

An **ApplicationSet** generates many **Application** CRs from a template (see [definitions](#key-concepts-definitions)). Controllers create/update/delete the children; you sync the generated Applications like any other app.

```bash
kubectl -n argocd get applicationsets
kubectl -n argocd describe applicationset <name>
kubectl -n argocd get applications -l argocd.argoproj.io/application-set-name=<name>

# After editing the ApplicationSet in Git / cluster
kubectl -n argocd apply -f applicationset.yaml
# Generated Applications appear as Application CRs; sync them as usual
```

---

## kubectl Shortcuts (CRDs)

Argo CD’s API kinds are installed as **CRDs**. Use these when the CLI isn’t available or you want raw object status.

kubectl -n $ARGOCD_NS get applications
kubectl -n $ARGOCD_NS get app          # short name if available
kubectl -n $ARGOCD_NS get applicationsets
kubectl -n $ARGOCD_NS get appprojects

kubectl -n $ARGOCD_NS get application <app> -o yaml
kubectl -n $ARGOCD_NS describe application <app>

# Sync status / health from status subresource
kubectl -n $ARGOCD_NS get application <app> \
  -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}'

kubectl -n $ARGOCD_NS get application <app> \
  -o jsonpath='{.status.sync.revision}{"\n"}'

# Control plane pods
kubectl -n $ARGOCD_NS get pods
kubectl -n $ARGOCD_NS get svc argocd-server
```

| CRD | Short / kind |
|-----|----------------|
| Application | `applications.argoproj.io` |
| ApplicationSet | `applicationsets.argoproj.io` |
| AppProject | `appprojects.argoproj.io` |

---

## Logs & Debugging

```bash
# Controllers
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-application-controller --tail=100 -f
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-repo-server --tail=100 -f
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-server --tail=100 -f
kubectl -n argocd logs -l app.kubernetes.io/name=argocd-applicationset-controller --tail=50

# App-level
argocd app get <app>
argocd app actions list <app>
argocd app logs <app>                         # if supported for workload logs
argocd app resources <app>

# Common failure checks
argocd app get <app> --hard-refresh
argocd app diff <app>
argocd repo list                              # credentials / connection
kubectl -n argocd get events --sort-by=.lastTimestamp | tail -30
```

| Symptom | Try |
|---------|-----|
| ComparisonError / Unknown | Repo URL, credentials, path, `targetRevision`; repo-server logs |
| OutOfSync stuck | `argocd app diff`; prune; `ignoreDifferences` for controller fields |
| Sync failed | Application events + application-controller logs |
| Progressing forever | Destination pods Pending/CrashLoop; imagePullSecrets; probes |
| Permission denied | AppProject source/destination whitelist; Argo CD RBAC |

---

## Sync Status & Health Legend

| Sync | Meaning |
|------|---------|
| Synced | Live matches desired Git state |
| OutOfSync | Drift or pending Git changes |
| Unknown | Can’t compare (repo/render error) |

| Health | Meaning |
|--------|---------|
| Healthy | Resources ready |
| Progressing | Rolling out / starting |
| Degraded | Failures (crash, failed sync hooks, …) |
| Suspended | Spec paused (e.g. Deployment suspended) |
| Missing | Resource in Git not found live |
| Unknown | No health check / not evaluated |

---

## Useful One-Liners

```bash
# All apps OutOfSync
argocd app list -o json | jq -r '.[] | select(.status.sync.status!="Synced") | .metadata.name'

# All apps not Healthy
argocd app list -o json | jq -r '.[] | select(.status.health.status!="Healthy") | "\(.metadata.name)\t\(.status.health.status)"'

# Sync everything in a project
argocd app list -p team-ai -o name | xargs -n1 argocd app sync

# Hard refresh all apps
argocd app list -o name | xargs -n1 -I{} argocd app get {} --hard-refresh

# Current Git revision for an app
argocd app get <app> -o json | jq -r '.status.sync.revision'

# Rendered manifests to a file
argocd app manifests <app> > /tmp/<app>-manifests.yaml

# Who am I in Argo CD?
argocd account get-user-info

# Reset local admin password (break-glass; server pod exec — check current docs)
# Prefer SSO; rotate and remove initial secret after bootstrap
```

---

## Aliases (optional)

```bash
alias acd='argocd'
alias acdapps='argocd app list'
alias acdg='argocd app get'
alias acds='argocd app sync'
alias acdd='argocd app diff'
alias acdw='argocd app wait'
```

---

*Companion to `argocd-integration-guide.md` and `kubernetes-cheat-sheet.md`. Adjust server URLs, namespaces, and project names for your platform.*
