# Shoppy Cart Full Documentation

**Production-Grade GitOps Platform on AWS EKS**

---

## 1. One-Paragraph Summary

This is a complete, production-shaped DevOps platform built on AWS EKS. It takes an 12-service microservices application (Google's Online Boutique) from `git push` to a live, observable, auto-scaling deployment at `https://faekcorp.site` — fully automated, with rollback safety, centralized logging, metrics, and alerting. The platform is split across five working repositories with clear ownership boundaries, driven by GitOps (ArgoCD), and secured with EKS Pod Identity (no OIDC/IRSA — see `09-whats-not-included.md`). Every component is implemented, validated, and documented with commands, checks, expected results, and real gotchas.

---

## 2. Architecture — ASCII Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  Developer pushes code to shoppy-cart-app                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  GitHub Actions (CI) on self-hosted runner                      │
│    ├── Detect changed services (git diff)                       │
│    ├── Build Docker images                                      │
│    ├── Trivy scan (advisory)                                    │
│    └── Push to ECR (git-<sha>)                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  CI update-gitops job                                           │
│    └── Opens PR in shoppy-cart-gitops (updates values.yaml)     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Human reviews + merges PR                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  ArgoCD (CD) — watches shoppy-cart-gitops                       │
│    ├── Renders Helm chart + Kustomize                           │
│    └── Auto-syncs cluster to Git                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  EKS cluster (shoppy-cart namespace)                            │
│    ├── 12 microservices + Redis                                 │
│    ├── HPA (frontend auto-scales 1→6 on CPU 50%)                │
│    └── loadgenerator (simulated traffic)                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Networking                                                     │
│    Gateway API → AWS ALB → Route53 → https://faekcorp.site     │
│    ExternalDNS (auto DNS) + ACM cert (apex + wildcard)          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Observability                                                  │
│    Metrics: Prometheus + Grafana + Alertmanager → Slack          │
│    Logs:    Elasticsearch + Filebeat + Kibana                    │
│    Scaling: Metrics Server → HPA                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. The 5 Pillars

| Pillar | What it delivers | Key tools |
|---|---|---|
| **Infrastructure** | Reproducible AWS environment (VPC, EKS, ECR, IAM, bastion, runner) | Terraform, AWS |
| **CI (Continuous Integration)** | Code → tested, scanned, immutable image in ECR | GitHub Actions, Docker, Trivy, ECR |
| **CD (Continuous Delivery)** | Git is truth; cluster converges to Git automatically | ArgoCD, Helm, Kustomize, Image Updater |
| **Observability** | Know if it's healthy, why it broke, get alerted before users notice | Prometheus, Grafana, Alertmanager, ELK |
| **Reliability** | Handle traffic spikes; recover from bad deploys in seconds | Metrics Server, HPA, ArgoCD rollback |

---

## 4. The 5 Working Repos + 1 Docs Repo

| Repo | Responsibility | Owner (role) | URL |
|---|---|---|---|
| **shoppy-cart-infra** | AWS infrastructure via Terraform | Cloud/Infra team | https://github.com/shoppy-cart-org/shoppy-cart-infra |
| **shoppy-cart-app** | 12 microservices source + CI | App Dev team | https://github.com/shoppy-cart-org/shoppy-cart-app |
| **shoppy-cart-gitops** | Kubernetes desired state (Helm, ArgoCD, Gateway API, HPA) | GitOps team | https://github.com/shoppy-cart-org/shoppy-cart-gitops |
| **shoppy-cart-platform** | ArgoCD governance (`AppProject`, guardrails) | Platform team | https://github.com/shoppy-cart-org/shoppy-cart-platform |
| **shoppy-cart-observability** | Monitoring + logging stacks | SRE/Observability team | https://github.com/shoppy-cart-org/shoppy-cart-observability |
| **shoppy-cart-doc** | This documentation | Everyone | https://github.com/shoppy-cart-org/shoppy-cart-doc |

**Boundary principle:** Each repo has one responsibility. Code ≠ config ≠ infra ≠ platform ≠ observability. The `shoppy-cart-gitops` repo is the app's runtime desired state; the `shoppy-cart-platform` repo defines the guardrails (`AppProject`) the app must deploy within. The `AppProject` is the contract between the two teams.

---

## 5. The End-to-End Flow

```
git push → CI build+scan → ECR image (git-<sha>)
    → CI opens PR in gitops repo (updates values.yaml)
    → You review + merge
    → ArgoCD syncs cluster to Git
    → Pods running new image
    → ExternalDNS creates Route53 record
    → Gateway/ALB routes https://faekcorp.site
    → Prometheus scrapes metrics, Filebeat ships logs
    → Grafana/Kibana show health; Alertmanager → Slack
    → HPA scales under load
```

**Result:** Zero-touch deploy from commit to production, with observability and rollback safety.

---

## 6. Key Lessons (Top 5)

| # | Lesson | Phase |
|---|---|---|
| 1 | **Pod Identity, not OIDC/IRSA** — Builder ID accounts cannot create OIDC providers. EKS Pod Identity is the correct fallback. | 4, 10, 12, 18 |
| 2 | **Manual ServiceAccount creation** for Pod Identity — the Terraform association is AWS-side only; the K8s SA must exist. | 10, 12, 18 |
| 3 | **Gateway listeners materialize only when an HTTPRoute is attached** — a `Programmed=True` Gateway with zero listeners is expected until a route is applied. | 11 |
| 4 | **Image Updater must let `create-pull-request` handle branch/commit/push** — manual `git` steps before the action cause it to skip. | 15 |
| 5 | **Rollback with auto-sync requires both** ArgoCD rollback **and** `git revert` — otherwise auto-sync re-deploys the bad image. | 16 |

Full list in `docs/07-gotchas.md`.

---

## 7. How to Follow

Read the docs in order:

1. `docs/01-overview.md` — what and why
2. `docs/02-architecture.md` — the 7 layers
3. `docs/03-repo-structure.md` — the 6 repos and their boundaries
4. `docs/04-prerequisites.md` — AWS, domain, org, tools, keys
5. `docs/05-build-order.md` — **the 19-phase implementation guide**
6. `docs/06-credential-matrix.md` — Pod Identity, PATs, scoped credentials
7. `docs/07-gotchas.md` — the 15 lessons (symptom → cause → fix)
8. `docs/08-runbooks.md` — rollback, troubleshooting, cleanup
9. `docs/09-whats-not-included.md` — honesty + next steps

---

## 8. What's NOT Included

This is a learning platform, not a complete production system. See `docs/09-whats-not-included.md` for the full list. Key simplifications:

- **No OIDC/IRSA** — EKS Pod Identity used instead (Builder ID account limitation).
- **No real databases** — Redis only, no RDS/Postgres.
- **No real payment/email/shipping** — mocks in the app.
- **No network policies / Pod Security Standards / Kyverno** — none.
- **No multi-region / disaster recovery.**
- **Single NAT Gateway** — cost trade-off, AZ failure risk documented.
- **Trivy is advisory** — CVEs in upstream base images do not fail the build.

---

## 9. Tech Stack

| Layer | Tools |
|---|---|
| **Infrastructure** | Terraform, AWS VPC, EKS, ECR, IAM, EC2, Route53, ACM, S3, EBS CSI |
| **CI** | GitHub Actions (self-hosted runner), Docker, Trivy, ECR |
| **CD** | ArgoCD, ArgoCD Image Updater, Helm, Kustomize |
| **Networking** | Gateway API, AWS Load Balancer Controller, ExternalDNS, Route53, ACM |
| **Observability** | kube-prometheus-stack (Prometheus, Grafana, Alertmanager), ECK (Elasticsearch, Kibana, Filebeat), Metrics Server |
| **Scaling** | HPA, loadgenerator (Locust) |
| **App** | Online Boutique (11 services, 6 languages) |
| **Identity** | EKS Pod Identity (no OIDC/IRSA) |

---

## 10. License

Apache-2.0 (matches the Online Boutique application license).

---

# docs/01-overview.md

# Overview

## What Is This Project?

This is a **production-grade GitOps platform** built on AWS EKS. It deploys Google's Online Boutique — an 11-service microservices e-commerce demo — and wraps it in the full enterprise DevOps toolchain: Terraform infrastructure, CI/CD with GitHub Actions and ArgoCD, Gateway API networking with automatic DNS, Prometheus/Grafana monitoring, ELK logging, HPA autoscaling, and rollback safety.

The application is the payload. **The platform is the deliverable.**

## Why This Project?

The goal is to demonstrate — and teach — the skills that separate a tutorial from a real platform:

- **Enterprise repo structure** — five working repos with clear ownership boundaries, plus a docs repo.
- **GitOps as truth** — the cluster converges to Git; every deploy is a reviewable commit.
- **Pod Identity over OIDC** — a real-world fallback for accounts that cannot create OIDC providers.
- **Observability as a first-class concern** — metrics, logs, and alerting, installed and verified.
- **Reliability** — autoscaling proven under load; rollback proven with a simulated bad deploy.
- **Honesty** — what is simplified and what a real production system would add.

## The 5 Pillars

| Pillar | What | Tools |
|---|---|---|
| **Infrastructure** | Reproducible AWS environment | Terraform, VPC, EKS, ECR, IAM, bastion, runner |
| **CI** | Code → image in ECR | GitHub Actions, Docker, Trivy, ECR |
| **CD** | Git → cluster | ArgoCD, Image Updater, Helm, Kustomize |
| **Observability** | Metrics + logs + alerts | Prometheus, Grafana, Alertmanager, Elasticsearch, Filebeat, Kibana |
| **Reliability** | Autoscale + rollback | Metrics Server, HPA, ArgoCD history, `git revert` |

## The 6 Repos

| Repo | Responsibility | URL |
|---|---|---|
| shoppy-cart-infra | AWS infrastructure (Terraform) | https://github.com/shoppy-cart-org/shoppy-cart-infra |
| shoppy-cart-app | App code + CI | https://github.com/shoppy-cart-org/shoppy-cart-app |
| shoppy-cart-gitops | Kubernetes desired state | https://github.com/shoppy-cart-org/shoppy-cart-gitops |
| shoppy-cart-platform | ArgoCD governance | https://github.com/shoppy-cart-org/shoppy-cart-platform |
| shoppy-cart-observability | Monitoring + logging | https://github.com/shoppy-cart-org/shoppy-cart-observability |
| shoppy-cart-doc | This documentation | https://github.com/shoppy-cart-org/shoppy-cart-doc |

## The End Result

After following `05-build-order.md` end-to-end, you will have:

- A live app at `https://faekcorp.site` (apex and `www`), with a valid ACM certificate.
- Zero-touch CI/CD: `git push` → image in ECR → GitOps PR → merge → ArgoCD deploy.
- Metrics dashboards at `https://grafana.faekcorp.site` and `https://prometheus.faekcorp.site`.
- Log search at `https://kibana.faekcorp.site`.
- Slack alerts on critical conditions.
- HPA scaling proven under load (1 → 3 → 6 replicas).
- Rollback proven with a simulated bad deploy.
- ArgoCD UI at `https://argocd.faekcorp.site`.

## Who Is This For?

- **Junior DevOps** — follow the commands and checks in `05-build-order.md` to implement the project.
- **Senior DevOps** — read the decisions and gotchas to understand *why* each choice was made.
- **Future you** — re-implement the project from scratch using the doc as the map and the repos as the code.
- **Recruiters** — the README and this overview give the story in 2 minutes.

---

# docs/02-architecture.md

# Architecture

## The 7 Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 7: RELIABILITY                                           │
│  HPA (auto-scale) + Rollback (ArgoCD history + git revert)      │
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 6: OBSERVABILITY                                         │
│  Prometheus/Grafana/Alertmanager + Elasticsearch/Filebeat/Kibana│
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 5: CD (GitOps)                                           │
│  ArgoCD + Image Updater + Helm + Kustomize                      │
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4: CI                                                    │
│  GitHub Actions + Docker + Trivy + ECR                          │
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3: NETWORKING & DNS                                      │
│  Gateway API + AWS LB Controller + ExternalDNS + Route53 + ACM  │
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2: KUBERNETES PLATFORM                                   │
│  EKS + node groups + addons + Pod Identity                      │
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: INFRASTRUCTURE                                        │
│  Terraform: VPC, subnets, IAM, ECR, bastion, runner             │
└─────────────────────────────────────────────────────────────────┘
```

Each layer depends on the one below it. You cannot deploy Layer 5 (CD) without Layer 2 (EKS) and Layer 3 (networking). The build order in `05-build-order.md` follows this dependency chain exactly.

## Layer 1: Infrastructure (Terraform)

**What it provisions:** VPC (3 AZs, public + private app + private data subnets), IGW, single NAT Gateway + VPC endpoints, security groups, IAM roles, ECR repos, bastion, self-hosted runner, key pairs, EKS cluster.

**Why it exists:** You need servers, networks, and identities before anything can run. Terraform makes the infrastructure reproducible, versioned, and destroyable in one command.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra

## Layer 2: Kubernetes Platform (EKS)

**What it runs:** EKS cluster v1.36 with a managed node group (3 × m7i-flex.large, one per AZ). Addons: vpc-cni, coredns, kube-proxy, ebs-csi, pod-identity-agent.

**Why it exists:** Kubernetes gives you self-healing, rolling updates, service discovery, and a huge ecosystem that expects it. EKS removes the control-plane operational burden.

**Identity:** **EKS Pod Identity** — not IRSA, not GitHub OIDC (see `06-credential-matrix.md` and `09-whats-not-included.md`).

## Layer 3: Networking & DNS (Gateway API + ExternalDNS)

**What it provides:** A `Gateway` with four listeners (HTTP/HTTPS × apex/wildcard), an `HTTPRoute` per app, an AWS ALB (created by the AWS Load Balancer Controller), an ACM certificate covering `faekcorp.site` + `*.faekcorp.site`, and automatic Route53 records (created by ExternalDNS).

**Why it exists:** Users reach the app at a real domain over HTTPS, with DNS and load balancer provisioning fully automated from Kubernetes resources.

**Key insight:** Listeners materialize only when an HTTPRoute is attached. A `Programmed=True` Gateway with zero listeners is expected until a route is applied.

## Layer 4: CI (GitHub Actions)

**What it does:** On push to `main`, detects changed services, builds Docker images, scans with Trivy (advisory), pushes to ECR with `git-<sha>` tags, and opens a PR in the GitOps repo updating `values.yaml`.

**Why it exists:** Builds must be consistent, scanned, and traceable to a commit. The GitOps PR is the human review gate before deploy.

## Layer 5: CD (ArgoCD)

**What it does:** Watches the GitOps repo. Renders the Helm chart and Kustomize overlays. Syncs the cluster to Git. Detects and reverts drift. Keeps history for rollback.

**Why it exists:** Git is the single source of truth. Every deploy is a reviewable commit. Rollback is a Git revert or an ArgoCD history rollback.

## Layer 6: Observability (Prometheus + ELK)

**What it does:** Prometheus scrapes metrics, Grafana visualizes them, Alertmanager routes critical alerts to Slack. Elasticsearch stores logs, Filebeat ships them, Kibana searches them.

**Why it exists:** You cannot run production blind. Metrics tell you *that* something is wrong; logs tell you *why*; alerting tells you *before users notice*.

## Layer 7: Reliability (HPA + Rollback)

**What it does:** Metrics Server collects CPU/memory; HPA scales pods when CPU exceeds the target. ArgoCD history + `git revert` allow rollback of bad deploys.

**Why it exists:** Traffic spikes are handled automatically. Bad deploys are recoverable in seconds — with the knowledge that auto-sync requires *both* an ArgoCD rollback *and* a `git revert` to avoid re-deploying the bad image.

## The Flow (End-to-End)

```
git push → CI build+scan → ECR image (git-<sha>)
    → CI opens PR in gitops repo → you review + merge
    → ArgoCD syncs cluster → pods running new image
    → ExternalDNS creates Route53 record
    → Gateway/ALB routes https://faekcorp.site
    → Prometheus scrapes, Filebeat ships logs
    → Grafana/Kibana show health; Alertmanager → Slack
    → HPA scales under load
```

---

# docs/03-repo-structure.md

# Repository Structure

## Why 5 Working Repos (Plus 1 Docs Repo)?

Each repo has **one responsibility**. Code, configuration, infrastructure, governance, and observability are different concerns with different lifecycles, different owners, and different change cadences. Bundling them creates coupling and confusion. The enterprise pattern is one repo per responsibility.

| Repo | Owner (role) | Responsibility | URL |
|---|---|---|---|
| **shoppy-cart-infra** | Cloud/Infra team | AWS infrastructure via Terraform | https://github.com/shoppy-cart-org/shoppy-cart-infra |
| **shoppy-cart-app** | App Dev team | 12 microservices source + CI | https://github.com/shoppy-cart-org/shoppy-cart-app |
| **shoppy-cart-gitops** | GitOps team | Kubernetes desired state (Helm, ArgoCD, Gateway API, HPA) | https://github.com/shoppy-cart-org/shoppy-cart-gitops |
| **shoppy-cart-platform** | Platform team | ArgoCD governance (`AppProject`, guardrails) | https://github.com/shoppy-cart-org/shoppy-cart-platform |
| **shoppy-cart-observability** | SRE/Observability team | Monitoring + logging | https://github.com/shoppy-cart-org/shoppy-cart-observability |
| **shoppy-cart-docs** | Everyone | This documentation | https://github.com/shoppy-cart-org/shoppy-cart-doc |

## Repo 1: shoppy-cart-infra

**What it owns:** VPC, subnets, NAT, VPC endpoints, security groups, IAM roles (runner, Terraform, bastion, app CI, EKS cluster, EKS node, EBS CSI Pod Identity, AWS LB Controller Pod Identity, ExternalDNS Pod Identity), ECR repos, bastion EC2, runner EC2, key pairs, EKS cluster + node group + addons + access entries + Pod Identity associations, Terraform CI/deploy workflows.

**What it does NOT own:** App source code, Kubernetes manifests, ArgoCD governance, observability stacks.

**How to validate:** `terraform init && terraform validate && terraform plan` from `environments/dev`. Apply via the `Terraform Deploy` workflow (manual trigger, `apply` confirmation).

**URL:** https://github.com/shoppy-cart-org/shoppy-cart-infra

## Repo 2: shoppy-cart-app

**What it owns:** 12 microservices source code, Dockerfiles, protos, CI workflow (`.github/workflows/ci.yml`), known-limitations doc.

**What it does NOT own:** Infrastructure, Kubernetes manifests, ArgoCD config, observability.

**How to validate:** Push a `src/` change; CI builds, scans, pushes to ECR, opens a PR in the GitOps repo.

**URL:** https://github.com/shoppy-cart-org/shoppy-cart-app

## Repo 3: shoppy-cart-gitops

**What it owns:** Gateway API resources (`GatewayClass`, `LoadBalancerConfiguration`, `Gateway`), ArgoCD config (values, `TargetGroupConfiguration`), Helm chart for the app, Kustomize networking overlays, HPA templates, loadgenerator parameterization, ArgoCD `Application` CRD for the app.

**What it does NOT own:** App source code, AWS infrastructure, ArgoCD `AppProject` governance, observability stacks.

**How to validate:** ArgoCD UI shows `online-boutique` `Synced` / `Healthy`; `kubectl get pods -n shoppy-cart` shows all services Running; `curl -I https://faekcorp.site` returns HTTP/2 200.

**URL:** https://github.com/shoppy-cart-org/shoppy-cart-gitops

## Repo 4: shoppy-cart-platform

**What it owns:** ArgoCD `AppProject` (guardrails: allowed repos, allowed destinations, allowed resource kinds), platform policies, onboarding docs.

**What it does NOT own:** The app's `Application` CRD (that's GitOps team), the app manifests, infrastructure.

**The contract:** The `AppProject` defines what the app's `Application` is allowed to do. The GitOps team's `Application` must comply.

**How to validate:** `kubectl get appproject -n argocd` shows the project; ArgoCD accepts the `online-boutique` Application under it.

**URL:** https://github.com/shoppy-cart-org/shoppy-cart-platform

## Repo 5: shoppy-cart-observability

**What it owns:** Monitoring stack (`kube-prometheus-stack` values, Grafana/Prometheus HTTPRoutes + TGCs), logging stack (StorageClass, ECK Helm values, Elasticsearch/Kibana/Filebeat Helm values, Kibana HTTPRoute + TGC), runbook.

**What it does NOT own:** App deployment, infrastructure, ArgoCD governance.

**How to validate:** `kubectl get pods -n monitoring` and `kubectl get pods -n logging` all Running; Grafana/Prometheus/Kibana accessible via their URLs; Slack Watchdog alert fired.

**URL:** https://github.com/shoppy-cart-org/shoppy-cart-observability

## Repo 6: shoppy-cart-docs

**What it owns:** This documentation (README + `docs/01`–`09` + `LICENSE`).

**URL:** https://github.com/shoppy-cart-org/shoppy-cart-doc

## The Boundary Principle

```
┌──────────────────────┐   ┌──────────────────────┐
│  shoppy-cart-infra   │   │  shoppy-cart-app     │
│  (Cloud/Infra)       │   │  (Dev team)          │
│  Terraform           │   │  Source + CI         │
└──────────┬───────────┘   └──────────┬───────────┘
           │                          │
           │ provisions               │ builds → ECR
           ▼                          ▼
┌──────────────────────┐   ┌──────────────────────┐
│  AWS (VPC, EKS, ECR) │   │  ECR (images)        │
└──────────┬───────────┘   └──────────┬───────────┘
           │                          │
           │ runs                     │ pulled by
           ▼                          ▼
┌──────────────────────┐   ┌──────────────────────┐
│  shoppy-cart-gitops  │◄──│  ArgoCD (in cluster) │
│  (GitOps team)       │   │  watches gitops repo │
│  Helm, ArgoCD, K8s   │   └──────────────────────┘
└──────────┬───────────┘
           │
           │ governed by
           ▼
┌──────────────────────┐   ┌──────────────────────┐
│  shoppy-cart-        │   │  shoppy-cart-        │
│  platform            │   │  observability       │
│  (Platform team)     │   │  (SRE team)          │
│  AppProject          │   │  Monitoring + Logging│
└──────────────────────┘   └──────────────────────┘

┌──────────────────────┐
│  shoppy-cart-docs    │
│  (this repo)         │
│  The story + links   │
└──────────────────────┘
```

---

# docs/04-prerequisites.md

# Prerequisites

Everything you need before starting Phase 1.

## AWS Account

- An AWS account with **AdministratorAccess** for the bootstrap IAM user (used only to create the initial Terraform infrastructure).
- **Region:** `eu-north-1` (Stockholm). The doc assumes this region throughout.
- **Important limitation:** This project was built on an AWS Builder ID / simplified account that **cannot create OIDC providers**. As a result, we use **EKS Pod Identity** instead of IRSA and GitHub OIDC. If your account supports OIDC, you could substitute IRSA/GitHub OIDC, but the doc follows the Pod Identity path that was actually used.
- **AWS Budget alert** set (e.g., $50) to avoid surprise bills.

## Domain + DNS

- A registered domain: `faekcorp.site` (replace with your own).
- A Route53 hosted zone for the domain.
- **Nameservers delegated** from your registrar to Route53 (verify with `dig NS faekcorp.site` or `nslookup -type=NS faekcorp.site`).
- An **ACM certificate** in `eu-north-1` covering **both** `faekcorp.site` and `*.faekcorp.site` (status `ISSUED`). Validation is via DNS CNAME records in Route53.

## GitHub Organization

- A GitHub organization: `shoppy-cart-org` (replace with your own).
- The five working repos created **empty** (no README, no `.gitignore`):
  - `shoppy-cart-infra`
  - `shoppy-cart-app`
  - `shoppy-cart-gitops`
  - `shoppy-cart-platform`
  - `shoppy-cart-observability`
- The docs repo: `shoppy-cart-docs`.
- **Self-hosted runner registered at the organization level** with labels `self-hosted, Linux, X64, aws, shoppy-cart` (the runner is installed in Phase 6).
- **PATs (fine-grained)** with the following scopes (all on the specific repos, not org-wide):
  - `GITOPS_PAT` — `Contents: Read and write`, `Pull requests: Read and write` on `shoppy-cart-gitops` (used by the app CI's `update-gitops` job).
  - ArgoCD repo read PAT — `Contents: Read-only` on `shoppy-cart-gitops` (used by ArgoCD to clone).
  - `ARGOCD_IMAGE_UPDATER_PAT` (if using Image Updater with write-back) — `Contents: Read and write` on `shoppy-cart-gitops`.

## Local Tools

| Tool | Version | Purpose |
|---|---|---|
| Terraform | >= 1.10.0 | Infrastructure |
| AWS CLI | v2 | AWS access |
| kubectl | matching EKS 1.36 | Cluster access (from bastion) |
| Helm | v3.14+ | Charts |
| `yq` | v4 | YAML editing in CI |
| `jq` | 1.6+ | JSON in CI |
| `git` | latest | Version control |
| `ssh` | OpenSSH | Bastion access |
| Docker (optional) | latest | Local image testing |

## SSH Key Pairs

Two key pairs are created by Terraform (Phase 6.5) and stored in the Terraform state:
- `sc-bastion-keypair` — laptop → bastion
- `sc-runner-keypair` — bastion → runner

Retrieve the private key with `terraform output -raw bastion_private_key > ~/.ssh/sc-bastion-keypair.pem` (or configure the module to use an existing key).

## Cost Estimate

Running cost per day (approx.) while the platform is up:

| Resource | Approx. daily cost |
|---|---|
| EKS control plane | $2.40 |
| 3 × m7i-flex.large nodes | $5.00 |
| NAT Gateway + data | $1.20 |
| ALB | $0.60 |
| EBS volumes (Elasticsearch 10Gi + others) | $0.10 |
| Bastion (t3.micro) + runner (t3.medium) | $0.80 |
| **Total** | **~$10 / day** |

**Destroy at the end of each session** (`08-runbooks.md` → Cleanup) to avoid ongoing cost.

---

# docs/05-build-order.md

# Build Order

## How to Use This Doc

Each phase has:

- **What** — one sentence.
- **Depends on** — which phases must be complete first.
- **Code** — the repo URL + folder path. Click to see the full implementation.
- **Commands** — what to run.
- **Checks** — what to verify, and the expected output.
- **Result** — a sample output showing what success looks like.
- **Gotchas** — what breaks, linked to `07-gotchas.md`.

**No full code is included here.** The repo is the source of truth for code. This doc is the map.

## The 19 Phases (Overview)

| Phase | What | Depends on | Milestone |
|---|---|---|---|
| 0 | Prerequisites (account, domain, org, tools, keys) | — | AWS CLI works, domain delegated, org created |
| 1 | Terraform foundation (S3 backend, providers, variables) | 0 | `terraform init` + `validate` + `plan` pass |
| 2 | VPC (subnets, IGW, NAT, endpoints, route tables) | 1 | Plan shows ~27 resources |
| 3 | Security groups (bastion, runner, ALB, workers) | 2 | Plan shows 4 SGs + rules |
| 4 | IAM (roles, policies, instance profiles, Pod Identity roles) | 1 | Plan shows IAM resources |
| 5 | Bastion (EC2, instance profile, tools) | 2, 3, 4 | SSH to bastion works; `kubectl` installed |
| 6 | Runner (EC2, tools, GitHub runner) | 2, 3, 4 | Runner Idle in GitHub org |
| 6.5 | Key pairs (bastion + runner) | 2 | Both key pairs in state |
| 7 | EKS (cluster, node group, addons, access entries) | 2, 3, 4, 5 | `kubectl get nodes` → 3 Ready (from bastion) |
| 8 | CI/CD workflows (Terraform plan/deploy) | 6, 7 | CI runs green on PR; deploy succeeds manually |
| 9 | ECR (12 repos, lifecycle, scan) | 4 | 12 ECR repos exist |
| 10 | AWS LB Controller (Pod Identity + Helm) | 7, 9 | 2 controller pods Running; Gateway API feature enabled |
| 11 | Gateway API (GatewayClass, LBConfig, Gateway) | 10 | Gateway `Programmed=True` |
| 12 | ExternalDNS (Pod Identity + Helm) | 11 | ExternalDNS pod Running; watching HTTPRoutes |
| 13 | ArgoCD (Helm + HTTPRoute + TGC) | 10, 11, 12 | ArgoCD UI accessible at `argocd.faekcorp.site` |
| 14 | App deployment (Helm + ArgoCD Application) | 13 | `online-boutique` Synced/Healthy; app at `faekcorp.site` |
| 15 | Image Updater (CI → GitOps PR) | 14 | Push a `src/` change → CI opens PR → merge → ArgoCD deploys |
| 16 | Rollback (runbook + test) | 14 | Simulate bad deploy → rollback → app recovers |
| 17 | Monitoring (kube-prometheus-stack + Slack) | 13 | Grafana/Prometheus accessible; Watchdog alert in Slack |
| 18 | Logging (ECK + Elasticsearch + Kibana + Filebeat) | 17 | Kibana shows logs from `shoppy-cart` namespace |
| 19 | Scaling (Metrics Server + HPA + loadgenerator) | 14, 17 | HPA scales frontend 1→3 (or 6) under load |

---

## Phase 0: Prerequisites

**What:** Account, domain, GitHub org, local tools, SSH keys.

**Depends on:** Nothing.

**Code:** N/A (manual setup).

### Commands

```bash
# Verify AWS CLI
aws sts get-caller-identity

# Verify domain delegation
dig NS faekcorp.site

# Verify ACM cert status
aws acm list-certificates --region eu-north-1 \
  --query 'CertificateSummaryList[?DomainName==`faekcorp.site`]'

# Verify Terraform
terraform version
```

### Checks

| Check | Command | Expected |
|---|---|---|
| AWS identity | `aws sts get-caller-identity` | Your account ID returned |
| Domain delegated | `dig NS faekcorp.site` | 4 AWS nameservers |
| ACM issued | `aws acm list-certificates ...` | `Status: ISSUED` |
| Terraform available | `terraform version` | >= 1.10.0 |

### Result

```
$ aws sts get-caller-identity
{
    "UserId": "AIDA...",
    "Account": "138798371673",
    "Arn": "arn:aws:iam::138798371673:user/fouda-admin"
}
```

---

## Phase 1: Terraform Foundation

**What:** S3 backend (created manually in Phase 0), provider versions, variables, backend config, empty `main.tf` / `outputs.tf`.

**Depends on:** Phase 0.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/environments/dev

### Commands

From your laptop, in `shoppy-cart-infra/environments/dev`:

```bash
terraform init
terraform validate
terraform plan
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Backend connected | `terraform init` | "Successfully configured the backend" |
| Config valid | `terraform validate` | "Success! The configuration is valid." |
| Nothing to create | `terraform plan` | "No changes." |

### Result

```
$ terraform plan
No changes. Your infrastructure matches the configuration.
```

### Gotchas

- S3 bucket must exist before `terraform init`. See `07-gotchas.md` #3 (backend).

---

## Phase 2: VPC

**What:** VPC `10.70.0.0/16`, 3 public + 3 private app + 3 private data subnets across 3 AZs, IGW, single NAT Gateway + EIP, 4 VPC endpoints (ECR API, ECR Docker, STS interface + S3 gateway), route tables and associations.

**Depends on:** Phase 1.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/vpc

### Commands

```bash
cd environments/dev
terraform fmt -recursive
terraform init
terraform validate
terraform plan
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Plan resource count | `terraform plan` | ~27 resources |
| VPC CIDR | Check plan output | `10.70.0.0/16` |
| NAT Gateway | Check plan | 1 NAT + 1 EIP |
| VPC endpoints | Check plan | 4 endpoints |

### Result

```
Plan: 27 to add, 0 to change, 0 to destroy.
```

### Gotchas

- Removing `region` from `aws_vpc`; region belongs in provider. See `07-gotchas.md` (general Terraform hygiene).

---

## Phase 3: Security Groups

**What:** Four security groups: `sc-bastion-sg` (SSH from your IP), `sc-runner-sg` (no internet inbound; outbound all), `sc-alb-sg` (HTTP/HTTPS inbound), `sc-eks-node-sg` (node-to-node, ALB → nodes, bastion → nodes).

**Depends on:** Phase 2.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/security-groups

### Commands

```bash
cd environments/dev
terraform fmt -recursive
terraform init
terraform validate
terraform plan
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Plan resource count | `terraform plan` | ~15 resources (4 SGs + 11 rules) |
| Bastion ingress | Check plan | 22/tcp from your `/32` |
| Node self-ref | Check plan | workers → workers |

### Result

```
Plan: 42 to add, 0 to change, 0 to destroy.   # 27 VPC + 15 SG
```

### Gotchas

- Unique rule names required; `from_port`/`to_port` omitted for `ip_protocol = "-1"`.

---

## Phase 4: IAM

**What:** `ScRunnerRole` (EC2 trust), `ScTerraformRole` (trusts runner role, full infra permissions), `ScBastionRole` (`eks:DescribeCluster`), `ScAppCiRole` (ECR push, trusts runner role), EKS cluster role, EKS node role, EBS CSI Pod Identity role, AWS LB Controller Pod Identity role, ExternalDNS Pod Identity role. Customer-managed policies for each.

**Depends on:** Phase 1.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/iam

### Commands

```bash
cd environments/dev
terraform fmt -recursive
terraform init
terraform validate
terraform plan
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Plan resource count | `terraform plan` | IAM roles + policies + attachments + instance profiles |
| Runner role trusts EC2 | `aws iam get-role --role-name ScRunnerRole --query 'Role.AssumeRolePolicyDocument'` | `ec2.amazonaws.com` |
| Terraform role trusts runner | `aws iam get-role --role-name ScTerraformRole --query 'Role.AssumeRolePolicyDocument'` | Runner role ARN |
| App CI role trusts runner | `aws iam get-role --role-name ScAppCiRole --query 'Role.AssumeRolePolicyDocument'` | Runner role ARN |

### Result

```
$ aws iam get-role --role-name ScTerraformRole --query 'Role.AssumeRolePolicyDocument.Statement[0].Principal'
{
    "AWS": "arn:aws:iam::138798371673:role/ScRunnerRole"
}
```

### Gotchas

- `sts:TagSession` must be in the trust policy for role chaining. See `07-gotchas.md` #1.
- Pod Identity roles trust `pods.eks.amazonaws.com`, not an OIDC provider.

---

## Phase 5: Bastion

**What:** t3.micro EC2 in a public subnet, `sc-bastion-keypair`, `ScBastionRole` instance profile, `install-tools.sh` (kubectl, helm, argocd CLI, terraform, docker, aws-cli, kustomize, yq).

**Depends on:** Phases 2, 3, 4.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/bastion

### Commands

```bash
# From laptop
cd environments/dev
terraform apply

# Get bastion IP
terraform output bastion_public_ip

# SSH
ssh -i ~/.ssh/sc-bastion-keypair.pem ubuntu@<bastion_public_ip>
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Bastion reachable | `ssh ...` | Ubuntu prompt |
| Tools installed | `kubectl version --client && helm version && argocd version --client` | Version output |
| AWS identity | `aws sts get-caller-identity` | Bastion role ARN |

### Result

```
$ kubectl version --client
Client Version: v1.36.x
$ helm version
version.BuildInfo{Version:"v3.x"}
```

### Gotchas

- `install-tools.sh` may not run at first boot (cloud-init timing). If tools are missing, run `~/install-tools.sh` manually after SSH. See `07-gotchas.md` (known limitation).

---

## Phase 6: Runner

**What:** t3.medium EC2 in a private subnet, `sc-runner-keypair`, `ScRunnerRole` instance profile, `install-tools.sh` + `install-runner.sh` + `register-runner.sh`. Registered as an **org-level** self-hosted runner with labels `self-hosted, Linux, X64, aws, shoppy-cart`.

**Depends on:** Phases 2, 3, 4.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/runner

### Commands

```bash
# SSH via bastion
ssh -J ubuntu@<bastion> -i ~/.ssh/sc-runner-keypair.pem ubuntu@<runner_private_ip>

# On the runner, get a fresh registration token from GitHub org settings
# Then run:
~/actions-runner/register-runner.sh   # prompts for token
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Runner registered | GitHub org → Settings → Actions → Runners | `shoppy-cart-runner-01` Idle |
| Identity chain | (on runner) `aws sts get-caller-identity` | `ScRunnerRole` |

### Gotchas

- The runner registration token is short-lived and **must not be committed**. The registration is manual. See `07-gotchas.md` (security).

---

## Phase 6.5: Key Pairs

**What:** Two AWS key pairs created by Terraform (`sc-bastion-keypair`, `sc-runner-keypair`). Runner module retrofitted with `key_name = "sc-runner-keypair"`.

**Depends on:** Phase 2.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/keypair

### Checks

| Check | Command | Expected |
|---|---|---|
| Key pairs exist | `aws ec2 describe-key-pairs --region eu-north-1` | Both names present |

---

## Phase 7: EKS

**What:** EKS cluster v1.36 with private endpoint + restricted public endpoint, 3 × m7i-flex.large nodes (one per AZ, private subnets, AL2023, IMDSv2 required), addons (vpc-cni, coredns, kube-proxy, ebs-csi, **pod-identity-agent**), bastion access entry (`AmazonEKSClusterAdminPolicy`), EKS API SG rule for bastion.

**Depends on:** Phases 2, 3, 4, 5.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/eks

### Commands

From the bastion:

```bash
aws eks update-kubeconfig --name sc-eks-cluster --region eu-north-1
kubectl get nodes
kubectl get pods -n kube-system
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Nodes Ready | `kubectl get nodes` | 3 nodes, `Ready` |
| Addons Running | `kubectl get pods -n kube-system` | coredns, vpc-cni, kube-proxy, ebs-csi, pod-identity-agent all Running |
| Bastion can reach API | `kubectl get ns` | List of namespaces |

### Result

```
$ kubectl get nodes
NAME                                          STATUS   ROLES    AGE   VERSION
ip-10-70-21-213.eu-north-1.compute.internal   Ready    <none>   54m   v1.36.5
ip-10-70-22-83.eu-north-1.compute.internal    Ready    <none>   54m   v1.36.5
ip-10-70-23-43.eu-north-1.compute.internal    Ready    <none>   54m   v1.36.5
```

### Gotchas

- Bastion cannot reach EKS API without the SG rule. See `07-gotchas.md` #5.
- Pod Identity agent must be installed **before** any Pod Identity workloads. See `07-gotchas.md` #1.

---

## Phase 8: CI/CD Workflows (Infra)

**What:** Two GitHub Actions workflows in `shoppy-cart-infra`:
- `terraform-ci.yml` — PR: fmt, validate, plan, comment.
- `terraform-deploy.yml` — manual: plan, upload artifact, apply artifact after approval.

**Depends on:** Phases 6, 7.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/.github/workflows

### Commands

```bash
# Create a branch and push a trivial change to trigger CI
git switch -c test/ci
git commit --allow-empty -m "test: trigger CI"
git push -u origin test/ci
```

### Checks

| Check | Command | Expected |
|---|---|---|
| CI runs | GitHub Actions | `Terraform CI` workflow green |
| Plan posted | PR comments | Plan summary posted |
| Deploy works | Actions → Terraform Deploy → Run workflow → `apply` | Both jobs green |

### Gotchas

- The Terraform role needs `sts:TagSession`. See `07-gotchas.md` #1.
- `set +e` around `terraform plan` (exit code 2 = changes). See `07-gotchas.md` #2.

---

## Phase 9: ECR

**What:** 12 private ECR repos (`shoppy-cart/<service>`), immutable tags, scan on push, lifecycle policy (30 tagged, 7-day untagged).

**Depends on:** Phase 4.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/ecr

### Commands

Via the `Terraform Deploy` workflow after merging the ECR PR. Then from laptop:

```bash
aws ecr describe-repositories --region eu-north-1 \
  --query 'repositories[?starts_with(repositoryName, `shoppy-cart/`)].repositoryName'
```

### Checks

| Check | Command | Expected |
|---|---|---|
| 12 repos | `aws ecr describe-repositories ...` | 12 names |
| Scan on push | `aws ecr describe-repositories --query '...imageScanningConfiguration'` | `scanOnPush: true` |
| Immutable tags | `...imageTagMutability` | `IMMUTABLE` |

### Gotchas

- `depends_on = [module.iam]` on the ECR module to avoid a race with the IAM policy update. See `07-gotchas.md` #3.

---

## Phase 10: AWS Load Balancer Controller

**What:** `ScAwsLbControllerRole` (trusts `pods.eks.amazonaws.com`), official AWS policy + `ec2:ValidateSecurityGroupQuotasForInterface`, Pod Identity association, Helm install of `aws-load-balancer-controller` (v3.6.2, later rolled back and re-upgraded in troubleshooting), manual ServiceAccount creation, `ALBGatewayAPI=true` feature flag.

**Depends on:** Phases 7, 9.

**Code:**
- IAM: https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/iam
- EKS Pod Identity association: https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/eks

### Commands

On the bastion:

```bash
kubectl create sa aws-load-balancer-controller -n kube-system

helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --version 3.6.2 \
  --set clusterName=sc-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=eu-north-1 \
  --set vpcId=<vpc-id> \
  --set featureGates.ALBGatewayAPI=true
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Pods Running | `kubectl get pods -n kube-system \| grep aws-load-balancer` | 2 pods Running |
| ServiceAccount annotated | `kubectl get sa aws-load-balancer-controller -n kube-system -o yaml` | `eks.amazonaws.com/role-arn` |

### Gotchas

- Manual SA creation required — the Pod Identity association is AWS-side only. See `07-gotchas.md` #6.
- The official policy predates `ec2:ValidateSecurityGroupQuotasForInterface` (added Sept 2026). See `07-gotchas.md` #4.

---

## Phase 11: Gateway API

**What:** `GatewayClass` (`aws-alb-gateway-class`), `LoadBalancerConfiguration` (internet-facing, ACM cert), `Gateway` with four listeners (http-apex, http-wildcard, https-apex, https-wildcard).

**Depends on:** Phase 10.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-gitops/tree/main/infrastructure/gateway-api

### Commands

On the bastion:

```bash
kubectl apply -f infrastructure/gateway-api/gateway-class.yaml
kubectl apply -f infrastructure/gateway-api/alb-config.yaml
kubectl apply -f infrastructure/gateway-api/gateway.yaml
```

### Checks

| Check | Command | Expected |
|---|---|---|
| GatewayClass accepted | `kubectl get gatewayclass aws-alb-gateway-class` | `ACCEPTED=True` |
| Gateway programmed | `kubectl get gateway app-alb-gateway -n default` | `PROGRAMMED=True` |
| ALB active | `aws elbv2 describe-load-balancers --region eu-north-1 --query 'LoadBalancers[?contains(LoadBalancerName, \`app-alb-gateway\`)]'` | `State: active` |

### Result

```
$ kubectl get gateway app-alb-gateway -n default
NAME              CLASS                   ADDRESS                                                                  PROGRAMMED   AGE
app-alb-gateway   aws-alb-gateway-class   k8s-default-appalbga-...elb.amazonaws.com                                True         27m
```

### Gotchas

- Listeners materialize only when an HTTPRoute is attached. See `07-gotchas.md` #5.

---

## Phase 12: ExternalDNS

**What:** `ScExternalDnsRole` (trusts `pods.eks.amazonaws.com`, Route53 `ChangeResourceRecordSets` on your zone), Pod Identity association, Helm install of `external-dns` v1.20.0 with `sources: service, ingress, gateway-httproute`, `policy: upsert-only`, `domainFilters: faekcorp.site`, manual ServiceAccount creation.

**Depends on:** Phase 11.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-infra/tree/main/modules/iam

### Commands

On the bastion:

```bash
kubectl create namespace external-dns
kubectl create sa external-dns -n external-dns

helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update
helm install external-dns external-dns/external-dns \
  -n external-dns \
  --version 1.20.0 \
  --set provider.name=aws \
  --set policy=upsert-only \
  --set 'sources[0]=service' \
  --set 'sources[1]=ingress' \
  --set 'sources[2]=gateway-httproute' \
  --set 'domainFilters[0]=faekcorp.site' \
  --set aws.region=eu-north-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Pod Running | `kubectl get pods -n external-dns` | 1/1 Running |
| Watching HTTPRoutes | `kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns` | "Applying provider record filter for domains: [faekcorp.site.]" |

### Gotchas

- Manual SA creation, same as LB Controller. See `07-gotchas.md` #6.

---

## Phase 13: ArgoCD

**What:** Helm install of `argo-cd` v10.14.0 (app v3.6.1) with `server.insecure: true`, `kustomize.buildOptions: --enable-helm`, HTTPRoute at `argocd.faekcorp.site`, `TargetGroupConfiguration` (`targetType: ip`), repo registration (read-only PAT).

**Depends on:** Phases 10, 11, 12.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-gitops/tree/main/infrastructure/argocd

### Commands

On the bastion:

```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argo-cd argo/argo-cd -n argocd -f argocd-values.yaml --version 10.14.0
kubectl apply -f target-grp-config.yaml

# Register the GitOps repo (read-only PAT)
argocd login localhost:8080 --username admin --password <password> --insecure
argocd repo add https://github.com/shoppy-cart-org/shoppy-cart-gitops.git \
  --username <your-github-username> --password <read-only-PAT>
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Pods Running | `kubectl get pods -n argocd` | 7 pods Running |
| HTTPRoute accepted | `kubectl describe httproute -n argocd argo-cd-argocd-server` | `Accepted: True`, `ResolvedRefs: True` |
| DNS record | `aws route53 list-resource-record-sets --hosted-zone-id <id> --query 'ResourceRecordSets[?Name==`argocd.faekcorp.site.`]'` | A + AAAA to ALB |
| UI reachable | `curl -I https://argocd.faekcorp.site` | HTTP/2 200 |
| Admin password | `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" \| base64 -d` | string |

### Gotchas

- ArgoCD cannot access a private repo without credentials. Register it with a read-only PAT. See `07-gotchas.md` #7.
- The `TargetGroupConfiguration` must use `gateway.k8s.aws/v1` (not `v1beta1`) for LBC v3.6.2.

---

## Phase 14: App Deployment

**What:** Helm chart for Online Boutique (12 services + Redis) with ECR images, Kustomize networking (HTTPRoute for `faekcorp.site` + `www`), multi-source ArgoCD `Application`, namespace `shoppy-cart`, `syncPolicy.automated` with `prune` + `selfHeal`.

**Depends on:** Phase 13.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-gitops/tree/main/apps/online-boutique

### Commands

On the bastion:

```bash
kubectl apply -f argocd-apps/online-boutique.yaml
kubectl get application -n argocd online-boutique -w
```

### Checks

| Check | Command | Expected |
|---|---|---|
| App Synced | `argocd app get online-boutique` | `Sync Status: Synced`, `Health: Healthy` |
| Pods Running | `kubectl get pods -n shoppy-cart` | 12 services + redis Running |
| DNS record | `aws route53 list-resource-record-sets --hosted-zone-id <id> --query 'ResourceRecordSets[?Name==`faekcorp.site.`]'` | A to ALB |
| App reachable | `curl -I https://faekcorp.site` | HTTP/2 200 |

### Result

```
$ kubectl get pods -n shoppy-cart
NAME                          READY   STATUS    RESTARTS   AGE
adservice-...                 1/1     Running   0          10m
cartservice-...               1/1     Running   0          10m
... (12 services + redis)
```

### Gotchas

- Multi-source `Application` (Helm + Kustomize) required. See `07-gotchas.md` #8.
- `CreateNamespace=true` in `syncOptions` or pre-create the `shoppy-cart` namespace.

---

## Phase 15: Image Updater

**What:** Add `update-gitops` job to `shoppy-cart-app/.github/workflows/ci.yml` (runs after `build-and-push`, on push to `main`, only changed services). Clones the GitOps repo with `GITOPS_PAT`, updates `values.yaml` with `yq`, uses `peter-evans/create-pull-request@v6` to open a PR. `GITOPS_PAT` is a fine-grained PAT with `Contents: Read and write` + `Pull requests: Read and write` + `Issues: Write` on `shoppy-cart-gitops`.

**Depends on:** Phase 14.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-app/tree/main/.github/workflows

### Commands

```bash
# Trigger a build: push a change to src/frontend/
git switch -c test/full-loop
echo "# test" >> src/frontend/README.md
git commit -am "test: trigger CI -> GitOps PR -> deploy"
git push -u origin test/full-loop
# Open + merge PR on GitHub → push to main triggers CI
```

### Checks

| Check | Command | Expected |
|---|---|---|
| App CI builds frontend | GitHub Actions | `build-and-push` green |
| `update-gitops` opens PR | GitHub → shoppy-cart-gitops PRs | `chore: update images to git-<sha>` |
| PR diff clean | PR files changed | only `values.yaml`, only changed services |
| After merge, ArgoCD syncs | `argocd app get online-boutique` | Synced, new pod age |
| Pod uses new image | `kubectl describe pod -n shoppy-cart -l app=frontend \| grep image` | `git-<new-sha>` |

### Gotchas

- `create-pull-request` expects **uncommitted** changes. Do **not** `git checkout -b` / `git commit` / `git push` before calling the action. See `07-gotchas.md` #9.
- `GITOPS_PAT` needs `Pull requests: Write` and `Issues: Write`, not just `Contents`. See `07-gotchas.md` #10.

---

## Phase 16: Rollback

**What:** Document and test the rollback procedure. The runbook lives at `shoppy-cart-gitops/docs/runbooks/rollback.md`.

**Depends on:** Phase 14.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-gitops/tree/main/docs/runbooks

### Commands

**Simulate a bad deploy:**

```bash
cd shoppy-cart-gitops
yq -i '.images.frontend.tag = "git-bad0000000000000000000000000000000000"' apps/online-boutique/helm/values.yaml
git add apps/online-boutique/helm/values.yaml
git commit -m "test: simulate bad deploy"
git push origin main
```

**Observe:** `kubectl get pods -n shoppy-cart -l app=frontend` → `ImagePullBackOff`; app **still up** thanks to the old pod.

**Rollback (ArgoCD UI or CLI):**

```bash
argocd app set online-boutique --sync-policy none
argocd app history online-boutique
argocd app rollback online-boutique 1
```

**Fix Git:**

```bash
git revert <bad-commit-sha>
git push origin main
```

**Re-enable auto-sync:**

```bash
argocd app set online-boutique --sync-policy automated --auto-prune --self-heal
argocd app sync online-boutique
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Bad pod fails | `kubectl get pods -n shoppy-cart -l app=frontend` | `ImagePullBackOff` |
| App still accessible | `curl -I https://faekcorp.site` | HTTP/2 200 (old pod serving) |
| After rollback, pods recover | `kubectl get pods ...` | Running |
| Good image restored | `kubectl describe pod ... \| grep image` | good tag |
| ArgoCD Healthy | `argocd app get online-boutique` | Synced/Healthy |

### Gotchas

- With auto-sync on, ArgoCD rollback alone is not enough — the next sync re-deploys the bad image. You must **also** `git revert`. See `07-gotchas.md` #11.

---

## Phase 17: Monitoring

**What:** `kube-prometheus-stack` v92.21.1 in `monitoring` namespace, Slack webhook as k8s secret, Alertmanager configured to route critical alerts to Slack, HTTPRoutes for `grafana.faekcorp.site` and `prometheus.faekcorp.site`, `TargetGroupConfiguration` (`targetType: ip`) for both.

**Depends on:** Phase 13.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-observability/tree/main/monitoring

### Commands

On the bastion:

```bash
kubectl create namespace monitoring
kubectl create secret generic alertmanager-slack-webhook \
  --from-literal=slack-webhook-url='<YOUR_SLACK_WEBHOOK_URL>' \
  -n monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade -i kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 92.21.1 \
  -f monitoring/helm-values/kube-prom-stack.yaml \
  -n monitoring

kubectl apply -f monitoring/httproute-grafana.yaml
kubectl apply -f monitoring/httproute-prometheus.yaml
kubectl apply -f monitoring/targetgrp-grafana.yaml
kubectl apply -f monitoring/targetgrp-prometheus.yaml
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Pods Running | `kubectl get pods -n monitoring` | 8 pods Running |
| Grafana reachable | `curl -I https://grafana.faekcorp.site` | 302 → /login |
| Prometheus reachable | `curl -I https://prometheus.faekcorp.site` | 405 (HEAD not allowed) |
| Slack alert | Slack channel | `[FIRING:1] Watchdog` message |

### Gotchas

- The default Filebeat config from Elastic's chart is more robust than the raw CRD version — but install either way.
- The Slack webhook must be in a secret, not in values.yaml.

---

## Phase 18: Logging

**What:** ECK Operator v3.5.0 in `elastic-system` (or `logging`), StorageClass `ebs-sc` (gp3, encrypted, `WaitForFirstConsumer`), Elasticsearch 9.5.x (single node, 10Gi EBS), Filebeat DaemonSet (all nodes), Kibana, HTTPRoute for `kibana.faekcorp.site`, `TargetGroupConfiguration` (`targetType: ip`, `protocol: HTTPS`, `healthCheckConfig` HTTPS `/api/status`).

**Depends on:** Phase 17.

**Code:** https://github.com/shoppy-cart-org/shoppy-cart-observability/tree/main/logging

### Commands

On the bastion:

```bash
kubectl create namespace logging
kubectl apply -f logging/storageclass.yaml

helm repo add elastic https://helm.elastic.co
helm repo update

helm install eck-operator elastic/eck-operator -n logging --version 3.5.0

helm install eck-elasticsearch elastic/eck-elasticsearch -n logging \
  --version 0.20.0 -f logging/helm-values/eck-elasticsearch.yaml

helm install eck-beats elastic/eck-beats -n logging \
  --version 0.20.0 -f logging/helm-values/eck-beats.yaml

helm install eck-kibana elastic/eck-kibana -n logging \
  --version 0.20.0 -f logging/helm-values/eck-kibana.yaml

kubectl apply -f logging/httproute-kibana.yaml
kubectl apply -f logging/targetgrp-kibana.yaml
```

### Checks

| Check | Command | Expected |
|---|---|---|
| ECK Running | `kubectl get pods -n logging -l app.kubernetes.io/name=elastic-operator` | Running |
| ES Running | `kubectl get elasticsearch -n logging` | HEALTH green |
| Kibana Running | `kubectl get kibana -n logging` | HEALTH green |
| Filebeat Running | `kubectl get pods -n logging -l common.k8s.elastic.co/type=beat` | 3 pods (or 2/3 if a node is full — documented) |
| Kibana reachable | `curl -I https://kibana.faekcorp.site` | 302 → /login |
| Logs visible | Kibana → Discover → `filebeat-*` | Logs from `shoppy-cart` namespace |

### Gotchas

- Kibana Service is `eck-kibana-kb-http` (not `shoppy-kibana-kb-http`). See `07-gotchas.md` #12.
- The `TargetGroupConfiguration` health check must be nested under `healthCheckConfig`. See `07-gotchas.md` #13.
- A DaemonSet pod can be Pending if the node is at pod capacity (29/29). See `07-gotchas.md` #15.

---

## Phase 19: Scaling

**What:** Metrics Server v3.14.0 (`kube-system`), HPA for frontend (`min: 1`, `max: 6`, CPU `50%`) in the Helm chart (`hpa-frontend.yaml`), `loadGenerator.users` parameterized in `values.yaml` (set to `150`). HPA must be in the Helm chart, not `kubectl edit`.

**Depends on:** Phases 14, 17.

**Code:**
- Metrics Server install: (bastion)
- HPA template: https://github.com/shoppy-cart-org/shoppy-cart-gitops/tree/main/apps/online-boutique/helm/templates
- Values: https://github.com/shoppy-cart-org/shoppy-cart-gitops/tree/main/apps/online-boutique/helm/values.yaml

### Commands

On the bastion:

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm install metrics-server metrics-server/metrics-server \
  -n kube-system --version 3.14.0

kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes
```

Then commit the HPA + `loadGenerator.users: 150` changes to the GitOps repo, merge, and watch:

```bash
kubectl get hpa -n shoppy-cart -w
kubectl get pods -n shoppy-cart -l app=frontend -w
kubectl top pods -n shoppy-cart -l app=frontend
```

### Checks

| Check | Command | Expected |
|---|---|---|
| Metrics Server works | `kubectl top nodes` | CPU/Memory per node |
| HPA exists | `kubectl get hpa -n shoppy-cart` | `frontend-hpa` |
| HPA scaling | `kubectl get hpa -n shoppy-cart -w` | `cpu: >50%` → replicas increase |
| Pods created | `kubectl get pods -n shoppy-cart -l app=frontend` | 2–6 Running |
| Scale down | Reduce `USERS` to 10 → merge → wait | HPA scales back to 1 after stabilization |

### Result

```
$ kubectl get hpa -n shoppy-cart
NAME           REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
frontend-hpa   Deployment/frontend   cpu: 76%/50%    1         6         2          10m
$ kubectl get hpa -n shoppy-cart
NAME           REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
frontend-hpa   Deployment/frontend   cpu: 54%/50%    1         6         3          15m
```

### Gotchas

- HPA must be in the Helm chart. `kubectl edit` will be reverted by ArgoCD. See `07-gotchas.md` #14.
- `loadGenerator.users` must be parameterized in `values.yaml`, not hardcoded in the template. See `07-gotchas.md` #14.

---

# docs/06-credential-matrix.md

# Credential Matrix

## Why Scoped Credentials?

Every credential has one job. If it is compromised, the blast radius is limited to that job. The matrix below lists every credential in the platform, its scope, and who sets it.

**Important:** This project uses **EKS Pod Identity**, not IRSA or GitHub OIDC. The Builder ID account used to build this project could not create OIDC providers (see `09-whats-not-included.md`). Pod Identity is the AWS-recommended modern alternative and works without OIDC. The trust principal for Pod Identity roles is `pods.eks.amazonaws.com`.

## The Matrix

| Repo / Component | Credential | Type | Scope | Who sets it |
|---|---|---|---|---|
| shoppy-cart-infra (bootstrap) | Laptop admin IAM user | Long-lived (bootstrap only) | AdministratorAccess — used only to create the initial Terraform infra | You |
| shoppy-cart-infra (CI) | `ScRunnerRole` | IAM role, assumed by EC2 instance profile | Machine identity for the runner | Terraform |
| shoppy-cart-infra (CI) | `ScTerraformRole` | IAM role, assumed by runner role | Full Terraform permissions | Terraform |
| shoppy-cart-app (CI) | `ScAppCiRole` | IAM role, assumed by runner role | ECR push on `shoppy-cart/*` | Terraform |
| shoppy-cart-app (CI) | `GITOPS_PAT` | Fine-grained PAT (secret) | `Contents: RW`, `Pull requests: RW`, `Issues: Write` on `shoppy-cart-gitops` | You (GitHub secret in `shoppy-cart-app`) |
| EKS addons (EBS CSI, LB Controller, ExternalDNS) | Pod Identity roles | IAM roles, trusted by `pods.eks.amazonaws.com` | Scoped to each addon's need | Terraform |
| ArgoCD (repo clone) | Read-only PAT | Fine-grained PAT | `Contents: Read` on `shoppy-cart-gitops` | You (ArgoCD repo registration) |
| ArgoCD Image Updater (optional) | Write PAT | Fine-grained PAT | `Contents: RW` on `shoppy-cart-gitops` | You (k8s secret in argocd) |
| Runner → GitHub | Runner registration token | Short-lived token | One-time, to register the runner | You |
| Bastion → EKS | `ScBastionRole` | IAM role, EC2 instance profile | `eks:DescribeCluster` + EKS access entry | Terraform |

## Pod Identity (For Cluster Addons)

The EBS CSI driver, AWS Load Balancer Controller, and ExternalDNS each run as pods. They need AWS permissions. **Pod Identity** links a Kubernetes ServiceAccount to an IAM role without an OIDC provider:

```
Pod → ServiceAccount → EKS Pod Identity Association → IAM Role → AWS
```

**Key facts:**
- The IAM role trusts `pods.eks.amazonaws.com` (not an OIDC provider).
- The trust policy includes `sts:TagSession` (required for Pod Identity).
- The `eks-pod-identity-agent` addon must be installed on the cluster.
- A `aws_eks_pod_identity_association` resource in Terraform links the role to `(namespace, service_account)`.
- **The Kubernetes ServiceAccount must exist** — the Terraform association is AWS-side only. Create it manually (`kubectl create sa ...`) or via Helm `serviceAccount.create=true`.

## GitHub PATs (For CI and ArgoCD)

| PAT | Scope | Used by |
|---|---|---|
| `GITOPS_PAT` | `shoppy-cart-gitops`: `Contents: RW`, `Pull requests: RW`, `Issues: Write` | App CI's `update-gitops` job (opens PR) |
| Read-only PAT | `shoppy-cart-gitops`: `Contents: Read` | ArgoCD repo registration |
| Write PAT (optional) | `shoppy-cart-gitops`: `Contents: RW` | ArgoCD Image Updater, if using write-back |

**Store PATs as GitHub secrets** (`GITOPS_PAT` in `shoppy-cart-app`) or as Kubernetes secrets (ArgoCD repo registration). **Never commit a PAT.**

## The Principle

- **One credential per job.**
- **Scoped to the minimum** (specific repo, specific actions).
- **Short-lived where possible** (Pod Identity temporary credentials, not long-lived keys).
- **No static AWS keys** anywhere except the one-time bootstrap admin user.

---

# docs/07-gotchas.md

# Gotchas

## How to Use This Doc

Each gotcha: **symptom → cause → fix → phase**. If you hit an issue during the build, search here first.

---

## #1 — `sts:TagSession` Required for Role Chaining

**Symptom:** `aws sts assume-role` fails with `AccessDenied` even though the trust policy looks correct.

**Cause:** Role chaining (runner role → Terraform role) requires `sts:TagSession` in the **trust policy** of the target role, in addition to `sts:AssumeRole`.

**Fix:** Add `"sts:TagSession"` to the `Action` array in the target role's `assume_role_policy`. Same applies to Pod Identity roles (trust `pods.eks.amazonaws.com` with `sts:AssumeRole` + `sts:TagSession`).

**Phase:** 4, 10, 12.

---

## #2 — `set +e` Around `terraform plan` in CI

**Symptom:** CI fails with "Process completed with exit code 2" even when the plan succeeded.

**Cause:** `terraform plan` returns exit code `2` when changes are present (not an error). GitHub Actions treats non-zero exit as failure.

**Fix:** Wrap the plan step with `set +e` and check `$?` explicitly, or use `terraform plan -detailed-exitcode || [ $? -eq 2 ]`.

**Phase:** 8.

---

## #3 — `depends_on = [module.iam]` to Avoid Race

**Symptom:** First ECR apply fails with `AccessDenied` on `ecr:CreateRepository` even though the policy is correct.

**Cause:** The IAM policy update (adding `ecr:*`) and the ECR repo creation happen in parallel. The policy may not have propagated when ECR creation runs.

**Fix:** Add `depends_on = [module.iam]` to the ECR module call in `environments/dev/main.tf`.

**Phase:** 9.

---

## #4 — `ec2:ValidateSecurityGroupQuotasForInterface`

**Symptom:** AWS LB Controller fails to create ALB listeners with `AccessDenied` on `ec2:ValidateSecurityGroupQuotasForInterface`.

**Cause:** The official AWS LB Controller policy (from v2.13.3) predates this action (added by AWS in September 2026). The controller calls it when provisioning listeners.

**Fix:** Add `"ec2:ValidateSecurityGroupQuotasForInterface"` to the `ec2` statement in `modules/iam/policies/aws-lb-controller-policy.json.tftpl`.

**Phase:** 10.

---

## #5 — Gateway Listeners Materialize Only When HTTPRoute Attached

**Symptom:** Gateway shows `Programmed=True`, ALB is `Active`, but the ALB has **zero listeners**.

**Cause:** The AWS LB Controller creates the ALB first, then materializes listeners when it processes a valid HTTPRoute attached to the Gateway.

**Fix:** This is expected behavior. Apply an HTTPRoute (even a test one) to trigger listener creation. In production, the app's HTTPRoute triggers it.

**Phase:** 11.

---

## #6 — Manual ServiceAccount Creation for Pod Identity

**Symptom:** Pod Identity workloads (LB Controller, ExternalDNS) fail with `serviceaccount not found` or `no identity found`.

**Cause:** The Terraform `aws_eks_pod_identity_association` creates the AWS-side association. It does **not** create the Kubernetes ServiceAccount.

**Fix:** Create the SA manually (`kubectl create sa <name> -n <ns>`) **before** installing the Helm chart, and set `serviceAccount.create=false`. Or use `serviceAccount.create=true` in the Helm chart (but then verify the Pod Identity annotation is present).

**Phase:** 10, 12, 18.

---

## #7 — ArgoCD Repo Registration (Read-Only PAT or SSH Key)

**Symptom:** ArgoCD Application shows `Repository not found` for a private repo.

**Cause:** ArgoCD cannot clone a private repo without credentials.

**Fix:** Register the repo with a read-only PAT (`argocd repo add https://... --username <user> --password <PAT>`) or an SSH deploy key. If using SSH, update the `Application`'s `repoURL` to the SSH URL — ArgoCD treats HTTPS and SSH as different repositories.

**Phase:** 13.

---

## #8 — Multi-Source Application (Helm + Kustomize)

**Symptom:** ArgoCD Application fails to render the Helm chart and Kustomize overlays together.

**Cause:** A single-source Application can only point at one path. The chart lives in `apps/online-boutique/helm`, networking in `apps/online-boutique/networking`.

**Fix:** Use the multi-source `Application` syntax: `spec.sources[]` with two entries. The first is the Helm chart (`path: apps/online-boutique/helm`), the second is Kustomize (`path: apps/online-boutique/networking`).

**Phase:** 14.

---

## #9 — `create-pull-request` Expects Uncommitted Changes

**Symptom:** CI `update-gitops` job fails; PR not created; branch already pushed but no PR.

**Cause:** The `peter-evans/create-pull-request` action is designed to detect **uncommitted changes** in the working directory, create its own branch, commit, and push. If you manually `git checkout -b`, `git commit`, and `git push` before calling the action, the action sees no uncommitted changes and skips its own work.

**Fix:** Clone the repo, edit the file (uncommitted), and call the action. Do **not** manually create a branch, commit, or push.

**Phase:** 15.

---

## #10 — `GITOPS_PAT` Needs `Pull requests: Write` + `Issues: Write`

**Symptom:** `update-gitops` can clone and commit but cannot open a PR.

**Cause:** A fine-grained PAT with only `Contents: RW` can push a branch but cannot open a pull request or create labels.

**Fix:** Add `Pull requests: Read and write` and `Issues: Write` to the `GITOPS_PAT`.

**Phase:** 15.

---

## #11 — Rollback with Auto-Sync Requires Both ArgoCD Rollback and `git revert`

**Symptom:** After ArgoCD rollback, the bad pod comes back seconds later.

**Cause:** Auto-sync is enabled. ArgoCD sees Git still has the bad image and re-syncs to it.

**Fix:** Disable auto-sync → ArgoCD rollback → `git revert` the bad commit → push → re-enable auto-sync. Both steps are required.

**Phase:** 16.

---

## #12 — Kibana Service Is `eck-kibana-kb-http`

**Symptom:** Kibana HTTPRoute shows `BackendNotFound`.

**Cause:** The Service name follows the Helm release name. If you installed with `helm install eck-kibana`, the Service is `eck-kibana-kb-http` — not `shoppy-kibana-kb-http`.

**Fix:** Set `backendRefs[0].name: eck-kibana-kb-http` in the HTTPRoute.

**Phase:** 18.

---

## #13 — TargetGroupConfiguration Health Check Must Be Nested Under `healthCheckConfig`

**Symptom:** ALB health checks fail for Kibana; targets unhealthy; 502/504.

**Cause:** Kibana serves HTTPS on port 5601. The TGC must specify `protocol: HTTPS` and `healthCheckProtocol: HTTPS` with path `/api/status`. The `healthCheckConfig` fields must be **nested** under `healthCheckConfig`, not flat.

**Fix:** Use the correct nesting:

```yaml
defaultConfiguration:
  targetType: ip
  protocol: HTTPS
  healthCheckConfig:
    healthCheckProtocol: HTTPS
    healthCheckPath: /api/status
```

**Phase:** 18.

---

## #14 — HPA Must Be in Helm Chart; `USERS` Parameterized

**Symptom:** HPA created with `kubectl apply` disappears; `USERS` change doesn't trigger scaling.

**Cause:** ArgoCD reverts drift. Anything `kubectl apply`-ed outside Git is reverted on the next sync. Also, `USERS` in `templates/loadgenerator.yaml` is hardcoded.

**Fix:** Add `hpa-frontend.yaml` to the Helm chart and `hpa:` config to `values.yaml`. Parameterize `USERS` in `templates/loadgenerator.yaml` (`{{ .Values.loadGenerator.users | default 10 }}`) and set `users: 150` in `values.yaml`.

**Phase:** 19.

---

## #15 — DaemonSet Pod Pending When Node Is Full

**Symptom:** Filebeat DaemonSet shows `DESIRED 3, READY 2`; one pod Pending with "Too many pods" in events.

**Cause:** The node's maximum pod count (29/29 for m7i-flex.large with the default VPC CNI) is reached. The DaemonSet cannot schedule its pod on that node.

**Fix:** For learning, accept 2/3 coverage. For full coverage, scale the node group (add a 4th node) or enable **Prefix Delegation** on the VPC CNI to raise the pod limit from 29 to ~110 (requires node replacement). See `09-whats-not-included.md`.

**Phase:** 18.

---

# docs/08-runbooks.md

# Runbooks

## Rollback (Bad Deployment)

**When:** Pods in `CrashLoopBackOff`, `ImagePullBackOff`, app returns 5xx, users report errors.

**Steps:**

1. **Disable auto-sync** (ArgoCD UI: App Details → Sync Policy → toggle off; or CLI: `argocd app set online-boutique --sync-policy none`).
2. **Rollback** (ArgoCD UI: History and Rollback → previous revision → Rollback; or CLI: `argocd app history online-boutique` then `argocd app rollback online-boutique <revision>`).
3. **Fix Git** (`cd shoppy-cart-gitops && git pull && git log --oneline -5 && git revert <bad-commit> && git push origin main`).
4. **Re-enable auto-sync** (ArgoCD UI: toggle back on; or CLI: `argocd app set online-boutique --sync-policy automated --auto-prune --self-heal && argocd app sync online-boutique`).
5. **Verify** (`kubectl get pods -n shoppy-cart`, `kubectl describe pod ... | grep image`, `curl -I https://faekcorp.site`).

**Why both steps:** With auto-sync on, ArgoCD rollback alone is not enough — the next sync re-deploys the bad image. You must also revert Git.

## Troubleshoot: Pods Pending

| Cause | Fix |
|---|---|
| Node at pod capacity | `kubectl describe nodes` — check "Allocated resources" vs "Capacity". Scale node group. |
| Insufficient CPU/memory | Reduce pod resource requests or scale node group. |
| Node affinity mismatch | Check `kubectl describe pod <pod>` events. |
| PVC Pending (Elasticsearch) | Check StorageClass exists (`kubectl get sc`) and EBS CSI driver is running. |

## Troubleshoot: ImagePullBackOff

| Cause | Fix |
|---|---|
| Wrong image tag | Check the Deployment's image tag matches an ECR tag. |
| ECR auth failure | Check the node role has ECR pull permissions; check image pull secret. |
| Image not in ECR | Push it via CI. |

## Troubleshoot: Gateway Not Programmed

| Cause | Fix |
|---|---|
| SG rule missing | Check bastion / controller SG rules allow 443 to EKS API. |
| Wrong listener name | Check `sectionName` in HTTPRoute matches a Gateway listener (`https-apex`, `https-wildcard`). |
| HTTPRoute not attached | `kubectl get httproute -A` — check `PARENTS` column. |

## Troubleshoot: DNS Not Created

| Cause | Fix |
|---|---|
| ExternalDNS not running | `kubectl get pods -n external-dns` |
| Domain filter mismatch | Check ExternalDNS logs for "Applying provider record filter". |
| IAM missing Route53 | Check `ScExternalDnsRole` policy has `route53:ChangeResourceRecordSets` on the hosted zone. |
| Hosted zone ID mismatch | Confirm ExternalDNS is watching the correct zone. |

## Troubleshoot: ArgoCD OutOfSync

| Cause | Fix |
|---|---|
| Git changed | Check `git log` in the GitOps repo; `argocd app diff` |
| Repo registration failed | `argocd repo list` — check status. |
| `AppProject` blocks the resource | Check `kubectl get appproject -n argocd -o yaml` allowed kinds. |

## Cleanup (Destroy Everything)

**Order matters:**

1. **Delete the ALB** (AWS Console → EC2 → Load Balancers) — ArgoCD-created, Terraform doesn't know about it.
2. **Delete the ALB security groups** (AWS Console → EC2 → Security Groups).
3. **Delete orphaned EBS volumes** (if any — `aws ec2 describe-volumes --filters Name=status,Values=available`).
4. **`terraform destroy`** from `shoppy-cart-infra/environments/dev`.
5. **Verify** (`aws ec2 describe-vpcs --filters "Name=tag:Project,Values=shoppy-cart"` → empty).
6. **Delete the S3 state bucket** (`aws s3 rb s3://shoppy-cart-s3-terraform-state --force`).
7. **Delete Route53 hosted zone** (optional — keep if rebuilding).
8. **Remove runner** (GitHub org → Settings → Actions → Runners → Remove).

---

# docs/09-whats-not-included.md

# What's NOT Included

## Why This Doc Exists

This is a **learning project**, built on a **Builder ID AWS account**, designed to teach enterprise DevOps patterns. It is not a complete production system. This doc is honest about what is simplified, and what a real production system would add.

## What's Simplified

| What | Why | Production would add |
|---|---|---|
| **No OIDC / IRSA** | Builder ID account cannot create OIDC providers | GitHub OIDC for CI, IRSA for cluster addons (or Pod Identity, which is the modern alternative) |
| **No real databases** | Only Redis for cart; all other services are stateless or mock | RDS/Postgres for catalog, orders, users; managed Redis (ElastiCache) |
| **No real payment/email/shipping** | App services are mocks | Real payment gateway, email service (SES), shipping API |
| **No network policies** | Flat network inside the cluster | Calico/Cilium network policies, default-deny |
| **No Pod Security Standards** | Default PodSecurity admission | Restricted PSS, securityContext everywhere, read-only root filesystems |
| **No Kyverno / OPA** | No policy-as-code | Kyverno for image signing, resource limits, label requirements |
| **No multi-region / DR** | Single region, single cluster | Multi-region active-passive, Route53 health checks, cross-region backups |
| **Single NAT Gateway** | Cost trade-off (~$35/mo saved) | 3 NAT Gateways (one per AZ) for AZ failure resilience |
| **Trivy is advisory** | CVEs in upstream base images (Alpine, OpenSSL) do not fail the build | Block on CRITICAL, update base images, use `.trivyignore` for accepted risks |
| **No External Secrets** | Secrets in k8s secrets | External Secrets Operator + AWS Secrets Manager or Vault |
| **No cost optimization** | Fixed node group | Karpenter for right-sizing, Spot instances, cluster autoscaler |
| **No multi-tenancy** | One app, one team | Namespace-per-team, resource quotas, ArgoCD Projects per team |
| **Filebeat 2/3 coverage** | One node at pod capacity (29/29) | Enable Prefix Delegation on VPC CNI, or add nodes |
| **No real observability SLOs** | Default dashboards and alerts | SLOs, error budgets, burn-rate alerts, PagerDuty integration |

## What a Real Production System Would Add

1. **Secrets management:** External Secrets Operator + AWS Secrets Manager or HashiCorp Vault. No secrets in Git or k8s secrets.
2. **Real databases:** RDS (Postgres) for catalog, orders; ElastiCache for cart; backups, PITR.
3. **Policy as code:** Kyverno or OPA Gatekeeper. Image signing (Cosign), admission control.
4. **Network security:** Cilium network policies, default-deny, egress restrictions.
5. **Pod Security Standards:** Restricted profile, `runAsNonRoot`, read-only rootfs, dropped capabilities.
6. **Multi-region:** Active-passive, Route53 health checks, cross-region read replicas, DR runbook.
7. **Cost optimization:** Karpenter, Spot for non-critical workloads, savings plans, budget alerts.
8. **Observability SLOs:** SLOs, error budgets, burn-rate alerts, PagerDuty, runbook links from alerts.
9. **CI hardening:** Block on CRITICAL CVEs, signed images, SBOMs, provenance (SLSA).
10. **Compliance:** Audit logging, retention, access reviews, least-privilege IAM boundaries.

## What You Should Do Next

After completing this project, the highest-value next steps are:

1. **Add a real database** — RDS Postgres, connect the catalog service to it. Learn VPC peering or PrivateLink.
2. **Add External Secrets** — move the Slack webhook and any other secrets to AWS Secrets Manager; use External Secrets Operator.
3. **Add Kyverno** — require resource limits, labels, image provenance. Learn policy-as-code.
4. **Add Karpenter** — right-size nodes, use Spot for non-critical. Learn cost optimization.
5. **Add a second environment** — staging. Promote images from dev → staging → prod via Git. Learn environment promotion.
6. **Add network policies** — Calico/Cilium, default-deny, allow only what's needed. Learn zero-trust networking.
7. **Add SLOs** — define SLOs for the frontend, alert on burn rate. Learn reliability engineering.

## What This Project Proves

Despite the simplifications, this project demonstrates hands-on, production-shaped skills:

- **IaC:** Terraform modules, remote state, multi-module composition, CI/CD for infrastructure.
- **Kubernetes platform:** EKS, node groups, Pod Identity, access entries, addons.
- **Networking:** Gateway API, AWS LB Controller, ExternalDNS, Route53, ACM.
- **CI:** GitHub Actions, Docker, Trivy, ECR, self-hosted runner.
- **CD (GitOps):** ArgoCD, Image Updater, Helm, Kustomize, multi-source Application, repo registration.
- **Observability:** Prometheus, Grafana, Alertmanager, Slack, Elasticsearch, Filebeat, Kibana, ECK.
- **Reliability:** HPA, Metrics Server, rollback with auto-sync handling.
- **Troubleshooting:** 15 real gotchas, each with symptom, cause, and fix.
- **Honest assessment:** what's simplified and what to add next.

