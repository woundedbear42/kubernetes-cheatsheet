# Kubernetes Cheat Sheet

Quick reference for everyday `kubectl` plus advanced commands for GPU / AI / ML workloads.

---

## Table of Contents

1. [Setup & Context](#setup--context)
2. [Cluster State in etcd](#cluster-state-in-etcd)
3. [Backing Up & Restoring etcd](#backing-up--restoring-etcd)
4. [Upgrading Kubernetes](#upgrading-kubernetes)
5. [Namespaces](#namespaces)
6. [Get / Inspect Resources](#get--inspect-resources)
7. [Pods](#pods)
8. [Deployments & Workloads](#deployments--workloads)
9. [Services & Networking](#services--networking)
10. [ConfigMaps & Secrets](#configmaps--secrets)
11. [Storage](#storage)
12. [Scaling & Rollouts](#scaling--rollouts)
13. [Debugging & Troubleshooting](#debugging--troubleshooting)
14. [Node Taints & Tolerations](#node-taints--tolerations)
15. [RBAC & Security](#rbac--security)
16. [Jobs & CronJobs](#jobs--cronjobs)
17. [Advanced: AI / GPU Workloads](#advanced-ai--gpu-workloads)
18. [Useful One-Liners](#useful-one-liners)

---

## Setup & Context

```bash
# Cluster info
kubectl cluster-info
kubectl version --short
kubectl api-resources
kubectl api-versions

# Contexts (kubeconfig)
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>
kubectl config set-context --current --namespace=<ns>
kubectl config view --minify

# Short aliases (optional)
alias k=kubectl
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kgs='kubectl get svc'
```

---

## Cluster State in etcd

Kubernetes keeps **desired and reported cluster state** in **etcd** — a consistent key-value store. Almost every `kubectl get` / `apply` ultimately reads or writes that store through the API server.

### How the path works

```
kubectl / controllers / operators
        │
        ▼
   kube-apiserver   ←── only component that talks to etcd
        │
        ▼
      etcd          ←── source of truth for API objects
```

| Layer | Role |
|-------|------|
| Clients & controllers | Read/watch/write via the Kubernetes API (never talk to etcd directly) |
| `kube-apiserver` | Authn/authz, validation, admission, then persist to etcd; serves watches |
| etcd | Stores serialized API objects; quorum writes for consistency |

Controllers (scheduler, kubelet via node authorizer, deployments controller, etc.) **watch** the API, compare desired vs actual, and write updates back. etcd itself does not “run” reconciliation — it only stores state.

### What lives in etcd (and what does not)

| In etcd | Not in etcd |
|---------|-------------|
| Pods, Deployments, Services, Jobs, … | Container logs (node filesystem / log backend) |
| ConfigMaps, Secrets (often encrypted at rest) | Image layers / registry content |
| RBAC, namespaces, CRDs & CR instances | Persistent volume *data* (CSI / cloud disks) |
| Node and Lease objects, EndpointSlices | Metrics time-series (metrics-server / Prometheus) |
| Controller revisions / history metadata | etcd’s own member config outside the KV data |

Objects are stored under keys like `/registry/<resource>/<namespace>/<name>` (cluster-scoped resources omit the namespace segment), typically as protobuf (sometimes JSON for older / atypical paths).

### Consistency & watches

- Writes go through etcd’s Raft quorum — a successful API write is durable on a majority of etcd members.
- Each object has a **`resourceVersion`**; list/watch uses it so controllers can resume without full resyncs when possible.
- `kubectl apply` is optimistic concurrency: conflicting concurrent updates can get a `409 Conflict` when `resourceVersion` does not match.

### Inspect & operate (control-plane access)

Direct etcd access is for break-glass / backup — prefer the API for day-to-day work.

```bash
# Typical kubeadm static-pod etcd on a control-plane node
ETCDCTL="etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

# Member health & endpoints
$ETCDCTL endpoint health
$ETCDCTL member list -w table

# Browse keys (prefix)
$ETCDCTL get /registry/namespaces --prefix --keys-only
$ETCDCTL get /registry/pods/ai-workloads --prefix --keys-only
```

```bash
# Encryption at rest: Secrets may be encrypted in etcd (EncryptionConfiguration on the API server).
# If enabled, raw etcd values are ciphertext — decrypt only via the API:
kubectl get secret <name> -n <ns> -o yaml
```

**Do not** edit etcd keys by hand to “fix” objects — you can desync controllers and corrupt cluster state. Change resources through `kubectl` / the API; use snapshots for disaster recovery.

---

## Backing Up & Restoring etcd

An etcd snapshot is the cluster’s disaster-recovery checkpoint for **API object state**. It does **not** back up PV contents, node disks, or external databases — back those up separately.

### What to back up together

| Asset | Why |
|-------|-----|
| etcd snapshot (`.db`) | All Kubernetes API objects |
| `/etc/kubernetes/pki` (or equivalent certs) | API / etcd / admin certs needed after restore |
| kubeadm / cluster config | How the control plane was built |
| EncryptionConfig + encryption keys | Required if Secrets are encrypted at rest in etcd |
| CSI / cloud volume snapshots | Application data on PVs |

### Take a snapshot (kubeadm-style stacked etcd)

Run on a healthy control-plane node (or against a reachable etcd endpoint):

```bash
ETCDCTL="etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

# Confirm cluster is healthy first
$ETCDCTL endpoint health --cluster

# Write a consistent snapshot (safe on a live cluster)
mkdir -p /var/backups/etcd
$ETCDCTL snapshot save /var/backups/etcd/etcd-$(date +%F-%H%M).db

# Verify the file is a valid snapshot
etcdctl snapshot status /var/backups/etcd/etcd-YYYY-MM-DD-HHMM.db -w table

# Copy off-box (example)
scp /var/backups/etcd/etcd-*.db backup-host:/backups/k8s/
tar -C /etc/kubernetes -czf pki-$(date +%F).tar.gz pki
```

Managed Kubernetes (EKS/GKE/AKS) usually handles etcd for you — use the provider’s control-plane backup / restore features instead of raw `etcdctl`.

### Restore overview (stacked etcd)

Exact steps vary by install (kubeadm stacked vs external etcd). Typical flow for a **single** control-plane disaster restore:

```bash
# 1. Stop API server / etcd static pods (move manifests aside)
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak
# wait until etcd / kube-apiserver containers stop

# 2. Restore snapshot into a new data dir
etcdctl snapshot restore /var/backups/etcd/etcd-YYYY-MM-DD-HHMM.db \
  --data-dir=/var/lib/etcd-restore

# 3. Point etcd at the restored data (update manifest hostPath / --data-dir)
#    For multi-member etcd: restore as a new single-member cluster, then re-add members
#    per the current Kubernetes / etcd restore docs — do not reuse old member IDs blindly.

# 4. Restore manifests and wait for control plane
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests
kubectl get nodes
kubectl get cs 2>/dev/null; kubectl get --raw='/readyz?verbose'
```

**Practice restores in a lab.** A snapshot you have never restored is not a tested backup.

### Backup hygiene

```bash
# Before upgrades, node replacements, or etcd changes — always:
$ETCDCTL snapshot save /var/backups/etcd/pre-upgrade-$(date +%F-%H%M).db

# Keep retention off-node; encrypt backups if they contain Secrets (they do).
# After restore with encryption-at-rest, the same EncryptionConfiguration keys must be available.
```

---

## Upgrading Kubernetes

Upgrade **one minor version at a time** (e.g. 1.28 → 1.29 → 1.30). Skipping minors is unsupported and breaks the version skew policy.

### Version skew (remember these)

| Component | Rule of thumb |
|-----------|----------------|
| kube-apiserver | Newest component — upgrade control plane first |
| kube-controller-manager / scheduler | Same version as kube-apiserver |
| kubelet | May be up to **two** minor versions behind apiserver; never newer |
| kubectl | Preferably within one minor of the cluster |
| kube-proxy / CNI | Follow their upgrade notes; often with node / control plane |

### Pre-upgrade checklist

```bash
# 1. Snapshot etcd (+ PKI / encryption keys) — see previous section
# 2. Check current versions & node health
kubectl version
kubectl get nodes -o wide
kubectl get pods -A | grep -E 'kube-system|CrashLoop|Pending'

# 3. Drain impact: PDBs, singleton pods, GPU jobs
kubectl get pdb -A
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>

# 4. Read release notes for removals (APIs, feature gates, CNI, device plugins)
# 5. Match addon versions (CoreDNS, CNI, CSI, metrics-server, GPU operator) to the target minor
```

### kubeadm upgrade path (control plane, then workers)

```bash
# --- On a control-plane node ---
apt-get update && apt-get install -y kubeadm=<target>.x-00   # or yum/dnf equivalent
kubeadm upgrade plan
kubeadm upgrade apply v<target>          # first control plane
# additional control planes:
kubeadm upgrade node

# Upgrade kubelet + kubectl on that node, then restart kubelet
apt-get install -y kubelet=<target>.x-00 kubectl=<target>.x-00
systemctl daemon-reload && systemctl restart kubelet
kubectl get nodes   # control plane should show new version

# --- Workers (one by one, or per surge pool) ---
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# on the node:
apt-get install -y kubeadm=<target>.x-00
kubeadm upgrade node
apt-get install -y kubelet=<target>.x-00 kubectl=<target>.x-00
systemctl daemon-reload && systemctl restart kubelet
# back on admin machine:
kubectl uncordon <node>
```

### Managed clusters

```bash
# Upgrade via cloud control plane first, then node pools / node groups
# EKS example concepts: update cluster version → update add-ons → rotate node groups
# GKE: control plane upgrade → node pool upgrade (surge / blue-green)
# AKS: az aks upgrade → node image / pool upgrade

# Always confirm:
kubectl get nodes
kubectl get pods -A
kubectl get --raw='/readyz?verbose'
```

### Post-upgrade

```bash
# API deprecations still in use?
kubectl get --raw=/metrics | grep apiserver_requested_deprecated_apis || true

# Critical addons
kubectl -n kube-system get pods
kubectl get ds -A
# GPU clusters: recheck allocatable GPUs after kubelet / device-plugin restart
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu
```

**Order that usually works:** etcd backup → control plane → CNI/CSI/core addons → workers (drain/uncordon) → operators (GPU, ingress, service mesh) → re-validate workloads.

---

## Namespaces

```bash
kubectl get namespaces
kubectl create namespace ai-workloads
kubectl delete namespace ai-workloads

# Work in a namespace without changing context
kubectl get pods -n ai-workloads
kubectl get all -n ai-workloads

# All namespaces
kubectl get pods -A
kubectl get pods --all-namespaces
```

---

## Get / Inspect Resources

```bash
# Wide output, labels, YAML/JSON
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get pod <name> -o yaml
kubectl get pod <name> -o json
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Describe (events + status)
kubectl describe pod <name>
kubectl describe node <node>
kubectl describe deploy <name>

# Labels & selectors
kubectl get pods -l app=inference
kubectl get pods -l 'gpu=true,tier in (train,serve)'
kubectl label pod <name> env=prod
kubectl label pod <name> env-   # remove label

# Watch live
kubectl get pods -w
kubectl get events --sort-by=.lastTimestamp
```

---

## Pods

```bash
# Create / run
kubectl run nginx --image=nginx --port=80
kubectl run debug --image=busybox -it --rm --restart=Never -- sh
kubectl create -f pod.yaml
kubectl apply -f pod.yaml

# Lifecycle
kubectl delete pod <name>
kubectl delete pod <name> --grace-period=0 --force
kubectl delete pods -l app=old-job

# Logs
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> -f                    # follow
kubectl logs <pod> --previous            # crashed container
kubectl logs -l app=inference --tail=100
kubectl logs <pod> --since=1h

# Exec / attach
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -c <container> -- sh
kubectl attach <pod> -c <container> -it

# Port forward
kubectl port-forward pod/<pod> 8080:80
kubectl port-forward svc/<svc> 8000:80
```

---

## Deployments & Workloads

```bash
# Create
kubectl create deployment web --image=nginx --replicas=3
kubectl apply -f deployment.yaml

# Inspect
kubectl get deploy,rs,pods
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>

# Update image
kubectl set image deployment/<name> <container>=<image>:<tag>
kubectl set env deployment/<name> MODEL_PATH=/models/llama

# Edit live
kubectl edit deployment/<name>
kubectl patch deployment <name> -p '{"spec":{"replicas":5}}'

# Delete
kubectl delete deployment <name>
kubectl delete -f deployment.yaml
```

---

## Services & Networking

```bash
kubectl get svc,endpoints,ingress -A
kubectl expose deployment web --port=80 --type=ClusterIP
kubectl expose deployment web --port=80 --type=LoadBalancer
kubectl expose deployment web --port=80 --type=NodePort

# DNS inside cluster: <svc>.<ns>.svc.cluster.local
kubectl get endpoints <svc>
kubectl describe ingress <name>

# Network debug
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
# then: curl, dig, traceroute, etc.
```

---

## ConfigMaps & Secrets

```bash
# ConfigMaps
kubectl create configmap app-config --from-literal=LOG_LEVEL=info
kubectl create configmap app-config --from-file=./config/
kubectl get configmap app-config -o yaml

# Secrets (prefer sealed-secrets / external-secrets in prod)
kubectl create secret generic hf-token --from-literal=token=hf_xxx
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass>
kubectl get secret hf-token -o jsonpath='{.data.token}' | base64 -d

# Mount / env from secret in a Deployment (YAML)
# envFrom: [ secretRef: { name: hf-token } ]
```

---

## Storage

```bash
kubectl get pv,pvc,sc
kubectl get storageclass
kubectl describe pvc model-cache

# Create PVC (example)
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
spec:
  accessModes: ["ReadWriteMany"]   # or ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: <your-sc>
EOF

# Copy files to/from pods (models, checkpoints)
kubectl cp ./model.safetensors <ns>/<pod>:/models/model.safetensors
kubectl cp <ns>/<pod>:/outputs/checkpoint.pt ./checkpoint.pt
```

---

## Scaling & Rollouts

```bash
kubectl scale deployment/<name> --replicas=5
kubectl autoscale deployment/<name> --min=2 --max=10 --cpu-percent=70

# Rollout control
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
kubectl rollout restart deployment/<name>
```

---

## Debugging & Troubleshooting

```bash
# Why isn't it scheduling / starting?
kubectl describe pod <name>          # check Events
kubectl get events -n <ns> --field-selector involvedObject.name=<pod>
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*].state}'

# Node pressure / capacity
kubectl describe node <node> | less
kubectl top nodes
kubectl top pods -A --containers
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory,GPU:.status.allocatable.nvidia\.com/gpu

# Resource usage
kubectl top pod -n <ns>
kubectl top pod --sort-by=memory

# Ephemeral debug container (requires EphemeralContainers feature)
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl debug node/<node> -it --image=busybox

# Pending pod reasons
kubectl get pods --field-selector=status.phase=Pending
```

---

## Node Taints & Tolerations

A **taint** marks a node so the scheduler *repels* pods that do not **tolerate** it. Use taints to reserve nodes (GPU pools, dedicated tenants, control-plane) or to evict pods during maintenance.

Taints are `key=value:effect` (value is optional). Effects:

| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New pods without a matching toleration are not scheduled. Existing pods stay. |
| `PreferNoSchedule` | Soft avoid — scheduler tries not to place non-tolerating pods, but may still. |
| `NoExecute` | New pods blocked **and** running pods without a toleration are evicted (optionally after `tolerationSeconds`). |

```bash
# Inspect current taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
kubectl describe node <node> | grep -A5 Taints

# Add a taint (reserve node for GPU workloads)
kubectl taint nodes <node> nvidia.com/gpu=true:NoSchedule

# Prefer to keep general workloads off this node (soft)
kubectl taint nodes <node> dedicated=batch:PreferNoSchedule

# Evict pods that do not tolerate the taint
kubectl taint nodes <node> maintenance=true:NoExecute

# Remove a specific taint (trailing '-')
kubectl taint nodes <node> nvidia.com/gpu=true:NoSchedule-
kubectl taint nodes <node> maintenance=true:NoExecute-

# Overwrite an existing taint with the same key/effect
kubectl taint nodes <node> dedicated=gpu:NoSchedule --overwrite
```

Pods need a matching **toleration** to land on (or stay on) a tainted node:

```yaml
spec:
  tolerations:
  # Match key + value + effect
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  # Or tolerate any value for that key
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  # Stay up to 5 minutes after a NoExecute taint is applied
  - key: maintenance
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 300
```

```bash
# Quick check: can this pod schedule onto a tainted GPU node?
# Pending + "had taint {…} that the pod didn't tolerate" in Events → missing toleration
kubectl describe pod <name>
```

**Taint vs cordon**

| Action | What it does |
|--------|----------------|
| `kubectl cordon <node>` | Marks node unschedulable (`SchedulingDisabled`) for *all* new pods |
| `kubectl taint …:NoSchedule` | Blocks only pods that lack a matching toleration |
| `kubectl drain <node>` | Cordons + evicts pods (respects PodDisruptionBudgets) |

Control-plane nodes are often tainted with `node-role.kubernetes.io/control-plane:NoSchedule` so workloads stay on workers unless they explicitly tolerate it.

---

## RBAC & Security

```bash
kubectl auth can-i create pods
kubectl auth can-i get secrets --as=system:serviceaccount:<ns>:<sa>
kubectl create serviceaccount training-sa -n ai-workloads
kubectl create rolebinding training-edit \
  --clusterrole=edit \
  --serviceaccount=ai-workloads:training-sa \
  -n ai-workloads

kubectl get sa,role,rolebinding,clusterrole,clusterrolebinding
```

---

## Jobs & CronJobs

```bash
# One-shot training / batch job
kubectl create job train-run --image=my-trainer:latest -- python train.py
kubectl apply -f training-job.yaml
kubectl get jobs,pods
kubectl logs job/train-run

# Cron
kubectl create cronjob nightly-eval --image=eval:latest --schedule="0 2 * * *" -- python eval.py
kubectl get cronjobs
kubectl delete cronjob nightly-eval
```

---

## Advanced: AI / GPU Workloads

### Check GPU capacity & drivers

```bash
# Nodes with GPU allocatable resources (NVIDIA device plugin)
kubectl get nodes -o json | jq -r '
  .items[] |
  select(.status.allocatable["nvidia.com/gpu"] != null) |
  "\(.metadata.name)\tGPU=\(.status.allocatable["nvidia.com/gpu"])"'

# Or without jq
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
GPU:.status.allocatable.nvidia\.com/gpu,\
MIG:.status.allocatable.nvidia\.com/mig-1g\.5gb

# Confirm NVIDIA device plugin / GPU Operator pods
kubectl get pods -n gpu-operator
kubectl get pods -A | grep -E 'nvidia|gpu'
kubectl describe daemonset -n gpu-operator nvidia-device-plugin-daemonset

# On a GPU node (via debug)
kubectl debug node/<gpu-node> -it --image=nvidia/cuda:12.4.1-base-ubuntu22.04 -- nvidia-smi
```

### Request GPUs in a pod / Job

```yaml
# gpu-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke
spec:
  restartPolicy: Never
  containers:
  - name: cuda
    image: nvidia/cuda:12.4.1-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1          # whole GPU
      # For AMD: amd.com/gpu: 1
      # For Intel: gpu.intel.com/i915: 1
```

```bash
kubectl apply -f gpu-pod.yaml
kubectl logs gpu-smoke
kubectl delete pod gpu-smoke
```

### Training Job with multiple GPUs

```yaml
# training-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: llm-finetune
  labels:
    workload: training
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 86400
  template:
    metadata:
      labels:
        workload: training
    spec:
      restartPolicy: Never
      runtimeClassName: nvidia          # if using NVIDIA RuntimeClass
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      containers:
      - name: trainer
        image: registry.example.com/trainer:v1
        command: ["torchrun", "--nproc_per_node=4", "train.py"]
        env:
        - name: NCCL_DEBUG
          value: INFO
        - name: HF_HOME
          value: /models/hf-cache
        resources:
          requests:
            cpu: "16"
            memory: 64Gi
            nvidia.com/gpu: 4
          limits:
            cpu: "16"
            memory: 64Gi
            nvidia.com/gpu: 4
        volumeMounts:
        - name: models
          mountPath: /models
        - name: dshm
          mountPath: /dev/shm          # critical for PyTorch DataLoader / NCCL
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: model-cache
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: 16Gi
```

```bash
kubectl apply -f training-job.yaml
kubectl get job llm-finetune -w
kubectl logs -f job/llm-finetune
kubectl delete job llm-finetune
```

### Inference Deployment (vLLM / TGI style)

```bash
# Example: scale a serving Deployment that holds 1 GPU per replica
kubectl apply -f inference-deploy.yaml
kubectl set image deployment/vllm-serve vllm=vllm/vllm-openai:latest
kubectl scale deployment/vllm-serve --replicas=2
kubectl rollout status deployment/vllm-serve

# Port-forward OpenAI-compatible API
kubectl port-forward svc/vllm-serve 8000:8000
curl http://localhost:8000/v1/models
```

Minimal inference fragment:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
env:
- name: CUDA_VISIBLE_DEVICES
  value: "0"
- name: HUGGING_FACE_HUB_TOKEN
  valueFrom:
    secretKeyRef:
      name: hf-token
      key: token
```

### Node affinity / taints for GPU pools

```bash
# Label GPU nodes
kubectl label node <node> workload=gpu accelerator=a100

# Taint so only GPU pods land there
kubectl taint nodes <node> nvidia.com/gpu=true:NoSchedule

# Schedule only on A100s
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: a100-job
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values: ["a100"]
  containers:
  - name: train
    image: trainer:latest
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
```

### MIG / fractional GPUs (NVIDIA)

```bash
# Inspect MIG profiles exposed by device plugin
kubectl describe node <node> | grep -i mig

# Request a MIG slice instead of a full GPU
# resources.limits: { nvidia.com/mig-1g.5gb: 1 }
```

### Shared Memory, NCCL, multi-node training

```bash
# Always give training pods large /dev/shm (see emptyDir above)
# Multi-node: use JobSet, Kubeflow MPIJob, or Volcano / Kueue

# Example: check NCCL env on a running trainer
kubectl exec -it <train-pod> -- env | grep -E 'NCCL|CUDA|MASTER'

# Common NCCL knobs (as env)
# NCCL_IB_DISABLE=0
# NCCL_SOCKET_IFNAME=eth0
# NCCL_DEBUG=INFO
```

### Queueing & fair-share (Kueue / Volcano)

```bash
# Kueue — list queues / admitted workloads
kubectl get clusterqueues,localqueues,workloads -A
kubectl describe workload <name> -n <ns>

# Volcano
kubectl get queue,podgroup,vcjob -A
```

### Model cache & Hugging Face on cluster

```bash
# Prefetch models onto a PVC with a one-shot Job
kubectl apply -f - <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: hf-prefetch
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: prefetch
        image: python:3.11-slim
        command: ["bash","-lc","pip install -q huggingface_hub && huggingface-cli download meta-llama/Meta-Llama-3-8B --local-dir /models/llama3-8b"]
        env:
        - name: HF_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-token
              key: token
        volumeMounts:
        - name: models
          mountPath: /models
        resources:
          requests:
            cpu: "2"
            memory: 8Gi
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: model-cache
EOF
```

### ResourceQuota / LimitRange for AI namespaces

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ai-quota
  namespace: ai-workloads
spec:
  hard:
    requests.nvidia.com/gpu: "8"
    limits.nvidia.com/gpu: "8"
    requests.cpu: "64"
    requests.memory: 256Gi
    pods: "40"
EOF

kubectl describe quota -n ai-workloads
```

### Useful AI-oriented inspections

```bash
# All pods requesting GPUs
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.containers[].resources.limits["nvidia.com/gpu"] != null) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# GPU allocation per node
kubectl get pods -A -o json | jq -r '
  .items[] |
  . as $p |
  (.spec.containers // [])[] |
  select(.resources.limits["nvidia.com/gpu"] != null) |
  "\($p.spec.nodeName // "UNSCHEDULED")\t\($p.metadata.namespace)/\($p.metadata.name)\t\(.resources.limits["nvidia.com/gpu"])"'

# Find OOMKilled trainers
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.status.containerStatuses[]?.lastState.terminated.reason == "OOMKilled") |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

### Common AI stack operators (install references)

```bash
# NVIDIA GPU Operator (Helm)
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm install gpu-operator nvidia/gpu-operator -n gpu-operator --create-namespace

# NVIDIA Device Plugin only (lighter)
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.16.2/deployments/static/nvidia-device-plugin.yml

# KServe (model serving)
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.13.0/kserve.yaml

# Inspect CRDs after install
kubectl get crd | grep -E 'nvidia|serving|kueue|volcano|ray'
```

---

## Useful One-Liners

```bash
# Delete all completed Jobs
kubectl delete jobs --field-selector=status.successful=1 -A

# Restart all pods in a Deployment
kubectl rollout restart deployment/<name>

# Force reschedule of Pending GPU pods after node recovery
kubectl delete pod -l workload=training --field-selector=status.phase=Pending

# Export a live resource to YAML (strip status)
kubectl get deploy <name> -o yaml | yq 'del(.status, .metadata.uid, .metadata.resourceVersion, .metadata.managedFields)'

# Shell into the first pod matching a label
kubectl exec -it $(kubectl get pod -l app=vllm -o jsonpath='{.items[0].metadata.name}') -- bash

# Wait until Job completes
kubectl wait --for=condition=complete job/llm-finetune --timeout=6h

# Diff live vs local manifest
kubectl diff -f deployment.yaml
```

---

## Quick Reference: Resource Names

| Short | Full |
|-------|------|
| `po` | pods |
| `deploy` | deployments |
| `rs` | replicasets |
| `svc` | services |
| `ing` | ingresses |
| `cm` | configmaps |
| `sec` | secrets |
| `ns` | namespaces |
| `no` | nodes |
| `pv` / `pvc` | persistentvolumes / claims |
| `sa` | serviceaccounts |
| `cj` | cronjobs |
| `job` | jobs |

```bash
kubectl get po,deploy,svc -n ai-workloads
```

---

## Tips for AI clusters

1. **Always mount `/dev/shm` as a Memory emptyDir** for PyTorch / NCCL — default is often 64Mi and will crash trainers.
2. **Request GPUs only in `limits`** (device plugins typically ignore requests, but keep them equal).
3. **Use taints + tolerations** so CPU workloads do not land on expensive GPU nodes.
4. **Prefetch models to a PVC** instead of downloading on every pod start.
5. **Prefer Jobs / JobSets** for training; Deployments for long-running inference.
6. **Watch node allocatable GPUs** after installs — if `nvidia.com/gpu` is missing, the device plugin is not healthy.
7. **Set ResourceQuotas** on AI namespaces to prevent a single team from exhausting the GPU pool.

---

*Generated as a practical field reference. Adjust image tags, CRD versions, and storage classes for your cluster.*
