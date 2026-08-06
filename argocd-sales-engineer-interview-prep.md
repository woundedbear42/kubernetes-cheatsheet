# Argo CD Sales Engineer — Recruitment Call Prep

Practical prep guide for a sales engineer (SE) recruitment call focused on Argo CD / GitOps. A recruiter call is mostly fit and story, not a deep Argo CD whiteboard. Prepare to sound credible on GitOps and sellable on why you’d be good with customers.

Related technical refs in this repo:
- [`argocd-cheat-sheet.md`](./argocd-cheat-sheet.md) — CLI / day-to-day commands
- [`argocd-integration-guide.md`](./argocd-integration-guide.md) — install, GitOps layout, sync/promotion

---

## Table of Contents

1. [What They’ll Evaluate](#what-theyll-evaluate)
2. [Argo CD Fluency (Enough for Round 1)](#argo-cd-fluency-enough-for-round-1)
3. [Sales-Engineer Story to Rehearse](#sales-engineer-story-to-rehearse)
4. [Discovery Questions Worth Knowing](#discovery-questions-worth-knowing)
5. [Competitive / Commercial Awareness](#competitive--commercial-awareness)
6. [Questions You Should Ask Them](#questions-you-should-ask-them)
7. [Practical Prep Plan](#practical-prep-plan)
8. [Bottom Line](#bottom-line)

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
- **Nearby products**: ApplicationSets, AppProjects/RBAC, Rollouts (progressive delivery), vs Flux as the common alternative

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
- Multi-tenancy and RBAC requirements?
- Progressive delivery / canaries needed?
- Air-gapped, SSO, or policy / Gatekeeper constraints?

---

## Competitive / Commercial Awareness

Depending on the employer (often Akuity or a partner/vendor in the Argo ecosystem), know:

- Open-source Argo CD strengths and gaps (enterprise support, UX, multi-cluster ops, SSO/RBAC at scale, managed offering)
- Flux as the main GitOps alternative
- Broader “why not just CI pipelines + kubectl/Helm?” objection

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
3. Skim official docs on Applications, ApplicationSets, and sync policies (plus this repo’s Argo CD guides).
4. Pick 2–3 customer-value stories from your past (even if not Argo-specific).
5. Research the company’s packaging (OSS vs enterprise vs managed) so your “why here” isn’t generic.

---

## Bottom Line

For the recruitment call, lead with clear communication and customer outcomes; prove Argo literacy at the “I can teach GitOps and run a basic demo” level, not controller-internals depth. Save architecture deep-dives for the SE/panel rounds.
