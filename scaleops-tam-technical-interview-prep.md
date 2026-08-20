# ScaleOps TAM — Technical Conversation Prep (Peer TAM)

You cleared the recruiter screen. This guide preps you for a **technical conversation with another ScaleOps Technical Account Manager** — usually a peer (sometimes a TAM Lead). Expect less “why are you looking?” and more **“how would you run this account / POC / escalation?”**

Start from [`scaleops-tam-interview-prep.md`](./scaleops-tam-interview-prep.md) if you need the company pitch and recruiter framing. This doc goes deeper.

Related refs:
- [`kubernetes-cheat-sheet.md`](./kubernetes-cheat-sheet.md) — requests/limits, HPA, debugging, GPU
- [`kubernetes-rbac-advanced-guide.md`](./kubernetes-rbac-advanced-guide.md) — enterprise install / least-privilege conversations
- [`argocd-integration-guide.md`](./argocd-integration-guide.md) — GitOps coexistence (ScaleOps markets GitOps-native control)

---

## Table of Contents

1. [What This Round Is Testing](#what-this-round-is-testing)
2. [How the Conversation Usually Flows](#how-the-conversation-usually-flows)
3. [Technical Fluency They Expect](#technical-fluency-they-expect)
4. [Walkthrough Scenarios (Rehearse Out Loud)](#walkthrough-scenarios-rehearse-out-loud)
5. [POC & Onboarding Playbook](#poc--onboarding-playbook)
6. [Troubleshooting & Escalation Judgment](#troubleshooting--escalation-judgment)
7. [Customer Conversation Role-Plays](#customer-conversation-role-plays)
8. [Stories to Prepare](#stories-to-prepare)
9. [Questions to Ask the Peer TAM](#questions-to-ask-the-peer-tam)
10. [Day-Of Checklist](#day-of-checklist)
11. [Bottom Line](#bottom-line)

---

## What This Round Is Testing

A peer TAM is hiring a future teammate. They’re asking:

| Signal | What “good” sounds like |
|--------|-------------------------|
| **K8s resource model fluency** | You can explain requests vs usage vs node cost without hand-waving |
| **Product-accurate mental model** | Automation + coexistence with HPA/Karpenter — not “another VPA” |
| **Production safety instincts** | Phased rollout, guardrails, observe → automate, rollback thinking |
| **TAM craft** | Discovery, success criteria, champions, QBRs, expansion |
| **Judgment under ambiguity** | When to push automation vs slow down; when to escalate to Support/Eng |
| **Peer fit** | Clear, calm, collaborative — someone they’d put on a shared Slack with a customer |

They are **not** usually grading CKA-level debugging speed. They **are** grading whether a platform director would trust you in a war room.

---

## How the Conversation Usually Flows

Typical 45–60 minutes:

| Block | What happens |
|-------|----------------|
| 5 min | Their background + role reality (post-sale vs hybrid with sales) |
| 10 min | Your background — dig into **one** technical customer story |
| 15–20 min | Scenario / whiteboarding: rightsizing, POC, or “customer is scared of prod automation” |
| 10 min | Product / competitive / “how would you explain X?” |
| 5–10 min | Your questions |

**Communication tip:** narrate trade-offs. Peer TAMs hate binary answers (“just turn it all on”) and fluff (“we drive synergy”). Prefer: *observe → prove → expand → QBR*.

---

## Technical Fluency They Expect

### 1. The problem ScaleOps exists to solve

Be able to teach this in ~2 minutes on a whiteboard (mental or literal):

```
App owners set requests for "never page me"
        ↓
Scheduler / Karpenter reserve that capacity
        ↓
Nodes look full; utilization is low; bill is high
        ↓
Platform files tickets; YAML churn; still wrong next week
```

**Punchline:** horizontal autoscaling and node autoscaling amplify **bad requests**. Fixing the pod layer unlocks density → fewer nodes → real $ savings.

### 2. ScaleOps capability model (peer depth)

Know each lever and what layer it touches:

| Lever | Layer | What a TAM says |
|-------|-------|-----------------|
| **Pod rightsizing** | Vertical (CPU/mem requests/limits) | Continuous match to live demand + cluster context |
| **In-place resize** | Running pods (when cluster supports it) | Adjust without restart/eviction/rollout when possible |
| **Admission-time patching** | New pods | Rightsized values applied as pods are created |
| **Replica optimization** | Horizontal (min/max / triggers) | Coordinates with — does not replace — HPA/KEDA intent |
| **Smart placement / bin packing** | Scheduling density | Better packing so nodes can drain empty |
| **Node / Karpenter optimization** | Supply side | Better instance choice / consolidation *with* existing autoscalers |
| **Spot optimization** | Cost/risk | Shift tolerant workloads; respect interruption risk |
| **GPU rightsizing / sharing** | Accelerators | Raise utilization on expensive capacity |
| **Cost monitoring** | FinOps proof | Cluster/ns/team/app breakdown for QBRs |
| **Healing / burst reaction** | Reliability | React to OOM, throttle, noisy neighbor, spikes |

### 3. ScaleOps vs VPA (they will ask)

| Topic | Native VPA (typical production pain) | ScaleOps positioning |
|-------|--------------------------------------|----------------------|
| HPA coexistence | Classic conflict / thrash risk | Vertical under HPA/KEDA; horizontal stays customer-owned |
| Update disruption | Evict/recreate modes common | Prefer non-disruptive paths; **in-place resize** when available |
| Signal quality | Mostly historical usage histograms | Live cluster context + events (node pressure, neighbors, etc.) |
| Workload coverage | Limited / awkward on custom controllers | Custom workloads / CRDs called out as first-class |
| Ops model | Recommendations + human toil | Autonomous continuous management with policies |

**Soundbite:** “VPA taught the industry vertical rightsizing; ScaleOps is the production operating model — continuous, context-aware, HPA-safe.”

### 4. HPA / KEDA coexistence (must be crisp)

```
HPA / KEDA  →  how many replicas
ScaleOps    →  how big each replica (and related efficiency)
Karpenter/CA → how many / which nodes
```

Savings compound when vertical rightsizing + replica optimization + node consolidation stack (public customer narratives often cite this). If you only shrink requests but never free nodes, **cloud bill barely moves** — say that out loud; peer TAMs love that nuance.

### 5. Kubernetes vocabulary to use correctly

| Term | Use it correctly |
|------|------------------|
| **Request** | Scheduling reservation + (usually) HPA/utilization denominator context |
| **Limit** | Cap; CPU throttle when hit; memory → OOMKill risk if exceeded |
| **Throttle vs OOM** | CPU limit pressure vs memory kill — different symptoms, different customer fear |
| **QoS** | Guaranteed / Burstable / BestEffort — rightsizing changes risk profile |
| **PDB** | Limits disruption; matters if any eviction/recreate path exists |
| **Unevictable pods** | DaemonSets, pods without disruption budget room — block consolidation |
| **In-place pod resize** | Mutate resources on running pod via resize subresource (cluster version dependent) |
| **GitOps drift** | If Git owns requests, discuss how ScaleOps + Argo/Flux stay aligned (ScaleOps markets GitOps-native control) |

Refresh commands: [`kubernetes-cheat-sheet.md`](./kubernetes-cheat-sheet.md).

### 6. Install / security talking points (high level)

Enough for a peer chat — not a security questionnaire:

- Self-hosted via **Helm**; agent/components run in customer cluster
- Enterprise cares about: RBAC scope, network egress, data leaving cluster (usually: metrics stay local), air-gap, SSO to UI if applicable
- Start read-only / recommendation-style observation when customers need trust; enable automation per ns / workload class

---

## Walkthrough Scenarios (Rehearse Out Loud)

Practice answering in this shape: **clarify → diagnose → plan → prove → expand**.

### Scenario A — “We have Karpenter and HPA; why do we need you?”

**Clarify:** cloud (EKS?), Spot?, who sets requests, any Kubecost/VPA history, Sev frequency.

**Teach:**

1. HPA scales replicas on metrics; if each pod requests 4 CPU and uses 0.5, you multiply waste.
2. Karpenter provisions for **requests**, not average usage.
3. ScaleOps corrects the reservation continuously so Karpenter can consolidate.

**Close:** “We don’t replace Karpenter — we make its input honest and help consolidation actually happen.”

### Scenario B — “Automation in prod will page us”

**Plan:**

1. Install → observe-only window (enough history to trust recommendations).
2. Pick low-risk namespaces (non-stateful, good telemetry, willing champion).
3. Set policies / min-max guardrails; exclude sacred workloads initially.
4. Enable automation with clear rollback (disable automation / pin / revert policy).
5. Define success: CPU/mem request reduction, node count / $, error budget / golden signals unchanged.
6. Expand cluster-by-cluster; bring FinOps to QBR with proof.

**Peer signal:** you care more about **trust** than max savings in week one.

### Scenario C — “VPA burned us with restarts”

**Empathy + contrast:** acknowledge eviction pain on stateful / long-lived connections.

**Differentiate:** HPA coexistence, in-place resize where supported, admission-time sizing for new pods, real-time reaction to OOM/throttle vs slow histogram lag.

**Ask:** which workloads failed, K8s version, were they on `InPlace`/`Recreate`, did HPA fight VPA?

### Scenario D — “GitOps owns all manifests — won’t this drift?”

**Frame:** desired state vs runtime optimization. ScaleOps positions **GitOps-native control** (works with Argo CD / Flux / pipelines).

**TAM answer:** “We design the operating model with the platform team — policies as code, what Git owns vs what the controller may mutate, and how reviews/approvals work for exclusions.” Don’t invent a proprietary sync algorithm; show you know the **ownership** problem is the real customer issue.

### Scenario E — “Show me the money path”

Whiteboard the chain:

```
Rightsize requests → higher density → empty nodes → CA/Karpenter removes nodes → lower bill
         +
Replica optimization (avoid permanent over-min)
         +
Spot where safe
         +
GPU utilization (if AI estate)
```

Call out **vanity metrics**: “CPU request down 40% with same node count” is incomplete until nodes or committed spend move.

### Scenario F — Stateful / JVM / Kafka fear

**Acknowledge:** startup CPU spikes, heap vs RSS, brokers hate restarts.

**Talk:** boot-time optimization, workload-type aware policies, prefer in-place paths, exclude or conservative mode until confidence, watch OOM/throttle/probe failures as healing signals.

---

## POC & Onboarding Playbook

Peer TAMs live in this motion. Memorize a sane default plan you can customize:

### Discovery checklist (first calls)

- Clusters / versions / clouds; Karpenter vs Cluster Autoscaler
- HPA/KEDA/VPA/custom rightsizers today
- Cost tool today; monthly K8s compute spend (order of magnitude)
- Who owns requests (app vs platform) and change process (GitOps?)
- Prod constraints: PDBs, stateful sets, regulated ns, change windows
- Success owner: platform director? FinOps? both?

### POC success criteria (write them down with the customer)

| Criterion | Example target |
|-----------|----------------|
| Coverage | N namespaces or X% of CPU-hours under management |
| Efficiency | ≥Y% reduction in CPU and/or memory requests (or allocatable waste) |
| Bill / capacity | Node count or $ run-rate reduction (or validated projection) |
| Reliability | No increase in OOM/throttle/Sev-1; SLOs hold |
| Ops | Platform time on rightsizing tickets drops |
| Decision | Go / no-go date + expansion plan |

### Phased technical path

```
1. Helm install (non-prod or prod observe)
2. Baseline: waste, top offenders, node utilization
3. Observe period (trust the recommendations)
4. Automate pilot namespaces
5. Validate golden signals + savings
6. Enable more ns / replica / placement features
7. Prod expansion + QBR narrative
8. Multi-cluster / GPU / Spot land-and-expand
```

### Stakeholders map

| Persona | What they need from you |
|---------|-------------------------|
| Platform / SRE | Safety, RBAC, coexistence with Karpenter/HPA, runbooks |
| App teams | “You’re not going to break my deploy” |
| FinOps | Credible $ and unit-cost story |
| Security | Self-hosted posture, least privilege, data boundaries |
| Executive sponsor | Risk removal + savings without a headcount tax |

---

## Troubleshooting & Escalation Judgment

They may ask: *“Customer says ScaleOps caused OOMs / pending pods / Karpenter thrash — what do you do?”*

### Triage order (say this)

1. **Stabilize** — confirm blast radius; pause automation on affected workloads if needed.
2. **Facts** — timeline, which ns/workloads, K8s events (`OOMKilled`, `FailedScheduling`), HPA activity, node pressure, recent deploys.
3. **Hypothesis** — limit too low? request drop → packing onto pressured nodes? HPA metric distortion? PDB blocking drain?
4. **Mitigate** — raise floors/guardrails, exclude workload, pin previous posture.
5. **Escalate** — Support/Eng with crisp repro packet; keep AM informed on risk to renewal.
6. **Learn** — update account runbook; feed Product if gap.

### Escalation packet quality (peer love this)

- Cluster version, ScaleOps version/chart, cloud
- Workload kind + criticality
- Before/after resources, events, metrics screenshots
- Whether observe vs automate; which policies
- Customer business impact + next sync time

**Judgment test:** you neither blame the customer nor defend the product blindly — you **protect production first**, then find root cause.

---

## Customer Conversation Role-Plays

Practice short answers (60–90 seconds):

**1. “Is this just Kubecost with a button?”**  
“Kubecost-class tools are excellent at allocation and showing waste. ScaleOps is built to **continuously manage** resources in production so the waste doesn’t require a ticket queue. Many customers keep visibility tools *and* add automation.”

**2. “Will this fight our HPA?”**  
“HPA owns replicas; we own vertical sizing underneath. That’s the opposite of the classic VPA conflict. As pods rightsize, HPA often behaves more sanely because the per-pod baseline is honest.”

**3. “What’s the risk?”**  
“Any automation that touches resources has risk. We manage it with observe-first, guardrails, workload-aware policies, non-disruptive resize paths where the cluster allows, and clear rollback. My job is to expand coverage only as fast as evidence allows.”

**4. “How fast to value?”**  
“Observation can show opportunity quickly; automation value often shows as density and node count move — days to weeks depending on change control. I’ll never trade a Sev-1 for a pretty savings slide.”

**5. “We need GPU savings for AI.”**  
“Same class of problem on more expensive capacity: idle GPUs and wrong packing. We talk utilization, sharing/rightsizing, and keeping training/inference SLOs intact — then prove it on a pilot namespace of jobs.”

---

## Stories to Prepare

Bring **two deep stories** (even if not ScaleOps). Peer TAMs will interrupt for technical detail.

### Story 1 — Technical adoption / POC

- Environment (K8s scale, cloud, autoscaling stack)
- Problem (waste, toil, reliability)
- What you installed / changed
- How you proved safety
- Quantified outcome ($ , %, time, incidents)
- What you’d do differently

### Story 2 — Production incident / hard objection

- What broke or what scared the customer
- Your triage and communication
- How you partnered with Support/Eng/AM
- Customer trust after

### Optional Story 3 — Expansion / QBR

- How you turned one cluster into many
- FinOps + platform dual-threading
- Champion building

**STAR with numbers.** If you lack $ savings, use engineer hours, MTTR, ticket volume, or capacity recovered.

---

## Questions to Ask the Peer TAM

Ask things only an insider can answer:

- What’s the real split on this team: **post-sale vs selling POCs** with AEs?
- What does a **healthy book of business** look like (accounts, clusters, travel, Slack load)?
- Where do new TAMs struggle in the first 90 days?
- What’s the standard **POC → production** playbook today?
- How do you partner with SE / Support / Product on escalations?
- Which objections are still hardest (GitOps drift, stateful, VPA PTSD, security)?
- How are TAMs measured (adoption, validated savings, NRR, CSAT)?
- What changed for customers after recent product bets (in-place resize, GPU, Karpenter)?

---

## Day-Of Checklist

**Night before**
- [ ] Re-read recruiter guide pitch ([link](./scaleops-tam-interview-prep.md))
- [ ] Rehearse Scenarios A–C out loud (5 min each)
- [ ] Refresh requests/limits, HPA vs VPA, Karpenter mental model
- [ ] Skim ScaleOps rightsizing page + one case study (Adobe / Roku / Wiz)
- [ ] Pick 2 stories with metrics; write bullet prompts only

**On the call**
- [ ] Clarify their day-to-day (so you mirror the role)
- [ ] Prefer whiteboard structure over feature laundry lists
- [ ] Say “I don’t know the internal implementation, here’s how I’d validate with the customer/Support” when needed — honesty beats bluffing
- [ ] End with sharp questions that show you’ll learn the playbook fast

**Red flags to avoid**
- Promising “80% savings” as a guarantee
- Trashing VPA/Kubecost/Cast without nuance
- Ignoring reliability when chasing cost
- Sounding like pure SE with no adoption follow-through (or pure CSM with no K8s)

---

## Bottom Line

A peer ScaleOps TAM wants a teammate who can **teach the resource → density → node bill chain**, run a **safe observe → automate → expand** motion, handle **HPA/Karpenter/GitOps objections** calmly, and own the customer when something looks scary in prod. Lead with judgment and crisp Kubernetes fundamentals; use product vocabulary accurately; prove you’ll protect production while still driving measurable value.
