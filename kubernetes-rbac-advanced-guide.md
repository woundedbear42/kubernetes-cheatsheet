# Advanced Kubernetes RBAC Guide

A practical deep-dive into Role-Based Access Control for clusters — from core primitives to production patterns, debugging, and AI/ML workload isolation.

---

## Table of Contents

1. [Mental Model](#1-mental-model)
2. [Core Objects](#2-core-objects)
3. [API Groups, Verbs & Resources](#3-api-groups-verbs--resources)
4. [Built-in ClusterRoles](#4-built-in-clusterroles)
5. [Impersonation](#5-impersonation)
6. [Aggregation & Composition](#6-aggregation--composition)
7. [ServiceAccounts in Depth](#7-serviceaccounts-in-depth)
8. [Least-Privilege Patterns](#8-least-privilege-patterns)
9. [Multi-Tenancy & Namespace Isolation](#9-multi-tenancy--namespace-isolation)
10. [AI / GPU Workload RBAC](#10-ai--gpu-workload-rbac)
11. [Admission, Policies & Guardrails](#11-admission-policies--guardrails)
12. [Auditing & Continuous Verification](#12-auditing--continuous-verification)
13. [Debugging RBAC Failures](#13-debugging-rbac-failures)
14. [Anti-Patterns](#14-anti-patterns)
15. [Reference Command Sheet](#15-reference-command-sheet)

---

## 1. Mental Model

RBAC answers one question:

> **Who** can perform **which verbs** on **which resources** (optionally in **which namespace**)?

| Piece | Object | Scope |
|-------|--------|-------|
| Who | User, Group, or ServiceAccount | Identity |
| What they can do | Role / ClusterRole | Permission set |
| Binding | RoleBinding / ClusterRoleBinding | Grants Role → identity |

```
Identity ──binds──► Role/ClusterRole ──allows──► API verbs on resources
```

**Important distinctions**

- A **Role** is namespaced; a **ClusterRole** is cluster-scoped.
- A **RoleBinding** can reference a Role *or* ClusterRole, but the grant is always **limited to that binding’s namespace**.
- A **ClusterRoleBinding** grants ClusterRole permissions **cluster-wide**.
- Users/groups are not Kubernetes objects — they come from the authenticator (OIDC, certs, webhook, cloud IAM). ServiceAccounts *are* objects.

```bash
# Who am I?
kubectl auth whoami
kubectl config view --minify -o jsonpath='{.contexts[0].context.user}{"\n"}'
```

---

## 2. Core Objects

### Role (namespaced)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: ai-team
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### ClusterRole (cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
# Can also include namespaced resources — useful when bound via RoleBinding
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: ai-team
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: ai-engineers
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: training-runner
  namespace: ai-team
roleRef:
  kind: Role                 # or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-node-viewers
subjects:
- kind: Group
  name: sre-oncall
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

### Imperative creation

```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods,pods/log \
  -n ai-team

kubectl create clusterrole node-viewer \
  --verb=get,list,watch \
  --resource=nodes

kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=alice@example.com \
  --group=ai-engineers \
  --serviceaccount=ai-team:training-runner \
  -n ai-team

kubectl create clusterrolebinding cluster-node-viewers \
  --clusterrole=node-viewer \
  --group=sre-oncall
```

---

## 3. API Groups, Verbs & Resources

### Verbs

| Verb | Meaning |
|------|---------|
| `get` | Read a single named object |
| `list` | List collections (often implies get) |
| `watch` | Stream changes |
| `create` | Create |
| `update` | Full replace |
| `patch` | Partial update |
| `delete` | Delete one |
| `deletecollection` | Delete a list/collection |
| `*` | All verbs |

Special / subresource verbs you will hit often:

| Resource / subresource | Notes |
|------------------------|-------|
| `pods/log` | Read container logs |
| `pods/exec` | `kubectl exec` |
| `pods/portforward` | Port-forward |
| `pods/attach` | Attach to process |
| `pods/ephemeralcontainers` | `kubectl debug` |
| `services/proxy` | Proxy through API |
| `nodes/proxy` | Node proxy (dangerous) |
| `deployments/scale` | Scale subresource |
| `secrets` | Treat as sensitive — never grant casually |

### Discovering resources & API groups

```bash
kubectl api-resources
kubectl api-resources -o wide
kubectl api-resources --namespaced=true
kubectl api-resources --api-group=batch

# Exact resource name for RBAC (plural)
kubectl api-resources | grep -i nvidia
kubectl explain pods --api-version=v1
```

### Rule anatomy (advanced)

```yaml
rules:
# Core API group is ""
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
  resourceNames: ["gpu-smoke"]   # optional: only this name

# Non-resource URLs (ClusterRole only) — healthz, metrics, etc.
- nonResourceURLs: ["/healthz", "/metrics"]
  verbs: ["get"]

# Subresources
- apiGroups: [""]
  resources: ["pods/exec", "pods/log"]
  verbs: ["create", "get"]

# Wildcards (use sparingly)
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### `escalate` and `bind` — privileged verbs

Granting these is almost as powerful as cluster-admin:

| Verb | Resource | Risk |
|------|----------|------|
| `bind` | `roles` / `clusterroles` | Subject can bind any Role they can reference |
| `escalate` | `roles` / `clusterroles` | Subject can create Roles with *more* power than they have |
| `impersonate` | `users` / `groups` / `serviceaccounts` | Act as anyone |

```yaml
# Only platform admins should have this
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["bind", "escalate"]
```

---

## 4. Built-in ClusterRoles

| ClusterRole | Typical use |
|-------------|-------------|
| `cluster-admin` | Full control — break-glass only |
| `admin` | Full access within a namespace (via RoleBinding) |
| `edit` | Read/write most namespaced resources; **no** roles/rolebindings |
| `view` | Read-only most namespaced resources; **no** secrets by default |
| `system:node` | Kubelet |
| `system:controller:*` | Controllers |
| `system:discovery` | API discovery |

```bash
kubectl get clusterrole view -o yaml
kubectl get clusterrole edit -o yaml
kubectl get clusterrole admin -o yaml
kubectl describe clusterrole cluster-admin
```

**Gotcha:** Binding `view` via RoleBinding does **not** include Secrets. Binding `edit` does include Secrets — be careful in shared namespaces.

```bash
# Give a team edit in one namespace without cluster-admin
kubectl create rolebinding ai-edit \
  --clusterrole=edit \
  --group=ai-engineers \
  -n ai-team
```

---

## 5. Impersonation

Impersonation lets you test “as” another identity without their credentials — essential for RBAC design and support.

```bash
# As a user
kubectl auth can-i list pods -n ai-team --as=alice@example.com

# As a group
kubectl auth can-i create deployments -n ai-team --as=bob --as-group=ai-engineers

# As a ServiceAccount
kubectl auth can-i get secrets -n ai-team \
  --as=system:serviceaccount:ai-team:training-runner

# Run a full command as SA
kubectl get pods -n ai-team \
  --as=system:serviceaccount:ai-team:training-runner
```

### Granting impersonation (break-glass / platform ops)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
- apiGroups: ["authentication.k8s.io"]
  resources: ["userextras/scopes"]   # if using extras
  verbs: ["impersonate"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sre-impersonators
subjects:
- kind: Group
  name: sre-leads
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: impersonator
  apiGroup: rbac.authorization.k8s.io
```

Prefer **narrow** impersonation with `resourceNames` when possible:

```yaml
- apiGroups: [""]
  resources: ["serviceaccounts"]
  resourceNames: ["training-runner"]
  verbs: ["impersonate"]
```

---

## 6. Aggregation & Composition

ClusterRoles can **aggregate** other ClusterRoles via labels — how `admin` / `edit` / `view` grow when CRDs install.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: training-crds
  labels:
    rbac.example.com/aggregate-to-ai-admin: "true"
rules:
- apiGroups: ["kubeflow.org"]
  resources: ["mpijobs", "notebooks"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ai-admin
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-ai-admin: "true"
rules: []   # controller fills this in — do not edit manually
```

```bash
# See what aggregated into view/edit/admin
kubectl get clusterrole -l rbac.authorization.k8s.io/aggregate-to-view=true
kubectl get clusterrole -l rbac.authorization.k8s.io/aggregate-to-edit=true
kubectl get clusterrole -l rbac.authorization.k8s.io/aggregate-to-admin=true
```

**Rule:** If a ClusterRole has `aggregationRule`, the apiserver owns `.rules`. Manual edits are overwritten.

---

## 7. ServiceAccounts in Depth

### Create & use

```bash
kubectl create serviceaccount training-runner -n ai-team
kubectl get sa training-runner -n ai-team -o yaml
```

```yaml
# Pod spec
spec:
  serviceAccountName: training-runner
  automountServiceAccountToken: false   # prefer explicit mount when possible
```

### Token projection (modern default)

Since 1.24+, long-lived SA secrets are not auto-created. Use **TokenRequest** / projected volumes:

```yaml
volumes:
- name: sa-token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: api
```

```bash
# Bound token for external tooling (CI, scripts)
kubectl create token training-runner -n ai-team --duration=1h
```

### Disable default SA automount (namespace hygiene)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: ai-team
automountServiceAccountToken: false
```

Or per-pod: `automountServiceAccountToken: false`.

### Image pull secrets on SA

```bash
kubectl patch sa training-runner -n ai-team -p '
{"imagePullSecrets":[{"name":"regcred"}]}'
```

---

## 8. Least-Privilege Patterns

### Pattern A — Namespace power user (humans)

```bash
kubectl create rolebinding alice-admin \
  --clusterrole=admin \
  --user=alice@example.com \
  -n ai-team
```

Prefer `edit` over `admin` unless they must manage RoleBindings.

### Pattern B — CI deployer (pipeline SA)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: ai-team
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "deployments/scale"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Intentionally NO secrets/* wildcards — mount specific secrets via separate Role if needed
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["hf-token", "regcred"]
  verbs: ["get"]
```

### Pattern C — Read-only observability

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods", "nodes", "namespaces"]
  verbs: ["get", "list", "watch"]
```

### Pattern D — Secret-specific access

Never grant `secrets` with `*` names in shared namespaces. Pin `resourceNames`:

```yaml
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["hf-token"]
  verbs: ["get"]
```

**Caveat:** `list` / `watch` on secrets can still expose names (and sometimes data depending on client). Prefer avoiding `list` when pinning names.

### Pattern E — Subresource-only exec for break-glass

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-exec
  namespace: ai-team
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

### Validate before shipping

```bash
# Matrix of critical checks for an SA
for verb in get list watch create update patch delete; do
  for res in pods deployments jobs secrets; do
    printf '%-8s %-12s -> ' "$verb" "$res"
    kubectl auth can-i "$verb" "$res" -n ai-team \
      --as=system:serviceaccount:ai-team:training-runner
  done
done
```

---

## 9. Multi-Tenancy & Namespace Isolation

RBAC scopes *who* can call the API. Tenant isolation is broader: it also covers network paths, node/kernel blast radius, storage, and admission. Treat isolation as layered controls, not a single RoleBinding.

### Isolation pattern spectrum

| Pattern | Trust model | Control-plane share | Data-plane share | Typical use |
|---------|-------------|---------------------|------------------|-------------|
| Soft multi-tenancy (namespace-per-tenant) | Trusted / semi-trusted teams | Shared | Shared nodes | Internal platform, cost efficiency |
| Hierarchy (HNC / sub-namespaces) | Trusted org with nested teams | Shared | Shared nodes | Enterprise org → team → env trees |
| Virtual clusters (vCluster / loft) | Semi-trusted tenants needing own APIs | Shared host, virtualized API | Usually shared nodes | SaaS-ish platforms, tenant admins |
| Hard multi-tenancy (cluster-per-tenant) | Untrusted / regulated | Separate | Separate | Hostile tenants, strong compliance |

Pick the weakest pattern that still meets your threat model. Stronger isolation costs ops and money; weaker isolation needs stricter policy.

### Pattern A — Soft multi-tenancy (shared cluster)

One cluster, one namespace (or small set) per tenant. Isolation is policy-enforced, not physical.

| Control | Purpose |
|---------|---------|
| Namespace per team/tenant | Primary RBAC and quota boundary |
| RoleBinding to `edit`/`view` (or custom) | Human access scoped to their ns |
| No ClusterRoleBindings for tenants | Prevent privilege escape |
| ResourceQuota / LimitRange | Capacity fairness / noisy-neighbor control |
| NetworkPolicy (default-deny + allowlists) | East-west traffic isolation |
| Pod Security Admission / Kyverno / OPA | Block privileged pods, hostPath, hostNetwork |
| Separate SAs per workload | Limit token blast radius inside the ns |

```bash
# Tenant should NOT be able to do this:
kubectl auth can-i create clusterrolebindings --as=alice@example.com
# no

kubectl auth can-i create namespaces --as=alice@example.com
# no

kubectl auth can-i get pods -n other-tenant --as=system:serviceaccount:team-a:app
# no
```

**What soft multi-tenancy does *not* give you:** kernel/node isolation. A container escape or privileged misconfig can still reach other tenants on the same node.

### Pattern B — Hierarchical namespaces

Use **Hierarchical Namespace Controller (HNC)** when tenants are nested (org → team → env) and you want:

- Sub-namespaces that inherit labels/policies from a parent
- Propagated Roles/RoleBindings (or deliberate non-propagation)
- Admin boundaries that mirror the org chart without a ClusterRoleBinding per leaf

RBAC tip: bind humans at the parent or leaf namespace with RoleBindings; keep cluster-scoped privileges on platform groups only. Propagate *read* more freely than *bind/escalate*.

### Pattern C — Virtual clusters

**vCluster** (and similar: loft, kamaji-style setups) gives each tenant a virtual control plane:

- Tenant gets admin-like RBAC *inside* their virtual API server
- Host cluster maps synced resources into a host namespace (or set of namespaces)
- Host RBAC for the syncer/agent stays tightly scoped; tenants never get host `cluster-admin`

Use when tenants need CRDs, their own ClusterRoles, or “looks like my own cluster” UX, but you still want one physical fleet. Still combine with NetworkPolicy + PSA on the host; the virtual API is not a node security boundary by itself.

### Pattern D — Hard multi-tenancy (cluster-per-tenant)

Separate clusters (or dedicated node pools + separate control planes) when tenants are untrusted or regulated:

- Separate etcd / API servers → no shared RBAC mistakes across tenants
- Optional separate accounts/projects in the cloud IAM layer
- Higher cost; simpler mental model for blast radius

Even here, use least-privilege for platform automation that touches many clusters (fleet controllers should impersonate or use per-cluster credentials, not one global `cluster-admin`).

### Layered controls (apply on every pattern)

```
Identity (OIDC/IAM)
    → RBAC (Role/RoleBinding per tenant ns)
        → Admission (PSA / ValidatingAdmissionPolicy / Kyverno / OPA)
            → NetworkPolicy (+ optionally service mesh authz)
                → Runtime (seccomp, drop caps, no privileged)
                    → Node / cluster separation (when trust is low)
```

| Layer | Soft ns | Virtual cluster | Dedicated cluster |
|-------|---------|-----------------|-------------------|
| RBAC namespace boundary | Required | Host + virtual | Per cluster |
| Deny tenant ClusterRoleBindings | Required | On host | N/A (they own theirs) |
| Default-deny NetworkPolicy | Strongly recommended | Strongly recommended | Per threat model |
| PSA `restricted` or equivalent | Strongly recommended | On host synced pods | Recommended |
| ResourceQuota | Required for fairness | Per virtual / host ns | Optional |
| Dedicated nodes / sandboxes (gVisor, Kata) | Optional hardening | Optional | Optional |

### Example: tenant namespace bootstrap (soft pattern)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    tenant: team-a
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    persistentvolumeclaims: "10"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# Follow with allow-DNS / allow-same-ns / allow-egress-to-platform policies.
```

```bash
# Bind the tenant group to namespaced edit — not cluster-admin
kubectl create rolebinding team-a-edit \
  --clusterrole=edit \
  --group=team-a@example.com \
  -n team-a

# Verify isolation
kubectl auth can-i create deployments -n team-a --as=jane@example.com --as-group=team-a@example.com
# yes
kubectl auth can-i create deployments -n team-b --as=jane@example.com --as-group=team-a@example.com
# no
```

### Hardening checklist per tenant namespace

```bash
# 1. Create ns + quota + LimitRange + default-deny NetworkPolicy
# 2. Label ns for PSA (restricted) or enforce via Kyverno/OPA
# 3. Bind group to edit/view/custom Role — never cluster-admin
# 4. Disable default SA token automount; create purpose-built SAs
# 5. Restrict Role/RoleBinding creation if tenants shouldn't manage RBAC
# 6. Block privileged pods, hostPath, hostNetwork, NodePort abuse
# 7. Deny creating ClusterRoleBindings / touching cluster-scoped resources
# 8. Audit: can-i checks for tenant users AND their workload SAs
```

### Choosing a pattern (quick guide)

- **Same company, shared ops, cost-sensitive** → Pattern A (+ PSA + NetworkPolicy + quotas)
- **Nested teams / delegated ns admin** → Pattern B (HNC) on top of A
- **Tenants need their own CRDs / cluster-scoped objects** → Pattern C (vCluster)
- **Untrusted tenants, strong compliance, or hostile code** → Pattern D (dedicated cluster), optionally with sandbox runtimes

RBAC alone is **not** a security boundary against a privileged container escape. Combine API authorization with admission, network, and (when needed) node or cluster separation.

---

## 10. AI / GPU Workload RBAC

### Recommended identities

| Actor | Identity | Binding |
|-------|----------|---------|
| Data scientist (human) | OIDC user/group | RoleBinding → `edit` or custom in `ai-team` |
| Training Job | SA `training-runner` | Role: create Jobs, get pods/logs, get specific secrets, use PVC |
| Inference controller | SA `inference-controller` | Role: manage Deployments/Services in serve ns |
| GPU operator | SA in `gpu-operator` | ClusterRoles shipped by operator — leave alone |
| Platform SRE | group `sre` | ClusterRoleBinding → limited view + impersonate |

### Example: training runner Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: training-runner
  namespace: ai-team
rules:
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["hf-token"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: training-runner
  namespace: ai-team
subjects:
- kind: ServiceAccount
  name: training-runner
  namespace: ai-team
roleRef:
  kind: Role
  name: training-runner
  apiGroup: rbac.authorization.k8s.io
```

### Nodes / GPU visibility for schedulers

Custom schedulers or queue controllers (Kueue, Volcano) often need:

```yaml
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["scheduling.k8s.io"]
  resources: ["priorityclasses"]
  verbs: ["get", "list", "watch"]
```

Install operators via their Helm charts — **don’t hand-roll ClusterRoles** unless you must.

### Who can `kubectl debug` GPU nodes?

```yaml
# Highly privileged — limit to SRE
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods", "pods/exec"]
  verbs: ["create", "get"]
# plus permission to create pods in the target namespace for node debug
```

---

## 11. Admission, Policies & Guardrails

RBAC authorizes the API request; **admission** can still reject the object.

| Layer | Examples |
|-------|----------|
| RBAC | Can create Pods? |
| ValidatingAdmissionPolicy / OPA / Kyverno | Privileged? hostPath? missing limits? |
| Pod Security Admission | `restricted` / `baseline` / `privileged` |
| ResourceQuota | Over GPU quota? |

### Example: PSA labels on AI namespace

```bash
kubectl label ns ai-team \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted
```

GPU training often needs `baseline` or exceptions for `privileged` device access — prefer **RuntimeClass** + device plugin over full privileged.

### Prevent self-escalation with policy

Deny RoleBindings that reference `cluster-admin` in tenant namespaces (Kyverno/OPA). Example Kyverno intent:

```yaml
# Pseudopolicy: block RoleBindings/ClusterRoleBindings to cluster-admin
# except in kube-system / platform namespaces
```

---

## 12. Auditing & Continuous Verification

### API audit (server-side)

Enable audit policy on the apiserver to log:

- `impersonate`
- secrets `get`/`list`
- RBAC object mutations (`roles`, `rolebindings`, …)
- `create` on `pods/exec`

### Find overly broad bindings

```bash
# Who has cluster-admin?
kubectl get clusterrolebinding -o json | jq -r '
  .items[] |
  select(.roleRef.name=="cluster-admin") |
  "\(.metadata.name): \([.subjects[]? | "\(.kind)/\(.name)"] | join(", "))"'

# All ClusterRoleBindings for a group
kubectl get clusterrolebinding -o json | jq -r '
  .items[] |
  select([.subjects[]? | select(.kind=="Group" and .name=="ai-engineers")] | length > 0) |
  .metadata.name'

# RoleBindings in a namespace
kubectl get rolebinding,clusterrolebinding -n ai-team -o wide
```

### SubjectAccessReview (programmatic)

```bash
kubectl create -f - -o yaml <<'EOF'
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: system:serviceaccount:ai-team:training-runner
  resourceAttributes:
    namespace: ai-team
    verb: create
    resource: jobs
    group: batch
EOF
```

### SelfSubjectRulesReview — dump effective rules

```bash
kubectl create -f - -o yaml <<'EOF'
apiVersion: authorization.k8s.io/v1
kind: SelfSubjectRulesReview
spec:
  namespace: ai-team
EOF
# Or as another user:
kubectl create -f - -o yaml --as=system:serviceaccount:ai-team:training-runner <<'EOF'
apiVersion: authorization.k8s.io/v1
kind: SelfSubjectRulesReview
spec:
  namespace: ai-team
EOF
```

```bash
# Convenience
kubectl auth can-i --list -n ai-team
kubectl auth can-i --list -n ai-team \
  --as=system:serviceaccount:ai-team:training-runner
```

---

## 13. Debugging RBAC Failures

### Symptoms

```
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:ai-team:default"
cannot list resource "pods" in API group "" in the namespace "ai-team"
```

### Debug sequence

```bash
# 1. Confirm identity
kubectl auth whoami
# or from the error message: system:serviceaccount:NS:SA

# 2. Ask the authorizer directly
kubectl auth can-i list pods -n ai-team \
  --as=system:serviceaccount:ai-team:default -v=8
# -v=8 shows RBAC evaluation details in client logs

# 3. List effective rules
kubectl auth can-i --list -n ai-team \
  --as=system:serviceaccount:ai-team:default

# 4. Find bindings for that SA
kubectl get rolebinding,clusterrolebinding -A -o json | jq -r '
  .items[] |
  select([.subjects[]? |
    select(.kind=="ServiceAccount" and .name=="default" and .namespace=="ai-team")]
    | length > 0) |
  "\(.kind)/\(.metadata.namespace // "CLUSTER")/\(.metadata.name) -> \(.roleRef.kind)/\(.roleRef.name)"'

# 5. Inspect the Role rules
kubectl get role <name> -n ai-team -o yaml
kubectl describe clusterrole <name>
```

### Common footguns

| Problem | Cause |
|---------|-------|
| Works for you, fails in Job | Pod uses `default` SA — set `serviceAccountName` |
| Bound ClusterRole but still denied | Used RoleBinding? OK for namespaced resources. Cluster-scoped resources need ClusterRoleBinding |
| `resourceNames` set | `list`/`create` ignore names; `get`/`update`/`delete`/`patch` need exact name |
| Wrong apiGroup | `jobs` need `batch`, not `""` |
| CRD not covered | Built-in `edit` may lack your CRD until aggregation labels applied |
| Typo in subject namespace | SA subjects **must** include `namespace:` |

---

## 14. Anti-Patterns

1. **Binding `cluster-admin` to humans or CI** — use break-glass accounts with MFA and audit.
2. **Reusing `default` ServiceAccount** with privileges — create purpose-built SAs.
3. **Granting `secrets` list/watch broadly** — data exfil risk.
4. **Wildcards (`*`/`*`) in tenant Roles** — privilege creep.
5. **Handing out `bind` / `escalate` / `impersonate`** without review.
6. **Editing aggregated ClusterRole `.rules`** — they get overwritten.
7. **Assuming RBAC = tenant isolation** — combine with PSA, NetworkPolicy, quotas.
8. **Long-lived SA tokens in git / CI variables** — use short-lived `kubectl create token` or cloud identity federation (IRSA, Workload Identity).

---

## 15. Reference Command Sheet

```bash
# Identity
kubectl auth whoami
kubectl auth can-i --list -n <ns>
kubectl auth can-i create pods -n <ns>
kubectl auth can-i create pods -n <ns> --as=system:serviceaccount:<ns>:<sa>
kubectl auth can-i '*' '*' --all-namespaces   # am I cluster-admin?

# Create
kubectl create sa <name> -n <ns>
kubectl create role <name> --verb=... --resource=... -n <ns>
kubectl create clusterrole <name> --verb=... --resource=...
kubectl create rolebinding <name> --role=<role> --serviceaccount=<ns>:<sa> -n <ns>
kubectl create clusterrolebinding <name> --clusterrole=<cr> --group=<group>
kubectl create token <sa> -n <ns> --duration=1h

# Inspect
kubectl get sa,role,rolebinding -n <ns>
kubectl get clusterrole,clusterrolebinding
kubectl describe rolebinding <name> -n <ns>
kubectl get clusterrolebinding -o wide | grep cluster-admin

# Apply / diff
kubectl apply -f rbac.yaml
kubectl diff -f rbac.yaml
kubectl delete rolebinding <name> -n <ns>
```

### Minimal AI-team bootstrap

```bash
NS=ai-team
kubectl create namespace "$NS"
kubectl create sa training-runner -n "$NS"
kubectl create sa inference-controller -n "$NS"

kubectl create rolebinding "${NS}-engineers-edit" \
  --clusterrole=edit \
  --group=ai-engineers \
  -n "$NS"

kubectl create role training-runner \
  --verb=get,list,watch,create,delete \
  --resource=jobs \
  -n "$NS"
# then apply the fuller Role YAML from section 10 and bind it
```

---

## Quick Decision Tree

```
Need access to one namespace only?
  └─ Yes → Role + RoleBinding (or RoleBinding to ClusterRole)
  └─ No  → ClusterRole + ClusterRoleBinding

Human vs workload?
  └─ Human → User/Group from OIDC
  └─ Workload → dedicated ServiceAccount

Need secrets?
  └─ Pin resourceNames; avoid list/watch
  └─ Prefer external secrets + mount, not API get

Need to manage RBAC inside the ns?
  └─ Yes → bind ClusterRole admin (still ns-scoped via RoleBinding)
  └─ No  → bind edit or custom Role

Still Forbidden?
  └─ kubectl auth can-i ... --as=... && kubectl auth can-i --list
```

---

## Further Reading

- [Kubernetes RBAC docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [ServiceAccount tokens](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

---

*Companion to `kubernetes-cheat-sheet.md`. Adjust group names, namespaces, and CRD apiGroups for your platform.*
