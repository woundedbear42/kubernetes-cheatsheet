# Argo CD Sales Engineer — Recruitment Call Prep

Practical prep guide for a sales engineer (SE) recruitment call focused on Argo CD / GitOps. A recruiter call is mostly fit and story, not a deep Argo CD whiteboard. Prepare to sound credible on GitOps and sellable on why you’d be good with customers.

Related technical refs in this repo:
- [`argocd-cheat-sheet.md`](./argocd-cheat-sheet.md) — CLI / day-to-day commands
- [`argocd-integration-guide.md`](./argocd-integration-guide.md) — install, GitOps layout, sync/promotion
- [`kubernetes-deployment-models.md`](./kubernetes-deployment-models.md) — rolling, blue-green, canary, progressive delivery

This guide also covers **Kustomize** (manifest overlays) and **Kargo** (continuous promotion) as adjacent topics interviewers often expect.

---

## Table of Contents

1. [What They’ll Evaluate](#what-theyll-evaluate)
2. [Argo CD Fluency (Enough for Round 1)](#argo-cd-fluency-enough-for-round-1)
3. [Deployment Models with Argo CD](#deployment-models-with-argo-cd)
4. [Kustomize (Manifest Composition)](#kustomize-manifest-composition)
5. [Kargo (Continuous Promotion)](#kargo-continuous-promotion)
6. [Sales-Engineer Story to Rehearse](#sales-engineer-story-to-rehearse)
7. [Discovery Questions Worth Knowing](#discovery-questions-worth-knowing)
8. [Competitive / Commercial Awareness](#competitive--commercial-awareness)
9. [Questions You Should Ask Them](#questions-you-should-ask-them)
10. [Practical Prep Plan](#practical-prep-plan)
11. [Bottom Line](#bottom-line)

---

## What They’ll Evaluate

- Can you explain Argo CD / GitOps clearly to a non-engineer?
- Do you have enough hands-on or adjacent platform experience to earn trust later?
- Have you done discovery, demos, POCs, or objection handling before?
- Why this company / this product, not “any SE job”?

---

## Argo CD Fluency (Enough for Round 1)

Be ready to explain in ~60 seconds:

- **GitOps**: desired state in Git; cluster converges to that state
- **Argo CD core loop**: watch Git → compare live vs desired → sync/heal
- **Why buyers care**: auditability, faster/safer deploys, multi-cluster consistency, fewer snowflake clusters
- **Where it sits**: CD layer after CI; not a replacement for build/test
- **Nearby products**: ApplicationSets, AppProjects/RBAC, Rollouts (progressive delivery), **Kustomize** (manifest overlays), **Kargo** (stage-to-stage promotion), vs Flux as the common alternative

Know these phrases cold:

| Term | Why it matters in a sales conversation |
|------|----------------------------------------|
| Sync status | Desired (Git) vs live cluster match |
| Health status | Whether resources are actually healthy |
| Auto-sync | Continuous convergence from Git |
| Self-heal | Reverts manual/cluster drift |
| Drift | Live state diverged from Git |
| Multi-tenancy | Isolating teams/apps safely |
| SSO / OIDC | Enterprise identity requirements |
| App of Apps / ApplicationSets | Scaling app definitions across clusters/teams |

---

## Deployment Models with Argo CD

In customer conversations, “deployment model” usually means **two different layers**. Be explicit which one you’re talking about:

| Layer | Question it answers | Argo CD’s role |
|-------|---------------------|----------------|
| **GitOps / platform model** | How do we organize apps, envs, and clusters? | Applications, App of Apps, ApplicationSets, sync policy |
| **In-cluster release model** | How does version N become N+1 with controlled risk? | Syncs desired state; Rollouts/Flagger (or native Deployment) do traffic shifting |

```
CI builds image → Git updated (tag/digest) → Argo CD syncs manifests
                                              │
                                              ▼
                                    Deployment rolling update
                                    or Argo Rollouts canary / blue-green
```

### 1. How you structure apps in Argo CD

| Model | How it works | When to recommend |
|-------|--------------|-------------------|
| **One Application per service/env** | Explicit `Application` CR per combo | Small estates; easy to explain in demos |
| **App of Apps** | Root app points at a folder of Application manifests; bootstraps the platform | Platform bootstrap, “install everything from Git” story |
| **ApplicationSets** | Generators (list, cluster, git, SCM/PR) create many Applications from a template | Multi-env, multi-cluster, PR preview envs at scale |

**SE soundbite:** App of Apps is great for *bootstrap*; ApplicationSets are better for *scale and templating*.

### 2. Multi-cluster topologies

| Topology | Pattern | Buyer trade-off |
|----------|---------|-----------------|
| **Hub-and-spoke** | Central Argo CD manages many remote clusters | One control plane, simpler ops; hub is a blast-radius / HA concern |
| **Per-cluster Argo CD** | Argo CD runs in each cluster (or per region) | Stronger isolation / air-gap fit; more instances to upgrade and secure |
| **Hybrid** | Hub for most clusters; dedicated Argo for regulated / prod | Common enterprise compromise |

Ask early: *How many clusters today, and who is allowed to deploy to prod?*

### 3. Environment promotion models

Promotion should update **Git**, not `kubectl set image` on the cluster.

| Model | How promotion works | Typical sync stance |
|-------|---------------------|---------------------|
| **Path-per-env** | `overlays/dev` → `staging` → `prod`; PR bumps the next overlay’s image | Dev auto-sync; prod manual or PR-gated |
| **Branch-per-env** | Merge forward `dev` → `staging` → `prod` branches | Same idea with branch as the gate |
| **Kargo-orchestrated** | Warehouse detects artifacts → Freight promoted Stage → Stage; Kargo writes Git (often Kustomize image bump) | Argo CD still syncs; Kargo owns *when/what* promotes |
| **Progressive in-cluster** | Git jumps to the new digest once; **Argo Rollouts** ramps traffic inside the env | Argo CD syncs the Rollout CR; Rollouts owns canary steps |

Common enterprise pattern:

| Env | Git source | Sync |
|-----|------------|------|
| Dev | `overlays/dev` | Automated + self-heal |
| Staging | `overlays/staging` | Auto or manual after smoke tests |
| Prod | `overlays/prod` | Manual sync and/or gated PR |

### 4. Sync models (control plane behavior)

| Sync model | Meaning | Sales angle |
|------------|---------|-------------|
| **Manual sync** | Human (or pipeline) clicks/CLI syncs | Change control, regulated prod |
| **Automated sync** | Merge → cluster converges | Velocity, fewer snowflake clusters |
| **Auto-sync + prune + self-heal** | Full GitOps convergence | Strongest “desired state” story; needs good Git hygiene |
| **Sync windows / hooks / waves** | Timeboxed syncs; ordered PreSync → Sync → PostSync | Migrations, maintenance windows, multi-resource ordering |

### 5. In-cluster release strategies (what happens after sync)

Argo CD delivers desired manifests. **How pods and traffic flip** is mostly the workload controller:

| Strategy | Mechanism | Pair with Argo CD when… |
|----------|-----------|-------------------------|
| **Rolling update** | Native `Deployment` default | Stateless APIs; compatible N and N+1 |
| **Recreate** | Kill old, then start new | Dev, exclusive locks, hard version incompatibility |
| **Blue-green** | Two full versions; flip Service / Ingress | Need instant cutover + easy rollback flip |
| **Canary** | Shift a % of traffic; analyze; promote/abort | High-revenue paths with good metrics |
| **A/B / header routing** | Route by identity/header, not only % | Dogfood cohorts, mobile app versions |
| **Shadow / mirror** | Copy traffic to new version without user impact | Soak before a real canary |

**Argo Rollouts** is the natural upsell next to Argo CD: Git still owns desired state; Rollouts owns progressive delivery (analysis, promote, abort). Flagger is the common alternative in the same category.

Deeper patterns and commands: [`kubernetes-deployment-models.md`](./kubernetes-deployment-models.md).

### 6. Which model to recommend (quick decision tree)

```
Many services × many clusters?
├─ yes → ApplicationSets (+ cluster generator)
└─ no  → plain Applications; App of Apps for bootstrap

Manifests: Helm charts from vendors / complex templates?
├─ yes → Helm (Argo CD source type)
└─ no / env overlays of the same app → Kustomize base + overlays

Promotion today = brittle CI scripts / manual PRs across envs?
├─ yes → Kargo Stages + Freight (Argo CD still deploys)
└─ no  → path/branch promotion may be enough

Need progressive delivery / metric gates?
├─ yes → Argo CD + Argo Rollouts (or Flagger)
└─ no  → Argo CD syncing Deployments (rolling) is enough

Prod change control required?
├─ yes → path/branch or Kargo promotion + manual/gated sync or Stage verification
└─ no  → auto-sync with prune + self-heal
```

### 7. Phrases that land with buyers

- “CI builds; Git records the desired digest; Argo CD makes the cluster match.”
- “Promotion is a Git merge, not a SSH into prod.”
- “Kustomize overlays keep env diffs boring and reviewable.”
- “Argo CD deploys; Kargo promotes — stop encoding promotion in CI glue.”
- “Rolling is table stakes; canary/blue-green is Argo Rollouts on top of the same GitOps flow.”
- “App of Apps gets you started; ApplicationSets keep you from drowning in YAML at scale.”

---

## Kustomize (Manifest Composition)

**Kustomize** is Kubernetes’ native way to compose YAML: a shared **base** plus **overlays** that patch per environment — no template language required. Argo CD renders Kustomize natively (`kustomization.yaml` in the Application path).

### Mental model

```
apps/my-service/
  base/                 # Deployment, Service, shared labels
    kustomization.yaml
  overlays/
    dev/                # fewer replicas, debug flags, image tag
    staging/
    prod/               # HPA, stricter resources, pinned digest
```

Argo CD Application for prod typically points at `path: apps/my-service/overlays/prod`.

### Why buyers use it with Argo CD

| Strength | Sales angle |
|----------|-------------|
| Overlays = env diffs as patches | Reviewable PRs; less “copy the whole chart” drift |
| Built into `kubectl` / Argo CD | No extra render plugin for the common case |
| Image transformers | `kustomize edit set image …` (or Kargo doing that) updates the digest in Git |
| Works with raw YAML teams | Good landing zone when Helm feels heavy |

### Kustomize vs Helm (one-liner contrast)

| | **Kustomize** | **Helm** |
|--|---------------|----------|
| Style | Patch/overlay composition | Templates + values |
| Best when | Same app shape across envs; platform-owned YAML | Vendor charts, many optional features, value-driven installs |
| With Argo CD | `path` to overlay | `helm.valueFiles` / `helm.parameters` on the Application |

**SE soundbite:** Helm packages and parameterizes; Kustomize overlays and patches. Many platforms use **both** (Helm for third-party, Kustomize for first-party services).

### Demo talking points

1. Show `base` + `overlays/dev` vs `overlays/prod` (replicas, resources, image).
2. Change image in the overlay → commit → Argo CD shows OutOfSync → sync → Healthy.
3. Mention promotion often = “update the next overlay’s image” — manually, via CI, or via **Kargo**.

---

## Kargo (Continuous Promotion)

**Kargo** is a GitOps-native **continuous promotion** layer from the Argo / Akuity ecosystem. It does **not** replace Argo CD.

| Tool | Job |
|------|-----|
| **Argo CD** | Make *this* cluster match *this* Git path (deploy / reconcile) |
| **Kargo** | Decide *what* artifact set moves *which* Stage → Stage, and write that into Git |
| **Argo Rollouts** | How traffic shifts *inside* an environment after sync |

```
Warehouse watches images/Git/charts
        │
        ▼
     Freight  (versioned bundle: image + commits + charts)
        │
        ▼
 Stage: dev ──verify──► Stage: staging ──verify──► Stage: prod
        │                      │                        │
        └──────── updates Git (e.g. Kustomize image) ───┘
                              │
                              ▼
                         Argo CD syncs
```

### Core terms to know cold

| Term | Meaning |
|------|---------|
| **Warehouse** | Subscribes to artifact sources (container images, Git, Helm charts) and discovers new versions |
| **Freight** | The unit of promotion — a specific, versioned bundle of those artifacts |
| **Stage** | An environment (or logical step) in the promotion pipeline (dev → staging → prod) |
| **Promotion** | Moving Freight into a Stage (often Git commit + optional Argo CD refresh/sync) |
| **Verification** | Gates before further promotion (health, metrics, tests, manual approval) |

### Problem it solves (buyer language)

Argo CD fixed “deploy from Git.” Teams still hack **promotion** with:

- CI jobs that open PRs / push tags per env
- Humans cherry-picking image digests across overlays
- One-off scripts that become unowned platform debt

Kargo makes promotion a first-class, auditable pipeline: same GitOps principles, fewer custom scripts.

### How it pairs with Kustomize + Argo CD

Common happy path:

1. CI builds and pushes `ghcr.io/org/api@sha256:…`
2. Kargo Warehouse sees the new image → creates Freight
3. Promote to **dev** Stage → Kargo updates `overlays/dev` (e.g. `kustomize edit set image`) and commits
4. Argo CD syncs dev; verification passes
5. Promote Freight to **staging** / **prod** the same way (auto or approval-gated)

### When to bring Kargo into the conversation

| Signal from discovery | Pitch |
|-----------------------|-------|
| “We promote with Jenkins/GitHub Actions glue” | Replace fragile promotion scripts with Stages + Freight |
| Many services × many envs | Consistent promotion UX; less per-team snowflake CI |
| Need approvals / evidence between stages | Verification and gated promote without abandoning GitOps |
| Already on Argo CD / Akuity path | Natural adjacent product; same ecosystem mental model |

**Not** for: replacing Argo CD, building images, or in-cluster canary math (that’s Rollouts).

### Phrases that land

- “Argo CD reconciles; Kargo promotes.”
- “Freight is the release candidate that moves through Stages — not a one-off kubectl.”
- “Your overlays stay the source of truth; Kargo just automates the boring, error-prone Git updates.”

---

## Sales-Engineer Story to Rehearse

Have crisp answers for:

1. Walk me through a technical sale or POC you influenced.
2. Hardest customer objection you’ve handled.
3. How you partner with AEs (discovery → demo → POC → close).
4. A time you said “no” or scoped down a deal.
5. Why Argo / GitOps now (platform teams, Kubernetes sprawl, compliance, AI infra, multi-cloud).

Frame outcomes in buyer language:

- Lead time
- Change failure rate
- MTTR
- Blast radius
- Audit / compliance
- Platform team leverage

---

## Discovery Questions Worth Knowing

Recruiters love hearing that you ask good questions. Examples:

- How many clusters / environments / teams?
- Who owns the platform vs app teams?
- Current CD path (Helm / Kustomize / CI pipelines / Spinnaker / Flux)?
- Helm, Kustomize, or both for first-party apps?
- How do you promote today (same image tag across envs, or rebuild per env)? Manual PRs, CI scripts, or a promotion tool?
- Multi-tenancy and RBAC requirements?
- Progressive delivery / canaries needed?
- One Argo control plane for many clusters, or Argo per cluster / region?
- Air-gapped, SSO, or policy / Gatekeeper constraints?

---

## Competitive / Commercial Awareness

Depending on the employer (often Akuity or a partner/vendor in the Argo ecosystem), know:

- Open-source Argo CD strengths and gaps (enterprise support, UX, multi-cluster ops, SSO/RBAC at scale, managed offering)
- **Kustomize vs Helm** as the default “how do you render manifests?” conversation
- **Kargo** as continuous promotion next to Argo CD (especially relevant at Akuity); don’t confuse it with Rollouts
- Flux as the main GitOps alternative
- Broader “why not just CI pipelines + kubectl/Helm?” objection — answer includes Git as desired state **and** promotion without CI owning cluster credentials / env diffs

---

## Questions You Should Ask Them

- Is this more new-logo hunting, expansion, or partner-led?
- Typical buyer (platform eng, SRE, DevOps, security)?
- Expected demo/POC ownership vs solutions architect split?
- Ramp: product certs, shadowing, quota expectations?
- What makes a great SE fail or succeed on this team?

---

## Practical Prep Plan

1. Rehearse a 2-minute “who I am + why Argo SE” pitch.
2. Rebuild one tiny Argo CD demo mentally: app from Git → sync → drift → heal.
3. Skim Applications, ApplicationSets, sync policies, [Kustomize](#kustomize-manifest-composition), [Kargo](#kargo-continuous-promotion), and [deployment models](#deployment-models-with-argo-cd) (plus this repo’s Argo CD / K8s guides).
4. Be ready with one-sentence contrasts: App of Apps vs ApplicationSets; Kustomize vs Helm; Argo CD vs Kargo vs Rollouts; rolling vs canary.
5. Pick 2–3 customer-value stories from your past (even if not Argo-specific).
6. Research the company’s packaging (OSS vs enterprise vs managed, and whether Kargo is in the portfolio) so your “why here” isn’t generic.

---

## Bottom Line

For the recruitment call, lead with clear communication and customer outcomes; prove Argo literacy at the “I can teach GitOps and run a basic demo” level, not controller-internals depth. Know that **Kustomize** composes env overlays and **Kargo** promotes across stages while **Argo CD** reconciles clusters. Save architecture deep-dives for the SE/panel rounds.
