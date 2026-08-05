# Argo CD Cheat Sheet

Quick reference for everyday `argocd` / `kubectl` commands when operating Argo CD Applications.

For install, GitOps layout, and promotion workflows, see [`argocd-integration-guide.md`](./argocd-integration-guide.md).

---

## Table of Contents

1. [Install CLI & Login](#install-cli--login)
2. [Apps — List / Get / Status](#apps--list--get--status)
3. [Sync, Refresh & Wait](#sync-refresh--wait)
4. [Diff, History & Rollback](#diff-history--rollback)
5. [Create / Update / Delete Apps](#create--update--delete-apps)
6. [Repos & Clusters](#repos--clusters)
7. [Projects](#projects)
8. [ApplicationSets](#applicationsets)
9. [kubectl Shortcuts (CRDs)](#kubectl-shortcuts-crds)
10. [Logs & Debugging](#logs--debugging)
11. [Sync Status & Health Legend](#sync-status--health-legend)
12. [Useful One-Liners](#useful-one-liners)

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

```bash
# Namespace where Argo CD lives (default)
export ARGOCD_NS=argocd

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
