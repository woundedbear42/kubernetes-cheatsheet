# CKA Study Guide — Drills & Sample Questions

Hands-on prep for the **Certified Kubernetes Administrator (CKA)** exam. This guide maps to the [official CNCF CKA curriculum](https://github.com/cncf/curriculum) (Kubernetes **v1.35** as of the Linux Foundation exam page), with timed drills and exam-style tasks you can run on a local cluster.

Related refs in this repo:
- [`kubernetes-cheat-sheet.md`](./kubernetes-cheat-sheet.md) — kubectl, etcd backup/restore, upgrades
- [`kubernetes-rbac-advanced-guide.md`](./kubernetes-rbac-advanced-guide.md) — RBAC deep dive
- [`kubernetes-deployment-models.md`](./kubernetes-deployment-models.md) — rollouts and release models

> **Disclaimer:** Not affiliated with CNCF/LF. Weights and topics change; always verify against the [current curriculum PDF](https://github.com/cncf/curriculum) and [LF exam page](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/).

---

## Table of Contents

1. [Exam Snapshot](#exam-snapshot)
2. [How to Study](#how-to-study)
3. [Domain 1 — Troubleshooting (30%)](#domain-1--troubleshooting-30)
4. [Domain 2 — Cluster Architecture, Installation & Configuration (25%)](#domain-2--cluster-architecture-installation--configuration-25)
5. [Domain 3 — Services & Networking (20%)](#domain-3--services--networking-20)
6. [Domain 4 — Workloads & Scheduling (15%)](#domain-4--workloads--scheduling-15)
7. [Domain 5 — Storage (10%)](#domain-5--storage-10)
8. [Timed Drills](#timed-drills)
9. [Sample Exam Tasks (with solutions)](#sample-exam-tasks-with-solutions)
10. [Exam-Day Checklist](#exam-day-checklist)
11. [Quick Command Muscle Memory](#quick-command-muscle-memory)

---

## Exam Snapshot

| Item | Detail |
|------|--------|
| Format | Online, proctored, **performance-based** (no multiple choice) |
| Duration | **2 hours** |
| Tasks | ~15–20 practical problems |
| Passing score | **66%** |
| Kubernetes version | Aligns to recent minor (currently advertised as **v1.35**) |
| Allowed docs | kubernetes.io docs, Kubernetes blog, Helm docs, Gateway API docs |
| Retake / simulator | Two exam attempts; Killer.sh simulator (2 sessions) included with voucher |

### Domain weights (official LF page)

| Domain | Weight | Study priority |
|--------|--------|----------------|
| Troubleshooting | **30%** | Highest — practice broken clusters daily |
| Cluster Architecture, Installation & Configuration | **25%** | kubeadm, etcd, RBAC, upgrades, Helm/Kustomize |
| Services & Networking | **20%** | Services, NetworkPolicy, Ingress, **Gateway API**, CoreDNS |
| Workloads & Scheduling | **15%** | Deployments, ConfigMaps/Secrets, HPA, affinity/taints |
| Storage | **10%** | PV / PVC / StorageClass |

**Speed rule:** average ~6–8 minutes per task. Skip and flag anything that stalls past 10 minutes; return if time remains.

---

## How to Study

### Lab setup (pick one)

```bash
# Kind (fast local multi-node)
kind create cluster --name cka --config multi-node.yaml

# Or kubeadm VMs (closest to exam control-plane tasks)
# Or killercoda / killer.sh practice environments
```

Enable imperative muscle memory:

```bash
export do="--dry-run=client -o yaml"
alias k=kubectl
# Example: k run nginx --image=nginx $do > pod.yaml
```

### Suggested cadence

| Phase | Focus |
|-------|--------|
| Foundation | Workloads, Services, Storage YAML from scratch; kubectl imperative shortcuts |
| Cluster ops | kubeadm install/upgrade path, etcd snapshot/restore, RBAC, Helm/Kustomize |
| Networking depth | NetworkPolicy from scratch, Ingress + Gateway API, DNS debugging |
| Troubleshooting reps | Broken pods/nodes/control-plane components under a timer |
| Mocks | Killer.sh + at least one full 2-hour mock; review every miss |

### What “good enough” looks like

- Create common resources **without memorizing full YAML** (`run`, `create deploy`, `expose`, `set image`, then edit).
- Navigate kubernetes.io docs in under 30 seconds to the right page.
- Restore etcd and upgrade a kubeadm control plane at least twice from cold.
- Write a deny-all NetworkPolicy and selective allow rules without looking up the API shape more than once.

---

## Domain 1 — Troubleshooting (30%)

Highest weight. Expect broken apps, NotReady nodes, dead control-plane pods, bad Services, and DNS failures.

### Competencies

- Troubleshoot clusters and nodes
- Troubleshoot cluster components (API server, controller-manager, scheduler, kubelet, etcd)
- Monitor resource usage
- Evaluate container logs / output streams
- Troubleshoot Services and networking

### Diagnostic ladder (memorize this order)

```
1. kubectl get / describe  →  Events & status
2. kubectl logs / logs -p  →  App & previous crash
3. kubectl get events -A --sort-by=.lastTimestamp
4. Node: kubectl describe node ; journalctl -u kubelet
5. Control plane static pods: /etc/kubernetes/manifests/
6. Network: endpoints, NetworkPolicy, CoreDNS, CNI
```

### High-yield commands

```bash
# Cluster health
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep -v Running
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Pod failure modes
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> -c <container> --tail=100
kubectl logs <pod> -n <ns> --previous
kubectl get events -n <ns> --field-selector involvedObject.name=<pod>

# Service / endpoints
kubectl get svc,ep,endpointslices -n <ns>
kubectl get endpointslices -l kubernetes.io/service-name=<svc> -n <ns>

# Node / kubelet (on the node)
sudo systemctl status kubelet
sudo journalctl -u kubelet -xe --no-pager | tail -100
crictl ps -a
crictl logs <container-id>

# Static pod / control plane
sudo ls /etc/kubernetes/manifests/
sudo crictl pods | grep -E 'kube-apiserver|etcd|scheduler|controller'
```

### Common failure → fix map

| Symptom | Likely cause | First check |
|---------|--------------|-------------|
| `ImagePullBackOff` | Bad image/tag or pull secret | `describe` Events |
| `CrashLoopBackOff` | App exit / bad config | `logs` + `logs --previous` |
| `Pending` | Resources, PVC, affinity, taints | `describe` Events |
| `Node NotReady` | kubelet, CNI, disk pressure | `describe node`, kubelet logs |
| Service has no endpoints | Label selector mismatch | `get pods --show-labels` vs Service selector |
| DNS fails in pods | CoreDNS, NetworkPolicy, ndots | `kubectl -n kube-system get pods -l k8s-app=kube-dns` |

### Drill A — Broken workload (10 min)

1. Deploy an app with a wrong image tag → fix to Running.
2. Break the Service selector → restore endpoints.
3. Add a resource request the node cannot meet → observe Pending → fix.

### Drill B — Node / control plane (15 min)

1. Stop kubelet on a worker → diagnose NotReady → restart.
2. Typo a static pod manifest (API server or scheduler) → recover from `/etc/kubernetes/manifests`.
3. Find which pod is OOMing with `kubectl top` and Events.

---

## Domain 2 — Cluster Architecture, Installation & Configuration (25%)

### Competencies

- RBAC
- Prepare infrastructure; create/manage clusters with **kubeadm**
- Cluster lifecycle (upgrade, certificate renew)
- Highly available control plane concepts
- Helm and Kustomize for cluster components
- Extension interfaces: CNI, CSI, CRI
- CRDs / operators (install & configure)

### kubeadm — install & upgrade muscle memory

```bash
# Version skew: upgrade control plane first, then workers; kubelet ≈ apiserver minor
sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get install -y kubeadm=<target>
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.XX.Y
# Then upgrade kubelet/kubectl on that node; drain workers before worker upgrades
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# upgrade kubelet/kubectl → uncordon
kubectl uncordon <node>
```

### etcd backup & restore (must be automatic)

```bash
# Backup (endpoints/certs often under /etc/kubernetes/pki/etcd/)
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

etcdctl snapshot status /opt/etcd-backup.db -w table

# Restore pattern (exam-style): restore to new data-dir, point etcd static pod at it,
# or follow the official restore procedure in kubernetes.io docs exactly.
```

See also [`kubernetes-cheat-sheet.md`](./kubernetes-cheat-sheet.md#backing-up--restoring-etcd).

### RBAC quick pattern

```bash
kubectl create sa deploy-sa -n app
kubectl create role deploy-role -n app --verb=get,list,watch,create,update,patch --resource=deployments,pods
kubectl create rolebinding deploy-rb -n app --role=deploy-role --serviceaccount=app:deploy-sa

# Cluster-scoped
kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding bob-nodes --clusterrole=node-reader --user=bob

# Auth can-i
kubectl auth can-i create deployments --as=system:serviceaccount:app:deploy-sa -n app
```

### Helm & Kustomize

```bash
# Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-nginx bitnami/nginx -n web --create-namespace
helm list -A
helm upgrade my-nginx bitnami/nginx -n web --set replicaCount=3
helm rollback my-nginx 1 -n web

# Kustomize
kubectl kustomize ./overlays/prod | kubectl apply -f -
kubectl apply -k ./overlays/prod
```

Minimal `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
images:
  - name: myapp
    newTag: "1.2.3"
replicas:
  - name: myapp
    count: 3
```

### CRDs / operators

```bash
kubectl apply -f https://example.com/operator.yaml   # often installs CRDs + controller
kubectl get crds
kubectl explain myresource.example.com --recursive | less
kubectl get <crd-plural> -A
```

### Drill C — Cluster ops (20 min)

1. Take an etcd snapshot to a specified path; verify with `snapshot status`.
2. Create SA + Role + RoleBinding granting only ConfigMap get/list in one namespace; prove with `auth can-i`.
3. Install a chart with Helm into a new namespace; scale via `helm upgrade --set`.
4. Apply a Kustomize overlay that changes image tag and replica count.

---

## Domain 3 — Services & Networking (20%)

### Competencies

- Pod-to-pod connectivity
- NetworkPolicies
- ClusterIP, NodePort, LoadBalancer, Endpoints
- **Gateway API** for Ingress traffic
- Ingress controllers and Ingress resources
- CoreDNS

### Services

```bash
kubectl expose deploy/web --port=80 --target-port=8080 --type=ClusterIP
kubectl create service nodeport web --tcp=80:8080   # or edit type/nodePort
kubectl get svc web -o yaml
kubectl get endpoints web
```

| Type | Use |
|------|-----|
| ClusterIP | Default in-cluster access |
| NodePort | Expose on each node IP:port |
| LoadBalancer | Cloud / MetalLB external VIP |
| ExternalName | DNS CNAME to external host |

### NetworkPolicy (write from scratch)

Default-deny ingress in a namespace, then allow from a labeled client:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
      ports:
        - protocol: TCP
          port: 80
```

Remember: policies are additive; an empty `podSelector: {}` selects all pods in the namespace; egress deny is separate (`policyTypes: ["Egress"]`).

### Ingress vs Gateway API

**Ingress** (still required):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

**Gateway API** (now on curriculum — know the hierarchy):

```
GatewayClass → Gateway → HTTPRoute (and siblings)
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: edge
  namespace: default
spec:
  gatewayClassName: cilium   # or whatever the exam cluster provides
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
spec:
  parentRefs:
    - name: edge
  hostnames: ["www.example.com"]
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web
          port: 80
```

### CoreDNS

```bash
kubectl -n kube-system get deploy coredns -o yaml
kubectl -n kube-system edit configmap coredns
# Test from a debug pod:
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default.svc.cluster.local
```

### Drill D — Networking (15 min)

1. Expose a Deployment as ClusterIP, then change to NodePort with a fixed `nodePort`.
2. Apply deny-all NetworkPolicy; prove curl fails; add allow from one labeled pod; prove success.
3. Create an Ingress (or HTTPRoute if Gateway is installed) for host `cka.local` → Service port 80.
4. Break CoreDNS ConfigMap briefly in a lab, observe failures, restore.

---

## Domain 4 — Workloads & Scheduling (15%)

### Competencies

- Deployments: rolling updates and rollbacks
- ConfigMaps and Secrets
- Workload autoscaling (HPA)
- Self-healing primitives
- Pod admission & scheduling: limits, node affinity, taints/tolerations

### Deployments / rollouts

```bash
kubectl create deploy web --image=nginx:1.25 --replicas=3
kubectl set image deploy/web nginx=nginx:1.26
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web
kubectl rollout undo deploy/web --to-revision=2
kubectl scale deploy/web --replicas=5
```

### ConfigMaps & Secrets

```bash
kubectl create cm app-config --from-literal=LOG_LEVEL=info --from-file=nginx.conf=./nginx.conf
kubectl create secret generic db --from-literal=password=s3cret
# Mount as envFrom / volume in Pod spec; know both patterns
```

### Scheduling primitives

```yaml
# requests/limits
resources:
  requests: { cpu: "100m", memory: "128Mi" }
  limits:   { cpu: "500m", memory: "256Mi" }

# nodeSelector
nodeSelector:
  disk: ssd

# affinity (required)
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: ["zone-a"]

# taint / toleration
# kubectl taint nodes node1 key=value:NoSchedule
tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

### HPA

```bash
kubectl autoscale deploy web --cpu-percent=70 --min=2 --max=10
kubectl get hpa
# Requires metrics-server in the cluster
```

### Drill E — Workloads (12 min)

1. Create Deployment → update image → rollback to previous revision.
2. Inject ConfigMap as env and Secret as volume; verify inside the pod.
3. Taint a node; schedule a Pod with matching toleration only on that node.
4. Create an HPA and confirm it appears with expected min/max.

---

## Domain 5 — Storage (10%)

### Competencies

- StorageClasses and dynamic provisioning
- Volume types, access modes, reclaim policies
- PersistentVolumes and PersistentVolumeClaims

### Objects & binding flow

```
Pod → PVC (request) → binds → PV (provisioned statically or via StorageClass)
```

```bash
kubectl get sc,pv,pvc -A
kubectl describe pvc data -n app
```

### Static PV + PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/pv-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
  namespace: app
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
  # storageClassName: ""   # often needed to bind to static PV with empty class
```

### Dynamic provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner   # exam may use a real provisioner name
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
# PVC with storageClassName: fast → PV created on demand
```

Access modes: `ReadWriteOnce` (RWO), `ReadOnlyMany` (ROX), `ReadWriteMany` (RWX), `ReadWriteOncePod`.

### Drill F — Storage (10 min)

1. Create StorageClass → PVC → Pod mounting PVC at `/data`; write a file; confirm PV bound.
2. Create static hostPath PV + PVC; force bind with matching size/access mode / StorageClass name.
3. Change reclaim policy on a PV and explain Delete vs Retain behavior.

---

## Timed Drills

Run these under a timer. Grade yourself: Done correctly / Partial / Failed. Redo any Failed within 24 hours.

### Drill set 1 — 30 minutes (warm-up)

| # | Task | Target |
|---|------|--------|
| 1 | Namespace `drill1`; Deployment `web` image `nginx:1.25`, 3 replicas, labels `app=web` | 4 min |
| 2 | ClusterIP Service `web` port 80 → container 80 | 2 min |
| 3 | ConfigMap `web-cm` with `PAGE=ok`; inject as env `PAGE` | 4 min |
| 4 | PVC `web-pvc` 100Mi RWO using default StorageClass; mount at `/usr/share/nginx/html` | 6 min |
| 5 | NetworkPolicy: only pods with `access=granted` can reach `app=web` on TCP/80 | 8 min |
| 6 | Scale to 5; change image to `nginx:1.26`; confirm rollout | 4 min |

### Drill set 2 — 45 minutes (cluster + RBAC)

| # | Task | Target |
|---|------|--------|
| 1 | etcd snapshot to `/tmp/etcd-snap.db`; show status | 8 min |
| 2 | SA `cicd` in `drill2`; Role allowing pods/deployments create,get,list,watch,patch,update,delete | 6 min |
| 3 | Bind SA; verify `auth can-i` true for create pods, false for nodes | 4 min |
| 4 | Taint `sku=gpu:NoSchedule` on one node; DaemonSet or Pod that tolerates and lands there | 8 min |
| 5 | Helm install (any small chart) into `drill2`; list release | 6 min |
| 6 | Kustomize overlay: bump replicas to 4 and apply | 6 min |
| 7 | Find all non-Running pods cluster-wide; fix one intentional failure you planted | 7 min |

### Drill set 3 — 60 minutes (mock half-exam)

Mix 8–10 tasks across troubleshooting, networking (Ingress or Gateway), storage, and a kubeadm/etcd style task if your lab supports it. Practice **context switching**:

```bash
kubectl config get-contexts
kubectl config use-context <cluster>
# Many CKA tasks require changing context — do this first on every question
```

---

## Sample Exam Tasks (with solutions)

These mimic exam wording. Try each **before** reading the solution. Adjust image names / paths to your lab.

### Task 1 — Multi-context Deployment

**Prompt:** Use context `cka-cluster`. In namespace `work` (create if needed), create Deployment `front` with image `nginx:1.25`, 2 replicas. Expose it as Service `front-svc` of type NodePort on port 80.

<details>
<summary>Solution</summary>

```bash
kubectl config use-context cka-cluster
kubectl create ns work
kubectl create deploy front --image=nginx:1.25 --replicas=2 -n work
kubectl expose deploy/front --name=front-svc --port=80 --target-port=80 --type=NodePort -n work
kubectl get svc front-svc -n work   # note nodePort
```
</details>

### Task 2 — RBAC least privilege

**Prompt:** Create ServiceAccount `deployer` in `work`. Create a Role granting create/get/list/update/delete on Deployments and Pods only in `work`. Bind it. Confirm the SA cannot list Secrets.

<details>
<summary>Solution</summary>

```bash
kubectl create sa deployer -n work
kubectl create role deployer-role -n work \
  --verb=create,get,list,update,delete \
  --resource=deployments,pods
kubectl create rolebinding deployer-rb -n work \
  --role=deployer-role --serviceaccount=work:deployer

kubectl auth can-i list secrets -n work --as=system:serviceaccount:work:deployer
# → no
kubectl auth can-i create deployments -n work --as=system:serviceaccount:work:deployer
# → yes
```
</details>

### Task 3 — NetworkPolicy

**Prompt:** In `work`, label pod/deployment so app pods have `tier=backend`. Create NetworkPolicy `backend-netpol` that allows Ingress to those pods **only** from pods with `tier=frontend` in the same namespace on TCP port 80. Deny other ingress.

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
  namespace: work
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl label deploy front tier=backend -n work --overwrite   # if using front as backend stand-in
kubectl apply -f backend-netpol.yaml
```
</details>

### Task 4 — Storage

**Prompt:** Create StorageClass `cka-slow` with provisioner `kubernetes.io/no-provisioner` and `volumeBindingMode: WaitForFirstConsumer`. Create PV `cka-pv` 200Mi, RWO, hostPath `/mnt/data`, reclaim `Retain`, StorageClassName `cka-slow`. Create PVC `cka-pvc` in `work` requesting 200Mi from `cka-slow`. Create Pod `cka-pod` mounting the PVC at `/data` using image `busybox` with command `sleep 3600`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cka-slow
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cka-pv
spec:
  capacity: { storage: 200Mi }
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cka-slow
  hostPath: { path: /mnt/data }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cka-pvc
  namespace: work
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: cka-slow
  resources:
    requests: { storage: 200Mi }
---
apiVersion: v1
kind: Pod
metadata:
  name: cka-pod
  namespace: work
spec:
  containers:
    - name: box
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: cka-pvc
```
</details>

### Task 5 — Rollout & rollback

**Prompt:** Update `front` to image `nginx:1.26`. Wait for rollout success. Then roll back to the previous revision.

<details>
<summary>Solution</summary>

```bash
kubectl set image deploy/front nginx=nginx:1.26 -n work
kubectl rollout status deploy/front -n work
kubectl rollout undo deploy/front -n work
kubectl rollout status deploy/front -n work
kubectl get deploy front -n work -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```
</details>

### Task 6 — Scheduling

**Prompt:** Label node `<node>` with `disk=ssd`. Create Pod `ssd-pod` in `work` that **must** schedule on nodes with that label, image `nginx`.

<details>
<summary>Solution</summary>

```bash
kubectl label node <node> disk=ssd --overwrite
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  namespace: work
spec:
  nodeSelector:
    disk: ssd
  containers:
    - name: nginx
      image: nginx
```

Or affinity `requiredDuringSchedulingIgnoredDuringExecution` with the same key/value.
</details>

### Task 7 — Service troubleshooting

**Prompt:** Deployment `api` exists but Service `api-svc` has no endpoints. Fix labels/selectors so endpoints populate without changing the Service name or port.

<details>
<summary>Solution approach</summary>

```bash
kubectl get deploy api -n <ns> --show-labels
kubectl get svc api-svc -n <ns> -o yaml   # check .spec.selector
kubectl get pods -n <ns> -l <current-selector> --show-labels
# Align either pod template labels or Service selector so they match
kubectl edit svc api-svc -n <ns>
# or kubectl label pods ... / edit deploy template
kubectl get endpoints api-svc -n <ns>
```
</details>

### Task 8 — Ingress

**Prompt:** Create Ingress `front-ing` in `work` for host `front.cka.local`, path `/` Prefix, backend Service `front-svc:80`. Use IngressClass `nginx` (or the class present in the cluster).

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: front-ing
  namespace: work
spec:
  ingressClassName: nginx
  rules:
    - host: front.cka.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: front-svc
                port:
                  number: 80
```
</details>

### Task 9 — etcd snapshot

**Prompt:** On the control plane, take an etcd snapshot saved to `/opt/cka/etcd-backup.db`. Use the cluster’s etcd PKI certs.

<details>
<summary>Solution</summary>

```bash
sudo mkdir -p /opt/cka
sudo ETCDCTL_API=3 etcdctl snapshot save /opt/cka/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
sudo ETCDCTL_API=3 etcdctl snapshot status /opt/cka/etcd-backup.db -w table
```

Paths may differ; confirm with docs / existing etcd static pod manifest.
</details>

### Task 10 — Gateway API HTTPRoute

**Prompt:** Assuming Gateway `edge` exists in `default` with an HTTP listener, create HTTPRoute `front-route` attaching to that Gateway, hostname `front.cka.local`, forwarding PathPrefix `/` to Service `front-svc` port 80 in `work`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: front-route
  namespace: work
spec:
  parentRefs:
    - name: edge
      namespace: default
  hostnames:
    - front.cka.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: front-svc
          port: 80
```
</details>

### Task 11 — CrashLoop diagnosis

**Prompt:** Pod `broken` in `work` is CrashLoopBackOff. Find the cause from logs/events and fix so it stays Running. (Lab setup: bad command or missing env.)

<details>
<summary>Solution approach</summary>

```bash
kubectl describe pod broken -n work
kubectl logs broken -n work
kubectl logs broken -n work --previous
# Typical fixes: correct command/args, add required env from ConfigMap/Secret, fix image
kubectl edit pod broken -n work   # often easier to delete + recreate with fixed YAML
```
</details>

### Task 12 — HPA

**Prompt:** Create HorizontalPodAutoscaler for Deployment `front` in `work`: CPU 65%, min 2, max 8.

<details>
<summary>Solution</summary>

```bash
kubectl autoscale deploy front -n work --cpu-percent=65 --min=2 --max=8
kubectl get hpa -n work
```
</details>

---

## Exam-Day Checklist

### Before the exam

- [ ] ID ready; PSI/exam desktop requirements met; quiet room
- [ ] Killer.sh both sessions used; note your weak domains
- [ ] Practiced with only allowed bookmarks (kubernetes.io, blog, Helm, Gateway API)
- [ ] Comfortable with copy/paste, terminal tabs, and `kubectl` imperative flags

### During every task

1. **Read the whole question** — namespace, context, names, ports are graded literally.
2. `kubectl config use-context …` first when required.
3. Create namespace if stated.
4. Prefer imperative + `$do` YAML edit over writing from zero.
5. **Verify** with `get`/`describe` before moving on (partial credit is real; wrong namespace = zero).
6. Flag and skip after ~8–10 minutes if stuck.

### Time boxing (2 hours)

| Window | Focus |
|--------|--------|
| 0:00–0:05 | Open notes bookmarks; breathe; skim all questions if UI allows |
| First pass | Clear low-hanging fruit (workloads, RBAC, simple Services) |
| Middle | Storage, NetworkPolicy, Ingress/Gateway |
| Hard last | etcd restore, upgrades, gnarly troubleshooting |
| Last 10 min | Re-verify named resources exist in the right namespaces |

### Allowed documentation tips

- Search kubernetes.io for the **object kind** (`NetworkPolicy`, `PersistentVolume`, `kubeadm upgrade`).
- Use `kubectl explain pod.spec.tolerations` when docs are slow.
- Gateway API docs are allowed separately — bookmark the HTTPRoute reference.

---

## Quick Command Muscle Memory

```bash
# Dry-run YAML
k run nginx --image=nginx $do > pod.yaml
k create deploy web --image=nginx $do > dep.yaml
k expose deploy web --port=80 $do > svc.yaml
k create cm cfg --from-literal=a=b $do > cm.yaml
k create secret generic s --from-literal=p=x $do > sec.yaml
k create role r --verb=get --resource=pods $do > role.yaml

# Explain / search
kubectl explain deploy.spec.strategy
kubectl api-resources | grep -i network
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -20

# Force replace after edit
kubectl replace --force -f pod.yaml

# Copy files out of pods if needed
kubectl cp work/cka-pod:/data/file ./file
```

### Imperative cheat aliases (optional local only)

```bash
alias k=kubectl
alias kgp='kubectl get pods -o wide'
alias kgpa='kubectl get pods -A -o wide'
alias kgn='kubectl get nodes -o wide'
alias kd='kubectl describe'
alias kl='kubectl logs'
```

---

## Cross-links & next steps

1. Drill from this guide daily; track miss rates per domain.
2. Use [`kubernetes-cheat-sheet.md`](./kubernetes-cheat-sheet.md) for etcd/upgrade command detail.
3. Use [`kubernetes-rbac-advanced-guide.md`](./kubernetes-rbac-advanced-guide.md) if RBAC scenarios feel slow.
4. Schedule Killer.sh early enough to leave a week for weak-domain remediation.
5. Re-check [CNCF curriculum](https://github.com/cncf/curriculum) the week of your exam for weight/topic changes.

**Bottom line:** CKA rewards fast, correct `kubectl` under pressure — especially troubleshooting, kubeadm/etcd, and networking. Practice timed drills until verification is habit, not an afterthought.
