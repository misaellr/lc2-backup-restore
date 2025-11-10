# Liferay Cockpit Requirements

## Table of Contents

- [Concepts — Liferay Cockpit](#concepts-liferay-cockpit)
- [The Product — Liferay Cockpit](#the-product-liferay-cockpit)
- [Entrenchment Clauses — Liferay Cockpit](#entrenchment-clauses-liferay-cockpit)
- [Persona — Liferay Cockpit personas](#persona-liferay-cockpit-personas)
- [Operations — Liferay Cockpit Operations](#operations-liferay-cockpit-operations)
- [Architecture — Liferay Cloud 2.0 (LC2)](#architecture-liferay-cloud-20-lc2)
- [Infrastructure — Liferay Cloud 2.0 (LC2)](#infrastructure-liferay-cloud-20-lc2)
- [Core Platform Scaffolding — LC2](#core-platform-scaffolding-lc2)
- [Kyverno Policy Model & Governance — LC2](#kyverno-policy-model--governance-lc2)
- [Technology Stack — Liferay Cockpit](#technology-stack-liferay-cockpit)
- [Screens — Liferay Cockpit UI](#screens-liferay-cockpit-ui)
- [API — Liferay Cockpit Control Plane API](#api-liferay-cockpit-control-plane-api)
- [Auth and Project Creation — Authentication & Project Creation](#auth-and-project-creation-authentication-project-creation)
- [CLI — LC2 Command-Line Interface](#cli-lc2-command-line-interface)
- [Local Dev Model — Liferay Cockpit Local Development Model](#local-dev-model-liferay-cockpit-local-development-model)
- [Portability — Infrastructure Dependencies & Options](#portability-infrastructure-dependencies-options)
- [LC2 Skills — Automation & Governance](#lc2-skills--automation--governance)


# Concepts — Liferay Cockpit

**Status:** Draft (MVP‑aligned)  
**Philosophy:** *GitOps‑first • Kubernetes‑native • Multi‑tenant isolation • KISS • Local↔Cloud parity • Kubernetes stateless by default*

> **Simplicity & KISS.** Simple things should be simple; complex things possible. We follow **KISS — keep it simple and straightforward (KISS)** and avoid unnecessary complexity.

---

## 1) Project Name
**Liferay Cloud 2.0 (LC2)** — a GitOps‑first, Liferay‑specialized PaaS that enforces hard tenant isolation and standardizes DevOps workflows, observability, logs, and backup/restore across **local** and **cloud** Kubernetes with the fewest necessary moving parts. **Cockpit** is the Java + Next.js management UI/API within LC2 used by Owner and Member personas to operate their projects.

---

## 2) Executive Summary

Liferay Cloud 2.0 (LC2) treats **Git as the source of truth** and uses **Argo CD** to reconcile desired state into Kubernetes. **Cockpit** (the Java + Next.js management interface) enables Owners and Members to operate their projects without kubectl access. All config/policy changes flow through **Pull Requests**; runtime operations (sync/refresh/rollback) are executed **server‑side** by the platform — end‑user devices never hold cluster‑admin credentials.

**Isolation is foundational.** Each project is a tenant; each environment (dev/uat/prd) maps to a dedicated namespace with strong guardrails (AppProject fences, RBAC, quotas/limits, default‑deny NetworkPolicy, PSA, admission control).

**Kubernetes is kept as stateless as possible.** Primary durable data (databases, file store, search) lives **outside** the cluster on managed services (RDS/Azure PG, S3/Blob, OpenSearch/Elastic). PVCs are **exceptions**, not the baseline, and must be explicitly justified and documented.

**Cloud + Local parity.** The same workflows operate **locally** and on **EKS/AKS**. When in the cloud, Cockpit **prefers managed services** and links to them instead of building custom UI.

**Core capabilities**  
- **Isolation (foundational)** — strict project/environment boundaries by namespace and policy.  
- **DevOps workflow** — PR‑driven deployments and **promotion** dev → uat → prd with verification gates.  
- **Observability (UI‑first)** — Cockpit **deep‑links** to managed Grafana for metrics/alerts; no custom charts in Cockpit.  
- **Logs (CLI‑first)** — LC2 CLI tails/queries logs against the provider logging plane (CloudWatch/Log Analytics/Loki).  
- **Backup & restore (externalized)** — DB PITR (RDS/Azure PG), versioned object storage (S3/Blob), search snapshots/reindex.  
- **Portability** — consistent local↔cloud experience; minimal dependencies.  
- **Compliance** — immutable audit of actions. **SIEM export** is a **future improvement** (mention sparingly).

---

## 3) Problem Statement

Operating multiple Liferay DXP environments across regions commonly suffers from:  
- **Environment drift** from manual edits; fragile rollbacks and weak auditability.  
- **Inconsistent tenant onboarding** and project scaffolding.  
- **Multi‑tenant risk** (noisy neighbors, cross‑namespace mistakes) without hard guardrails.  
- **Secret sprawl** and brittle hand‑offs.  
- **Opaque operations** when correlating deploys, health, logs, and alerts.  
- **Dev/prd mismatch** — local setups behaving unlike cloud.  
- **Over‑engineering** — unnecessary services/features increasing cost and blast radius.

**Cockpit** fixes this with GitOps, enforceable isolation, ESO‑managed secrets, **managed** data services, lean integrations, and a small set of predictable golden‑path workflows that work the same locally and in the cloud.

---

## 4) Business Impact

### A) For PaaS Admin Businesses (Liferay & Partners operating the platform)
- **Faster tenant onboarding** via templates (ApplicationSet) and AppProject fencing.  
- **Lower risk & TCO** with isolation, quotas, admission controls, and cloud‑managed data services.  
- **Compliance‑ready** — PR history + immutable audit stream (optional SIEM export later).  
- **Minimal platform surface** — Cockpit links to managed consoles instead of rebuilding them.  
- **No cross‑project observability needed** for admins — they only stand up plumbing.

**Admin KPIs (examples)**: onboarding lead time; MTTR; change failure rate; % changes via PR (~100% target); DR drill pass rate; tenant density per cluster.

### B) For Customers (Project Owners & Members)
- **Self‑service without tickets** — change via PR, operate via UI/CLI, promote safely across envs.  
- **Predictable, auditable deployments** — no snowflake edits; every action traceable.  
- **Operational clarity** — resource tree, diffs, health/sync, **deep links** to dashboards; CLI for logs.  
- **Safety net** — externalized backups with clear RPO/RTO.  
- **Same workflow locally and in cloud** — less friction, faster onboarding.

**Customer KPIs (examples)**: lead time for change; deployment frequency; MTTR; automated rollbacks; time‑to‑first‑deploy; RPO/RTO adherence.

---

## 5) General Concepts & Terminologies

- **Project (Tenant)** — Logical customer space; users (Owners/Members) are scoped to their projects.  
- **Environment** — `dev`, `uat`, `prd`; **each maps to a dedicated namespace** (e.g., `proj-<id>-dev`).  
- **Isolation (Entrenchment)** — Users may not mutate outside their project namespaces. Enforced by **AppProject**, **RBAC**, **ResourceQuota/LimitRange**, **NetworkPolicy (default‑deny)**, **PSA**, and **admission control** (only Argo controller SA and platform operator SA may write).  
- **Kubernetes stateless by default (Entrenchment)** — Durable data lives out of cluster (DB, files, search). PVCs are exceptional and documented.  
- **kubectl policy (Entrenchment)** — **Customers (Owners & Members) cannot use kubectl.** Admins retain **audited break‑glass** kubectl; normal ops flow through GitOps/Cockpit.  
- **GitOps** — Pull Requests define desired state; Argo CD reconciles to actual; ops (sync/rollback) are executed server‑side.  
- **ApplicationSet / AppProject (Argo CD)** — generators for env/region apps; fences for repos/destinations/kinds.  
- **ESO (External Secrets Operator)** — syncs secrets from providers (AWS SM / Azure Key Vault / Vault) into K8s; **values never shown**.  
- **Observability (UI‑first)** —
  - **EKS:** Amazon Managed Service for Prometheus + Amazon Managed Grafana.  
  - **AKS:** Azure Managed Prometheus + Azure Managed Grafana.  
  - **Local:** Prometheus OSS + Grafana OSS. Cockpit provides **deep links** only.  
- **Logs (CLI‑first)** —
  - **EKS:** CloudWatch Logs (log group per project/env).  
  - **AKS:** Azure Log Analytics (workspace queries).  
  - **Local:** Loki (multi‑tenant via org/project labels). LC2 CLI brokers scope/access.  
- **Backup units (externalized)** — **DB (PostgreSQL)** via provider PITR; **Document Library** via S3/Blob versioning; **Search** via snapshots (or reindex).  
- **URL Pattern** — `<cloud>-<region>.<cockpit-domain>/projects/[projectId]`  
  - Examples: `aws-us-east-1.myliferaypaas.com/projects/retail` · `azure-westus2.partnerpaas.com/projects/intranet` · `local.localdev/projects/checkout`.

---

## 6) Operating Environments

- **Local**: k3d/kind (1‑node). Dev‑grade ingress/TLS; optional local Postgres/MinIO/OpenSearch for development only. Workflows mirror cloud.  
- **AWS (EKS)**: prefers **RDS PostgreSQL**, **OpenSearch (or Elastic Cloud)**, **S3**; ALB/Route53; ACM or cert‑manager. Observability = **Managed Prometheus + Managed Grafana**. Logs via **CloudWatch Logs**.  
- **Azure (AKS)**: prefers **Azure Database for PostgreSQL**, **OpenSearch/Elastic**, **Blob Storage**; AGIC/Azure DNS; Key Vault/cert‑manager. Observability = **Azure Managed Prometheus + Azure Managed Grafana**. Logs via **Azure Log Analytics**.

**Parity pledge:** keep GitOps, isolation, and user flows identical across local and cloud; only the managed backends differ.

---

## 7) Non‑Goals (MVP)

- Running primary databases/search/file stores **inside Kubernetes** for production.  
- Cross‑project admin dashboards.  
- Custom metric/log charts inside Cockpit (we link to managed tools).  
- Displaying plaintext secrets.  
- Adding message buses or niche dependencies by default.

---

**One‑liner**: *Liferay Cloud 2.0 (LC2) = Isolation first, GitOps always — with Kubernetes kept as stateless as possible, managed observability via deep links, CLI‑first logs, and externalized backups, all with local↔cloud parity and minimal dependencies. Cockpit is the Java + Next.js management UI/API for Owners and Members.*

# The Product — Liferay Cockpit

**Status:** Draft (MVP‑aligned)  
**Philosophy:** *GitOps‑first • Kubernetes‑native • Isolation by default • **KISS** • Local↔Cloud parity • Kubernetes stateless by default*  
**Core capabilities:** **Isolation**, **DevOps workflow (dev → uat → prd)**, **Observability (UI‑first)**, **Logs (CLI‑first)**, **Backup & Restore (externalized)**

> **Simplicity & KISS.** Simple things should be simple; complex things possible. We follow **KISS — keep it simple and straightforward (KISS)** and avoid unnecessary complexity.
> **Statelessness clause.** Kubernetes should be **as stateless as possible** (local and cloud). All primary durable data (databases, file store, search) lives **outside** the cluster. PVCs are **exceptions**, not the baseline, and must be explicitly justified and documented.

---

## 1) What is the Product? (Overview)

Liferay Cloud 2.0 (LC2) is a **GitOps‑first PaaS** purpose‑built for **Liferay DXP** operations at scale. It standardizes isolation, promotion (dev→uat→prd), observability access, logs, and backup/restore while minimizing the surface area we build. LC2 includes:
- **Infrastructure provisioning** (Terraform + Helm for EKS/AKS/GKE clusters and dependencies)
- **Cockpit** (Java + Next.js management UI/API for Owner/Member personas)
- **LC2 CLI** (command-line interface for operations and scripting)
- **Kubernetes resources** (Argo CD, Kyverno policies, ESO, observability agents)

The platform uses **Kubernetes** as the substrate and **Git** as the source of truth.

- All desired‑state changes are done via **Pull Requests** and reconciled by **Argo CD**.  
- Runtime operations (sync, refresh, rollback) are executed **server‑side** by the platform control plane—end‑user devices never hold cluster‑admin creds.  
- The most important guarantee is **isolation**: every project and environment is fenced to its **own namespace** with strong guardrails.  
- **Observability is UI‑first:** we **link** to managed Grafana per runtime; we do **not** build charts in Cockpit.  
- **Logs are CLI‑first:** users tail/search via LC2 CLI against the provider logging plane (CloudWatch, Azure Log Analytics, Loki).  
- **Backups are externalized by default:** DB **PITR** (RDS/Azure PG), versioned object storage (S3/Blob), and search snapshots/reindex. No PVCs by default.

---

## 2) Where it Runs

- **Local**: kind / k3d / minikube — developer parity for workflows.  
- **AWS**: Amazon **EKS** — prefers **Amazon RDS for PostgreSQL**, **Amazon OpenSearch (or Elastic Cloud)**, **S3** for DL and backups. Observability = **Amazon Managed Service for Prometheus + Amazon Managed Grafana**; logs = **CloudWatch Logs**.  
- **Azure**: **AKS** — prefers **Azure Database for PostgreSQL**, **OpenSearch/Elastic Cloud**, **Blob Storage**. Observability = **Azure Managed Prometheus + Azure Managed Grafana**; logs = **Azure Log Analytics**.

**Cockpit URL Structure**  
```
<cloud>-<region>.<cockpit-domain>/projects/[projectId]
```

**Examples**  
```
aws-us-east-1.myliferaypaas.com/projects/retail
azure-westus2.partnerpaas.com/projects/intranet
local.localdevelopment/projects/developer
```

> **Parity pledge:** local deployments should have as much feature parity as possible with cloud deployments (workflow semantics, isolation, observability access, backup/restore behavior).

---

## 3) Personas & Access Model

### Admin (Cluster Operator — local or cloud)
- Provisions clusters/regions, installs platform components (Argo CD, ESO, ingress), configures managed services and backup locations.  
- Creates **projects** (namespaces, quotas, AppProject), invites owners.  
- **kubectl policy:** may use `kubectl` for **break‑glass** operations only (audited). Day‑to‑day via GitOps/Cockpit.  
- Does **not** need cross‑project observability; only stands up plumbing and guardrails.

### Project Owner
- Manages their project’s environments via Cockpit UI/API/CLI and Git PRs.  
- Full operational surface for their project: scale, HPA bounds, rollout/rollback, promotion dev→uat→prd.  
- **kubectl policy:** **no direct kubectl** (including `kubectl logs`).

### Project Member
- Executes day‑to‑day tasks for their project (scale within bounds, check health, view dashboards, tail/search logs).  
- Cannot change Owner/Member permissions.  
- **kubectl policy:** **no direct kubectl**.

> Owners/Members may belong to multiple projects. **All actions are scoped** to the environments of the project they are operating.

---

## 4) Entrenchment (Isolation & Simplicity Are Non‑Negotiable)

**Isolation clause**  
- Projects and environments **must be isolated**. Under no circumstances can a tenant mutate **outside** the namespaces of a project to which they have permissions.

**Simplicity & Statelessness**  
- **KISS + “simple things simple; complex things possible.”** Choose managed services and links over custom builds.  
- **Kubernetes stateless by default.** Durable data (DB, file store, search) is **external**. PVCs are documented **exceptions** with explicit guardrails.

**Enforcement mechanisms**  
- **Argo CD AppProject** per project (repo allowlists; destination namespaces; optional deny of cluster‑scoped kinds).  
- **Admission control** (Kyverno) to **deny all mutations** unless they originate from:  
  - Argo CD application controller service account, or  
  - Liferay Cockpit operator service account.  
- **RBAC & Namespace boundaries** per environment (`proj-<id>-dev|uat|prd`), **ResourceQuota/LimitRange**, **default‑deny NetworkPolicy**, **PSA=restricted**.  
- **Secrets**: managed by **External Secrets Operator**; UI/CLI show metadata/status only.

---

## 5) Kubernetes‑ & GitOps‑Based Operating Model

- **Packaging**: Helm (primary) with optional Kustomize overlays.  
- **GitOps flow**: PR → review/merge → Argo CD reconciles → audit trail maintained.  
- **Runtime ops**: Sync / Refresh / Rollback invoked via UI/CLI → executed server‑side via Argo.  
- **Secrets**: ESO/Vault references committed; rotations triggered from Cockpit; **values never exposed**.  
- **Backups (externalized)**: databases use **provider PITR** (RDS/Azure PG); file store uses **S3/Blob versioning**; search uses **snapshot** (or **reindex**). **No PVCs by default**. If a documented exception needs a PVC, snapshot/restore policy is defined for it.

---

## 6) Key Capabilities

1) **Isolation (foundational)**  
   Hard multi‑tenant isolation by project/environment through namespaces and policy. Cockpit constantly validates fences.

2) **DevOps Workflow (dev → uat → prd)**  
   Promotion via PR with verification gates (Healthy, Synced, no firing alerts). Rollbacks are PR/CLI‑driven and audited.

3) **Observability (UI‑first, managed)**  
   Cockpit provides **deep links** to the right dashboards per runtime:  
   - **EKS:** Managed Prometheus + Managed Grafana.  
   - **AKS:** Managed Prometheus + Managed Grafana.  
   - **Local:** Prometheus OSS + Grafana OSS.  
   *We do not build charts in Cockpit.*

4) **Logs (CLI‑first)**
   LC2 CLI tails/queries logs against the provider logging plane with strict scoping:
   - **EKS:** CloudWatch Logs (log group per project/env).
   - **AKS:** Azure Log Analytics (workspace queries/KQL).
   - **Local:** Loki (multi‑tenant).

5) **Backup & Restore (externalized)**  
   DB PITR (RDS/Azure PG), DL object versioning (S3/Blob), search snapshots/reindex. PVCs only by exception.

6) **Compliance & Audit**  
   Immutable audit of all actions (who/what/when/scope/result). **SIEM export** is a **future improvement** and referenced sparingly.

---

## 7) Day‑to‑Day Workflows

**For Owners/Members**  
1. **Connect repo** and map branches/paths to environments (namespaces).  
2. **Change via PR** (values, overlays, manifests, policies).  
3. **Merge & reconcile** — Argo CD applies; health/sync shown in Cockpit.  
4. **Promote via PR** (dev→uat→prd) with verification gates.  
5. **Observe** via **Open in Grafana** deep links; triage with LC2 CLI **logs**.  
6. **Backup/Restore** — select unit (DB/DL/Search) and restore target; operations are audited. PVC exceptions, if any, follow documented snapshot/restore policy.

**For Admins**  
- **Provision** regions/clusters; install platform components; create projects (namespaces, AppProject, quotas), set registry allowlist.  
- **Configure** managed services (RDS/OpenSearch or Azure equivalents) and backup locations (S3/Blob).  
- **Define** policy baselines; approve policy changes via PR; run DR drills. No cross‑project observability required.

---

## 8) Integration & Dependencies (Keep It Simple)

- **Required (baseline):** Argo CD, External Secrets Operator, Prometheus (per runtime: managed on cloud, OSS locally), ingress controller.  
- **Optional (by runtime or customer choice):** Managed Grafana (cloud) / Grafana OSS (local), OpenSearch/Elastic + Kibana (BYO), ExternalDNS, cert‑manager.  
- **Provider preferences:** use **managed services** on EKS/AKS for durability and observability; ensure **local↔cloud parity** of workflows.  
- Avoid low‑adoption/experimental dependencies; complex tasks can be performed via Git/CLI without new UI.

---

## 9) Compliance, Security & Audit (At a Glance)

- **Least privilege & persona guardrails:** **Customers (Owners & Members) cannot use kubectl.** Admins keep **audited break‑glass** kubectl.  
- **Immutable audit** of all Cockpit actions (scope included).  
- **Secrets safety:** values never rendered; ESO sources (AWS SM/Azure KV) retain their own audit.  
- **Network & pod security:** default‑deny NetworkPolicy; PSA=restricted; image allowlists; resource quotas/limits.  
- **TLS everywhere** (ingress + managed endpoints).  
- **SIEM export**: **future improvement** (note once; avoid repeating across docs).

---

## 10) Non‑Goals (MVP)

- Running primary databases/search/file stores inside Kubernetes for production.  
- Cross‑project admin dashboards.  
- Custom metric/log charts inside Cockpit (we link to managed tools).  
- Displaying plaintext secrets.  
- Introducing Kafka/message buses by default.

---

**One‑liner**: *A simple‑first, GitOps‑native PaaS that keeps Kubernetes **stateless by default**, enforces hard isolation, links to managed observability, provides CLI‑first logs and externalized backups—working the same locally and on EKS/AKS with minimal dependencies.*

# Entrenchment Clauses — Liferay Cockpit

**Status:** Binding (Non‑negotiable)  
**Applies to:** Product design, implementation, operations, and governance

---

## 0) Purpose
Protect the core values of Liferay Cockpit: **Isolation first**, **GitOps always**, **KISS**, **Kubernetes stateless by default**, and **local↔cloud parity**. These clauses constrain decisions across the product.

> **simplicity & KISS.** simple things should be simple; complex things possible. we follow **KISS — keep it simple and straightforward (KISS)** and avoid unnecessary complexity.

---

## 1) GitOps‑Only Change Paths
**What**
- Declarative changes (manifests, Helm values, policies) **must** be performed via **Git Pull Requests**.
- Runtime operations (sync/refresh/rollback) are executed **server‑side** by Cockpit → **Argo CD**.

**Prohibitions**
- **Owners & Members** (project personas) **cannot use kubectl** (including `kubectl logs`).  
- End‑user CLIs/browsers must not hold cluster‑admin credentials.

**Enforcement**
- Admission policies **deny all mutations** not originating from the **Argo CD application controller** or the **Cockpit operator** service accounts.
- Argo **AppProject** fences restrict repos/destinations and may deny cluster‑scoped kinds.

---

## 2) Isolation Above All (Projects & Environments)
**What**
- Each **project** is a tenant; each **environment** (dev/uat/prd) maps to a **dedicated namespace**: `proj-<id>-<env>`.
- Under no circumstances may an **Owner/Member** mutate resources **outside** their project’s namespaces.

**Enforcement**
- AppProject per project; namespace‑scoped RBAC; **ResourceQuota/LimitRange**; **default‑deny NetworkPolicy**; **PSA=restricted**; image/registry allowlists.
- Admission policies block cross‑namespace/project mutations from end users.

---

## 3) Kubernetes Should Be as Stateless as Possible (Local & Cloud)
**What**
- All **primary durable data** (databases, file store, search) lives **outside** Kubernetes on managed services (e.g., RDS/Azure PG, S3/Blob, OpenSearch/Elastic). Pods are disposable.
- **PVCs are exceptions**, not baseline. Any PVC usage must be explicitly justified, documented, and protected (snapshot/restore policy).

**Why**
- Simpler scaling, faster recovery, fewer moving parts, and consistent parity between local and cloud.

---

## 4) Local ↔ Cloud Parity
**What**
- Maintain as much feature parity as practical between **local** (k3d/kind) and **cloud** (EKS/AKS).
- Core experiences (GitOps flows, isolation semantics, observability access) must behave consistently.

**How**
- Keep provider choices **pluggable**, workflows/guardrails **identical**.

---

## 5) Minimal Cockpit Surface Area
**What**
- Prefer **links** to managed consoles over building custom UI.

**Rules**
- **Observability is UI‑first:** open managed Grafana (cloud) / Grafana OSS (local). **No custom charts** in Cockpit.
- **Logs are CLI‑first:** LC2 CLI tails/queries provider logging planes (CloudWatch / Log Analytics / Loki) with strict scoping.
- **Backups are externalized:** DB PITR (RDS/Azure PG); file store via S3/Blob versioning; search snapshots/reindex. **No PVCs by default.**

---

## 6) Prefer Cloud‑Managed Services (When on EKS/AKS)
**What**
- On **AWS (EKS)**, prefer **Amazon RDS for PostgreSQL**, **Amazon OpenSearch (or Elastic Cloud)**, and **S3** for object storage.
- On **Azure (AKS)**, prefer **Azure Database for PostgreSQL**, **OpenSearch/Elastic Cloud**, and **Blob Storage**.

**How**
- Bind applications via **External Secrets Operator (ESO)**; secret values never transit Cockpit.

---

## 7) Secrets Handling (No Plaintext Exposure)
**What**
- Secrets are managed by **ESO** with cloud/vault providers; Cockpit shows **metadata/status only**.
- PRs/diffs/logs must redact sensitive values.

**Rotation**
- Initiated from Cockpit and fulfilled by the provider/ESO.

---

## 8) Auditability
**What**
- Every change/operation generates an immutable audit: **who/what/when/where/scope/result**, with correlationId and links to PR/Argo.
- Secrets are redacted; allow/deny decisions are recorded.

---

## 9) Access & Operational Boundaries
**What**
- **No Owner/Member kubeconfigs.** All write/read operations are mediated by Cockpit (UI/API/CLI) and executed by Argo/Cockpit operator SAs.
- Owners/Members may belong to multiple projects; permissions are **project‑scoped**. **Members cannot remove Owners.**

---

## 10) No Cross‑Project Observability for Admins
**What**
- **Admins (cluster)** stand up plumbing and guardrails; they **do not** need cross‑project dashboards.

**Why**
- Reduces scope and risk; keeps responsibility with project teams.

---

## 11) Security Baseline
**What**
- TLS everywhere; **default‑deny NetworkPolicy** with curated egress; **PSA=restricted**; image policy enforcement; resource quotas/limits; signed images where available.
- Drift detection and optional self‑heal are Argo‑managed and auditable.
- No cross‑tenant data exposure; logs/dashboards scoped per project/env.

---

## 12) Dependency Policy
**What**
- Choose **stable, widely adopted** OSS or cloud‑managed equivalents.
- New dependencies require: security review, maintenance plan, parity assessment (local vs cloud), and rollback plan.

**Remove/replace** dependencies that increase blast radius or violate parity/simplicity.

---

## 13) Amending These Clauses
**What**
- Changes require **platform governance approval** and a recorded design review.
- Clauses **1 (GitOps‑Only)** and **2 (Isolation)** are **foundational** and cannot be relaxed without a major‑version decision and **explicit stakeholder communication**.

---

## 14) Acceptance Checks (Must Always Hold)
- 100% of config changes originate from **PRs**; 100% of runtime ops originate from **Cockpit‑mediated Argo ops**.
- **No Owner/Member user/SA** (except Argo/Cockpit operator) can **mutate** resources in the API server.
- **Statelessness default holds:** no PVCs in projects unless a documented exception exists.
- Local builds demonstrate the **same workflows** as cloud (within practical limits).
- Audit sampling confirms end‑to‑end traceability and redaction.

# Persona — Liferay Cockpit personas

**Status:** Draft (MVP‑aligned)  
**Principles:** *GitOps‑first • isolation by default • simple > complex (KISS) • local↔cloud parity • kubernetes stateless by default*  
**Core capabilities in scope:** isolation, devops workflow (dev→uat→prd), observability (UI‑first), logs (CLI‑first), backup & restore (externalized)

---

## 1) overview
This document defines the actors of Liferay Cockpit, their goals and responsibilities, and the precise **boundaries** under which they operate. All personas follow the **GitOps‑only** model (PRs for desired state; Argo‑mediated runtime ops) and the **hard isolation** rule (actions confined to the namespaces of the projects they belong to). Kubernetes is kept **as stateless as possible** across local and cloud.

**personas**
- **Admin (cluster operator)** — runs the platform locally or on cloud (EKS/AKS), provisions projects, sets guardrails, integrates managed services, and governs policy. No cross‑project observability is required; admins stand up the plumbing.  
- **Project owner** — leads a tenant/project. Full operational control *within the project’s namespaces* and manages members & approvals.  
- **Project member** — operates workloads *within the project’s namespaces*; similar to Owner, except cannot remove Owners or transfer ownership.

> Owners and Members may belong to multiple projects and multiple environments per project. All changes are executed via **Git PRs** or **Cockpit‑mediated Argo operations**. **Owners/Members cannot use kubectl** (including `kubectl logs`). Admins retain **audited break‑glass** kubectl only.

---

## 2) admin (cluster operator — local or cloud)

**goals**
- Provide a safe, low‑friction multi‑tenant platform with **local↔cloud feature parity**.  
- Enforce guardrails (AppProject, quotas/limits, PSA, NetworkPolicy, registry allowlist) and **GitOps‑only** flows.  
- Prefer **managed services** for durable data (RDS/OpenSearch/S3 on AWS; Azure DB/OpenSearch/Blob on Azure).  
- Keep dependencies lean and well‑adopted (KISS).

**responsibilities**
- Provision regions/clusters; install Argo CD, External Secrets Operator (ESO), admission policies, ingress, and the observability/logging plumbing.  
- Create **projects**: namespaces per env (`proj-<id>-dev|uat|prd`), AppProject fencing, ResourceQuota/LimitRange, default‑deny NetworkPolicy; invite Owners.  
- Configure registry allowlists and managed service bindings; wire **observability** (managed Prometheus + managed Grafana) per runtime.  
- Define database/file/search backup locations and retention (externalized); run DR drills.  
- Approve policy changes via PR to the platform policy repo; perform upgrades and security patches.

**boundaries**
- Does **not** change tenant workloads directly; changes go through PRs or Argo ops with audit.  
- Does **not** expose plaintext secrets; ESO/provider manages values.  
- No cross‑project observability is required for the role.

**kpis**
- time to onboard a project; MTTR; change failure rate; % changes via PR; DR drill pass rate; upgrade lead time; tenant density per cluster.

---

## 3) project owner

**goals**
- Ship Liferay DXP and companion workloads safely and quickly.  
- Maintain consistent, auditable environments (dev/uat/prd).  
- Empower team members; approve promotions; keep kubernetes **stateless by default** (use managed DB/file/search).

**responsibilities**
- Connect repos; map branches/paths to envs; set sync policies.  
- Approve promotions (dev→uat→prd) per policy.  
- Manage project **members** and access tokens.  
- Operate via Cockpit: scale/HPA bounds, rollout/rollback, health/sync/diffs.  
- Use **observability (UI‑first)** via deep links to managed Grafana; triage **logs (CLI‑first)** through LC2 CLI (provider logging plane).  
- Initiate backups; request or perform **safe restores to a new namespace/env** inside the project (for verification/drills). In‑place or cross‑project/cluster restores require Admin.

**boundaries**
- Cannot operate outside the project namespaces.  
- Cannot view plaintext secrets; rotations go through ESO.  
- Must use **PRs** for config/policy changes; runtime ops via **Argo** only.  
- **No kubectl** (including `kubectl logs`).

**kpis**
- lead time for change; deployment frequency; MTTR; failed change rate; % automated rollbacks; RPO/RTO adherence.

---

## 4) project member

**goals**
- Operate and troubleshoot workloads within project namespaces.  
- Deliver features quickly with auditable changes; leverage managed services over in‑cluster state.

**responsibilities**
- Create PRs to change values/manifests/policies; operate via UI/CLI (sync/refresh/rollback).
- Triage using **logs (CLI‑first)** and **observability deep links** (managed Grafana).
- Initiate backup runs if policy allows; perform **safe restores to a new namespace/env** inside the project (for verification/drills). In‑place or cross‑project/cluster restores require Admin.

**boundaries**
- Same as Owner **within the project**, **except** cannot remove an Owner or transfer ownership.  
- No cross‑project operations; no plaintext secrets; **no kubectl**.

**kpis**
- time to first deploy; MTTR for incidents; PR review throughput.

---

## 5) permissions matrix (mvp)

| capability | admin | owner | member |
|---|:---:|:---:|:---:|
| view project & envs | ✅ | ✅ | ✅ |
| manage members/roles | ✅ | ✅ | ❌ |
| remove owner | ✅ | ❌ | ❌ |
| connect repo / set sync policy | ✅ | ✅ | ✅ |
| create PRs (values/overlays/policies) | ✅ | ✅ | ✅ |
| trigger argo ops (sync/refresh/rollback) | ✅ | ✅ | ✅ |
| approve promotions | ✅ | ✅ | ✅ *(policy‑gated)* |
| edit quotas/limits/network policy | ✅ *(via policy repo PR)* | ❌ | ❌ |
| rotate secrets (trigger ESO refresh) | ✅ | ✅ | ✅ |
| view plaintext secrets | ❌ | ❌ | ❌ |
| view dashboards (managed grafana) | ✅ | ✅ | ✅ |
| logs tail/query (lc2 CLI) | ✅ | ✅ | ✅ |
| create backups (db/dl/search) | ✅ | ✅ | ✅ *(policy‑gated)* |
| restore from backup | ✅ | ✅ *(safe clone/new env)* | ✅ *(safe clone/new env)* |
| use kubectl | ✅ *(break‑glass, audited)* | ❌ | ❌ |
| cross‑project or cross‑namespace writes | ❌ | ❌ | ❌ |

> All writes originate from **PR merges** or **Cockpit‑mediated Argo ops**. Admission control prevents any other mutation. Owners/Members have **no kubectl** access (including `kubectl logs`).

---

## 6) access patterns & scoping

- **touchpoints**: UI/CLI/Git only.  
- **logs**: CLI‑first; LC2 CLI brokers provider log access (CloudWatch Logs, Azure Log Analytics, Loki), always scoped to `{project, env}`.  
- **observability**: UI‑first; “open in grafana” deep links per project/env; no custom charts in Cockpit.  
- **url pattern**: `<cloud>-<region>.<cockpit-domain>/projects/[projectId]`  
  - examples: `aws-us-east-1.myliferaypaas.com/projects/retailproject`, `azure-westus2.partnerpaas.com/projects/intranetproject`, `local.localdevelopment/projects/developerproject`.  
- **scopes**: actions are always evaluated against `{project, env/namespace, region}` and persona role.

---

## 7) onboarding & offboarding

**onboarding**
1. Admin creates project (namespaces, AppProject, quotas, policies) and invites Owner.  
2. Owner adds Members, connects repos, sets sync policies.  
3. Initial deploy via PR; validate health, alerts, and externalized backups.

**offboarding**
1. Freeze project (disable promotions/syncs).  
2. Export audit logs; revoke tokens.  
3. Archive repos and tear down namespaces safely.

---

## 8) slos & expectations

- **UI/API availability**: 99.5% (MVP target).  
- **audit visibility**: near‑real‑time; immutable export within 1 hour.  
- **backup cadence**: nightly default for prd; restores rehearsed quarterly (policy‑driven).  
- **promotion gates**: Healthy, Synced, No firing alerts by default.

---

## 9) acceptance criteria (persona fit)

- Owners/Members can perform all project‑scoped operations via **PRs/Argo ops** without cluster‑admin.  
- Admins can onboard a new project quickly using templates and the policy repo.  
- Admission rules prevent any cross‑project mutation; audit entries capture who/what/when/where/why/result for every action.  
- Local environment demonstrates the same workflows as cloud (within practical limits).

---

**one‑liner**: *Admins run the platform and set guardrails. Owners and Members operate entirely via GitOps inside their project namespaces—with isolation, UI‑first observability, CLI‑first logs, and externalized backups across local and cloud.*

# Operations — Liferay Cockpit Operations

**Status:** Draft (MVP‑aligned)  
**Principles:** *GitOps‑first • Isolation by default • KISS • Local↔Cloud parity • Kubernetes stateless by default*  
**Core capabilities:** Isolation · DevOps workflow (dev→uat→prd) · Observability (UI‑first) · Logs (CLI‑first) · Backup & Restore (externalized)

---

## 1) Scope
This document defines **how Owners and Members operate their projects** in Liferay Cockpit. All mutations are either:
- **Declarative via Git PRs** (Helm values, overlays, manifests, policies), or
- **Runtime ops via Cockpit** (UI/CLI → server) which calls **Argo CD** (sync/refresh/rollback).

> **Isolation:** All actions are confined to **your project’s namespaces** (`proj-<id>-dev|uat|prd`). Cross‑project or cross‑namespace mutations are prohibited and technically blocked.  
> **Statelessness:** Kubernetes should be **as stateless as possible**. Durable data (DB, file store, search) lives **outside** the cluster on managed services.  
> **Infra abstraction:** **Admins** create projects and attach dependent services. **Owners/Members** never configure infrastructure and never need cloud credentials.

---

## 2) Access Pattern

**URL format**: `<cloud>-<region>.<cockpit-domain>/projects/[projectId]`  
Examples:  
- `aws-us-east-1.myliferaypaas.com/projects/retailproject`  
- `azure-westus2.partnerpaas.com/projects/intranetproject`  
- `local.localdevelopment/projects/developerproject`

Use the **Cockpit UI** for visibility/approvals and the **CLI** for scripted operations. All spec changes are **PRs**.

**CLI**: `lc2` (username/password login for MVP; OIDC post-MVP, project‑scoped)
```
lc2 login --region aws-us-east-1
lc2 use project retailproject
```

### 2.1 Environment readiness gates (feature gating)
Cockpit enables deploy/promotion features **only when dependencies are healthy** for that environment:

- **BindingsReady** (deploy gate): IaaS binding usable; **Database** reachable; **File store** write test passes; **Search** index/template checks pass; **ESO** shows Bound & fresh.  
- **PromotionReady** (promote gate): **Grafana** data source healthy; **Logs** provider plane returns data.

Until gates are green, dependent buttons are **disabled** with a clear hint. Owners/Members do not perform infra steps; Admins do.

---

## 3) Rule of Two Paths (and Boundaries)

1) **PR → merge → Argo reconcile** (spec/config/policy changes)  
2) **UI/CLI → Cockpit API → Argo** (sync/refresh/rollback runtime ops)

- **Owners & Members:** **no direct kubectl** (including `kubectl logs`).  
- **Admins:** **break‑glass kubectl only**, fully **audited**; day‑to‑day via GitOps/Cockpit.  
- No Owner/Member kubeconfigs are issued. Any optional debug exec is mediated by Cockpit and audited.  
- AppProject, RBAC, and admission control guarantee operations remain within project namespaces.  
- Secret **values** never appear in UI/CLI (ESO/provider vaults only).

---

## 4) Operations by Area

### 4.1 Deployments (GitOps Apps) & Promotion
**What**: Create/update Liferay DXP and companion workloads; promote dev→uat→prd.

**How (PR)**
- Edit Helm values/overlays (image tag, env, resources, service bindings).
- Raise **promotion PRs** to move a known‑good revision upward.

**How (Runtime)**
- **Sync/Refresh/Rollback** via UI/CLI; server executes Argo ops (audited).

**Verification gates (default)**
- Health = **Healthy**; Sync = **Synced**; **no firing alerts** in target env; **PromotionReady** satisfied.

**CLI examples**
```
lc2 values edit --app dxp --env dev
lc2 apps sync dxp --env uat
lc2 promote --app dxp --from dev --to uat
lc2 apps rollback dxp --env prd --to <git-sha>
```

**Repo sketch**
```
apps/
  liferay-dxp/
    charts/...            # or external chart reference
    values/
      dev.yaml
      uat.yaml
      prd.yaml
environments/
  <projectId>/
    applicationset.yaml   # generates per-env apps
    appproject.yaml       # repo+destination fences
```

---

### 4.2 Workloads & Pods (Triage)
**What**: Inspect rollout status, pods, events, and logs.

**How**
- UI: **Workloads** (desired/ready/available, restarts) → **Pods** (phase, restarts, events) → **Open Logs** (deep link or CLI hint).  
- Optional **Exec** (role‑gated) for debugging; executed by platform SA and audited (no user kubectl).

**CLI examples**
```
lc2 workloads list --env dev
lc2 pods list --env dev --filter crashloop
lc2 logs tail --env dev --app web --level error
```

> Scaling by replicas should be done via **HPA** (PR), not ad‑hoc manual scale.

---

### 4.3 Autoscaling (HPA)
**What**: Set min/max replicas and target metrics.

**How (PR)**
- Update HPA parameters in values/overlay; submit PR.

**CLI convenience**
```
lc2 hpa edit web --env prd --min 3 --max 10 --cpu 70
# creates a PR; platform never writes directly
```

**Runtime visibility**
- HPA scale events timeline, current/desired replicas, metric trends.

---

### 4.4 Networking (Ingress, Load Balancing, URL Routing & TLS)
**What**: Expose services via ingress/gateway controllers with consistent URL patterns, session management, TLS termination, and security controls across all runtimes.

**Philosophy**: Cloud‑agnostic, isolation‑first, GitOps‑always, KISS, local↔cloud parity.

#### 4.4.1 URL Structure & DNS Conventions
**Cockpit Host Format**
`<cloud>-<region>.<cockpit-domain>`

**Examples**
- `aws-us-east-1.myliferaypaas.com`
- `azure-westus2.partnerpaas.com`
- `gcp-us-central1.partnerpaas.com`
- `local.localdevelopment`

**Cockpit Routes**
- UI: `/` and SPA routes under `/projects/[projectId]`
- API: `/api/*` on the same host (default; subdomain only when isolation/externalization is required)

**Tenant Liferay Hosts (Per Environment)**
- `project.dev.sa.example.com`, `project.uat.sa.example.com`, `project.sa.example.com` (prd)

**Project ID Policy**
- Regex: `^[a-z0-9-]{2,40}$` (lowercase, digits, hyphen)

**DNS Records**
- One record per Cockpit region host to the LB
- Per‑tenant env records for Liferay
- Local maps `local.localdevelopment` → `127.0.0.1` via hosts file

#### 4.4.2 Ingress/Gateway Model & Controllers
**Entry Classes**
- `public`: Internet‑facing entry; TLS required; WAF enabled by default
- `private`: Internal entry (VPN/PSC/peering); WAF optional; internal DNS only

**Controllers by Runtime**
- **Local**: Ingress‑NGINX
- **AWS**: AWS Load Balancer Controller (ALB Ingress)
- **Azure**: Application Gateway for Containers (AGC) via Gateway API (preferred); AGIC as fallback
- **GCP**: GKE Gateway Controller (Gateway API)

**Ownership (GitOps)**
- **Admins**: Provide classes, controllers, cert issuers, global policies
- **Projects**: Own namespace‑scoped routes (Ingress/HTTPRoute) in their repos; constrained by AppProjects and policies

#### 4.4.3 Cockpit Exposure Requirements
**Host**: `https://<cloud>-<region>.<cockpit-domain>`

**Routing**
- `/api/*` → `cockpit-api` (ClusterIP)
- `/` and `/projects/*` → `cockpit-ui` (ClusterIP)
- No path rewrites; SPA must handle hard refreshes on deep links

**Sessions**: MUST be stateless; sticky sessions not required

**Protocols and Timeouts**
- HTTP/2 enabled
- WebSockets/SSE allowed (terminal, log stream)
- Idle/Read Timeouts: 120–300s (edge and controller)

**Exposure Class**: Default `public`; provide `private` variant if needed

#### 4.4.4 Tenant Liferay Exposure Requirements
**Hosts**: Per environment with clear isolation (dev/uat/prd)

**Session Policy**
- **Default**: LB/Ingress sticky sessions enabled for PRD
- **Exception**: Tomcat session replication MAY be enabled for small clusters (≤ 5 nodes) where stickiness is not acceptable; requires performance sign‑off and runbook
- **Cookie Name**: `DXP_STICKY`; **Default TTL**: `3600s`

**Rollouts**
- Use weighted traffic (Gateway API HTTPRoute weights or controller equivalent) for canary/blue‑green deployments
- Promotion steps: 5% → 25% → 50% → 100%, with guardrails on error rate and p95 latency

**Timeouts**: Moderate idle/read timeouts: 60–120s at the edge

#### 4.4.5 TLS, Certificates & Secure Headers
**TLS Termination**: MUST terminate at the edge (Gateway/Ingress); mTLS to backends is optional

**Certificates**
- **Cloud**: Provider‑managed certificates (or GCP Certificate Manager) bound to Gateway/Ingress
- **Local**: cert‑manager with ACME DNS‑01; pipeline promotes LE staging → prod

**HSTS and Security Headers**
- HSTS enabled on public entries
- Add common secure headers at the edge (X‑Content‑Type‑Options, X‑Frame‑Options, Referrer‑Policy, etc.)

#### 4.4.6 WAF, Rate Limiting & Edge Security
**WAF**: MUST be enabled for Cockpit public and Liferay PRD public entries

**Path Policies**
- `/api/*` MAY set stricter rules (auth headers required, request size limit, tighter timeouts)

**Exposure Controls**
- Optional IP allowlists for admin‑only endpoints (e.g., `/api/admin/*`)
- Rate limiting MAY be configured per tenant route where supported

#### 4.4.7 Health Checks, Draining & Availability
**Health Endpoints**: Edge health checks MUST hit explicit endpoints (not `/`)

**Rollout Draining**
- Connection draining MUST be enabled (e.g., ALB deregistration delay; AGC/GKE equivalents)
- PDBs ensure minimum availability during rollouts

**Topology**
- PRD clusters SHOULD be multi‑AZ (per infrastructure topology policy)
- DEV/UAT default single‑AZ

#### 4.4.8 Observability Requirements
**Access Logs**: MUST include host, path, status, latency; derive and log `projectId` for Cockpit `/projects/*` requests

**Metrics**: Edge/controller metrics MUST be exported to managed Prometheus backends

**Events and Errors**: Kubernetes events and ingress/controller errors MUST be shipped to cloud logging backends

**Correlation**: Add a correlation/request ID header at the edge and propagate to backends

#### 4.4.9 Reference Manifests by Runtime

**AWS (EKS + ALB Ingress) — Cockpit**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cockpit
  namespace: cockpit
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=300
spec:
  ingressClassName: alb-public
  tls:
  - hosts: [aws-us-east-1.myliferaypaas.com]
    secretName: cockpit-tls
  rules:
  - host: aws-us-east-1.myliferaypaas.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend: { service: { name: cockpit-api, port: { number: 8080 } } }
      - path: /
        pathType: Prefix
        backend: { service: { name: cockpit-ui,  port: { number: 3000 } } }
```

**AWS — Liferay PRD (Sticky Sessions)**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: liferay-prd
  namespace: project-prd
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/target-group-attributes: >
      stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=3600,
      deregistration_delay.timeout_seconds=30
spec:
  ingressClassName: alb-public
  tls:
  - hosts: [project.sa.example.com]
    secretName: project-prd-tls
  rules:
  - host: project.sa.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: dxp-web, port: { number: 8080 } } }
```

**Azure (AKS + AGC via Gateway API) — Cockpit**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cockpit-public
  namespace: cockpit
spec:
  gatewayClassName: azure-alb
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: azure-westus2.partnerpaas.com
    tls:
      mode: Terminate
      certificateRefs: [{ name: cockpit-tls }]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cockpit-route
  namespace: cockpit
spec:
  parentRefs: [{ name: cockpit-public }]
  hostnames: ["azure-westus2.partnerpaas.com"]
  rules:
  - matches: [{ path: { type: PathPrefix, value: "/api" } }]
    backendRefs: [{ name: cockpit-api, port: 8080 }]
  - matches: [{ path: { type: PathPrefix, value: "/" } }]
    backendRefs: [{ name: cockpit-ui,  port: 3000 }]
```

**GCP (GKE + GKE Gateway) — Cockpit**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cockpit-public
  namespace: cockpit
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: gcp-us-central1.partnerpaas.com
    tls:
      mode: Terminate
      certificateRefs: [{ name: cockpit-tls }]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cockpit-route
  namespace: cockpit
spec:
  parentRefs: [{ name: cockpit-public }]
  hostnames: ["gcp-us-central1.partnerpaas.com"]
  rules:
  - matches: [{ path: { type: PathPrefix, value: "/api" } }]
    backendRefs: [{ name: cockpit-api, port: 8080 }]
  - matches: [{ path: { type: PathPrefix, value: "/" } }]
    backendRefs: [{ name: cockpit-ui,  port: 3000 }]
```

**Local (k3d/k3s + Ingress‑NGINX) — Cockpit**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cockpit
  namespace: cockpit
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
spec:
  tls:
  - hosts: [local.localdevelopment]
    secretName: cockpit-tls
  rules:
  - host: local.localdevelopment
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend: { service: { name: cockpit-api, port: { number: 8080 } } }
      - path: /
        pathType: Prefix
        backend: { service: { name: cockpit-ui,  port: { number: 3000 } } }
```

**Local — Liferay (Sticky Test)**
```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "DXP_STICKY"
```

#### 4.4.10 Performance, Capacity & Limits
- **Header/Body Limits**: Set conservative maximums at edge; raise by exception
- **Compression**: Enable gzip/brotli for UI; ensure API responses only when safe
- **Burst Handling**: Validate HPA targets and scale‑up time; pre‑warm if large GC pauses are observed

#### 4.4.11 SLOs & Alerting
**Cockpit**
- 99.9% regional availability
- p95 < 500 ms for interactive views

**Liferay PRD**
- 99.95% (multi‑AZ)
- p95 < 800 ms core paths

**Error Budgets**
- 43.2 min/mo (99.9)
- 21.6 min/mo (99.95)

**Alerts**: Edge 5xx, p95 latency breaches, unhealthy backends, certificate expiry < 14 days

#### 4.4.12 Test & Validation Plan
**Functional**
- Hard refresh `/projects/[projectId]` returns 200 (SPA deep link OK)
- `/api/*` responds with correct CORS (same‑origin → minimal headers)
- WebSocket shell/log stream stable for ≥ 5 min

**Resilience**
- Rolling update of Cockpit and DXP yields no 5xx spikes; deregistration delay respected
- Canary: 5%→25%→50%→100% with live metrics gating

**Security**
- WAF active; test common rules
- HSTS present; TLS 1.2+ only; weak ciphers disabled (per provider defaults)
- Access logs populated with host/path/latency; `projectId` extraction validated

#### 4.4.13 Acceptance Checklist (Per Region/Cluster)
- [ ] DNS records for `<cloud>-<region>.<cockpit-domain>` and tenant env hosts
- [ ] TLS issued and attached (edge termination)
- [ ] Cockpit routes `/api/*` and `/projects/*` functional (hard refresh OK)
- [ ] WebSockets/SSE verified (shell/log streaming)
- [ ] WAF enabled on public entries (Cockpit, Liferay PRD)
- [ ] Liferay PRD stickiness verified (cookie present)
- [ ] Canary deployment validated (weighted traffic)
- [ ] Health checks & drains tuned (no 5xx during rollout)
- [ ] Access logs and metrics visible in vendor backends
- [ ] Namespace isolation & NetworkPolicies enforced
- [ ] All manifests under Git with PR history

#### 4.4.14 CLI Helpers
```bash
lc2 domain add web.prd.example.com --env prd
lc2 tls request --dns web.prd.example.com --env prd
```

> Keep dependencies lean: ExternalDNS and cert‑manager are optional; prefer cloud DNS/certs in EKS/AKS/GKE. Local uses cert‑manager with dev certs.

---

### 4.5 Network Policies (Isolation & Egress)
**What**: Default‑deny networking with explicit allow rules for multi‑tenant isolation.

**How (PR)**
- Edit or add **NetworkPolicy** to allow specific ingress/egress (DNS, DB, cache, external APIs)
- Routes are namespace‑scoped; tenants only manage their own routes
- Default‑deny NetworkPolicies at namespace boundary; egress allowlists enforced
- No cross‑tenant hostnames; each tenant uses its own environment hosts
- Allowed annotations/policies are whitelisted; unsafe/privileged attributes are rejected by policy

**CLI helper**
```
lc2 net allow egress postgres.prd.internal:5432 --env prd
# creates PR amending NetworkPolicy for that namespace
```

> AppProject + admission control prevent cross‑namespace exposure by default. NetworkPolicies enforce isolation at the network layer.

---

### 4.6 Secrets & Service Bindings (ESO)
**What**: Bind Liferay to DB/Search/Storage/Cache/Queue.

**How**
- **ExternalSecret** definitions in Git reference secret providers (AWS SM / Azure Key Vault / Vault) **without values**.
- Rotate by updating provider secret → trigger refresh from UI/CLI.

**CLI examples**
```
lc2 binding set db --engine postgres --secret-ref aws-sm://dxp/uat/db --env uat
lc2 secrets refresh --env prd --name db
```

> UI shows **status only** (Bound/Unbound, last refresh, secret name). Values are never displayed.

---

#### 4.6.1 DXP Licensing (Cluster Mode)
**What**: Handle Liferay DXP cluster licenses for deployments with `replicaCount > 1` or autoscaling (HPA) enabled.

**When Required**
- **Cluster mode requires a valid license**: Any deployment with multiple replicas must provide a **DXP cluster license** (XML format)
- Single-node non-clustered trials are out of scope; always plan for a real license when targeting clustered mode
- Running without a valid license will fail or fall back to trial behavior, which is explicitly disabled by policy

**License Artifact**
- **Format**: XML (stored as `license.xml`)
- **Source**: Customer portal (paid license) or developer cluster licenses for non-production evaluation
- **Lifecycle**: Platform admins manage acquisition, storage, rotation, and revocation workflows
- **Naming**: Keep consistent filename `license.xml` for mounts

**Approved Injection Methods**

**Production (Recommended): Kubernetes Secret**
```yaml
customEnv:
  x-license:
    - name: LIFERAY_DISABLE_TRIAL_LICENSE
      value: "true"

customVolumeMounts:
  x-license:
    - mountPath: /etc/liferay/mount/files/deploy/license.xml
      name: liferay-license
      subPath: license.xml

customVolumes:
  x-license:
    - name: liferay-license
      secret:
        secretName: liferay-license-prd
```

**Local/Dev: ConfigMap**
```yaml
configmap:
  data:
    license.xml: |
      <?xml version="1.0"?>
      <license> ... </license>

customEnv:
  x-license:
    - name: LIFERAY_DISABLE_TRIAL_LICENSE
      value: "true"

customVolumeMounts:
  x-license:
    - mountPath: /etc/liferay/mount/files/deploy/license.xml
      name: liferay-configmap
      subPath: license.xml
```

**Required Settings**
- **Mandatory flag**: `LIFERAY_DISABLE_TRIAL_LICENSE=true` (ensures no accidental trial license in clustered mode)
- **Mount path**: `/etc/liferay/mount/files/deploy/license.xml`

**Per-Environment Structure**
- One license secret per environment (dev/uat/prd) for independent rotation and access control
- Naming convention:
  - `liferay-license-dev`
  - `liferay-license-uat`
  - `liferay-license-prd`

**Runtime Specifics**

**Local (k3d)**
- Store license as local file (`license.xml`) for iteration
- Prefer ConfigMap for quick dev, or Secret for testing rotation/rollout behaviors
- Keep file out of repo; use `.gitignore` and local secret tooling (sops/age) if needed

**AWS (EKS) / Azure (AKS) / GCP (GKE)**
- Use **Secret** in project namespace with Helm values overlay per environment
- **Best practice**: Integrate **External Secrets Operator** with cloud secret manager (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager)
- License never committed to Git; rotated centrally via ESO refresh

**Clustering Guardrails**
- Set `replicaCount: 2+` or enable `autoscaling` (HPA)
- **Do not** use embedded database/search/file storage
- Always point to external shared services (RDS/Azure PG, S3/Blob, OpenSearch/Elastic)
- Ensure consistent portal and OSGi configs across replicas via chart values

**Rotation & Renewal**
- **Rotate by secret update**: Update secret (or backing entry in cloud secret manager)
- Trigger **rolling restart** via Helm upgrade or `kubectl rollout restart`
- Track license **expiry dates** in ops calendar
- Prepare maintenance window if vendor requires version changes alongside license updates
- Keep audit entry (commit + change request) for each rotation

**Security & Compliance**
- **Production**: Use Kubernetes Secrets or External Secrets; **never ConfigMaps** for licenses
- Restrict read access to license secret to Liferay service account only
- Avoid storing licenses in Git; if required, encrypt with sops/age and limit decryption to CI deployers
- Ensure backups exclude plaintext secrets or encrypt backups at rest with KMS

**Verification Checklist**
- [ ] Pods start without license errors in logs
- [ ] Login works and cluster services respond across replicas
- [ ] No trial warnings in system logs
- [ ] `LIFERAY_DISABLE_TRIAL_LICENSE=true` present in pod environment
- [ ] License mounted at `/etc/liferay/mount/files/deploy/license.xml`
- [ ] Secret-based injection used in cloud; ConfigMap not used in production
- [ ] (Optional) Smoke test: scale up/down and confirm continuous availability

**CLI Helpers**
```bash
# Refresh license secret (triggers rolling restart)
lc2 secrets refresh --env prd --name liferay-license

# Verify license mount and environment
lc2 pods list --env prd --app dxp
kubectl exec -it <pod> -- ls -la /etc/liferay/mount/files/deploy/
kubectl exec -it <pod> -- env | grep LIFERAY_DISABLE_TRIAL_LICENSE
```

**Operational Runbook**
1. Obtain or update correct `license.xml` from customer portal
2. Store in environment's secret store (K8s Secret or cloud secret manager)
3. Push values file change (if needed) and merge PR
4. `helm upgrade --reuse-values` the release (Argo CD handles in GitOps mode)
5. Verify health and absence of license warnings in logs

**Acceptance Criteria**
- Clustered Liferay starts and remains healthy with 2+ replicas
- License mounted at correct path
- Environment flag set correctly
- No license or trial warnings in logs
- Secret-based injection in cloud; ConfigMap only for local dev

---

### 4.7 Backup & Restore (Externalized)
**What**: Back up and restore **Database (PostgreSQL)**, **Document Library** (object storage), and **Search Indexes** (OpenSearch/Elastic). **No PVCs by default**; if an exception requires a PVC, follow the documented snapshot/restore path.

**Cloud**
- **AWS (EKS)**: **RDS** snapshots/PITR; **OpenSearch** snapshots; **S3** object versioning for DL.  
- **Azure (AKS)**: **Azure PostgreSQL** backups/PITR; **OpenSearch/Elastic** snapshots; **Blob** object versioning for DL.

**Local**
- Use local object storage (e.g., MinIO) or logical dumps for development only.

**How (PR/CLI)**
- Create a **BackupRequest** (YAML in repo) or invoke `lc2 backup create ...` which generates a PR/scheduled job committed to Git.
- Restores are requested via **PR** (select timestamp/snapshot) or `lc2 backup restore ...` (which creates the PR).

**CLI examples**
```
lc2 backup create db --env prd --retention 7d
lc2 backup create dl --env prd --bucket s3://dxp-prd-dl-backups
lc2 backup create search --env prd --repo opensearch-snapshots

lc2 backup restore db --env prd --point-in-time 2025-09-01T12:00:00Z
lc2 backup restore dl --env prd --snapshot dl-2025-09-01
lc2 backup restore search --env prd --snapshot snap-2025-09-01
```

**Audit & Policy**
- All backup/restore actions are audited (who/what/when/where/result).  
- RPO/RTO targets are defined per environment tier and validated during drills.

---

### 4.8 Observability & Alerts
**What**: Monitor health and metrics; triage alerts; open logs.

**How**
- **Deep links** to **Grafana** for dashboards/alerts (cloud: managed Grafana; local: Grafana OSS). Cockpit does **not** build charts.  
- **Open Logs** button routes to LC2 CLI guidance or provider log console (CloudWatch Logs / Azure Log Analytics) with the correct filters.

**CLI**
```
lc2 alerts list --env prd
lc2 open grafana --app dxp --env uat
lc2 logs query --env uat --match "level:error"
```

---

### 4.9 Policies & Guardrails
**What**: Quotas, Pod Security, allowed registries, admission rules.

**How (PR)**
- Request policy changes via PR to the **platform policy repo**; requires Admin approval.

**Effect**
- Violations are surfaced pre‑sync; forbidden mutations are denied at admission.

---

## 5) Approvals & Auditability

- **Approvals**: Promotions and sensitive ops can require Owner/Admin approval (policy‑driven).  
- **Audit**: Every action emits an immutable record with identity, resource, env, region, diffs (redacted), policy decisions, and outcome.

---

## 6) Error Handling & Rollback

- **Failed sync** → inspect **Diff**, **Events**, logs; rollback to last known good revision via UI/CLI.  
- **Policy deny** → audit entry includes rule and message; fix via PR and retry.  
- **Readiness gate** not met → dependent actions remain disabled; Admin resolves infra; try again.  
- **Backup/restore error** → review job logs/artifacts; re‑run after remediation.

---

## 7) Minimal Dependencies (Keep It Simple)

- **Required**: Argo CD, External Secrets Operator, Prometheus (+ Alertmanager), Ingress controller.  
- **Optional**: Managed Grafana (cloud) / Grafana OSS (local), OpenSearch/Kibana, ExternalDNS, cert‑manager.  
- Prefer **cloud‑managed services** on EKS/AKS for data/state; ensure **local↔cloud parity** of workflows.

---

## 8) Acceptance Criteria (Ops)

- 100% of desired state changes flow through **PRs**; runtime ops via **Argo** only.  
- No **Owner/Member**‑initiated cross‑namespace mutations are possible.  
- Promotion enforces verification gates and records outcomes.  
- HPA edits are PR‑based; scale events are visible with reasons.  
- Backups run on schedule and on‑demand; **restores** are PR‑driven and auditable (DB/DL/Search).  
- Observability **deep links** work with correct scoping; alerts triage is actionable.  
- Feature gating honors **BindingsReady/PromotionReady** and surfaces remediation hints.

---

## 9) Quick Reference — Common Commands

```
# Deployments & Promotion
lc2 values edit --app dxp --env dev
lc2 apps sync dxp --env dev
lc2 promote --app dxp --from dev --to uat
lc2 apps rollback dxp --env prd --to <git-sha>

# Workloads & Pods
lc2 workloads list --env dev
lc2 pods list --env dev --filter crashloop
lc2 logs tail --env uat --app web --level error

# Autoscaling
lc2 hpa edit web --env prd --min 3 --max 10 --cpu 70

# Networking
lc2 domain add web.prd.example.com --env prd
lc2 tls request --dns web.prd.example.com --env prd
lc2 net allow egress postgres.prd.internal:5432 --env prd

# Secrets & Bindings
lc2 binding set db --engine postgres --secret-ref aws-sm://dxp/uat/db --env uat
lc2 secrets refresh --env prd --name db

# Backup & Restore
lc2 backup create db --env prd --retention 7d
lc2 backup restore db --env prd --point-in-time 2025-09-01T12:00:00Z

# Observability
lc2 alerts list --env prd
lc2 open grafana --app dxp --env prd
lc2 logs query --env prd --match "level:error"
```

---

**One‑liner**: *Operate everything via PRs and a few safe Argo operations—within your namespaces only. Keep it simple, audit everything, link to managed observability, use CLI for logs, lean on provider backups, and let Cockpit gate features until dependencies are ready.*

# Architecture — Liferay Cloud 2.0 (LC2)

**Status:** Living document (LC2)
**Principles:** GitOps-first • Isolation by default • KISS • Local↔Cloud parity • Cloud-agnostic • Kubernetes stateless by default

---

## 1) System Overview

**Liferay Cloud 2.0 (LC2)** is a GitOps-first, multi-tenant PaaS for **Liferay DXP** and companion services. **Git** is the source of truth; **Argo CD** reconciles desired state; the **Cockpit** (UI/API) provides the management interface for Owners/Members to operate their projects, while **LC2 CLI** and provisioning scripts handle infrastructure and cluster setup. Projects map to tenants, and each project owns dedicated **dev/uat/prd namespaces** guarded by policy, quotas, and default-deny networking. Secrets flow through the **External Secrets Operator (ESO)**. Observability favors UI parity via **Grafana deep links** and CLI parity via cloud logging/search APIs. Backups and durable state live in managed services to keep Kubernetes stateless by default.

> **Statelessness:** Kubernetes hosts compute, routing, and telemetry agents. Durable data (DB, file store, search) resides in managed cloud services or explicit opt-in OSS backends. PVCs are the exception, not the baseline.

> **Infra abstraction:** Owners/Members never touch IaaS. **Admins** provision projects; Cockpit hides features until dependencies report **Ready**. Terraform fails the region bootstrap if Cockpit is unhealthy.

**Access pattern**
```
<cloud>-<region>.<cockpit-domain>/projects/<projectId>
```
Examples: `aws-us-east-1.myliferaypaas.com/projects/retail` · `azure-westus2.partnerpaas.com/projects/intranet` · `local.localdev/projects/checkout`

---

## 2) Layered Architecture

LC2 keeps a consistent four-layer model from local development through managed clouds.

### 2.1 Infrastructure (Local, AWS, Azure, GCP)
**Responsibilities**
- Networking & identity: VPC/VNet, subnets, routing, security groups/NSGs, OIDC/SSO, public DNS zones.
- TLS endpoints: ACME (Let's Encrypt) or cloud certificate services consumed by cert-manager.
- Managed data planes (preferred in cloud deployments):
  - **Metrics:** AWS AMP, GCP GMP, Azure-managed Prometheus.
  - **Logs:** AWS CloudWatch Logs, GCP Cloud Logging, Azure Log Analytics (default in cloud runtimes) with optional shared Loki for parity.
  - **Object storage:** S3 / GCS / Azure Blob (for optional Loki/Mimir).
  - **Secrets & databases:** AWS Secrets Manager / GCP Secret Manager / Azure Key Vault; RDS / Cloud SQL / Azure Database for PostgreSQL for Cockpit and tenant workloads.

**Local vs Cloud**
- **Local:** No managed services. **Grafana Alloy** forwards telemetry to in-cluster OSS backends (Loki, optional Mimir/Prometheus) backed by MinIO or dev object storage.
- **Cloud:** Keep **Alloy** agents; prefer managed metrics/logs accessed via tenant-aware gateways. Optional **dual-write** to Loki (through the same gateway) for dashboard/log-query parity when required.

**Interfaces Up/Down**
- Kubernetes ingress controllers request LB resources; **ExternalDNS** updates DNS; **cert-manager** uses ACME/cloud issuers; **ESO** reads from cloud secret stores; self-hosted backends write to object storage.

### 2.2 Kubernetes (K3D, GKE, AKS, EKS)
**Responsibilities**
- Provide the cluster substrate and a consistent add-on stack, installed via Helm and reconciled by Argo CD.

**Cluster-level add-ons (shared per cluster)**
- **Argo CD (OSS):** GitOps control plane; isolation through AppProjects + RBAC; ApplicationSet templatizes project apps.
- **Kyverno:** Admission policy (mutate/validate/generate) enforcing requests/limits, restricted registries, default-deny NetworkPolicies, ingress host naming, and "only Argo may deploy" guarantees.
- **Ingress + TLS + DNS:** NGINX or cloud ingress controllers, **cert-manager**, **ExternalDNS**.
- **External Secrets Operator:** Shared controller; projects define SecretStore/ExternalSecret manifests.
- **Collectors:** **Grafana Alloy** DaemonSet unified for logs/metrics/traces.
- Optional OSS backends: **Loki** (cluster shared, always fronted by the multi-tenant gateway enforcing `X-Scope-OrgID`) and **Mimir/Prometheus** when managed backends are unavailable or when dual-write parity is explicitly required.

**Local vs Cloud**
- **Local:** Same controllers; Loki/Prometheus run in-cluster when managed services are absent.
- **Cloud:** Controllers unchanged; Alloy remote-writes to managed services, optionally dual-writing to Loki.

**Interfaces**
- **Up to Infrastructure:** LB requests, DNS updates, certificate issuance, secret retrieval, object storage writes.
- **Down to PaaS:** Stable endpoints for Cockpit deep links (Argo CD, Grafana).
- **Down to Project:** Namespaces, quotas, RBAC roles, policy enforcement, secret sync, shared observability data planes.

### 2.3 PaaS (LC2 Management Plane)
**Responsibilities**
- **Cockpit (UI/API)**: Java + Next.js management interface for Owner/Member personas to operate their projects (deployments, scaling, promotions, observability access).
- **LC2 CLI & Provisioning Scripts**: Terraform and Helm-based infrastructure provisioning for EKS/AKS/GKE clusters, dependencies, and Kubernetes resources.
- Provision projects (namespaces, quotas, AppProjects), register SecretStores/ExternalSecrets, and surface deep links to Argo CD and Grafana.
- Maintain Cockpit metadata (PostgreSQL) and SSO. All changes flow through Git; Cockpit never mutates the Kubernetes API directly.
- Execute short-lived **Provisioner Jobs** (Terraform + cloud CLIs via workload identity) to set up per-project managed services; infra pipeline fails fast if Cockpit is unhealthy.

**Grafana (OSS or managed)**
- One Grafana per cluster: managed service in cloud, OSS locally. Each project uses its own **Organization** (or locked-down folder) with pre-provisioned data sources that embed scoped credentials.

**Interfaces**
- Writes manifests and values to Git (Argo CD syncs). Presents SSO-backed UI/CLI; emits audit logs for every operation.

### 2.4 Project (Environments & Applications)
**Responsibilities**
- Namespaces per environment: `<project>-dev`, `<project>-uat`, `<project>-prd`, each with quotas, LimitRanges, default-deny NetworkPolicies, and project-scoped RBAC.
- Workloads include Liferay DXP and companion services, deployed via Argo CD Applications bounded by the project's AppProject fences (repos, paths, namespaces).
- **ESO** syncs runtime secrets (ExternalSecret → Secret).
- **Grafana Alloy** scrapes/tails workloads and ships telemetry with per-project labels.

**Telemetry Wiring**
- **Logs:** Local → Alloy → gateway → Loki (shared) with enforced `X-Scope-OrgID`; Cloud → Alloy → gateway → provider logging plane (optional dual-write to Loki for parity).
- **Metrics:** Local → Alloy/Prometheus → gateway → Mimir/Prometheus; Cloud → Alloy/Prometheus remote_write → gateway → AMP/GMP/Azure-managed Prometheus.

---

## 3) Domains & URL Topology

**Declared once in Terraform**
- `cluster_domain` (cloud) e.g., `mycluster.example.com`
- `cluster_domain_local` (local) e.g., `localdev.me`

**Published (ExternalDNS for cloud; static for local)**
- `cockpit.<cluster-domain>`
- `*.dev|uat|prd.<project>.<cluster-domain>`

**Project UI routes:** `<cloud>-<region>.<cockpit-domain>/projects/<projectId>`  
Admin console: `<cloud>-<region>.<cockpit-domain>/admin/...`

---

## 4) Physical / Deployment Architecture

### Local (Developer)
- Single‑node Kubernetes (kind/k3d/minikube)
- Cockpit (UI/API) in `liferay-cockpit`; **local Postgres** for Cockpit DB
- Argo CD, ESO, Prometheus (+ Grafana OSS), **Loki** for logs
- Dev‑grade ingress (NGINX) and dev certificates/DNS
- Optional local Postgres/MinIO/OpenSearch for **development only**

### AWS (EKS)
- EKS cluster per region; node groups sized by tenant density
- Cockpit (UI/API) in `liferay-cockpit`; **RDS for PostgreSQL** for Cockpit DB
- **Managed services** per project: RDS for PostgreSQL, Amazon OpenSearch (or Elastic Cloud), S3
- Observability: **Amazon Managed Service for Prometheus + Amazon Managed Grafana**
- Logs: **CloudWatch Logs** (log group per project/env)
- Ingress: **ALB Ingress Controller** or NGINX + ALB; DNS via Route53; TLS via ACM or cert‑manager

### Azure (AKS)
- AKS cluster per region
- Cockpit (UI/API) in `liferay-cockpit`; **Azure Database for PostgreSQL** for Cockpit DB
- **Managed services** per project: Azure Database for PostgreSQL, OpenSearch/Elastic Cloud, Blob Storage
- Observability: **Azure Managed Prometheus + Azure Managed Grafana**
- Logs: **Azure Log Analytics** (workspace per region; queries scoped per project/env)
- Ingress: **AGIC** or NGINX; DNS via Azure DNS; TLS via Key Vault/Front Door or cert‑manager

---

## 5) Provisioning Architecture (Terraform + Cockpit)

### 5.1 Cluster & Cockpit Bootstrap (Terraform‑driven, Cockpit last)
1) **Network & Cluster** → VPC/VNet, EKS/AKS, node pools, storage classes.  
2) **Core add‑ons** → ingress, cert‑manager, ExternalDNS (cloud), ESO, metrics/logging.  
3) **Cockpit DB** → dedicated RDS/Azure PG for the management plane.  
4) **Cockpit** → Helm release with `wait=true`, values wired to Cockpit DB via **ESO**, ingress at `cockpit.<cluster-domain>`.

**Fail‑fast health gate**
- Terraform `helm_release.wait=true` (pod readiness).  
- Post‑apply **readiness Job** curls `https://cockpit.<cluster-domain>/healthz` until 200 (timeout).  
- On failure, **Terraform apply fails** (pipeline red).

### 5.2 Project Provisioning (Admin‑only, one click, no infra exposure)
- Admin uses **Project Wizard** (ID, display, preset, envs); region fixed to current host (until multi‑region auth).  
- Cockpit schedules a **short‑lived Provisioner Job** (in `liferay-cockpit`) that assumes a least‑priv **workload identity** and runs a single **`project-provisioner` Terraform stack**.

**Idempotent per‑project outcomes**
- **Prd**: dedicated DB, bucket/container, and search domain.
- **Dev/UAT**: one non‑prd DB (two DBs or per‑env schemas), one non‑prd bucket (prefixes), one non‑prd search domain (prefixes).  
- Provider secrets written to **Secrets Manager/Key Vault**; Cockpit commits **ExternalSecret** resources so **ESO** syncs them into `proj-<id>-<env>`.  
- Namespaces, RBAC, NetworkPolicy, quotas; minimal **Helm values overlays** referencing endpoints/secret names.  
- If GitOps is enabled, Cockpit commits overlays to Git and Argo reconciles.

### 5.3 Readiness & Auto First Run
- Cockpit computes **BindingsReady** per env from live probes: **IaaS binding**, **DB**, **file store**, **search**, **ESO freshness**.  
- For **promotion**, also checks **Grafana link** and **logs query**.  
- When **DEV = Ready**, Cockpit opens a **values PR** for the **latest approved Liferay** and deploys to `proj-<id>-dev` via Argo, exposing `portal.dev.<project>.<cluster-domain>`.

---

## 6) Repositories & Configuration Layout

```
platform-policy/
  kyverno/
  quotas-limitranges/
  networkpolicies/
  registries-allowlist/

environments/
  <projectId>/
    appproject.yaml
    applicationset.yaml

apps/
  liferay-dxp/
    charts/ (or chart dependency)
    values/
      dev.yaml
      uat.yaml
      prd.yaml
    serviceRefs/       # ExternalSecret definitions for DB/Search/Storage/Cache/Queue
```

> Backups/Restores are externalized and defined in provider tooling; if a project introduces a PVC exception, its snapshot/restore manifests live alongside the env config.

---

## 7) Data Flows & Sequences

### 7.1 Project Create (Admin)
```
Admin (UI) → Cockpit API: /admin/projects
Cockpit → K8s: schedule Provisioner Job (Terraform)
Provisioner → Cloud: create per‑project resources (prd dedicated; dev/uat shared), write secrets
Cockpit → Git/K8s: commit ExternalSecret + values overlays; create namespaces/RBAC/netpols/quotas
ESO → K8s: sync secrets into project namespaces
Cockpit → Probes: compute BindingsReady/PromotionReady
(If DEV Ready) Cockpit → Argo: deploy latest approved Liferay to DEV
```

### 7.2 Config Change (PR → Deploy)
```
Owner/Member → Git: Raise PR (values/overlay/manifest/policy)
Maintainer → Git: Merge PR
Argo CD → Cluster: Reconcile desired→live (namespaced)
Cockpit API → Audit: Record who/what/when/where/result
```

### 7.3 Promote dev → uat → prd
```
Owner/Member (UI/CLI) → Cockpit API: Generate promotion PR
Maintainer → Git: Merge PR
Argo CD → Cluster: Sync target env
Gates: Healthy + Synced + No firing alerts
Cockpit API → Audit: Record outcome
```

### 7.4 Runtime Operation (Sync / Rollback)
```
Owner/Member (UI/CLI) → Cockpit API: sync/rollback(target SHA)
Cockpit API → Argo API: execute under operator SA
Cockpit API → Audit: record operation & result
```

### 7.5 Secret Rotation
```
Admin/Owner → Provider vault: update secret
Admin/Owner (UI/CLI) → Cockpit API: trigger ESO refresh
ESO → K8s Secret: reconciles value
```

### 7.6 Logs (CLI‑first)
```
Owner/Member (CLI) → LC2 CLI: logs tail/query --project --env [filters]
LC2 CLI → Provider API: CloudWatch Logs / Log Analytics / Loki
```

---

## 8) Security & Identity

- **MVP auth (temporary)**: single regional **admin** (forced password change). No signup; Owners/Members logins arrive post‑MVP.  
- **Post‑MVP**: Global SSO (OIDC) with regional RBAC and JIT provisioning (documented separately).  
- **RBAC**: namespace‑scoped roles; Owners/Members have project‑scoped read/exec/log; writes via PRs/Argo ops only.  
- **Admission**: Kyverno denies CREATE/UPDATE/DELETE not from Argo controller SA or Cockpit operator SA.  
- **Secrets**: values never shown; logs/PRs redacted.  
- **Network**: default‑deny NetworkPolicy with curated egress; registry allowlist enforcement.  
- **Audit**: append‑only; every operation includes identity/scope/outcome; correlation IDs.

---

## 9) Observability Architecture

### Personas and access model
- **Admin (platform operator)**: Provisions/operates clusters and shared observability backends; sets defaults (limits, retention); can view and troubleshoot across projects. Changes must not weaken tenant isolation.
- **Project owner (tenant admin)**: Administers one project (dev/uat/prd namespaces). Can view only their project's telemetry, manage dashboards/alerts inside their project scope, and invite members.
- **Member (tenant user)**: Read-only visibility into their project's telemetry. No access to other projects or platform-level data.

**Isolation rules**
- Owners and members must never see logs/metrics for any other project.
- Cross-project visibility is reserved for admins only, and is kept outside tenant UIs.
- Cockpit deep-links a user into the correct Grafana **organization** for their project.

### Design principles
1. **Shared backends, strict isolation**: Prefer shared, horizontally scalable backends per runtime with strong logical isolation over one stack per project.
2. **Org-per-project in Grafana**: Each project maps to one Grafana organization. Users belong to exactly one org by default; admins can switch orgs for support.
3. **Header-enforced tenancy**: All reads/writes to multi-tenant backends go through a gateway that sets the tenant id header. Clients cannot set or override it.
4. **One click from Cockpit**: Cockpit links directly into project-scoped dashboards and Explore views; tenants do not need kubectl.

### Architecture overview (baseline)

**Logs**
- **Primary backend (cloud):** provider logging planes — AWS CloudWatch Logs, GCP Cloud Logging, Azure Log Analytics — remain the system of record for durability and compliance. Tenants reach them through the LC2 CLI and provider consoles.
- **Parity backend (optional):** a shared multi-tenant **Loki** per runtime, backed by object storage, is mandatory for local environments and opt-in for cloud runtimes that need Grafana query parity or offline troubleshooting. When enabled, Alloy dual-writes to both the provider backend and Loki.
- **Isolation and auth:** an authenticating **gateway** sits in front of Loki (and other multi-tenant endpoints) and injects the required tenant header (e.g., `X-Scope-OrgID=<project-id>`). Requests that lack or attempt to override the header are rejected.
- **Collection agents:** prefer **Grafana Alloy** (unified agent). Promtail remains supported during its LTS window; plan migration.
- **Storage and retention:** provider retention policies govern the primary backend; Loki retention is managed per tenant via `limits_config` and runtime overrides when dual-write is enabled.
- **UI:** a single **Grafana** per runtime with one **organization per project**. Grafana data sources point at the gateway-backed Loki when dual-write is enabled; otherwise Cockpit links tenants directly to provider log consoles.

**Metrics**
- **Per-project collectors:** **Prometheus Agent / Alloy** runs per project (namespace-scoped), scraping only that project.
- **Transport:** collectors use `remote_write` through a tenant-aware **gateway** to a shared multi-tenant metrics backend (preferred **Grafana Mimir**; alternative **Cortex**).
- **Rules and alerts:** recording/alerting rules reside in the backend **ruler**, partitioned per tenant. No tenant-visible global rules in baseline.
- **UI:** same Grafana org model; one Mimir/Cortex data source per org.

**Traces (later, not baseline)**
- Adopt **Tempo** with Alloy when needed.
- Mirror the logs/metrics pattern: tenant header at the gateway, org-scoped data sources.

### Multi-cloud posture (compatibility)
- **AWS:** Amazon Managed Prometheus (AMP) for metrics, CloudWatch Logs as the primary logging plane, and S3 for optional Loki object storage. Use gateways to enforce tenant headers; still keep one Grafana per runtime.
- **Azure:** Azure Managed Prometheus and Log Analytics as primary; Blob Storage backs optional Loki. Same gateway model applies.
- **GCP:** Managed Service for Prometheus and Cloud Logging as primary; GCS backs optional Loki. Same gateway model applies.

### Isolation model

**Mapping**
- Project id ↔ Grafana organization (1:1)
- User → belongs to one org (default) and inherits viewer/owner rights within that org
- Cockpit deep-links include the org context so users land in the right scope

**Enforcement controls**
- **Gateway:** only the gateway can set the tenant header; requests from clients that include the header are dropped or ignored.
- **RBAC:** Grafana org roles (viewer, editor, admin) limited to the org. Do not grant tenant users global admin.
- **Network:** namespace network policies restrict in-cluster access to backends via the gateway.
- **Storage:** tenant data is separated by index/prefix in backends. Retention is applied per tenant.
- **Limits:** per-tenant caps prevent noisy neighbors from degrading others (ingestion, queries, series/labels).

### Portability pattern (local ↔ cloud)
- **Local (k3d):** Alloy ships logs/metrics to in-cluster Loki (and optional Prometheus/Mimir). Grafana uses OSS data sources (Loki, Prometheus) for dashboards and Explore.
- **Cloud:** Alloy keeps shipping but targets managed backends (AMP/GMP/Azure Prometheus for metrics; CloudWatch/Cloud Logging/Log Analytics for logs). Grafana uses native provider plugins alongside optional Loki/Mimir data sources. When parity is required, Alloy dual-writes to both the managed backend and an in-cluster Loki so queries and dashboards behave identically across environments.

---

## 10) Backup & Restore (Externalized)

- **Database (PostgreSQL)**: RDS/Azure PG automated backups + PITR (prd).  
- **Document Library**: S3/Blob versioning + lifecycle.  
- **Search**: OpenSearch/Elastic snapshots.  
- **PVCs**: not used by default; any exception documents snapshot/restore.

---

## 11) API Surface (Control Plane)

- **GitOps & Argo Ops**: `POST /projects/{id}/apps/{app}/sync|rollback|refresh`; `GET /projects/{id}/apps/{app}/diff|tree`
- **PR Generators**: `POST /projects/{id}/values/pr`; `POST /projects/{id}/promotion/pr`
- **Observability Links**: `GET /projects/{id}/observability/grafana-link`; `GET /alerts`
- **Logs**: `POST /projects/{id}/logs/query|tail` (brokers provider APIs)
- **Audit**: `GET /projects/{id}/audit`; `GET /audit/{auditId}`; `GET /audit/stream` (SSE)

All endpoints enforce project/namespace scoping and emit audit records.

---

## 12) Failure Modes & Resilience

- **Cockpit not healthy at bootstrap** → Terraform run **fails** (pipeline red).  
- **Argo unavailable** → reconcile pauses; Cockpit reads cluster state and returns degraded; no writes allowed.  
- **Git outage** → no config changes; runtime ops continue; audits still emitted.  
- **Observability down** → Cockpit falls back to K8s events; deep links disabled.  
- **Provider logging error** → CLI surfaces error and suggested retries.  
- **Admission/policy misconfig** → safety checks and policy repo rollback.

---

## 13) Non‑Functional Targets (MVP)

- **Availability (Cockpit API/UI)**: 99.5%  
- **Performance**: P95 API < 500ms; link generation < 300ms  
- **Scale**: ≥ 50 projects, 150 envs, 500 deployments/day/cluster  
- **Security**: no Owner/Member direct writes; secrets redaction verified; TLS everywhere  
- **Portability**: local↔cloud parity of core flows  
- **Operability**: one‑command region bootstrap; project onboarding in minutes

---

## 14) Minimal Dependencies (Keep It Simple)

**Required**
- Kubernetes, Argo CD, Helm, External Secrets Operator, Prometheus (+Alertmanager), Ingress Controller

**Optional**
- Grafana, OpenSearch/Elastic + Dashboards, ExternalDNS, cert‑manager, OpenTelemetry (post‑MVP)

**Cloud‑managed preferred (EKS/AKS)**
- RDS/Azure DB for PostgreSQL, OpenSearch/Elastic, S3/Blob, DNS/cert services

Avoid low‑adoption/experimental projects that increase blast radius or break parity.

---

## 15) Open Decisions / Future Work

- Optional vCluster or per‑tenant clusters for higher isolation tiers.
- FinOps signals surfaced via links.
- Pipeline integrations beyond status surfacing.
- Tracing/OTel adoption (if needed) with managed exporters.

---

**One‑liner**: Terraform boots the cluster and Cockpit (with its own DB) and **fails** unless Cockpit is healthy. Then an **Admin clicks once** to create a project; a short‑lived in‑cluster Provisioner sets up external services and secrets, Cockpit runs readiness probes, auto‑deploys Liferay to **DEV**, and keeps Kubernetes **stateless by default** with infra fully abstracted from Owners/Members.

# Core Platform Scaffolding — LC2

**Status:** Living document (LC2)
**Principles:** KISS • Isolation by default • Cloud-agnostic • GitOps-first • Helm is primary packaging • Kyverno for admission/policy • Cockpit is the management plane

---

## Purpose
Define the minimum, repeatable Kubernetes/GitOps scaffolding that every LC2 **project** (tenant) receives by default. This sets strong isolation and clear guardrails while keeping the footprint small (KISS).

## What the scaffold creates
For each **project** (tenant) and its **environments** (dev, uat, prd):

1. **Namespaces & Labels**
   - `{project}-{env}` namespaces (e.g., `acme-dev`, `acme-uat`, `acme-prd`)
   - Required labels/annotations:
     - `project=<id>`, `env=<dev|uat|prd>`, `owner=<group>`, `cost-center=<tag>`

2. **Quotas & Limits**
   - `ResourceQuota` profiles (S/M/L) and `LimitRange` defaults for CPU/memory requests/limits.

3. **Network Policies (default-deny)**
   - Ingress default-deny; allow from project's ingress controller/namespace only.
   - Egress default-deny; allow DNS, required cloud services (DB endpoints, object store), and third-party endpoints approved per project.

4. **RBAC (project-scoped)**
   - Roles: `project-owner`, `project-deployer`, `project-viewer`.
   - Bindings to IdP groups mapped per project.
   - Argo CD AppProject enforces repo/path/namespace fences.

5. **Secrets & Config**
   - **External Secrets Operator** (ESO) wired to cloud secret store; no opaque secrets stored in Git.
   - Conventions for `ExternalSecret` and `SecretStore` per project.

6. **Argo CD AppProject + Applications**
   - App-of-apps per project (dev/uat/prd).
   - Promotion via PR (values files per environment).

7. **Observability hooks (minimal)**
   - Alloy/agent DaemonSets and project-scoped configs behind feature flags (enabled when observability is installed).

---

## Kubernetes Admission Control

### What admission control is
When you create/update Kubernetes resources, the API server runs **admission** steps before persisting them:
- **Mutating** admission (may change the request, e.g., inject defaults/sidecars).
- **Validating** admission (approve/deny requests that don't meet policy).

Admission policies enforce **safety and consistency**—e.g., "all pods must have CPU/memory limits," "no privileged containers," or "ingress hosts must be under our approved domain."

### Our decision: Kyverno-first
LC2 standardizes on **Kyverno** for admission (generate/mutate/validate). Reasons:
- **KISS authoring:** policies are YAML with Kubernetes-native patterns—easy for app teams to read and review.
- **Generate & mutate:** create scaffolding objects (e.g., default `NetworkPolicy`) and enforce/patch manifests.
- **Policy reports:** compliance surfaced as CRDs; easy to surface in Cockpit later.
- **Exception handling:** policy `exceptions` and namespace-scoped overrides keep operators productive.
- **Portability:** no cluster-specific code; works across AWS/Azure/GCP/local.

> We intentionally **do not** document ValidatingAdmissionPolicy/OPA in LC2. All baseline guardrails use Kyverno to reduce cognitive load and drift.

### Baseline Kyverno policy set (minimal but strong)
- **Security**
  - **Deny privileged** containers/escalations; require `runAsNonRoot`, `allowPrivilegeEscalation=false`.
  - **Drop dangerous capabilities**; restrict `hostPID`, `hostIPC`, `hostNetwork`.
  - **Seccomp/AppArmor** profiles required (runtime/default or baseline).
  - **Approved registries only** (e.g., ECR/ACR/GCR, plus allow-list for vendor images).

- **Resource hygiene**
  - **Require requests/limits** on all Pod controllers.
  - **Block hostPath** volumes by default.
  - **NodePort disabled**; LoadBalancer only via approved ingress controllers.
  - **Enforce labels/annotations** (`project`, `env`, `owner`, `cost-center`).

- **Network & ingress**
  - **Default-deny NetworkPolicy** present in every namespace (generated if missing).
  - **Ingress host rules** must match `{env}.{project}.example.com` or provider-specific domain policy.
  - **Egress allow-list** via labeled `NetworkPolicy` rules (DNS, DB endpoints, object store).

- **Secrets & config**
  - **Disallow Opaque `Secret` creation** by tenants; require **ExternalSecret** except for runtime bootstrap.
  - **Block cross-namespace Secret/ConfigMap references** (isolation).

- **Image & supply chain**
  - Optional: **image signature/policy** hooks (future), **pullPolicy** defaults.

---

## Bootstrap sequence (operator checklist)
1. Create `{project}-dev|uat|prd` namespaces with required labels.
2. Apply quota/LimitRange profile (S/M/L).
3. Bind IdP groups to `project-*` roles.
4. Register `SecretStore` for project with ESO.
5. Create Argo CD `AppProject` and app-of-apps in platform repo.
6. Enable baseline Kyverno policies (cluster-scoped) and project exceptions (if any).
7. Apply default-deny NetworkPolicy (generated/verified by Kyverno).
8. Wire Cockpit links to Argo, Grafana (org = project), and Observability (if enabled).

---

## Acceptance criteria
- Creating a Deployment without CPU/memory limits is **denied**.
- Namespaces show default-deny NetworkPolicy and quotas in place.
- A Pod in `acme-dev` **cannot** reach Services in `acme-uat` unless explicitly allowed.
- Secrets are created via **ExternalSecret**; attempting to apply Opaque Secrets is denied for tenants.
- Argo CD shows applications restricted to `{project}-{env}` namespaces only.
- Promotion dev→uat→prd occurs by PR; no kubectl required for tenants.

---

# Kyverno Policy Model & Governance — LC2

**Status:** Living document (LC2)
**Scope:** Comprehensive governance model for Kyverno, RBAC, NetworkPolicy/Pod Security, Argo CD, and ESO

---

## Goals and scope

- Guarantee **project isolation** across dev/uat/prod environments (security, network, resources)
- Enable **developers** (owners/members) to deploy any Helm chart without risking platform health
- Make behaviors **predictable** via admission policies, GitOps guardrails, and small opinionated defaults
- Keep the stack **portable** across local (k3d) and cloud (EKS/AKS/GKE), with consistent semantics

---

## Personas and responsibilities

### Admin (cluster operator)
- Provisions and operates cluster + shared platform controllers via GitOps
- Owns **cluster guardrails** (Kyverno ClusterPolicies), **AppProjects**, **namespace bootstrap**
- Configures **ESO** backends, **Argo CD** RBAC, and baseline **PSA/PSS** posture
- Curates policy library, approves **PolicyException** and monitors **PolicyReports**

### Project owner
- Full control of workloads **inside their namespaces** (e.g., `acme-dev|uat|prd`)
- Defines Argo **Applications** in their tenant **AppProject** and may add narrow NetPol allows
- Cannot change cluster-scoped resources or guardrails

### Project member
- Narrower verbs within project namespaces (deploy/sync/scale/restart) or read-only access

---

## Control model overview

- **RBAC** answers *who* can do what (cluster vs namespace scopes)
- **Admission** (Kyverno + PSA) answers *what* is allowed (mutate/validate/generate/verify)
- **NetworkPolicy** implements runtime **isolation** (default-deny + explicit allows)
- **GitOps** (Argo CD) defines **where** applications can deploy (AppProjects + RBAC)
- **Secrets** are pulled through **ESO** with strict namespace boundaries

Admission order is **mutate → validate**; violations **fail** admission for non-negotiables.

---

## Cluster guardrails (admin-owned, non-negotiable)

> Implement once, apply to all namespaces; keep the set small and clear.

1) **Pod security: restricted baseline**
   - Prefer **Pod Security Admission (PSA)** labels on namespaces to enforce **Restricted** profile; optionally complement with Kyverno checks for specific gaps.
   - Standardize namespace labels (e.g., `pod-security.kubernetes.io/enforce=restricted`).

2) **Default-deny network policy (generate + backfill)**
   - Generate a namespaced `NetworkPolicy` that denies all ingress/egress by default.
   - Apply to **new** and **existing** namespaces (set `generateExisting: true`).
   - Tenants may add **allow** rules; default-deny remains.

3) **Anti-noisy-neighbor (resource governance)**
   - Generate per-namespace **ResourceQuota** (CPU/RAM/PVC/object caps) and **LimitRange** (defaults and max/min).
   - Require every container to declare requests/limits.

4) **Supply-chain hygiene (verify images)**
   - Restrict image **registries** to approved allow-list.
   - Block `:latest`; prefer **digests** in UAT/PRD.
   - Enforce **signature/attestation** verification with `verifyImages` (Cosign/Notary).

5) **Service exposure and ingress**
   - Forbid **NodePort**; default **ClusterIP**.
   - Allow **LoadBalancer** only via explicit annotation/label.
   - Enforce **HTTPS** for UAT/PRD; allow HTTP only in DEV for speed.

6) **Exceptions and posture**
   - Allow **PolicyException** (namespaced, precisely scoped, time-boxed) with an approval flow.
   - Expose **PolicyReports** for cluster and namespaces; surface in the cockpit UI.

---

## Tenant policies (namespaced, owned by project)

1) **Label/annotation contract**
   - Validate/mutate required keys on workloads/services:
     - `app.kubernetes.io/name|component|instance`
     - `cockpit.liferay.io/project|env|app`
   - Enables routing, cost/chargeback, searchability, and dashboards.

2) **Egress allow-list**
   - Owners add targeted egress to SaaS/DB endpoints; deny wildcards like `0.0.0.0/0`.

3) **Secrets discipline (ESO)**
   - Prefer `ExternalSecret` over raw `Secret`; begin with **Audit** (warn) then flip to **Enforce**.
   - Use `SecretStore` per namespace when possible; **ClusterSecretStore** only with constraints.

4) **Image pull policy by environment**
   - DEV: `IfNotPresent`; UAT/PRD: use digests (and `Always` if tags are allowed).

---

## Argo CD governance (shared instance)

- One **AppProject per tenant** restricting:
  - **sourceRepos** (approved Git orgs)
  - **destinations** (tenant namespaces/clusters)
  - **clusterResourceWhitelist** and **namespaceResourceWhitelist** (allowed Kinds)
- Map SSO groups to roles via `argocd-rbac-cm` (`policy.csv`); keep default role minimal.
- Define Argo config **declaratively** in Git.
- Pair AppProjects with Kyverno guardrails to avoid cross-boundary writes.

---

## Rollout strategy

1) **Audit**: enable policies in `Audit` where useful; review **PolicyReports**; fix drift.
2) **Enforce core**: flip PSS/PSA, default-deny, quotas/limits, supply-chain policies to `Enforce`.
3) **Tighten**: enforce tenant convenience policies (secrets discipline, TLS rules) once green.
4) **Exceptions**: allow narrow, expiring **PolicyException** with approval trail.

---

# Technology Stack — Liferay Cockpit

**Status:** Draft (MVP‑aligned)  
**Principles:** *GitOps‑first • Kubernetes‑native • Isolation by default • KISS • Local↔Cloud parity • Kubernetes stateless by default*  
**Core capabilities:** Isolation · DevOps workflow (dev→uat→prd) · **Observability (UI‑first)** · **Logs (CLI‑first)** · **Backup & Restore (externalized)**

---

## 1) Overview

This document enumerates the **technologies** used by Liferay Cockpit across **local** and **cloud** (EKS/AKS) deployments. Choices follow **KISS**, enforce **hard isolation**, and keep **Kubernetes as stateless as possible** (durable data lives outside the cluster). We rely on **managed services where available** and avoid building UI where a cloud console already exists (Cockpit deep‑links instead). No message bus is required by default.

---

## 2) Layers at a Glance

| Layer | Required | Optional | Notes |
|---|---|---|---|
| Kubernetes | ✅ | — | Local (kind/minikube/k3d), AWS EKS, Azure AKS |
| GitOps Controller | ✅ Argo CD | — | AppProject, ApplicationSet, Application |
| Packaging | ✅ Helm | Kustomize | Helm primary; Kustomize overlays allowed |
| Policy/Admission | ✅ Kyverno | — | Enforce mutation source, PSA posture, registries, image rules |
| Secrets Mgmt | ✅ External Secrets Operator (ESO) | Vault/Provider plugins | Values never exposed in UI/CLI |
| Metrics & Alerting | ✅ Prometheus (+ alerting) | — | Managed on cloud (AMP/Azure Managed Prometheus); OSS locally |
| Dashboards | ✅ Grafana | — | Managed Grafana on cloud; Grafana OSS locally; Cockpit links—no custom charts |
| Logs | ✅ Provider logging plane | OpenSearch/Elastic + Kibana | EKS: CloudWatch Logs · AKS: Azure Log Analytics · Local: Loki. OpenSearch/Kibana are BYO if desired |
| Ingress | ✅ Ingress Controller | — | NGINX, ALB (AWS), AGIC (Azure) — pick per environment |
| DNS | — | ExternalDNS | Prefer cloud DNS (Route53/Azure DNS); ExternalDNS optional |
| TLS | — | cert‑manager | Or cloud certs (ACM / Key Vault + Front Door) |
| Backup & Restore | ✅ Provider backups/versioning | — | DB PITR (RDS/Azure PG), S3/Blob versioning for DL, search snapshots. If rare PVC exceptions exist, use provider/CSI snapshots only |
| Control Plane UI | ✅ **Next.js (TypeScript)** | — | “Cockpit” web app |
| Control Plane API | ✅ **Java 21 (Spring Boot 3.x)** | — | Authoritative backend; REST + SSE; owns domain, RBAC, audit, provisioning orchestration |
| CLI | ✅ `lc2` | — | Project‑scoped; username/password auth (MVP), OIDC (post‑MVP); wraps provider logging where applicable |
| Audit | ✅ Append‑only audit store | — | Records who/what/when/where/scope/result; redacts secrets |

---

## 3) Core Components

### 3.1 Kubernetes
- **Local**: kind/minikube/k3d (1‑node).  
- **Cloud**: AWS EKS / Azure AKS (regional).  
- **Baseline**: Namespaces per env, PSA=restricted, ResourceQuota/LimitRange, default‑deny NetworkPolicy.

### 3.2 GitOps — Argo CD
- **Objects**: AppProject (per project), ApplicationSet (per env/region), Application.  
- **Sync**: automated/prune/self‑heal configurable; history & rollback.  
- **Access**: Argo controller SA whitelisted by admission policy for mutations.

### 3.3 Packaging — Helm (primary)
- Liferay DXP profile as Helm values with schema validation.  
- Overlays via Kustomize (optional).

### 3.4 Policy/Admission
- **Kyverno** to deny CREATE/UPDATE/DELETE not from Argo controller/Cockpit operator; enforce registries allowlist, image policies, and PSA posture.

### 3.5 Secrets
- **External Secrets Operator (ESO)** with provider backends (AWS Secrets Manager / Azure Key Vault / Vault).  
- ExternalSecret manifests live in Git; values never appear in Cockpit.

### 3.6 Observability (UI‑first)
- **Metrics & Alerting**: Prometheus (managed on cloud; OSS locally).
- **Dashboards**: **Grafana** (Managed Grafana on cloud; Grafana OSS locally).
- Cockpit provides **deep links** by project/env; **no custom charts** in Cockpit.

**Guardrail: One managed stack per runtime**
- **One** Prometheus + Grafana instance per runtime (EKS cluster, AKS cluster, local k3d).
- Cockpit **links only**; no cross-project views for admins.
- Project teams access their own scoped dashboards via deep links.

### 3.7 Logs (CLI‑first)
- **EKS**: **CloudWatch Logs** (log group per project/env).
- **AKS**: **Azure Log Analytics** (workspace queries/KQL).
- **Local**: **Loki** (multi‑tenant via labels/tenant IDs).
- **Optional**: OpenSearch/Elastic + Kibana if the project brings them; LC2 CLI still enforces project/env scope.

**PoC: Loki + CockpitLogs (scoped)**
- **PoC only**: Loki + CockpitLogs (Next.js UI + Java API) for per-tenant/per-namespace log views.
- **No global live-tail**; all queries are scoped to `{project, env}`.
- Default remains **provider logging plane** (CloudWatch/Log Analytics for cloud).

### 3.8 Ingress, DNS, TLS
- **Ingress**: NGINX Ingress Controller (generic), or ALB (AWS), AGIC (Azure).  
- **DNS**: Route53 (AWS), Azure DNS (Azure). **ExternalDNS** optional for automation.  
- **TLS**: cert‑manager optional; otherwise ACM (AWS) / Key Vault + Front Door or App Gateway (Azure).

### 3.9 Backup & Restore (externalized)
- **Database (PostgreSQL)**: RDS (AWS) / Azure Database for PostgreSQL — **automated backups + PITR**.  
- **Document Library**: S3 (AWS) / Blob (Azure) — **versioning + lifecycle**.  
- **Search**: OpenSearch/Elastic — **snapshot repositories** to S3/Blob.  
- **PVCs**: **not used by default**. If a documented exception needs a PVC, use provider/CSI snapshots for that namespace.

### 3.10 Control Plane

#### UI — Next.js (TypeScript)
- App Router, SSR where needed, SSE/WebSockets for live status.  
- Project‑scoped UX; Admin‑only provisioning screens.  
- Strict CSP, HSTS; CSRF protection for POSTs to API.

#### API — **Java 21 / Spring Boot 3.x**
- REST + SSE; OpenAPI 3.1.  
- Modules: `api`, `core`, `provisioner-control`, `integration`.  
- Orchestrates Argo ops; schedules/streams Provisioner Jobs; computes readiness gates; emits audits.  
- Integrations: Kubernetes client, Argo CD API, provider logging planes, provider vaults, Terraform state backends.

### 3.11 CLI (`lc2`)
- **Auth**: username/password login (MVP); OIDC device/PKCE (post‑MVP).
- **Ops**: `apps sync|rollback|refresh`, `values edit`, `promote`, `logs`, `alerts`, `backup create|restore`.
- **Guarantee**: Spec changes → PRs; ops → API → Argo; no direct cluster writes; **Owners/Members have no kubectl**.

---

## 4) Local ↔ Cloud Mappings

| Capability | Local (Dev) | AWS (EKS) | Azure (AKS) |
|---|---|---|---|
| Kubernetes | kind/minikube/k3d | EKS | AKS |
| Ingress | NGINX | ALB or NGINX+ALB | AGIC or NGINX |
| DNS | Dev DNS/hosts | Route53 (ExternalDNS optional) | Azure DNS (ExternalDNS optional) |
| TLS | Dev certs/cert‑manager | ACM or cert‑manager | Key Vault/Front Door or cert‑manager |
| Cockpit DB | Local Postgres (StatefulSet) | RDS for PostgreSQL | Azure Database for PostgreSQL |
| DXP DB | Dev Postgres (container) | Amazon RDS for PostgreSQL | Azure Database for PostgreSQL |
| Search | Dev OpenSearch/Elastic (optional) | Amazon OpenSearch / Elastic Cloud (BYO) | OpenSearch/Elastic Cloud (BYO) |
| Doc Library | MinIO (dev) | S3 | Blob Storage |
| Metrics/Alerts | Prometheus + alerting | Managed Prometheus (AMP) | Managed Prometheus (Azure) |
| Dashboards | Grafana OSS | Amazon Managed Grafana | Azure Managed Grafana |
| Logs | **Loki** | **CloudWatch Logs** | **Azure Log Analytics** |
| Backups | Dev logical dumps/versioning | RDS PITR; S3 versioning; OS snapshots | Azure PG PITR; Blob versioning; OS/Elastic snapshots |
| GitOps | Argo CD (AppProject/ApplicationSet) | Same | Same |
| Secrets | K8s Secrets for dev; keep ExternalSecret CRs for parity | ESO + AWS SM | ESO + Azure Key Vault |

> **Parity:** Workflows are the same across columns (PRs, Argo ops, namespace isolation); only the backing services differ.

---

## 5) Versions & Compatibility (suggested MVP targets)

- **Java**: 21 (LTS) · **Spring Boot**: 3.x  
- **PostgreSQL**: 15+  
- **Node**: 20+ · **Next.js**: 15+  
- **Kubernetes**: v1.29+  
- **Argo CD**: v2.9+  
- **Helm**: v3.13+  
- **Kyverno**: v1.10+  
- **ESO**: v0.9+  
- **Prometheus Operator / kube‑prometheus‑stack**: recent stable  
- **Grafana**: Managed (cloud) / recent stable OSS (local)  
- **Loki**: recent stable (local only)  
- **OpenSearch/Elastic**: only if chosen; prefer managed LTS

> Choose LTS/stable channels to reduce drift; pin versions in `platform-policy` repo.

---

## 6) Dependency Policy (Entrenchment)

- Prefer **well‑adopted** OSS and **cloud‑managed** services.  
- Avoid introducing **low‑adoption/experimental** projects.  
- Each new dependency requires: threat model, maintenance plan, and parity assessment (local vs cloud).  
- Remove/replace dependencies that increase blast radius or break parity.

---

## 7) Minimal Footprints (Reference)

### Local (Developer)
- NGINX Ingress, Argo CD, ESO, **Prometheus (+ alerting)**, **Grafana OSS**, **Loki**, dev DNS/certs.  
- Optional for development only: local Postgres, MinIO, OpenSearch/Elastic + Kibana.

### AWS (EKS)
- ALB Ingress or NGINX+ALB, Argo CD, ESO, **Amazon Managed Service for Prometheus**, **Amazon Managed Grafana**, **CloudWatch Logs**, Route53, ACM (or cert‑manager), RDS PostgreSQL, S3.  
- Optional: Amazon OpenSearch (or Elastic Cloud BYO).

### Azure (AKS)
- AGIC or NGINX, Argo CD, ESO, **Azure Managed Prometheus**, **Azure Managed Grafana**, **Azure Log Analytics**, Azure DNS, Key Vault/Front Door (or cert‑manager), Azure Database for PostgreSQL, Blob Storage.  
- Optional: OpenSearch/Elastic (managed/marketplace).

---

## 8) Non‑Goals

- Full CI pipeline orchestration (we surface status only).  
- Granting cluster‑admin or cross‑project write privileges to **Owners/Members**.  
- Displaying plaintext secrets.  
- Requiring Kafka/Fluentd/Logstash for basic operation (optional integrations only).  
- Running production databases/search/file stores inside Kubernetes by default.

---

**One‑liner**: *A lean, portable stack: **Next.js** on the front‑end and **Java/Spring Boot** on the back‑end, with Kubernetes + Argo CD + Helm + ESO + managed Prometheus/Grafana + provider logging planes, and durable data kept outside the cluster—consistent from local to AWS or Azure.*

# Screens — Liferay Cockpit UI

**Status:** Draft (MVP-aligned)  
**Principles:** *GitOps-first • Isolation by default • KISS • Local↔Cloud parity • Kubernetes stateless by default*  
**Core capabilities surfaced:** Isolation · DevOps workflow (dev→uat→prd) · Observability (UI-first via deep links) · Logs (CLI-first) · Backup & Restore (externalized) · **Admin-only provisioning**

> The Cockpit favors **clarity over knobs**. Simple flows get first-class buttons; complex ops remain possible via **PRs** and **CLI**. Owners/Members are fully **abstracted from infrastructure**; **Admins** provision projects.

---

## 0) Entry & Admin Area (MVP)

### 0.1 Login (temporary, regional)
**Route**: `/login`  
**Purpose**: Minimal regional auth for MVP.

**Form**: username, password (`admin/admin` on first boot → **force change**).  
**UI**: region badge (e.g., `aws-us-east-1`), “Sign in” button.  
**States**: invalid creds (generic error), lockout after N attempts, password change flow on first login.  
**Redirects**: authenticated → `/admin` (if Admin) or `/projects` landing.

> No `/signup` before MVP. Multi-region SSO arrives post-MVP (see AUTH_AND_PROJECT_CREATION.md).

### 0.2 Admin Console
**Base**: `/admin`  
**Sections**:  
- **Projects**: list + **New Project** wizard.  
- **Provisioning Jobs**: history/status (short-lived in-cluster jobs).  
- **Health**: Cockpit health status (from `/healthz`), version, region.

### 0.3 Project Wizard (Admin-only)
**Route**: `/admin/projects/new`  
**Fields (minimal)**  
1) **Project ID** (e.g., `retail`)  
2) **Display name** (optional)  
3) **Sizing preset** (S / M / L)  
4) **Environments** (preselected: `dev`, `uat`, `prd`)  
5) **Region** (**fixed to current host** until multi-region auth)  

**Action**: **Create Project**

**What user sees next**: transitions to **Provisioning Progress** (below).

### 0.4 Provisioning Progress (Admin-only)
**Route**: `/admin/projects/<id>/provisioning`  
**Panels**
- **Jobs & Logs**: status of the in-cluster **Provisioner Job**; tail logs with timestamps; retry button (idempotent).  
- **Environment Readiness Matrix** (per env): IaaS binding · DB · File store · Search · ESO freshness · (Promotion) Grafana · Logs.  
- **DNS & Endpoints**: `portal.dev.<project>.<cluster-domain>`, Grafana link, logs plane link (read-only).  
- **Result banner**: “DEV Ready → Deploying latest approved Liferay to DEV…” then “DEV Deployed” with link.

**Failure UX**: single remediation hint + “Retry provisioning” (Admin-only). Owners/Members never see this page.

---

## 1) Navigation Model (Project-scoped)

- **Global shell:** region indicator, project switcher, full-text search, alerts bell, role chip (Admin/Owner/Member), user menu.  
- **URL pattern:** `<cloud>-<region>.<cockpit-domain>/projects/<projectId>`  
- **Environment pill:** Dev / UAT / Prd filter affects all lists/tables.  
- **Status chips:** Argo **Health** (Healthy/Degraded) and **Sync** (Synced/Out-of-Sync).  
- **Action bar:** Diff · Sync Now · Rollback · Open PR (contextual).  
- **Permissions hints:** disabled actions show tooltip (“Requires Owner or Admin”).

**Primary nav (project-scoped):**
1. Overview
2. Deployments
3. Workloads
4. Pods
5. Autoscaling
6. Repos & Config
7. Releases & Promotion
8. Observability
9. Alerts
10. Secrets & Service Bindings
11. Backup & Restore
12. Policies (read-only)
13. Audit & Approvals
14. Settings
15. Liferay DXP Profile

---

## 2) Screens

### 2.1 Overview
**Route**: `/projects/<id>`  
**Purpose**: High-signal health & recent activity; **readiness gating** banner.

**Widgets**
- **Env Health Tiles**: per env status + drift indicator; click → Deployments filtered by env.
- **Readiness Banner**: shows **BindingsReady / PromotionReady** with checks; if not Ready, **Deploy/Promote buttons disabled** with one-liner hint (“Admin must complete provisioning; see Admin → Provisioning”).  
- **Top Issues**: failed syncs, CrashLoops, firing alerts.
- **Recent Activity**: merges, syncs, rollbacks, alerts (24h); drill into details.
- **Commit→Deploy Timeline**: last N deploys w/ SHA, author, outcome.

**Quick actions**
- **Sync All**, **Open Diff**, **Rollback** (per app) — enabled only when **BindingsReady** is green for that env.
- **Open Grafana** (deep link pre-scoped to project/env).
- **Logs Help** → opens a CLI snippet drawer (scoped command templates).

**Empty state**: Connect a repo to get started (→ Repos & Config) — visible even if provisioning is in progress.

---

### 2.2 Deployments (GitOps Apps)
**Route**: `/projects/<id>/deployments` and `/projects/<id>/deployments/<app>`  
**Purpose**: Manage Argo Applications/ApplicationSets.

**List columns**: App · Env · Git SHA · Sync · Health · Desired/Ready/Available · Last Deploy.  
**Detail tabs**: **Resource Tree**, **Diff (desired vs live)**, **History**, **Targets**.

**Actions**: Diff · Sync Now · Rollback · Refresh — gated by **BindingsReady** for the env.  
**Notes**: autosync toggle maps to Argo syncPolicy; bulk sync is available.

---

### 2.3 Workloads
**Route**: `/projects/<id>/workloads`  
**Purpose**: Triage by controllers (Deployment/StatefulSet/Job/CronJob).

**Rows**: Kind · Name · Desired/Ready/Available · Health · Restarts(5m) · HPA chip (min-cur-des-max).  
**Row actions**: View Pods · **Open Logs** (opens provider console or CLI drawer) · Events · Open Diff · Sync (gated by readiness).

> Logs are **CLI-first** and/or opened in the provider logging plane (CloudWatch Logs, Azure Log Analytics, or Loki locally). No custom log viewer is built into Cockpit.

---

### 2.4 Pods
**Route**: `/projects/<id>/pods`  
**Purpose**: Pod-level diagnosis.

**Table**: Pod · Phase · Ready · Restarts · Node · Age (filters: Unready/CrashLoop/Recent restarts).  
**Drawer**: containers, last termination reason, **Events**, **Open Logs** button (provider console or CLI drawer), optional **Exec** (role-gated).

> Owners/Members do **not** get kubectl; any Exec is mediated by the platform service account and fully audited.

---

### 2.5 Autoscaling
**Route**: `/projects/<id>/autoscaling`  
**Purpose**: View/edit HPAs.

**List**: HPA · Target · Min/Max · Current/Desired · Last scale reason/time.  
**Detail**: replicas sparkline · scale events timeline.  
**Action**: **Edit via PR** (schema-aware values editor → preview diff → open PR).

---

### 2.6 Repos & Config
**Route**: `/projects/<id>/repos`  
**Purpose**: Connect Git, map branches/paths to envs, tweak sync policy.

**Panels**
- **Repository Connection**: provider, repo URL, branch, path.  
- **Sync Policy**: automated/prune/self-heal/retries.  
- **Values Editor**: schema-aware editor → **Preview Diff** → **Open PR**.

**Guardrails**: AppProject preview shows allowed repos/destinations.

---

### 2.7 Releases & Promotion
**Route**: `/projects/<id>/releases`  
**Purpose**: Promote revisions across envs with checks.

**Views**
- **Candidate Diff** dev→uat / uat→prd.  
- **Gates**: Healthy, Synced, **PromotionReady** (Grafana+Logs).  
- **History**: past promotions + outcomes.

**Actions**: **Promote** → creates PR; **Rollback** to prior revision. Buttons are disabled until gates pass.

---

### 2.8 Observability
**Route**: `/projects/<id>/observability`  
**Purpose**: Quick status + deep links (UI-first; no custom charts).

**Panels**
- **Status Summary**: basic health and recent alerts.  
- **Open Grafana**: single click to the managed Grafana (or OSS locally) pre-scoped to project/env.

> Cockpit **does not render charts**; it links to managed Grafana per runtime.

---

### 2.9 Alerts
**Route**: `/projects/<id>/alerts`  
**Purpose**: Triage active and recent alerts.

**Table**: Severity · Rule · App · Env · Since · Status.  
**Row actions**: Open Metrics (Grafana), **Open Logs** (provider plane/CLI drawer), Open Diff (if deploy-related).

---

### 2.10 Secrets & Service Bindings (ESO)
**Route**: `/projects/<id>/bindings`  
**Purpose**: Show wiring to DB/Search/Storage/Cache/Queue via ESO.

**Table**: Binding · Provider · Secret Name · Status (Bound/Unbound) · Last Refresh.  
**Actions**: **Rotate** (triggers ESO refresh).  
**Note**: secret **values are never shown**. Infra attachment is **Admin-only** and not visible here.

---

### 2.11 Backup & Restore
**Route**: `/projects/<id>/backups`  
**Purpose**: Manage DB, Document Library, and Search snapshots (externalized).

**Tabs**: **Database**, **Document Library**, **Search Indexes**.  
**Actions**: **Create Backup** (opens PR/plan), **Restore** (select snapshot/PITR → PR).  
**Cloud wiring**: shows target (RDS/OpenSearch/S3 or Azure equivalents); local uses MinIO for development.

**Tables**: Backup ID · Type · Target · Timestamp · Status · Retention · Initiator.  
**Safety**: destructive restores require confirmation + approval per policy.

---

### 2.12 Policies (Read-only)
**Route**: `/projects/<id>/policies`  
**Purpose**: Explain guardrails and violations.

**Sections**: AppProject fences, quotas/limits, NetworkPolicy defaults, PSA posture, allowed registries.  
**Violations**: pre-sync checks and last denied operations with rule messages.

---

### 2.13 Audit & Approvals
**Route**: `/projects/<id>/audit`  
**Purpose**: End-to-end traceability.

**Feed**: who · what · when · where · why/how · result; filters (time, user, action, env, app).  
**Inline**: diffs (redacted), policy rule context, PR/commit links, Argo op details.  
**Approvals**: pending promotions or manual syncs with reason and checklist.  
**Export**: NDJSON/CSV.

---

### 2.14 Settings
**Route**: `/projects/<id>/settings`  
**Purpose**: Project membership, tokens, quotas summary.

**Panels**: Members & roles, project-scoped API tokens, quotas/usage (read-only).  
**Owner-only**: manage members (cannot remove other Owners).

---

### 2.15 Liferay DXP Profile
**Route**: `/projects/<id>/dxp`  
**Purpose**: Opinionated summary for Liferay specifics.

**Sections**: image/tag, module pack, `portal-ext` preview (from ConfigMap), Ingress/TLS status, service bindings status.

---

## 3) UI States & Patterns

- **Gating**: If **BindingsReady** is not green, actions that mutate runtime (Sync/Rollback) or config (Values PR) are disabled; a single remediation hint points to Admin. **Promotion** additionally requires Grafana+Logs green.  
- **Loading**: skeletons for tables/cards; SSE for live updates.  
- **Empty**: “Connect your repo” or “No alerts in the last 24h”.  
- **Error**: toast + inline panel with remediation links (Diff/Logs/Docs).  
- **Permissions**: disabled actions with tooltip; hide controls the persona can never access.  
- **Dangerous ops**: 2-step confirm; audit note required.  
- **Time**: display in user locale; absolute UTC on hover.

---

## 4) Accessibility & i18n

- WCAG 2.2 AA targets; keyboard ops for Diff/Sync; ARIA on chips and tables.  
- Strings externalized; English default; add locales per demand.

---

## 5) Performance Targets

- P95 initial **Overview** load < 2s with cached health/sync summaries.  
- **Deployments** diff render < 1.5s typical for medium trees.  
- SSE updates within 5–10s of Argo status changes.

---

## 6) Acceptance Criteria (UI)

- **Login** exists (regional, temporary) and redirects to Admin console; no signup route.  
- **Admin Project Wizard** creates projects in current region; **Provisioning Progress** shows jobs, readiness, and DNS; DEV auto-deploy begins when Ready.  
- Overview reflects accurate Health/Sync and readiness and top issues within 10s.  
- Deployments can **Diff**, **Sync**, and **Rollback** with audit entries (gated by readiness).  
- Workloads/Pods show correct states and open logs via provider plane or CLI drawer; Exec is role-gated and audited.  
- Autoscaling edits create PRs and show scale events timeline.  
- Repos & Config supports values editing with **Preview Diff** → **Open PR**.  
- Releases & Promotion enforces gates and records outcomes.  
- Observability deep links preserve project/env scoping; no custom charts in Cockpit.  
- Alerts are actionable with working links to Logs/Metrics.  
- Secrets page never shows plaintext; **Rotate** triggers ESO refresh.  
- Backup & Restore creates PRs and shows job/snapshot status.  
- Audit & Approvals provides searchable, exportable trail with correlation IDs.

---

**One-liner**: *A focused cockpit with an Admin-only wizard for project provisioning, readiness-gated app operations for Owners/Members, managed-stack observability links, CLI-first logs, and auditable externalized backups—always within your namespaces.*


# API — Liferay Cockpit Control Plane API

**Status:** Draft (MVP‑aligned)  
**Principles:** GitOps‑first • Isolation by default • **KISS** • Local↔Cloud parity • Kubernetes **stateless by default**  
**Core capabilities:** DevOps workflow (dev → uat → prd), Observability (UI‑first links), Logs (CLI‑first helpers), Backup & Restore (externalized), Audit

---

## 1) Overview

The Liferay Cockpit Control Plane exposes a **project‑scoped** REST API used by the Web UI and the CLI.  
All **spec changes** are made via **PR generation endpoints**, and all **runtime ops** (sync/refresh/rollback) are executed **server‑side** against Argo CD.  
Every request is **RBAC‑checked** and produces an **audit record** with correlation IDs.

**Base URL pattern**
```
https://<cloud>-<region>.<cockpit-domain>/api
# examples
https://aws-us-east-1.myliferaypaas.com/api
https://azure-westus2.partnerpaas.com/api
https://local.localdevelopment/api
```

---

## 2) Auth, RBAC, Scopes

### 2.1 Authentication (MVP)
- **Session-based authentication** with username/password for MVP.
- Single regional **admin** user: `username=admin`, `password=admin` (must be changed on first login).
- Session tokens issued after successful authentication; include session cookie or `Authorization: Bearer <session-token>`.
- **Post-MVP**: Global SSO (OIDC) with regional RBAC will replace temporary admin authentication.

### 2.2 Roles & Scopes
- **Admin** — platform/cluster operations and project bootstrap.
- **Owner** — full control **within the project** (cannot remove other owners).
- **Member** — day‑to‑day operations **within the project**.

**Authorization scopes** (enforced per project):
- `project:read` — read status, alerts, audit.
- `project:operate` — trigger sync/rollback/refresh.
- `project:write` — generate PRs (values/policies/promotion/etc.).
- `project:secrets` — trigger ESO refresh/rotation (no secret values exposed).
- `admin:*` — platform administration (non‑tenant).

> **Isolation:** All endpoints are **namespaced to a project** and resolved to that project’s namespaces. Admission controls ensure only the Argo controller SA or the Cockpit operator SA mutate the cluster.

---

## 3) Conventions

- **Media types**: `application/json`; errors use RFC 7807 `application/problem+json`.
- **Timestamps**: ISO‑8601 UTC (`2025-10-11T02:34:56Z`).
- **Pagination**: `page_size` (default 50, max 200) + `next_page_token`.
- **Idempotency**: For mutating endpoints, send `Idempotency-Key: <uuid>`.
- **Correlation**: Responses include `X-Correlation-Id`; send your own via `X-Request-Id` to override.
- **SSE**: Server‑Sent Events for live streams use `text/event-stream`.

---

## 4) Error Model (RFC 7807)

```json
{
  "type": "https://cockpit.errors/forbidden",
  "title": "Forbidden",
  "status": 403,
  "detail": "User lacks project:operate for this project",
  "instance": "/projects/retailproject/apps/dxp/sync",
  "correlationId": "4f1a6c3b-..."
}
```

Common `type` values: `/validation`, `/forbidden`, `/not-found`, `/conflict`, `/rate-limit`, `/upstream`.

---

## 5) Projects & Environments

### 5.1 Get project summary
`GET /projects/{projectId}/summary`  _(scope: project:read)_

**Response**
```json
{
  "projectId": "retailproject",
  "environments": ["dev","uat","prd"],
  "health": { "dev": "Healthy", "uat": "Progressing", "prd": "Healthy" },
  "sync":   { "dev": "Synced",  "uat": "OutOfSync",  "prd": "Synced"  }
}
```

### 5.2 List environments
`GET /projects/{projectId}/environments`  _(project:read)_

```json
{ "items": [{"name":"dev"},{"name":"uat"},{"name":"prd"}] }
```

---

## 6) GitOps Apps (Deployments)

### 6.1 List applications
`GET /projects/{projectId}/apps`  _(project:read)_

Query: `env=dev|uat|prd`, `page_size`, `next_page_token`

**Response**
```json
{
  "items": [{"name":"dxp","env":"dev","gitSha":"abc...","health":"Healthy","sync":"Synced"}],
  "next_page_token": null
}
```

### 6.2 Application status
`GET /projects/{projectId}/apps/{app}/status?env={env}`  _(project:read)_

### 6.3 Resource tree
`GET /projects/{projectId}/apps/{app}/tree?env={env}`  _(project:read)_

### 6.4 Diff (desired vs live)
`GET /projects/{projectId}/apps/{app}/diff?env={env}`  _(project:read)_

> Secrets are redacted.

### 6.5 Sync (runtime op)
`POST /projects/{projectId}/apps/{app}/sync`  _(project:operate)_
```json
{ "env":"uat","targetRevision":"main@sha1:abc123" }
```

### 6.6 Rollback (runtime op)
`POST /projects/{projectId}/apps/{app}/rollback`  _(project:operate)_
```json
{ "env":"uat","toRevision":"main@sha1:def456" }
```

### 6.7 Refresh (runtime op)
`POST /projects/{projectId}/apps/{app}/refresh`  _(project:operate)_
```json
{ "env":"uat","hard":true }
```

**Responses for 6.5–6.7**
```json
{ "operationId":"op_7K9...", "status":"queued" }
```
Use **Operations** (section 12) or SSE to follow progress.

---

## 7) PR Generators (Spec Changes)

All generators **create a Git branch and PR** in the connected repo(s).

### 7.1 Values edit PR
`POST /projects/{projectId}/values/pr`  _(project:write)_
```json
{
  "app":"dxp",
  "env":"prd",
  "changes": {
    "image.tag":"7.4.13-gaX",
    "resources.requests.cpu":"500m"
  },
  "title":"DXP bump + CPU request",
  "description":"..."
}
```

### 7.2 Promotion PR (dev → uat → prd)
`POST /projects/{projectId}/promotion/pr`  _(project:write)_
```json
{ "app":"dxp","fromEnv":"dev","toEnv":"uat","strategy":"copy-values" }
```

### 7.3 HPA PR
`POST /projects/{projectId}/hpa/pr`  _(project:write)_
```json
{ "app":"web","env":"prd","min":3,"max":10,"targets":{"cpu":70} }
```

### 7.4 NetworkPolicy PR
`POST /projects/{projectId}/networkpolicy/pr`  _(project:write)_
```json
{ "env":"prd","allowEgress":[{"host":"postgres.prd.internal","port":5432,"protocol":"TCP"}] }
```

### 7.5 Service Binding PR (ESO)
`POST /projects/{projectId}/bindings/pr`  _(project:write, project:secrets)_
```json
{ "env":"uat","binding":"db","provider":"aws-sm","secretRef":"arn:aws:secretsmanager:...:secret:dxp-uat-db" }
```

**Response (7.1–7.5)**
```json
{ "prUrl":"https://git.example.com/org/repo/pull/123", "branch":"cockpit/PR-123", "commit":"abc..." }
```

---

## 8) Observability, Alerts & Logs

### 8.1 Project alerts (current + recent)
`GET /projects/{projectId}/alerts?env={env}&since=24h`  _(project:read)_

### 8.2 Grafana link (UI‑first)
`GET /projects/{projectId}/observability/grafana-link?app={app}&env={env}`  _(project:read)_

**Response**
```json
{ "url":"https://grafana.../d/liferay?var_project=retailproject&var_env=prd&var_app=dxp" }
```

### 8.3 Logs — provider console link (CLI‑first strategy)
`GET /projects/{projectId}/logs/console-link?env={env}&app={app}`  _(project:read)_

**Response**
```json
{ "url":"https://console.aws.amazon.com/cloudwatch/home#logsV2:log-groups/log-group/$PROJECT_ENV/..." }
```

### 8.4 Logs — CLI template (helper)
`GET /projects/{projectId}/logs/cli-template?env={env}&level=error&match={q}`  _(project:read)_

**Response**
```json
{
  "tool":"aws|az|logcli",
  "command":"aws logs tail /cockpit/retailproject/prd --since 1h --filter-pattern '"ERROR"'"
}
```

> Cockpit does not implement a log viewer; it provides **links/templates** to provider consoles/CLIs with the right scope.

---

## 9) Backup & Restore (Externalized)

### 9.1 Create backup PR
`POST /projects/{projectId}/backups/pr`  _(project:write)_
```json
{ "env":"prd","unit":"db|dl|search","retention":"7d","target":"s3://dxp-prd-dl","notes":"nightly" }
```

### 9.2 List backups
`GET /projects/{projectId}/backups?env={env}&unit=db|dl|search&page_size=50&next_page_token=...`  _(project:read)_

```json
{
  "items":[
    {
      "backupId":"bkp_2025-09-01T12:00:00Z",
      "unit":"db",
      "env":"prd",
      "target":"rds-snapshot:dxp-prd-...",
      "status":"Succeeded",
      "startedAt":"2025-09-01T12:00:00Z",
      "completedAt":"2025-09-01T12:05:20Z",
      "initiator":"alice@example.com"
    }
  ],
  "next_page_token": null
}
```

### 9.3 Restore PR
`POST /projects/{projectId}/restores/pr`  _(project:write)_
```json
{ "env":"prd","unit":"db","pointInTime":"2025-09-01T12:00:00Z","source":"rds-snapshot:dxp-prd-..." }
```

### 9.4 Get backup/restore status
`GET /projects/{projectId}/backups/{backupId}`  _(project:read)_  
`GET /projects/{projectId}/restores/{restoreId}`  _(project:read)_

---

## 10) Secrets (Operations Only)

### 10.1 Trigger ESO refresh
`POST /projects/{projectId}/secrets/refresh`  _(project:secrets)_
```json
{ "env":"prd","binding":"db" }
```

> The platform never returns secret **values**.

---

## 11) Audit & Approvals

### 11.1 Audit feed
`GET /projects/{projectId}/audit?from=2025-10-01T00:00:00Z&to=2025-10-11T00:00:00Z&user=&action=&env=&result=&page_size=100`  _(project:read)_

### 11.2 Audit record
`GET /audit/{auditId}`  _(project:read)_

### 11.3 Live stream (SSE)
`GET /projects/{projectId}/audit/stream`  _(project:read)_  
Events: `audit`, `op-status`, `alert`

### 11.4 Approvals
`GET /projects/{projectId}/approvals`  _(project:read)_  
`POST /projects/{projectId}/approvals/{approvalId}/approve`  _(project:write)_  
`POST /projects/{projectId}/approvals/{approvalId}/reject`  _(project:write)_

---

## 12) Operations (Long‑running)

Some actions (sync/rollback, backup/restore) return an **operation** handle.

### 12.1 Get operation
`GET /operations/{operationId}`  _(project:read)_
```json
{ "operationId":"op_7K9...", "status":"running|succeeded|failed", "percent":70,
  "startedAt":"...", "updatedAt":"...", "links":{"auditId":"aud_...","app":"dxp"} }
```

### 12.2 Stream operation (SSE)
`GET /operations/{operationId}/stream`

---

## 13) Repositories & Sync Policy

### 13.1 Connect repository
`POST /projects/{projectId}/repo/connect`  _(project:write)_
```json
{ "provider":"github","url":"https://github.com/org/repo","branch":"main","path":"apps/liferay-dxp" }
```

### 13.2 Update sync policy
`PATCH /projects/{projectId}/repo/sync-policy`  _(project:write)_
```json
{ "automated":true,"prune":true,"selfHeal":true,"retry":{"limit":3} }
```

---

## 14) Rate Limits

- **Default**: 600 requests/min per token; bursts allowed via token bucket.  
- `429 Too Many Requests` includes `Retry-After` and `X-RateLimit-*` headers.

---

## 15) Security Notes

- No endpoint returns plaintext secret values.  
- Admission policies ensure only Argo/Cockpit SAs can mutate the API server.  
- All responses include `X-Correlation-Id`; audits store request/response envelopes (redacted).

---

## 16) Examples (curl)

**Sync an app to prd**
```bash
curl -X POST "https://aws-us-east-1.myliferaypaas.com/api/projects/retailproject/apps/dxp/sync"   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json"   -H "Idempotency-Key: $(uuidgen)"   -d '{"env":"prd","targetRevision":"main@sha1:abc123"}'
```

**Open a values PR**
```bash
curl -X POST "https://aws-us-east-1.myliferaypaas.com/api/projects/retailproject/values/pr"   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json"   -d '{"app":"dxp","env":"uat","changes":{"image.tag":"7.4.13-gaX"},"title":"DXP tag bump"}'
```

**Create a DB backup PR**
```bash
curl -X POST "https://aws-us-east-1.myliferaypaas.com/api/projects/retailproject/backups/pr"   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json"   -d '{"env":"prd","unit":"db","retention":"7d"}'
```

**Get audit feed (last 24h)**
```bash
curl "https://aws-us-east-1.myliferaypaas.com/api/projects/retailproject/audit?since=24h"   -H "Authorization: Bearer $TOKEN"
```

---

## 17) Versioning

- Semantic versioning for the API: `v1` path when stabilized (`/api/v1/...`).  
- Breaking changes require a new version; old versions maintained per deprecation policy.

---

## 18) OpenAPI

An **OpenAPI 3.1** document is published at:  
`GET /openapi.json` (project‑agnostic parts) and `GET /projects/{projectId}/openapi.json` (project‑scoped subset).

---

### Future improvement (not in MVP)
- Optional **SIEM export** endpoints may be added later if justified; avoid mentioning/elaborating elsewhere to keep focus.

---

**One‑liner**: *A compact, project‑scoped API that turns PRs into deployments, links to the right observability view, provides CLI/log links, and enforces hard namespace isolation throughout.*


# Auth and Project Creation — Authentication & Project Creation

**Status:** Draft (MVP-aligned)  
**Principles:** *KISS • Isolation by default • GitOps-first • Kubernetes **stateless by default** • Local↔Cloud parity*

---

## 1) Purpose
Define how **authentication** and **project creation** work **before MVP** (temporary, regional login) and what the **post‑MVP** direction will be (global SSO with regional RBAC). Keep Owners/Members **fully abstracted from infrastructure**; only **Admins** create projects.

> **Entrenchment:** Infra Abstraction & Operator‑Only Provisioning — Owners/Members never configure IaaS/PaaS. Admins provision projects; Cockpit gates features until dependencies are **Ready** (DB, filestore, search, ESO, metrics, logs).

---

## 2) URL Pattern (recap)
Project UI lives at:
```
<cloud>-<region>.<cockpit-domain>/projects/[projectId]
```
Examples:  
`aws-us-east-1.myliferaypaas.com/projects/retail` · `azure-westus2.partnerpaas.com/projects/intranet` · `local.localdev/projects/checkout`

Admin console lives under the same host:  
`<cloud>-<region>.<cockpit-domain>/admin/...`

---

## 3) MVP Authentication (Temporary, Regional)
**Goal:** unblock development with the simplest possible login while we implement proper multi‑region SSO.

- **Single regional admin user only**: `username=admin`, `password=admin` **(must be changed on first login)**.  
- **No signup** (self‑registration) **before MVP**. No invites. Only the admin can access the console.  
- **Where credentials live:** Cockpit’s **management DB** (PostgreSQL) that is dedicated to Cockpit and provisioned by infra.  
- **Storage**: password stored as a salted hash; never plaintext.  
- **Scope:** authentication is **per region**. Local deployments use the same mechanism.  
- **TLS required**: all logins over HTTPS.  
- **Future**: replaced by **Global SSO + Regional RBAC** (section 7).

**Login page**  
- **Route**: `/login` (redirects to `/admin` if already authenticated).  
- **Fields**: username, password.  
- **States**: invalid creds, locked after N attempts (configurable), “change password” on first login.  
- **Branding**: region indicator (e.g., `aws-us-east-1`).

---

## 4) Admin Console & Project Wizard (MVP)
**Admin-only UI** to create projects. Owners/Members do not see this area.

- **Routes**:  
  - `/admin` — summary & quick links  
  - `/admin/projects` — list  
  - `/admin/projects/new` — wizard

**Wizard fields (minimal by design)**  
1) **Project ID** (e.g., `retail`)  
2) **Display name** (optional)  
3) **Sizing preset** (S / M / L)  
4) **Environments** (default: `dev`, `uat`, `prd` — preselected)  
5) **Region** — **fixed to current host’s region** until multi‑region auth is available

**What happens on “Create Project”**  
- Cockpit creates `proj-<id>-dev|uat|prd` namespaces, RBAC, quotas/limits, default‑deny NetworkPolicy, PSA posture.  
- Triggers the **in‑cluster provisioner** (admin‑only) to set up managed services per policy:
  - **Prd**: dedicated PostgreSQL, dedicated object storage, dedicated search.
  - **Dev/UAT**: shared PostgreSQL server with **per‑env schema**, shared bucket with prefixes, shared search domain with index prefixes.  
- Secrets are written to the cloud vault and **referenced** via **External Secrets Operator** (ESO). Values are never shown in Cockpit.  
- Cockpit evaluates **Readiness** (see section 5).  
- When **DEV** becomes `Ready`, Cockpit **auto‑deploys** the latest approved Liferay to DEV using Helm/Argo and exposes the default ingress host.

**Post‑create landing**  
- Redirect to: `<cloud>-<region>.<cockpit-domain>/projects/[projectId]` (Overview).  
- All app‑level controls remain **disabled** until Readiness is green.

---

## 5) Readiness & Feature Gating (per environment)
Cockpit holds and displays readiness state per env. Until **all** checks are green, Cockpit disables dependent features (Deploy/Sync/Promote).

**BindingsReady** (deploy gate)  
- **IaaS service account** (workload identity) installed and assumable.  
- **Database** reachable; credentials valid; (dev/uat) schema exists.  
- **File store** reachable; **write test** passes.  
- **Search** reachable; index/template test passes.  
- **Secrets sync** healthy (ESO bindings **Bound**, last refresh < SLA).

**PromotionReady** (promote gate)  
- **Metrics** data source healthy; **Grafana link** works.  
- **Logs** provider plane reachable; a scoped test query returns data.

> Owners/Members never configure infra. Admins handle provisioning; Cockpit performs the probes and gates the buttons.

---

## 6) Personas & Access (MVP)
- **Admin**: only authenticated user type in MVP. Can access `/admin`, run the Project Wizard, and view project UIs.  
- **Owner / Member**: **no login flow in MVP**. Their UIs exist but are used by the Admin for dry‑runs. Post‑MVP they log in via SSO.

**kubectl policy**  
- Admin retains **audited break‑glass** kubectl; normal operations flow through GitOps/Cockpit.  
- Owners/Members (post‑MVP) will have **no kubectl** access.

---

## 7) Post‑MVP Direction — Global SSO, Regional RBAC (Preview)
- **Authentication**: one **external IdP** (Okta/Azure AD/Keycloak) for all regions (OIDC).  
- **Regional RBAC**: each region maps IdP **groups → roles** (Admin/Owner/Member).  
- **JIT users**: region DBs store profiles/role bindings only; no passwords.  
- **Seamless UX**: visit any regional Cockpit URL → SSO; Admins use the same Project Wizard with region selected.  
- **CLI**: OIDC device/PKCE login per region.  
- **Security**: no long‑lived cloud keys; per‑project/env workload identities.

*(This section is informational; not part of MVP.)*

---

## 8) Security Notes (temporary auth)
- Force **password change** on first admin login; store salted hash.  
- Enforce **HTTPS**; set strict cookies; CSRF protection for form posts.  
- Rate‑limit `/login`; show generic error messages.  
- Expose `/healthz` and `/readyz` for Terraform/bootstrapping checks only (no user data).  
- This temporary auth is for **development/bootstrap only**; replace with SSO before production.

---

## 9) Acceptance Criteria (MVP)
- `/login` exists; `admin/admin` works **only until first forced password change**.  
- `/admin/projects/new` creates a project in the **current region**.  
- Namespaces, RBAC, quotas, and policies are created; ESO bindings reference provider secrets.  
- Readiness gates behave as specified; **DEV** auto‑deploys Liferay when ready.  
- Project Overview reachable at `<cloud>-<region>.<cockpit-domain>/projects/[projectId]`.  
- No `/signup` route exists; non‑admin access is blocked.

---

## 10) Non‑Goals (MVP)
- Multi‑region SSO & group mappings.  
- User invitations and self‑service signup.  
- Password reset flows.  
- Cross‑region project creation from a single session.

---

**One‑liner:** *Before MVP, authenticate with a single regional admin and create projects via a tiny Admin‑only wizard; Cockpit gates features on readiness and keeps Kubernetes **stateless by default**. After MVP, move to global SSO with regional RBAC and the same simple project creation flow.*

# CLI — LC2 Command-Line Interface

**Status:** Draft (MVP-aligned)  
**Principles:** *GitOps-first • Isolation by default • KISS • Local↔Cloud parity • Kubernetes stateless by default*

> **kubectl policy (standard text):** **Owners & Members** do **not** use `kubectl` (including `kubectl logs`). **Admins** may use `kubectl` for **break‑glass only** (audited). All normal operations flow through **Git PRs** and **Cockpit‑mediated Argo ops**.

---

## 1) Purpose & Scope
The LC2 CLI ("`lc2`") lets **Owners** and **Members** operate their **project**/**environment** safely without direct cluster access. It's project‑scoped, generates **PRs** for spec changes, and triggers **server‑side** Argo operations for runtime tasks. It also brokers provider integrations (logs, observability links) with correct scoping.

**Works the same** on **Local**, **EKS**, and **AKS**. Backends differ, not the workflow.

**Architecture Boundary**
- LC2 CLI talks **only to the Cockpit API**; it **never** hits Kubernetes, Loki, or cloud logging APIs directly.
- All operations are executed **server-side** by the Cockpit control plane and fully **audited**.
- Logs, observability links, and cluster state are brokered through the Cockpit API with strict project/environment scoping.

---

## 2) Install & Auth

### 2.1 Install
- macOS: `brew install lc2` *(placeholder)*  
- Linux: tarball / deb/rpm *(placeholder)*  
- Windows: winget / MSI *(placeholder)*

### 2.2 Login
```bash
lc2 login --region aws-us-east-1
# Opens login prompt (username/password for MVP; browser SSO post-MVP)
# Enter: username=admin, password=admin (change on first login)
# Stores a session token
```

### 2.3 Select project (scope everything)
```bash
lc2 use project retailproject    # sets default --project
lc2 use env prd                  # optional default --env
```

**Config file**: `~/.config/cockpit/config.yaml`
**Tokens**: `~/.config/cockpit/tokens.json` (session tokens; automatic refresh)

---

## 3) Conventions & Safety Rails
- Every command enforces `--project` and `--env` (from context or flags).  
- **Spec changes → PRs**. **Runtime ops → Cockpit API → Argo** (server‑side).  
- Outputs include **correlation IDs**; all actions are **audited**.  
- **Exit codes**: `0` success · `2` validation/permission · `3` provider/back‑end error · `4` rate‑limited.

Global flags:
```
--project <id>   Project scope (required if not set by `use project`)
--env <dev|uat|prd>   Environment scope
--output <text|json>   Default: text
--yes / --confirm      Confirm destructive ops
--request-id <uuid>    Set correlation id
```

---

## 4) Quick Start (common flow)
```bash
lc2 login --region aws-us-east-1
lc2 use project retailproject
lc2 env list

# Edit values (opens PR) and deploy
lc2 values edit --app dxp --env dev
lc2 apps sync dxp --env dev
lc2 promote --app dxp --from dev --to uat

# Triage
lc2 open grafana --app dxp --env uat
lc2 logs tail --env uat --app web --level error --since 1h
```

---

## 5) Projects & Environments
```
lc2 project list
lc2 env list [--project <id>]
```

---

## 6) Deployments (GitOps Apps)
Status, diffs, rollouts—backed by Argo.

```
lc2 apps list [--env <env>]
lc2 apps status <app> --env <env>
lc2 apps diff   <app> --env <env>
lc2 apps sync   <app> --env <env>
lc2 apps rollback <app> --env <env> --to <git-sha|tag>
lc2 apps refresh  <app> --env <env> [--hard]
```

**Promotion (dev → uat → prd)**
```
lc2 promote --app <app> --from <env> --to <env>
# creates a PR; gates: Healthy, Synced, No firing alerts
```

**Values (schema‑aware editor → PR)**
```
lc2 values edit --app <app> --env <env>
```

---

## 7) Workloads, Pods & Autoscaling
Triage and safe adjustments (HPA via PR).

```
# Workloads & Pods
lc2 workloads list --env <env>
lc2 pods list --env <env> [--filter crashloop|unready]

# Autoscaling (creates PR)
lc2 hpa edit <workload> --env <env> --min 2 --max 6 --cpu 70
```

> Manual scaling by replicas is discouraged—prefer HPA via PR.

---

## 8) Networking (Ingress, Load Balancing, URL Routing & TLS)
Create PRs to expose services and manage TLS with consistent patterns across all runtimes.

```
lc2 domain add <host> --env <env> --service <svc:port>
lc2 tls request --dns <host> --env <env>
lc2 net allow egress <host:port> --env <env>   # NetworkPolicy PR
```

**Key Points**
- Cloud‑agnostic with local↔cloud parity
- Controllers: Local (Ingress‑NGINX), AWS (ALB), Azure (AGC/AGIC), GCP (GKE Gateway)
- Cockpit: Stateless sessions, WebSocket/SSE support, 120-300s timeouts
- Liferay: Sticky sessions (PRD), canary/blue‑green rollouts, 60-120s timeouts
- TLS at edge (provider‑managed on cloud, cert‑manager on local)
- WAF required for Cockpit public and Liferay PRD public entries
- Multi‑AZ for PRD; single‑AZ for DEV/UAT

Cloud DNS/certs are preferred on EKS/AKS/GKE; ExternalDNS/cert‑manager are optional.

---

## 9) Secrets & Service Bindings (ESO)
Bindings to DB/Search/Storage/Cache are defined via ExternalSecret manifests (values never shown).

```
lc2 secrets status   --env <env>
lc2 secrets refresh  --env <env> --name <binding>
lc2 binding set db --env <env> --engine postgres --secret-ref aws-sm://dxp/uat/db
```

**DXP Licensing**
- Cluster mode (replicaCount > 1 or HPA enabled) requires a valid DXP cluster license (XML)
- Production: Use Kubernetes Secret or External Secrets Operator with cloud secret manager
- Local/Dev: ConfigMap acceptable for testing only
- License mounted at `/etc/liferay/mount/files/deploy/license.xml`
- Environment flag: `LIFERAY_DISABLE_TRIAL_LICENSE=true` (mandatory)
- Per-environment secrets: `liferay-license-{dev|uat|prd}` for independent rotation

```bash
# Refresh license secret (triggers rolling restart)
lc2 secrets refresh --env prd --name liferay-license

# Verify license configuration
lc2 pods list --env prd --app dxp
```

---

## 10) Observability (UI‑first)
Open the correct Grafana for your project/env; Cockpit does **not** render charts.

```
lc2 open grafana --env <env> [--app <app>]
lc2 alerts list    --env <env>
```

Backends by runtime:  
- **EKS:** Managed Prometheus + Managed Grafana  
- **AKS:** Azure Managed Prometheus + Azure Managed Grafana  
- **Local:** Prometheus OSS + Grafana OSS

---

## 11) Logs (CLI‑first)
Tail, query, and export logs **without** exposing provider creds directly. The CLI brokers calls to the right backend:

- **EKS:** CloudWatch Logs  
- **AKS:** Azure Log Analytics (KQL)  
- **Local:** Loki (via HTTP or `logcli`)

```
# Live tail
lc2 logs tail --env <env> [--app <name>] [--level <info|warn|error>] [--since 1h]

# Search
lc2 logs query --env <env> --match "payment failed" [--regex] [--since 2h]

# Export (time/size‑capped)
lc2 logs export --env <env> --since 24h --out logs.ndjson
```

Examples:
```
# EKS: errors from "web" in last hour
lc2 logs tail --env prd --app web --level error --since 1h

# AKS: look for timeouts in UAT
lc2 logs query --env uat --match "timeout" --since 2h
```

---

## 12) Backup & Restore (Externalized)
Databases use provider **PITR**; Document Library uses object storage **versioning**; Search uses **snapshots**/**reindex**. No PVCs by default.

```
# Scheduling
lc2 backup schedule set --env prd --time "02:15" --retain 30d

# On‑demand backups
lc2 backup create db      --env uat
lc2 backup create dl      --env prd --bucket s3://proj-prd-dl-backups
lc2 backup create search  --env prd --repo opensearch-snapshots

# Discover restore points
lc2 backup list           --env prd [--unit db|dl|search] [--since 7d]

# Restores
lc2 backup restore db     --env prd --at "2025-09-01T12:00:00Z" --to-new proj-prd-pitr --confirm
lc2 backup restore dl     --env prd --snapshot dl-2025-09-01 --confirm
lc2 backup restore search --env prd --snapshot snap-2025-09-01 --confirm
```

> **Owner/Member** can perform **safe** restores into a **new** env/namespace within their project. **Admins** are required for **in‑place** or cross‑project/cluster restores.

---

## 13) Policies & Guardrails
Read current fences and defaults; changes go via PR to the platform policy repo.

```
lc2 policy show --project <id>
```

---

## 14) Audit & Approvals
Searchable audit and simple approvals for gated actions.

```
lc2 audit list --since 24h [--user <email>] [--env <env>]
lc2 audit stream
lc2 approvals list
lc2 approvals approve <id>   # role‑gated
```

---

## 15) Errors & Troubleshooting
- **403**: missing scope or out‑of‑project access; check `--project/--env` and role.  
- **409**: policy/admission denial; run `lc2 apps diff` and fix via PR.  
- **429**: rate‑limited; retry after `Retry‑After`.  
- **Upstream**: provider backend down (Grafana/CloudWatch/Log Analytics); retry and/or use console link.

**Diagnostics**
```
lc2 doctor        # checks auth, project/env, backend reachability
lc2 config show   # prints current context (no secrets)
```

---

## 16) Examples by Runtime

### 16.1 Local
```
lc2 login --region local
lc2 use project myproj && cockpit use env dev
lc2 open grafana --env dev
lc2 logs tail --env dev --app web
```

### 16.2 EKS
```
lc2 login --region aws-us-east-1
lc2 use project retail && cockpit use env prd
lc2 promote --app dxp --from uat --to prod
lc2 logs query --env prd --match "ERROR" --since 1h
```

### 16.3 AKS
```
lc2 login --region azure-westus2
lc2 use project intranet && cockpit use env uat
lc2 alerts list --env uat
lc2 backup restore db --env uat --at "2025-10-11T13:45:00Z" --to-new intranet-uat-pitr --confirm
```

---

## 17) Command Reference (alphabetical)

```
apps        list | status | diff | sync | rollback | refresh
approvals   list | approve | reject
audit       list | stream
backup      schedule set | create (db|dl|search) | list | restore (db|dl|search)
binding     set
config      show
domain      add
env         list
hpa         edit
logs        tail | query | export
net         allow (egress)
open        grafana
pods        list
policy      show
promote     (dev→uat→prd)
project     list
secrets     status | refresh
use         project | env
values      edit
```

---

## 18) Non‑Goals (CLI)
- Exposing plaintext secrets.  
- Running arbitrary kubectl.  
- Building a custom log viewer (we link/use provider planes).  
- Managing cloud resources beyond what’s needed for scoping and deep links.

---

**One‑liner:** *Operate everything through safe, auditable commands: PRs for spec, Argo for runtime, deep links for observability, CLI for logs, and provider‑grade backups—identically on Local, EKS, and AKS.*

# Local Dev Model — Liferay Cockpit Local Development Model

**Status:** Draft (MVP-aligned)  
**Principles:** *KISS • GitOps-first • Isolation by default • Kubernetes **stateless by default** • Local↔Cloud parity • Infra abstraction (Admin-only)*

---

## 1) Purpose
Provide a **minimum-effort local workflow** that still respects our production principles. One command should create a working Cockpit control plane with GitOps guardrails; optional flags enable deeper parity (observability, sample Liferay stack) only when we need it.

---

## 2) Local topology at a glance
- **Base install (always on):** Cockpit UI/API + management Postgres in namespace `liferay-cockpit`, External Secrets Operator with a local SecretStore, NGINX ingress, Kyverno GitOps guardrail, and the supporting ConfigMaps/Secrets created via GitOps templates.
- **Optional extras:** Local Liferay data services (PostgreSQL, MinIO, OpenSearch), Argo CD for reconciliation testing, observability stack (Prometheus/Grafana OSS + Loki), and demo project/provisioner flows toggled per flag.

> **Stateless by default:** project data uses externalized services even locally; PVCs stay hidden behind explicit flags for dev-only experimentation.

---

## 3) URLs & DNS (local)
- Default domain: `*.localdev.me`. Fallback: `*.127.0.0.1.nip.io`.
- Cockpit entrypoint: `https://cockpit.localdev.me`.
- Project URLs (when extras are enabled): `https://portal.dev.<project>.localdev.me`.
- TLS is optional. The `--tls` flag currently checks for `mkcert` and prepares the workspace for upcoming cert automation; use HTTP by default.

---

## 4) Local Infrastructure Bootstrap with devctl

### Purpose

**IMPORTANT DISTINCTION**:
- **PLAN.md** = Implementation plan for **developing the Cockpit product** (API, UI, project provisioning features)
- **devctl** = Tool for **bootstrapping the local Kubernetes infrastructure** where Cockpit will be deployed

The `devctl` script automates the provisioning of a local k3d cluster with all foundational infrastructure components required for Cockpit development. This includes Kubernetes itself, ingress controllers, secret management, admission policies, and the base Cockpit application.

### Prerequisites

**Required Tools**:
- Docker v24+ (or Colima/Rancher Desktop)
- k3d v5.6+
- kubectl v1.28+
- helm v3.12+
- jq v1.6+
- Git v2.40+

**Optional Tools**:
- mkcert (for TLS support - future enhancement)
- logcli (for Loki CLI access)

**Environment Variables** (optional overrides):
```bash
export CLUSTER_NAME=cockpit-local        # k3d cluster name
export DEV_DOMAIN=localdev.me           # Ingress domain (resolves to 127.0.0.1)
export NAMESPACE_COCKPIT=liferay-cockpit
export NAMESPACE_PLATFORM=platform
export COCKPIT_CHART=./charts/cockpit
```

### devctl Commands

```bash
# Bootstrap complete local infrastructure stack
./scripts/local/devctl up

# Bootstrap with optional extras
./scripts/local/devctl up --extras=observability,argocd,liferay-sample

# Check cluster and component health status
./scripts/local/devctl status

# Run smoke tests to validate Cockpit functionality
./scripts/local/devctl smoke

# Remove Cockpit workloads (keep cluster running)
./scripts/local/devctl down

# Delete entire cluster
./scripts/local/devctl destroy

# Delete cluster and cached assets (TLS certs, etc.)
./scripts/local/devctl destroy --all
```

### What devctl Provisions

When you run `./scripts/local/devctl up`, the following infrastructure is automatically provisioned:

1. **k3d cluster** (cockpit-local)
   - Single control-plane node
   - LoadBalancer port mappings (80→30080, 443→30443)
   - Traefik disabled (using NGINX Ingress instead)

2. **NGINX Ingress Controller**
   - HTTP/HTTPS routing for local services
   - LoadBalancer service type
   - Admission webhook configured

3. **External Secrets Operator (ESO)**
   - Installed in external-secrets-system namespace
   - SecretStore configured with Kubernetes backend (local dev)
   - Ready for cloud provider integration (AWS Secrets Manager, etc.)

4. **Kyverno Policy Engine**
   - Installed in kyverno namespace
   - GitOps-only mutation policy enforced
   - Blocks direct kubectl mutations (except from approved ServiceAccounts)

5. **PostgreSQL for Cockpit**
   - Installed in platform namespace (cockpit-db-postgresql)
   - Database schema initialized (users, projects, project_environments, audit_logs)
   - Default admin user seeded (username: admin, password: admin)

6. **Cockpit Application**
   - API and UI deployed in liferay-cockpit namespace
   - Connected to PostgreSQL database
   - Accessible at http://cockpit.localdev.me

### Optional Add-ons (--extras flag)

| Flag | Description |
|------|-------------|
| `observability` | Installs Prometheus, Grafana, and Loki in observability namespace. Enables metrics and logging deep links in Cockpit UI. |
| `argocd` | Installs Argo CD in argocd namespace. Enables GitOps-based deployments and provides foundation for Phase 1 GitOps workflow. |
| `liferay-sample` | Installs PostgreSQL and MinIO in liferay-dev namespace for Liferay DXP development (OpenSearch coming in later phase). |
| `demo-project` | Creates demo project namespace (proj-demo-dev) with placeholder ConfigMap to simulate GitOps-managed project. |

**Example with extras**:
```bash
./scripts/local/devctl up --extras=observability,argocd
```

### Typical Development Workflow

```bash
# 1. Bootstrap local infrastructure
./scripts/local/devctl up --extras=argocd

# 2. Access Cockpit and change default password
# Open http://cockpit.localdev.me in browser
# Login: admin / admin (change on first login)

# 3. Set up Git repository for GitOps (see PLAN.md Phase 1, Task 1.2)
cd /path/to/parent
git clone https://github.com/yourorg/liferay-cockpit-gitops.git
cd liferay-cockpit-gitops
# ... create directory structure per PLAN.md ...

# 4. Configure Argo CD to watch repository (see PLAN.md Phase 1, Task 1.2)
kubectl apply -f repository-secret.yaml

# 5. Follow PLAN.md Phase 1+ for Cockpit development
# - Implement Cockpit API features (Phase 2)
# - Implement Cockpit UI features (Phase 5)
# - Implement project provisioning logic (Phase 3)
# - etc.

# 6. Validate infrastructure health
./scripts/local/devctl status

# 7. Run smoke tests
./scripts/local/devctl smoke

# 8. Teardown when done
./scripts/local/devctl destroy
```

### DNS Configuration

By default, `localdev.me` and `*.localdev.me` resolve to 127.0.0.1 via public DNS. No `/etc/hosts` modification needed.

For custom domains, add to `/etc/hosts`:
```
127.0.0.1 cockpit.local
127.0.0.1 retail-dev.local
```

### Access Points After Bootstrap

Once `devctl up` completes successfully:

- **Cockpit UI**: http://cockpit.localdev.me
- **Cockpit API**: http://cockpit.localdev.me/api
- **Argo CD UI** (if --extras=argocd): Port-forward via `kubectl port-forward svc/argocd-server -n argocd 8080:443`, then visit https://localhost:8080
- **Grafana** (if --extras=observability): Port-forward via `kubectl port-forward svc/cockpit-monitoring-grafana -n observability 3000:80`, then visit http://localhost:3000

**Default Credentials**:
- Cockpit: admin / admin (change on first login)
- Argo CD: admin / (retrieve via `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`)
- Grafana: admin / prom-operator

### GitOps Exception for Bootstrap

**IMPORTANT**: `devctl up` uses **direct Helm and kubectl commands** to provision infrastructure. This is a **documented exception** to the GitOps-Only principle (Entrenchment Clause #1).

**Rationale**:
- Local cluster bootstrap requires chicken-and-egg setup (Git repos, Argo CD, base infrastructure must exist before GitOps can work)
- Automation reduces human error and ensures consistent local environments
- Matches realistic production pattern (cluster init via Terraform/IaC, operations via GitOps)

**After Bootstrap**:
Once the infrastructure is operational, **ALL subsequent changes flow through Git → Argo CD**. This includes:
- Platform policy updates
- Project provisioning
- Application deployments
- Configuration changes

Kyverno admission control enforces this by blocking direct kubectl mutations (except from approved ServiceAccounts: Argo CD, Cockpit Operator, Kyverno, ESO).

### Relationship to PLAN.md

PLAN.md defines the **implementation plan for developing the Cockpit product**. The phases are:

- **Phase 1**: Foundation - Validate infrastructure and establish GitOps operational patterns
- **Phase 2**: Cockpit API - Implement backend services (authentication, Git operations, templates)
- **Phase 3**: Project Provisioning - Implement multi-tenant project creation
- **Phase 4**: Liferay Deployment - Deploy Liferay DXP clusters
- **Phase 5**: Cockpit UI - Implement frontend management interface
- **Phase 6-10**: Observability, logs, backups, cloud support, production readiness

`devctl` provides the **infrastructure foundation** for Phase 1+. It does NOT implement the Cockpit product features - those are developed by following PLAN.md phases.

### Troubleshooting

**devctl up fails**:
1. Verify prerequisites: `k3d version`, `kubectl version`, `helm version`, `jq --version`
2. Check Docker running: `docker ps`
3. Check port conflicts: `sudo lsof -i :80 -i :443 -i :8080` (or `lsof` without sudo on macOS)
4. Review error output from devctl console
5. Clean start: `./scripts/local/devctl destroy && ./scripts/local/devctl up`

**Common Issues**:
- **Port 80/443 in use**: Stop conflicting services (Apache, local web servers) or configure custom ports
- **Docker daemon not running**: Start Docker Desktop/Colima/Rancher Desktop
- **Image pull failures**: Check internet connection, proxy settings, Docker Hub rate limits
- **Helm timeout errors**: Increase timeout in devctl script or check cluster has sufficient resources
- **Kyverno blocks mutations**: Expected behavior! Use Argo CD to make changes (see PLAN.md Phase 1)

---

## 7) Developer workflow building blocks
- **UI/API hot reload:** port-forward the in-cluster deployment or run locally with `API_BASE_URL` pointing at `https://cockpit.localdev.me`.
- **Provisioner debugging:** when `demo-project` is enabled, view logs under Admin → Provisioning Progress; rerun jobs idempotently via CLI or UI.
- **GitOps validation:** with the `argocd` extra, every change flows through Git → Argo CD; use this to test drift/prune behaviour.
- **Logs/metrics:** enabling `observability` exposes Grafana OSS deep links and routes CLI log queries to Loki.

---

## 8) Troubleshooting
- `devctl status` shows Kubernetes context, ingress IP, and component health.
- Smoke test failures usually mean ESO did not sync; check `kubectl get externalsecret -n liferay-cockpit`.
- Extras can be re-applied: rerun `devctl up --extras=…` to reconcile missing components.
- mkcert issues? rerun without `--tls`; HTTP works.

---

## 9) Cleanup
- `scripts/local/devctl down` removes workloads but leaves Docker images for faster re-up.
- `scripts/local/devctl destroy --all` deletes the k3d cluster, cached certs, and temp files.

---

## 10) Notes & non-goals
- Local is for **productivity and UX parity**, not production load testing.
- No real cloud credentials required; workload identity is simulated.
- Owners/Members still have **no kubectl**; all operations go through Cockpit/Argo.
- Full parity (multi-project, advanced backups) remains optional and behind flags.

---

**One-liner:** *`devctl up` spins Cockpit, guardrails, and essentials in one shot; extras let us layer observability, GitOps, and sample projects only when needed.*

# Portability — Infrastructure Dependencies & Options

**Status:** Draft (MVP-aligned)  
**Principles:** *GitOps-first • Isolation by default • **KISS** • Local↔Cloud parity • Kubernetes **stateless by default***

> **Intent:** Document the **minimum** platform dependencies and the **equivalent options** across **Local**, **EKS**, and **AKS** so projects behave the same while swapping only the underlying services.

---

## 1) Scope & Assumptions

- **Stateless Kubernetes by default:** databases, file store, and search live **outside** the cluster (managed services). PVCs are **exceptions** and must be documented.  
- **Project isolation:** each `(project, env)` is mapped to its own namespace(s) and cloud resources with strict scoping.  
- **Low surface area:** prefer **managed** services and **links** to provider consoles; Cockpit avoids custom UI when an equivalent exists.  
- **Personas:** Admin (provisions plumbing), Owner (operates own project), Member (day-to-day within project). Owners/Members do **not** use `kubectl`.

---

## 2) Summary Matrix (capability → runtime choice)

| Capability | Local (Dev) | AWS EKS (Prd) | Azure AKS (Prd) | Notes |
|---|---|---|---|---|
| **Kubernetes** | k3d/kind/minikube | Amazon EKS | Azure AKS | Single region per cluster; namespaces per env |
| **GitOps** | Argo CD (AppProject/ApplicationSet) | Same | Same | Git = source of truth |
| **Packaging** | Helm (primary) + optional Kustomize | Same | Same | Schema-validated values |
| **Metrics & Alerts** | Prometheus OSS (+ Alertmanager) | Amazon Managed Service for Prometheus | Azure Monitor managed service for Prometheus | PromQL-compatible across environments |
| **Dashboards** | Grafana OSS | Amazon Managed Grafana | Azure Managed Grafana | Cockpit **deep links**; no custom charts |
| **Logs** | Loki (multi-tenant) | CloudWatch Logs | Azure Log Analytics | CLI-first; Cockpit brokers scope |
| **Database (DXP)** | Dev Postgres container | Amazon RDS for PostgreSQL (PITR) | Azure Database for PostgreSQL (PITR) | Managed RDBMS only for prd |
| **Document Library** | MinIO (dev) | S3 (versioning + lifecycle) | Blob (versioning + lifecycle) | Externalized file store |
| **Search** | OpenSearch/Elastic (dev) | Amazon OpenSearch **or** Elastic Cloud (BYO) | OpenSearch/Elastic Cloud (BYO) | Remote mode for DXP; reindexable |
| **Ingress** | NGINX | ALB Ingress (or NGINX+ALB) | AGIC (or NGINX) | TLS via cert-manager or cloud certs |
| **DNS** | Dev DNS/hosts | Route53 | Azure DNS | ExternalDNS optional |
| **TLS** | cert-manager (dev) | ACM or cert-manager | Key Vault/Front Door or cert-manager | Prefer cloud certs in prd |
| **Secrets** | External Secrets Operator (dev provider) | ESO → AWS Secrets Manager | ESO → Azure Key Vault | Values never shown in UI/CLI |
| **Backups** | Dev-only (logical dumps; MinIO versioning) | RDS PITR · S3 versioning · OpenSearch snapshots | Azure PG PITR · Blob versioning · OS/Elastic snapshots | PVCs only by exception; otherwise externalized |
| **Container Registry** | local registry | ECR | ACR | Image allowlists enforced |
| **Auth (Cockpit)** | Username/password (MVP); OIDC (post-MVP) | Username/password (MVP); OIDC (post-MVP) | Username/password (MVP); OIDC (post-MVP) | Session tokens for UI/CLI |

---

## 3) Runtime Profiles

### 3.1 Local (Developer / CI)
- **K8s**: k3d/kind/minikube (1-node).  
- **Observability**: Prometheus OSS + Grafana OSS; dashboards mirror cloud.  
- **Logs**: Loki; tenanting via labels/org ID.  
- **Data**: Dev Postgres/MinIO/OpenSearch only for **development**; never used for prd.  
- **Ingress/TLS**: NGINX + dev cert; simple DNS/hosts.  
- **Secrets**: ESO with a local/dev backend.  
- **Why**: Fast, reproducible, and mirrors cloud workflows.

### 3.2 AWS (EKS)
- **K8s**: EKS cluster per region.  
- **Observability**: **AMP** (managed Prometheus) + **Amazon Managed Grafana**.  
- **Logs**: **CloudWatch Logs** with project/env log groups.  
- **Data**: **RDS PostgreSQL** (PITR), **S3** with versioning/lifecycle, **Amazon OpenSearch** or Elastic Cloud (BYO).  
- **Ingress/TLS/DNS**: ALB Ingress + Route53; TLS via **ACM** or cert-manager.  
- **Secrets**: ESO → **AWS Secrets Manager**.  
- **Why**: Max leverage of managed services; fewer moving parts.

### 3.3 Azure (AKS)
- **K8s**: AKS cluster per region.  
- **Observability**: **Azure Managed Prometheus** + **Azure Managed Grafana**.  
- **Logs**: **Azure Log Analytics** (KQL).  
- **Data**: **Azure Database for PostgreSQL** (PITR), **Blob** versioning/lifecycle, **OpenSearch/Elastic Cloud** (BYO).  
- **Ingress/TLS/DNS**: **AGIC** + Azure DNS; TLS via **Key Vault/Front Door** or cert-manager.  
- **Secrets**: ESO → **Azure Key Vault**.  
- **Why**: Mirrors AWS profile using Azure-native services.

---

## 4) Feature Parity & Mapping

| Feature | Local | EKS | AKS | Parity Notes |
|---|---|---|---|---|
| **Deployments & Promotion** | Argo CD | Argo CD | Argo CD | Same PR/Argo flow |
| **Health/Sync/Diff** | Argo APIs | Argo APIs | Argo APIs | Same UI patterns |
| **Observability access** | Grafana OSS link | Managed Grafana link | Managed Grafana link | No custom charts |
| **Logs** | `lc2 logs` → Loki | `lc2 logs` → CloudWatch | `lc2 logs` → Log Analytics | CLI-first; scope enforced |
| **Backups** | Dev only | Managed services | Managed services | Externalized; PVCs by exception |
| **Secrets** | ESO (dev) | ESO → AWS SM | ESO → Azure KV | Values never shown |

---

## 5) Dependency Policies (Keep It Simple)

- **Prefer managed**: pick AMP/Managed Grafana (AWS) and Azure Managed Prometheus/Managed Grafana (Azure).  
- **Avoid bloat**: only add dependencies with strong adoption and clear ROI.  
- **Pluggable**: OpenSearch/Elastic are **BYO** options; Cockpit still enforces scoping.  
- **PVCs are exceptions**: must document snapshot/restore policy per namespace if used.

---

## 6) Networking & Security Defaults

- **Namespaces** per env: `proj-<id>-dev|uat|prd`.
- **RBAC** scoped to project; **Owners/Members** operate only via Cockpit/UI/CLI (no `kubectl`).
- **Admission** denies mutations not from Argo controller SA or Cockpit operator SA.
- **NetworkPolicy** default‑deny + explicit egress (DB, cache, external APIs).
- **PSA=restricted**, registry allowlists, resource quotas/limits, TLS everywhere.
- **Ingress/Gateway**:
  - Controllers per runtime: Local (Ingress‑NGINX), AWS (ALB), Azure (AGC/AGIC), GCP (GKE Gateway)
  - Public/private entry classes with WAF on public entries (Cockpit + Liferay PRD)
  - Cockpit: stateless sessions, WebSocket/SSE, 120-300s timeouts
  - Liferay: sticky sessions (PRD), canary rollouts (5%→25%→50%→100%), 60-120s timeouts
  - TLS termination at edge; provider‑managed certs on cloud, cert‑manager on local
  - Connection draining, PDBs, multi‑AZ for PRD

---

## 7) Cost & Operability Notes

- **Local**: zero cloud cost; good for iteration and CI validations.  
- **EKS/AKS**: choose **managed** services to reduce ops toil; pay‑as‑you‑scale for metrics/logs/dashboards and database storage.  
- **Grafana**: managed plans support SSO and data-source RBAC; OSS is fine for local.  
- **Backups**: RPO/RTO determined by provider configuration; Cockpit tracks status and restores via PR/ops.

---

## 8) Acceptance Criteria (Portability)

- A project that **works locally** deploys on **EKS/AKS** with **no manifest changes** beyond provider endpoints/secret refs.  
- Observability **deep links** work for all runtimes; no custom charting required.  
- `lc2 logs` returns data for all runtimes with the same arguments (only backend differs).  
- Backups/Restores are operational at the provider layer; Cockpit reflects status and orchestrates via PR/ops.  
- No Owner/Member `kubectl`; Admin’s break-glass access is audited.

---

**One-liner:** *Same workflows everywhere, different managed backends underneath. Keep Kubernetes stateless, lean on cloud services, and change only what’s necessary to swap Local ↔ EKS ↔ AKS.*

# Liferay Cockpit Application Requirements

**Version**: 0.11 (Draft)  
**Date**: 2025-10-12  
**Status**: Draft for Review  
**Project Code**: Liferay Cockpit (Next.js + Java + PostgreSQL)

---

## 1. purpose

Define the functional and non‑functional requirements for the **Liferay Cockpit** application, a lightweight management plane that orchestrates the DevOps lifecycle (Dev → UAT → PRD) for Liferay DXP workloads on Kubernetes. Cockpit runs **inside Kubernetes** (local k3d/kind/minikube for development; EKS/AKS for cloud) and keeps **Kubernetes as stateless as possible**—all durable data (DB, filestore, search) is **externalized**.

---

## 2. system overview

- **architecture pattern**: Modular application composed of a **Next.js** web experience (UI/BFF), a **Java** control‑plane API service (authoritative backend), and a **PostgreSQL** persistence layer **dedicated to Cockpit**. Cockpit runs in its **own namespace** and uses a **Terraform‑provisioned managed DB** in cloud (RDS/Azure PG).  
- **primary responsibilities**:
  - Provide **project‑centric** lifecycle management for Liferay environments.  
  - Orchestrate **project provisioning** (admin‑only) and **GitOps** delivery (Argo CD) for apps.  
  - **Gate features** on environment **readiness** (DB, filestore, search, ESO, metrics, logs).  
  - Surface **observability** via deep links; provide **CLI‑first** access for logs.  
- **core technologies**:
  - **UI**: Next.js 15 (TypeScript, App Router). UI never connects to Postgres.  
  - **Backend**: **Java 21 / Spring Boot 3.x** — owns domain model, RBAC, audit, idempotency; exposes REST + SSE.  
  - **Database**: PostgreSQL 15; migrations via Liquibase/Flyway.  
  - **Runtime**: Kubernetes + Helm; Argo CD for GitOps.  
- **design rules**:  
  - **Infra abstraction**: **Admins** provision; **Owners/Members** never configure IaaS.  
  - **kubectl policy**: Owners/Members **no kubectl** (including `kubectl logs`); **Admins** break‑glass only (audited).  
  - **Cockpit last**: Infra pipeline **fails** if Cockpit isn’t healthy (fail‑fast bootstrap).

---

## 3. deployment context & adaptation requirements

1. **infrastructure awareness**
   - Detect local vs. cloud runtime to adjust links (Grafana, logs), DNS/ingress, and defaults.
   - Source of truth: operator ConfigMap flags + cluster metadata.

2. **local mode behavior**
   - Use **local Postgres** (StatefulSet) for Cockpit DB if a managed endpoint is not provided.  
   - Install dev‑grade **Prometheus/Grafana OSS** and **Loki** only when external endpoints are absent.  
   - Use NGINX ingress with dev TLS (mkcert) and `*.localdev.me` by default.

3. **managed IaaS behavior**
   - Cockpit DB is **managed** (RDS/Azure PG) provisioned by Terraform.  
   - Use **managed Prometheus + Managed Grafana**; logs via **CloudWatch**/**Log Analytics**.  
   - DNS and certificates through provider services; ExternalDNS/cert‑manager are optional.

4. **project provisioner (in‑cluster)**
   - A **short‑lived Job** (inside `liferay-cockpit`) runs **Terraform + cloud CLIs** using **workload identity** (IRSA/Workload Identity).  
   - **One module**: `project-provisioner` (Terraform) creates per‑project services (prd dedicated; dev/uat shared within project) and writes secrets to the provider vault; Cockpit commits **ExternalSecret** references.

5. **terraform state & concurrency**
   - Remote backends with locking (S3+Dynamo / Azure Blob).  
   - Serialize **applies per project**; queue additional jobs.

---

## 4. personas & access model

| Persona | Scope | Can do | Cannot do |
|---|---|---|---|
| **Admin** | Whole cluster/region | Login to **/admin**, create projects (wizard), trigger provisioner jobs, view provisioning logs, manage policies; break‑glass kubectl (audited) | Expose secrets; bypass GitOps for app changes |
| **Owner** | One project | Operate apps (diff/sync/rollback), open observability links, request promotions/backups/restores | Any IaaS/PaaS configuration; kubectl |
| **Member** | One project | Day‑to‑day operations (view health, tail/query logs via CLI broker, open Grafana), contribute PRs | Provisioning; kubectl; policy changes |

**Authentication (MVP)**: **Regional login** with a single **admin** user (forced password change). **No signup**.  
**Post‑MVP**: **Global SSO** (OIDC) with **regional RBAC** and JIT users.

---

## 5. functional requirements

### 5.1 project & environment management (Admin‑only provisioning)
1. **Project creation** via **Project Wizard** (ID, display name, preset, envs; region fixed to current host in MVP).  
2. **Environment bootstrap** (Dev/UAT/PRD): create namespaces, RBAC, quotas/limits, default‑deny NetworkPolicies, PSA posture, Argo scaffolding.  
3. **Managed services** per policy: **Prd dedicated** DB/filestore/search; **Dev/UAT shared** server with per‑env DB/schema, bucket prefixes, and index prefixes.  
4. **Secrets** via **External Secrets Operator**; Cockpit stores **references only**.  
5. **Readiness gating**: Cockpit computes **BindingsReady** (IaaS binding, DB, filestore, search, ESO freshness) and **PromotionReady** (Grafana+logs) per env and **disables** dependent features until green.  
6. **Auto first run**: when **Dev = Ready**, deploy latest approved Liferay to Dev.

### 5.2 deployments & promotion (Owners/Members)
- **Deployments** managed by Argo CD Applications; **PRs** modify values/overlays.  
- **Promotion** Dev→UAT→PRD via PR generator; gates: Healthy, Synced, no firing alerts, **PromotionReady**.

### 5.3 logs & observability
- **Logs**: CLI‑first tail/query/export brokered to provider planes (CloudWatch/Log Analytics/Loki).  
- **Observability**: **Grafana deep links** (managed in cloud, OSS locally). Cockpit does **not** render charts.

### 5.4 backup visibility & recovery (externalized)
- Show **backup catalog** (DB PITR, object storage versioning, search snapshots).  
- **Restore requests** via PR/flow; execution via provider toolchain; audit everything.

### 5.5 audit & approvals
- Every mutation and runtime op emits audit events with identity, scope, and outcome.  
- Promotions and restores can require approvals by policy.

---

## 6. api requirements (java service)

1. **style**: REST JSON with OpenAPI 3.1; **SSE** for live updates.  
2. **authn**: MVP local login for admin; Post‑MVP OIDC (Authorization Code + PKCE).  
3. **authz**: Policy engine in Java; RBAC + resource scoping; group→role mapping post‑MVP.  
4. **idempotency**: `Idempotency-Key` on mutating endpoints; dedupe stored with TTL.  
5. **resilience**: Timeouts, retries, circuit breakers calling Kubernetes/Argo/provider APIs.  
6. **pagination/filtering**: cursor‑based; stable sorts; filters on project/env/status/date.  
7. **audit**: append‑only trail; export CSV/JSON.  
8. **argo integration**: register/view Applications/ApplicationSets; trigger sync/refresh/rollback; read health/sync.  
9. **provisioner control**: enqueue **Provisioner Jobs**; stream job logs/status; ensure single active job per project.

---

## 7. ui requirements (next.js)

1. **information architecture**
   - **Admin**: `/login`, `/admin`, `/admin/projects`, `/admin/projects/new`, **Provisioning Progress**.  
   - **Project workspace**: Overview, Deployments, Workloads, Pods, Autoscaling, Repos & Config, Releases & Promotion, Observability, Alerts, Secrets & Service Bindings, Backup & Restore, Policies (read‑only), Audit & Approvals, Settings, DXP Profile.
2. **gating UX**: Disabled actions with a **single remediation hint** when readiness checks fail.  
3. **real‑time**: SSE‑driven updates for health/sync/promotions.  
4. **accessibility**: WCAG 2.2 AA.  
5. **security**: Strict CSP, HSTS, SameSite=strict cookies, CSRF for POSTs to the Java API.

---

## 8. java service requirements

1. **framework**: **Spring Boot 3.x** modules: `api`, `core`, `provisioner-control`, `integration`.  
2. **kubernetes client**: Fabric8 or official client; respect least‑priv RBAC.  
3. **gitops**: Argo CD REST/gRPC; project‑scoped tokens; handle degraded read‑only modes.  
4. **provisioner orchestration**: schedule Jobs, stream logs, enforce **serialize applies** per project; interact with Terraform state backends.  
5. **background reconciliation**: refresh env readiness, observability links, backup catalog at intervals; backoff on failures.  
6. **db**: Spring Data; Liquibase/Flyway; HikariCP.  
7. **testing**: Testcontainers (Postgres), kind‑based tests for K8s/Argo paths; contract tests for API.

---

## 9. data model (postgresql)

Core entities (scoped by `project_id` where applicable): `projects`, `project_environments`, `deployments`, `jobs`, `approvals`, `audit_logs`, `observability_integrations`, `backups`, `settings`, `webhooks`, and **users/roles** (for MVP admin + future SSO). Enforce UUID PKs, timestamps, soft‑delete where needed, and append‑only for `audit_logs`.

---

## 10. integration requirements

- **Kubernetes**: namespace‑scoped operations; admission/PSA alignment.  
- **Argo CD**: ApplicationSet + Applications; diff/sync/status.  
- **Observability**: Grafana HTTP links; Prometheus HTTP API (status checks).  
- **Logs**: CloudWatch / Log Analytics / Loki APIs via CLI broker.  
- **Secrets**: External Secrets Operator; provider vaults hold values.  
- **Terraform**: in‑cluster **Provisioner Job** as default; remote runners optional later (Atlantis/HCP).

---

## 11. adaptation matrix

| capability | local cluster mode | managed IaaS mode |
|---|---|---|
| Cockpit DB | In‑cluster Postgres (StatefulSet) | Managed RDS/Azure PG via ESO |
| Observability | Prometheus + Grafana OSS | Managed Prometheus + Managed Grafana |
| Logs | Loki | CloudWatch Logs / Azure Log Analytics |
| DNS/TLS | NGINX + dev TLS (mkcert) | Provider DNS/certs; ExternalDNS/cert‑manager optional |
| Secrets | K8s Secrets for dev; keep ExternalSecret CRs for parity | ESO to provider vaults (no values in Cockpit) |
| Provisioning | Provisioner Job with local creds (simulated) | Provisioner Job with workload identity (least‑priv) |

---

## 12. non‑functional requirements

- **Availability (Cockpit UI/API)**: 99.5% target.  
- **Performance**: P95 API < 500 ms; Overview < 2 s for standard project.  
- **Scale**: ≥ 50 projects / 150 envs / 500 deploys per day per cluster.  
- **Security**: TLS 1.3; hashed passwords for MVP admin; secrets redaction; no long‑lived cloud keys.  
- **Observability**: Prometheus metrics; JSON logs with correlation IDs; (tracing optional post‑MVP).  
- **Resilience**: graceful degradation; retries/backoff; read‑only modes when dependencies are down.  
- **Portability**: identical UX across local/EKS/AKS; only backends differ.  
- **Supply chain**: SBOMs, signed images, admission policy checks.

---

## 13. testing & validation

- **E2E**: Cluster bootstrap (Cockpit last, fail‑fast), project provisioning (Admin), Dev auto‑deploy, promotion gates, observability links, logs CLI.  
- **Integration**: K8s/Argo/Terraform paths; ESO refresh; readiness probes.  
- **Performance**: load tests on Deployments/Overview.  
- **Chaos**: provisioner job timeout, Argo down, provider API failures (retries + surfacing).  
- **CI**: kind‑based jobs for client paths; Testcontainers for Java.

---

## 14. delivery

- **Packaging**: Docker images for Next.js (Node 20) and **Java** (distroless). Helm chart bundles both + Cockpit DB config.  
- **GitOps**: Argo CD App‑of‑Apps; Cockpit is **last stage** in Terraform infra; bootstrap fails if Cockpit health probe is not 200.  
- **Upgrades**: rolling; DB migrations as Jobs with rollback plan.  
- **Cockpit DB backup**: PITR procedures documented (provider‑native).

---

## 15. assumptions & out‑of‑scope (MVP)

- Multi‑region SSO, invites, and signup (post‑MVP).  
- Owner/Member `kubectl` (never planned).  
- In‑cluster stateful data services for production (PVCs are exceptions only).

---

**next steps**
1) Review personas and gating flows with stakeholders.
2) Finalize `project-provisioner` Terraform inputs/outputs.
3) Lock MVP scope for Java API, Admin Wizard, Provisioner orchestration, Argo wiring, audits, and readiness probes.

---

## 16. cloud native poc patterns

A functional PoC was developed to deploy Liferay DXP on AWS EKS using Terraform and Helm (`poc-dxp-cloud-native.txt`, 4382 lines). While the PoC is **single-tenant** and **IaC-driven** (not GitOps), it demonstrates several **production-ready patterns** that Cockpit will adopt. This section documents the findings and integration decisions.

### 16.1 poc architecture summary

**What the PoC Implements**:
- Single-tenant Liferay deployment on AWS EKS
- Terraform modules for VPC, EKS, RDS PostgreSQL, OpenSearch, S3
- Helm chart with Liferay clustering (JGroups DNS_PING)
- Blue/green database pattern for zero-downtime migrations
- Argo Workflows for backup/restore operations
- IRSA (IAM Roles for Service Accounts) for AWS authentication
- Go-based Kubernetes Operator (minimal/stub implementation)

**Architectural Differences from Cockpit**:
- ❌ Single-tenant vs. Cockpit's multi-tenant (namespace isolation)
- ❌ Manual `terraform apply` vs. Cockpit's GitOps-first (Argo CD)
- ❌ No admission control vs. Cockpit's Kyverno enforcement
- ❌ Direct Kubernetes Secrets vs. Cockpit's External Secrets Operator
- ❌ AWS-only vs. Cockpit's multi-cloud (AWS + Azure + local)

**Entrenchment Clause Alignment**: Only **EC10 (Declarative Manifests)** fully aligns. The PoC does not use GitOps, namespace isolation, ESO, or audit logging.

---

### 16.2 patterns to adopt

These patterns from the PoC will be integrated into Cockpit's GitOps-first, multi-tenant architecture:

#### Priority 1: Phase 2 (Liferay DXP Deployment - Local)

**✅ 16.2.1 JGroups DNS_PING Configuration**

Liferay clustering in Kubernetes requires JGroups to discover cluster members via DNS SRV records.

**PoC Implementation** (`cloud/helm/default/resources/unicast.xml`):
```xml
<dns.DNS_PING
    dns_query='_cluster._tcp.{{ include "liferay.name" $ }}-headless.{{ include "liferay.namespace" $ }}.svc.cluster.local'
    dns_record_type="SRV"
/>
```

**Cockpit Integration**:
- TemplateService generates `unicast.xml` per project
- Commit to Git: `environments/{project-id}/liferay-config/unicast.xml`
- Reference in Liferay StatefulSet as ConfigMap volume
- Requires headless Service for stable cluster DNS

**Requirement Alignment**: Supports LOCAL_DEV_MODEL clustering needs (line 2599).

---

**✅ 16.2.2 Liferay Portal Properties Template**

Production-ready Liferay configuration connecting to PostgreSQL, S3, and OpenSearch.

**PoC Implementation** (`cloud/helm/default/resources/portal-cloud.properties`):
```properties
# Database
jdbc.default.driverClassName=org.postgresql.Driver
jdbc.default.url=jdbc:postgresql://{{ .Values.database.endpoint }}:5432/{{ .Values.database.name }}
jdbc.default.username={{ .Values.database.username }}
jdbc.default.password={{ .Values.database.password }}

# S3 Document Library
dl.store.impl=com.liferay.portal.store.s3.S3Store
dl.store.s3.bucket.name={{ .Values.s3.bucketName }}
dl.store.s3.region={{ .Values.s3.region }}

# OpenSearch
elasticsearch7.http.enabled=true
elasticsearch.http.url=https://{{ .Values.opensearch.endpoint }}:443
elasticsearch.username={{ .Values.opensearch.username }}
elasticsearch.password={{ .Values.opensearch.password }}
```

**Cockpit Integration**:
- TemplateService generates `portal-cloud.properties` per project/environment
- Values sourced from ExternalSecrets (DB credentials, S3 bucket, etc.)
- Commit to Git: `environments/{project-id}/liferay-config/portal-cloud.properties`
- Mount as ConfigMap in Liferay pods

**Requirement Alignment**: Matches LOCAL_DEV_MODEL Liferay configuration (line 2598).

---

**✅ 16.2.3 StatefulSet for Liferay**

Liferay requires stable network identity for clustering.

**PoC Implementation**: Uses `StatefulSet` (not `Deployment`) with:
- Stable pod identifiers: `liferay-0`, `liferay-1`, etc.
- Headless Service for cluster communication
- PersistentVolumeClaims for Liferay home directory

**Cockpit Integration**:
- **Local (Phase 2)**: Use StatefulSet with PVCs for convenience
- **AWS (Phase 5)**: Use StatefulSet but minimize PVC usage (externalize to RDS/S3/OpenSearch)

**Requirement Alignment**:
- ✅ ENTRENCHMENT_CLAUSES line 339: "**PVCs are exceptions**, not baseline"
- ✅ REQUIREMENTS line 2605: "PVCs only for local dev convenience"

---

#### Priority 2: Phase 5 (AWS Cloud Support)

**✅ 16.2.4 Terraform Module Structure**

PoC demonstrates modular Terraform organization for reusability.

**PoC Structure**:
```
terraform/aws/
  eks/           # VPC, EKS cluster, security groups
  dependencies/  # RDS, OpenSearch, S3, IAM
  backup/        # AWS Backup vaults, plans
```

**Cockpit Integration**:
```
terraform/
  modules/
    vpc/
    eks-cluster/
    rds-postgres/
    opensearch/
    s3-bucket/
    iam-roles/
  stacks/
    aws/
      cluster-cockpit/    # Management plane
      project-resources/  # Per-project template (used by Provisioner Job)
```

**Provisioner Job Usage**:
1. Cockpit API schedules Job with project-specific tfvars
2. Job runs `terraform apply` in `project-resources/` stack
3. Job writes outputs to AWS Secrets Manager
4. Job generates ExternalSecret manifests
5. Job commits manifests to Git
6. Argo CD reconciles ExternalSecrets

**Requirement Alignment**: Matches ARCHITECTURE line 1078 (Provisioner Job pattern).

---

**✅ 16.2.5 IRSA (IAM Roles for Service Accounts)**

AWS best practice for pod-to-AWS authentication without long-lived credentials.

**PoC Implementation** (`cloud/terraform/aws/dependencies/liferay.tf`):
```hcl
resource "aws_iam_policy" "s3" {
  name = "${var.deployment_name}-s3-policy"
  policy = jsonencode({
    Statement = [{
      Action = ["s3:DeleteObject", "s3:GetObject", "s3:ListBucket", "s3:PutObject"]
      Resource = [module.s3_bucket.s3_bucket_arn, "${module.s3_bucket.s3_bucket_arn}/*"]
    }]
  })
}

resource "aws_iam_role" "liferay" {
  name = "${var.deployment_name}-liferay-role"
  assume_role_policy = data.aws_iam_policy_document.oidc_assume_role.json
}
```

**Cockpit Integration**:
1. Provisioner Job creates IAM role per project: `cockpit-proj-{id}-liferay`
2. Attach policies for RDS access, S3 access, OpenSearch access (least privilege)
3. Annotate ServiceAccount in Liferay namespace:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: liferay
  namespace: proj-{id}-dev
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/cockpit-proj-{id}-liferay
```
4. Liferay pods automatically assume role via EKS OIDC integration

**Requirement Alignment**: Implied in SECURITY section (line 1129: AWS Secrets Manager).

---

**✅ 16.2.6 Random Password Generation**

Secure credential generation via Terraform.

**PoC Implementation** (`cloud/terraform/aws/dependencies/passwords.tf`):
```hcl
resource "random_password" "postgres_password" {
  length = 16
  min_lower = 1
  min_numeric = 1
  min_special = 1
  min_upper = 1
  override_special = "!#%&*()-_=+[]{}<>:?"
  special = true
}
```

**Cockpit Integration** (adapted for ESO):
1. Provisioner Job generates random passwords via Terraform
2. Terraform writes to AWS Secrets Manager: `cockpit/proj-{id}/db-password`
3. Provisioner generates ExternalSecret manifest referencing AWS SM
4. Commits ExternalSecret to Git
5. ESO syncs secret into namespace

**Requirement Alignment**: Matches **EC7 (Secrets via ESO)**. PoC uses K8s Secrets; Cockpit uses ESO.

---

**✅ 16.2.7 Security Groups for Data Services**

AWS networking best practices for RDS and OpenSearch.

**PoC Implementation** (`cloud/terraform/aws/dependencies/liferay.tf`):
```hcl
resource "aws_security_group" "rds" {
  name = "${var.deployment_name}-rds-sg"
  vpc_id = local.vpc_config.vpc_id
}

resource "aws_vpc_security_group_ingress_rule" "rds_ingress" {
  cidr_ipv4 = data.aws_vpc.current.cidr_block  # Restrict to VPC CIDR only
  from_port = 5432
  to_port = 5432
  ip_protocol = "tcp"
  security_group_id = aws_security_group.rds.id
}
```

**Cockpit Integration**:
- Provisioner Job creates security groups per project
- Restrict ingress to EKS node security group only
- Apply to RDS, OpenSearch resources

**Requirement Alignment**: Matches SECURITY section (line 1130: private networking, least privilege).

---

#### Priority 3: Phase 8 (Advanced Features - Optional)

**⚠️ 16.2.8 Blue/Green Database Pattern**

PoC maintains two RDS instances (blue and green) for zero-downtime migrations.

**PoC Implementation** (`cloud/terraform/aws/dependencies/liferay.tf`):
```hcl
locals {
  is_active_data_blue = var.data_active == "blue"
  db_active = local.is_active_data_blue ? module.postgres_blue[0] : module.postgres_green[0]
  bucket_active = local.is_active_data_blue ? module.s3_bucket_blue : module.s3_bucket_green
}
```

**Cockpit Integration Decision**:
- ⚠️ **Trade-off**: 2x database cost for every project
- ⚠️ **Alternative**: Use RDS snapshots + Argo CD progressive rollout
- ⚠️ **Decision**: **Phase 8 (optional)** - Not in MVP scope

**Requirement Alignment**: Not explicitly required in REQUIREMENTS.md; post-MVP enhancement.

---

**⚠️ 16.2.9 Argo Workflows for Backup/Restore**

PoC uses Argo Workflows (not Argo CD) for orchestrating complex multi-step backup operations.

**PoC Implementation**: DAG workflow for restore:
1. Checkout Git repository (Terraform code)
2. Get current active/inactive database identifiers
3. Find peer recovery points (RDS snapshot + S3 recovery point)
4. Apply Terraform with `is_restoring=true`
5. Restart Liferay StatefulSet
6. Clean up inactive RDS instance

**Cockpit Integration Decision**:
- ⚠️ **Requirements State**: "**provider PITR** (RDS/Azure PG); **S3/Blob versioning**" (line 214)
- ⚠️ **Preference**: Use AWS Backup service for RDS PITR (simpler, provider-native)
- ⚠️ **Decision**: **Phase 8 (optional)** - Consider Argo Workflows for advanced restore scenarios only

**Requirement Alignment**: Partially aligned - Requirements prefer provider-native backup.

---

### 16.3 patterns to discard

These PoC patterns violate Cockpit's Entrenchment Clauses and will **not** be adopted:

**❌ 16.3.1 Manual Terraform Apply**

**PoC Approach**: Platform engineers run `terraform apply` manually.

**Why Discard**: Violates **EC1 (GitOps-Only)** and **EC9 (API-Driven)**.

**Cockpit Approach**: Cockpit API → Provisioner Job → Git commit → Argo CD reconciliation.

---

**❌ 16.3.2 Direct Kubernetes Secrets**

**PoC Approach**: Terraform creates `kubernetes_secret` resources directly.

**Why Discard**: Violates **EC7 (Secrets via ESO)**.

**Cockpit Approach**: Terraform → AWS Secrets Manager → ExternalSecret → ESO syncs.

---

**❌ 16.3.3 Single-Tenant Architecture**

**PoC Approach**: One Liferay deployment per cluster.

**Why Discard**: Cockpit is **multi-tenant** - many projects per cluster.

**Cockpit Approach**: Namespace isolation (`proj-{id}-dev/uat/prd`).

---

**❌ 16.3.4 Hardcoded Resource Limits**

**PoC Approach**: Helm values have hardcoded CPU/memory limits.

**Why Discard**: Violates **EC5 (Preset-Based Sizing)**.

**Cockpit Approach**: ResourceQuota at namespace level based on preset (S/M/L).

---

**❌ 16.3.5 No Admission Control**

**PoC Approach**: Platform engineers have full kubectl access; no enforcement of GitOps principles.

**Why Discard**: Violates **EC1 (GitOps-Only)** - need technical enforcement.

**Cockpit Approach**: Kyverno ClusterPolicy blocks non-GitOps mutations (Phase 1A).

---

**❌ 16.3.6 Kubernetes Operator (Stub Implementation)**

**PoC Approach**: Deploys Go-based operator that only logs "Hello, world!" when ConfigMaps change.

**Why Discard**: PoC operator has no real functionality; Cockpit API serves this orchestration role.

**Cockpit Approach**: Cockpit API is the control plane; standard controllers (Argo CD, ESO) are sufficient.

---

### 16.4 integration guidance

**Adapting PoC Patterns to GitOps Multi-Tenancy**:

1. **Templating**: PoC configurations (JGroups, portal properties) must be templated for per-project generation.

2. **Git Commit Flow**: All PoC Terraform/Helm outputs must be committed to Git (not applied directly):
   ```
   Cockpit API → Generate Manifests → Git Commit → Argo CD Reconcile → Resources Created
   ```

3. **Namespace Isolation**: PoC uses single namespace; Cockpit must adapt all resources to `proj-{id}-{env}` namespaces.

4. **Secrets Management**: Replace all PoC Kubernetes Secrets with ExternalSecrets referencing cloud provider secret stores.

5. **Provisioner Job Pattern**: PoC runs Terraform manually; Cockpit schedules in-cluster Job:
   - Job runs Terraform with project-specific tfvars
   - Job writes outputs to cloud provider secret store (AWS SM/Azure KV)
   - Job generates ExternalSecret manifests
   - Job commits manifests to Git
   - Argo CD reconciles ExternalSecrets

6. **Multi-Cloud Abstraction**: PoC is AWS-only; Cockpit must abstract provider-specific resources:
   - Phase 5: AWS (RDS, OpenSearch, S3, IRSA)
   - Phase 10: Azure (PostgreSQL, Cognitive Search, Blob Storage, Managed Identity)

---

**Implementation Phases**:

- **Phase 2**: Adopt JGroups config, portal properties, StatefulSet
- **Phase 5**: Adopt Terraform modules, IRSA, random passwords, security groups
- **Phase 8**: Consider blue/green DB and Argo Workflows (optional)

---

**Testing & Validation**:

Each adopted pattern must have test checkpoints:
- **Phase 2.4 Checkpoint**: Liferay cluster forms successfully (2 replicas, JGroups DNS_PING)
- **Phase 5.4 Checkpoint**: IRSA works (Liferay pod accesses S3 without credentials in manifest)
- **Phase 5.3 Checkpoint**: ESO syncs credentials from AWS Secrets Manager

---

**Documentation Requirements**:

PLAN.md will be updated to reference these PoC patterns explicitly in:
- Phase 2, Task 2.4: Liferay deployment (JGroups, portal properties, StatefulSet)
- Phase 5, Tasks 5.2-5.4: AWS provisioning (Terraform modules, IRSA, security groups)

---


# LC2 Skills — Automation & Governance

**Status:** Living document (LC2)
**Purpose:** Document available Claude Code skills for automating LC2 operations, policy generation, and governance

---

## Overview

LC2 provides specialized **Claude Code skills** for common platform operations. These skills are invoked automatically when working with LC2 repositories and can be called explicitly when needed. All skills follow the **KISS** principle and maintain **local↔cloud parity**.

---

## Available Skills

### 1. lc2-architecture
**Description:** Scaffold or audit the LC2 architecture (infrastructure, Kubernetes, PaaS Cockpit, project). Produces Architecture.md, ADRs, Argo CD app-of-apps skeleton, Kyverno policy index, naming conventions, and checklists.

**Use cases:**
- Initialize LC2 architecture for a new project
- Audit existing architecture for compliance with four-layer model
- Generate ADRs (Architecture Decision Records)
- Create Argo CD app-of-apps scaffolding

**Inputs:**
- Project ID (e.g., `acme`)
- Environments (default: `dev, uat, prd`)
- Runtime: `local | aws | azure | gcp`
- Cluster name and region (when cloud)
- Shared vs per-project toggles (Argo CD shared, Grafana OSS shared with org-per-project; Loki optional)

**Outputs:**
- `templates/docs/Architecture.md`
- `templates/docs/Principles.md`
- `templates/docs/Naming Conventions.md`
- `templates/docs/Checklists/Architecture Audit.md`
- `templates/docs/ADR/adr-template.md` and example ADRs
- `templates/argocd/app-of-apps.yaml` and `templates/argocd/appproject-<project>.yaml`
- `templates/kyverno/policy-index.md`
- `templates/kubernetes/project-layout.yaml`

**Example invocation:**
```
"initialize lc2 architecture for project acme on eks with managed prometheus and cloudwatch"
"audit architecture for project beta on gke; list argo fences and kyverno gaps"
```

---

### 2. kyverno-guardrails
**Description:** Generate and maintain Kyverno policy packs for LC2 with strong tenant isolation and KISS guardrails. Creates cluster-wide non-negotiables (PSS, default-deny, quotas, supply chain) and per-project conventions (labels, ESO-first secrets, egress allow-lists).

**Use cases:**
- Generate baseline Kyverno policies for cluster
- Create project-specific tenant policies
- Generate PolicyException definitions
- Create Kyverno test suites

**Inputs:**
- Project list (optional): `cockpit.projects.json` or folders under `projects/`
- Repo layout (e.g., `kyverno/`, `argocd/`, `charts/`)

**Outputs:**
- `kyverno/cluster/*.yaml` — ClusterPolicies (non-negotiables)
- `kyverno/tenant/*.yaml` — namespaced Policies (templated per project)
- `kyverno/README.md` — rationale, test commands, rollout (audit → enforce → tighten)
- `argocd/appprojects/<project>.yaml` (optional scaffold) and RBAC tips

**Cluster guardrails generated:**
1. Pod Security Standards — Restricted (Enforce)
2. Default-deny NetworkPolicy (Generate + Backfill)
3. Resource Governance (ResourceQuota + LimitRange)
4. Supply Chain Hygiene (registry allow-list, no :latest, verifyImages)
5. Service Exposure & Ingress (forbid NodePort, require TLS in UAT/PRD)
6. Policy Exceptions (namespaced, time-boxed)

**Tenant conventions generated:**
1. Labels & Annotations Contract (mutate/validate)
2. Egress allow-list
3. Secrets discipline (ESO-first)
4. Image pull policy by environment
5. Ingress TLS by environment

**Example invocation:**
```
"Generate cluster guardrails and tenant policies for project acme (dev/uat/prd)"
"Add verifyImages for ghcr.io/liferay/** with our Cosign public key"
"Switch secrets to ESO-first and warn on raw Secrets in UAT/PRD"
```

---

### 3. lc2-observability
**Description:** Configure or audit observability for Liferay Cloud 2.0 across local (k3d) and cloud (EKS/AKS/GKE). Generates Alloy configs, wires Loki/Mimir or managed Prometheus, provisions Grafana org-per-project, creates Argo CD apps, and enforces Kyverno guardrails.

**Use cases:**
- Set up observability for a new project
- Audit observability configuration and coverage
- Configure dual-write strategy (provider + Loki)
- Adjust per-tenant retention and limits

**Inputs:**
- Project ID: e.g., `acme`
- Environments: `dev, uat, prd` (default)
- Runtime: `local | aws | azure | gcp`
- Log backend: `loki | cloudwatch | cloud-logging | log-analytics` (optional dual-write to loki)
- Metrics backend: `mimir | cortex | managed-prometheus (amp/gmp/azure)`
- Retention (days): dev=7, uat=14, prd=30 (defaults)
- Per-tenant caps: streams, ingestion rate, query concurrency

**Outputs:**
- `observability/helm/alloy/values.yaml`
- `observability/helm/loki/values.yaml` and `gateway/values.yaml` (when using OSS Loki)
- `observability/helm/mimir/values.yaml` (when using OSS metrics backend)
- `observability/helm/grafana/provisioning/{datasources,dashboards}/*`
- `argocd/apps/observability/*.yaml`
- `kyverno/policies/observability/*.yaml`
- `overrides/<project>.yaml` (per-tenant limits + retention)

**Example invocation:**
```
"set up observability for project acme on eks (aws) with managed prom + cloudwatch"
"audit observability for project beta on gke; show coverage and isolation results"
"switch project acme logs to dual-write (cloudwatch + loki) and set prod retention to 90d"
```

---

### 4. lc2-entrenchment
**Description:** Establish, publish, and audit the Entrenchment Clauses for Liferay Cloud 2.0. Creates executive-level documents, guardrail matrices, PR templates, and scorecards. Tool-agnostic (not Kyverno/Argo specific) and never talks to a cluster.

**Use cases:**
- Create entrenchment clauses for platform governance
- Generate guardrails matrix
- Create PR templates with entrenchment checklist
- Audit repos for entrenchment compliance

**Inputs:**
- Product name and scope (default: Liferay Cloud 2.0)
- Governance owners (e.g., Platform PM, Staff Engineer, SRE Lead)
- Exception approvers and SLA (e.g., 7 days)
- Review cadence (e.g., quarterly)
- Repository paths where PR templates and CODEOWNERS should live

**Outputs:**
- `templates/docs/Entrenchment Clauses.md`
- `templates/docs/Guardrails Matrix.md`
- `templates/docs/Acceptance Checks.md`
- `templates/docs/Exceptions.md`
- `templates/docs/Decision Log.md`
- `templates/docs/Change Governance.md`
- `templates/Checklists/Quarterly Review.md`
- `.github/PULL_REQUEST_TEMPLATE.md`
- `CODEOWNERS`

**Example invocation:**
```
"create the lc2 entrenchment package and PR template for our repo (apply=false)"
"audit repos for entrenchment compliance and produce a scorecard (dry run)"
```

---

### 5. openai-mcp-assist
**Description:** When Claude Code encounters unexpected problems, is refining requirements during execution, or wants a second opinion, it consults the configured OpenAI MCP tool for a "second pair of eyes" to unblock execution errors, crystallize requirements, and sanity-check designs.

**Use cases:**
- Unblock execution errors
- Define/refine requirements and acceptance criteria
- Get second opinion on design decisions
- Diagnose complex issues

**When invoked:**
- Unexpected problem/issue: tool error, failing test, runtime exception, blocked dependency
- Requirements work: defining or refining scope, acceptance criteria, edge cases, test scenarios
- Doubt: uncertainty between options where critique/alternatives review would reduce risk

**Example invocation:**
This skill is invoked automatically by Claude Code when encountering issues. Not typically invoked directly by users.

---

## Skill Usage Guidelines

### When to use skills
- Use **lc2-architecture** when bootstrapping new projects or auditing existing architecture
- Use **kyverno-guardrails** when establishing or updating policy baselines
- Use **lc2-observability** when setting up or troubleshooting telemetry
- Use **lc2-entrenchment** when establishing governance or auditing compliance
- **openai-mcp-assist** is invoked automatically when needed

### Common workflows

**New project setup:**
```
1. Run lc2-architecture skill to scaffold project structure
2. Run kyverno-guardrails skill to generate policies
3. Run lc2-observability skill to configure telemetry
4. Run lc2-entrenchment skill to establish governance
```

**Audit existing project:**
```
1. Run lc2-architecture skill in audit mode
2. Run kyverno-guardrails skill to verify policy coverage
3. Run lc2-observability skill to check telemetry coverage
4. Run lc2-entrenchment skill to produce compliance scorecard
```

---

## Safety and Guardrails

All skills follow these safety principles:
1. **Never write secrets to Git** — reference External Secrets (ESO)
2. **Stage under scratch directories** — show diff before promoting
3. **Only run cluster commands when `apply=true`** is present
4. **Do not write outside project folder**
5. **Use normal casing throughout** — no camelCase-only patterns
6. **KISS principle** — simple things simple, complex things possible

---

## Skills Development

Skills are maintained in the `skills/` directory with the following structure:
```
skills/
  <skill-name>/
    SKILL.md           # Skill definition and documentation
    templates/         # Output templates
    reference.md       # Reference materials (optional)
```

---

**One-liner:** LC2 skills automate architecture scaffolding, policy generation, observability setup, and governance — following KISS, maintaining local↔cloud parity, and keeping Kubernetes stateless by default.

# Infrastructure — Liferay Cloud 2.0 (LC2)

**Status:** Living document (LC2)
**Principles:** Cloud-agnostic • Isolation First • GitOps Always • KISS • Local↔Cloud Parity

> **Reference:** Detailed infrastructure requirements in Infrastructure.md; cloud IAM/RBAC permissions in LC2_Cloud_Permissions.md

---

## 1. Scope & Responsibilities

LC2 infrastructure spans **Local** (k3d/k3s), **AWS** (EKS), **Azure** (AKS), and **GCP** (GKE) with consistent workflows across all runtimes. Infrastructure layer (Layer 1 in the 4-layer architecture) provides the foundation for Kubernetes, PaaS, and Project layers.

### Persona Separation

**Admin (Infrastructure & Platform)**
- Provisions cloud infrastructure with **Terraform**
- Creates/updates Kubernetes clusters (EKS/AKS/GKE)
- Installs and operates the **Cockpit** (management plane) and cluster-level components
- Defines global guardrails (RBAC, quotas, Pod Security Standards, Kyverno policies)
- Maintains networks, DNS, TLS, registries, secret backends, and observability plumbing

**Owner / Member (Project Team)**
- Owns one or more **projects**; deploys and manages **dev/uat/prd** via GitOps (Argo CD)
- Operates within allocated quotas and policies
- Views logs/metrics for their namespaces via **Cockpit**
- **Cannot** see underlying nodes, other projects, or cluster-wide resources

---

## 2. Target Runtimes & Parity

| Runtime | Kubernetes | Database | Search | Object Storage | Logs | Metrics |
|---------|------------|----------|--------|----------------|------|---------|
| **Local** | k3d/k3s | PostgreSQL (containerized) | OpenSearch (containerized) | MinIO | Loki | Prometheus |
| **AWS** | EKS | RDS PostgreSQL | OpenSearch Service / Elastic Cloud | S3 | CloudWatch Logs | AMP |
| **Azure** | AKS | Azure DB PostgreSQL | Elastic Cloud | Blob Storage | Log Analytics | Azure Managed Prometheus |
| **GCP** | GKE | Cloud SQL PostgreSQL | Elastic Cloud | Cloud Storage | Cloud Logging | Managed Prometheus |

**Consistency guarantee:** Same GitOps, policy, and namespace model across all runtimes. Provider differences handled via adapters and clear defaults.

---

## 3. Topology Options (Cluster-Wide)

LC2 supports user-selectable topologies during provisioning to balance cost and resilience:

### 3.1 Kubernetes Availability

1. **All single-AZ** — Lowest cost; development/pilot clusters
2. **DEV/UAT single-AZ, PRD multi-AZ** — Balanced (recommended baseline)
3. **All multi-AZ** — Highest resilience; regulated/mission-critical workloads

**Implementation:** Multi-AZ uses zone-spanning node pools with topology spread constraints, anti-affinity, and PodDisruptionBudgets.

### 3.2 Database Layout (PostgreSQL)

1. **Shared dev+uat, isolated prd** — Cost-optimized
2. **Dedicated per environment** — Strong isolation (recommended for production)

**Engines:** RDS (AWS), Azure Database for PostgreSQL, Cloud SQL (GCP)

### 3.3 Search Layout (OpenSearch/Elasticsearch)

1. **Shared dev+uat, isolated prd** — Cost-optimized
2. **Dedicated per environment** — Strong isolation (recommended)

**Engines:** OpenSearch Service (AWS), Elastic Cloud (Azure/GCP)

---

## 4. Networking & Security

**VPC/VNet Design**
- Private subnets for nodes and data planes
- Public subnets only for load balancers/ingress
- NAT gateways for egress; VPC endpoints/private links where sensible

**Ingress & Load Balancing** (Cloud‑agnostic, Isolation‑first)
- **Local:** Ingress‑NGINX with cert‑manager (self-signed or LE dev certs)
- **AWS:** AWS Load Balancer Controller (ALB Ingress)
- **Azure:** Application Gateway for Containers (AGC) via Gateway API (preferred); AGIC as fallback
- **GCP:** GKE Gateway Controller (Gateway API)

**Entry Classes**
- `public`: Internet‑facing; TLS required; WAF enabled by default
- `private`: Internal (VPN/PSC/peering); WAF optional; internal DNS only

**URL Patterns & Routing**
- **Cockpit**: `<cloud>-<region>.<cockpit-domain>` with `/api/*` and `/projects/*` routes
- **Liferay Tenants**: `project.{dev|uat|prd}.sa.example.com` per environment
- Project ID: `^[a-z0-9-]{2,40}$`

**Session Management**
- **Cockpit**: Stateless; no sticky sessions required
- **Liferay PRD**: Sticky sessions enabled (cookie `DXP_STICKY`, TTL 3600s)
- **Liferay DEV/UAT**: Optional stickiness per performance requirements

**Protocols & Timeouts**
- HTTP/2 enabled; WebSocket/SSE allowed
- **Cockpit**: 120-300s idle/read timeouts
- **Liferay**: 60-120s idle/read timeouts
- Connection draining enabled (ALB deregistration delay, AGC/GKE equivalents)

**Canary/Blue‑Green Deployments**
- Weighted traffic via Gateway API HTTPRoute weights or controller equivalent
- Promotion: 5% → 25% → 50% → 100% with error rate and p95 latency guardrails

**DNS & TLS**
- DNS: Route 53 (AWS), Azure DNS, Cloud DNS (GCP)
- **Cloud:** Provider‑managed certificates (ACM, Key Vault, GCP Certificate Manager)
- **Local:** cert‑manager with ACME DNS‑01 solver; LE staging→prod promotion in pipelines
- TLS termination at edge; mTLS to backends optional

**WAF & Security**
- WAF required for Cockpit public and Liferay PRD public entries
- HSTS enabled on public entries
- Secure headers at edge (X‑Content‑Type‑Options, X‑Frame‑Options, Referrer‑Policy)
- Optional IP allowlists for admin endpoints (`/api/admin/*`)
- Rate limiting per tenant route where supported

**Policy & Hardening**
- **Pod Security Standards** (namespace labels) enforced via **Kyverno**
- **RBAC:** Owners/Members restricted to project namespaces
- **Supply chain:** All release images **signed** (Cosign); admission verifies signatures (Kyverno verifyImages)
- **Default-deny NetworkPolicies** per namespace with explicit egress allowlists
- Routes namespace‑scoped; tenants manage only their own routes
- Allowed annotations/policies whitelisted; unsafe attributes rejected

**Observability**
- Access logs: host, path, status, latency; `projectId` extraction for Cockpit routes
- Metrics: edge/controller metrics to managed Prometheus backends
- Events: Kubernetes events and ingress/controller errors to logging backends
- Correlation ID header at edge propagated to backends

**High Availability**
- PRD: Multi‑AZ topology; PodDisruptionBudgets enforced
- DEV/UAT: Single‑AZ default
- Health checks to explicit endpoints (not `/`)

**SLOs**
- Cockpit: 99.9% availability, p95 < 500ms
- Liferay PRD: 99.95% availability, p95 < 800ms
- Error budgets: 43.2 min/mo (99.9%), 21.6 min/mo (99.95%)

---

## 5. Secrets & Configuration

**Standard:** **External Secrets Operator (ESO)** with ClusterSecretStore mapping to:
- **AWS:** Secrets Manager and/or SSM Parameter Store
- **Azure:** Key Vault
- **GCP:** Secret Manager

Projects declare `ExternalSecret` manifests to materialize runtime `Secret`s in their namespaces. Credentials rotated at provider; ESO handles refresh.

**DXP Licensing (Cluster Mode)**
- **Requirement**: Cluster deployments (replicaCount > 1 or HPA enabled) require valid DXP cluster license (XML)
- **Storage**:
  - **Production**: Kubernetes Secret or ESO-backed secret from cloud secret manager
  - **Local/Dev**: ConfigMap acceptable for testing only
- **Injection**: Mount `license.xml` at `/etc/liferay/mount/files/deploy/license.xml`
- **Environment**: Set `LIFERAY_DISABLE_TRIAL_LICENSE=true` (mandatory to prevent trial license in cluster mode)
- **Per-Environment**: Separate secrets per environment (`liferay-license-dev`, `liferay-license-uat`, `liferay-license-prd`)
- **Rotation**: Update secret → trigger rolling restart via Helm upgrade or GitOps sync
- **Security**: Restrict secret read access to Liferay service account only; never store in Git unencrypted
- **Compliance**: Track license expiry dates; audit all rotation operations

---

## 6. Cloud Permissions (IAM/RBAC)

### AWS - IRSA (IAM Roles for Service Accounts)

**Deployer permissions:**
- `ec2:*`, `eks:*`, `iam:*`, `elasticloadbalancing:*`
- `ssm:PutParameter`, `ssm:GetParameter`, `ssm:DeleteParameter`
- `secretsmanager:CreateSecret`, `secretsmanager:GetSecretValue`, `secretsmanager:DeleteSecret`
- Optional RDS: `CreateDBInstance`, `DescribeDBInstances`, `ModifyDBInstance`

**Runtime workloads:** OIDC-based IAM roles per service account with least-privilege policies (e.g., `secretsmanager:GetSecretValue`)

### Azure - Managed Identity

**Deployer:**
- **Contributor** on resource group
- Optional **User Access Administrator** (only if deployer grants roles)

**Runtime workloads:**
- **Network Contributor** on RG (for LoadBalancer/NIC operations)
- **Key Vault Secrets User** on specific vaults

### GCP - Workload Identity

**Deployer roles (project level):**
- `roles/compute.networkAdmin`
- `roles/container.clusterAdmin`
- `roles/iam.serviceAccountAdmin`
- `roles/secretmanager.admin`
- Optional: `roles/cloudsql.admin`

**Runtime workloads:** GSA bound to KSA with least-privilege (e.g., `roles/secretmanager.secretAccessor`)

> **Reference:** See LC2_Cloud_Permissions.md for complete IAM policy documents and verification checklists

---

## 7. Observability Infrastructure

**Metrics Collection**
- **Grafana Alloy** agents (DaemonSet) for unified log/metric/trace collection
- **Remote_write** to provider-managed Prometheus:
  - AWS: Amazon Managed Service for Prometheus (AMP)
  - Azure: Azure Monitor managed service for Prometheus
  - GCP: Managed Service for Prometheus

**Logs**
- Ship to provider logging planes (CloudWatch / Log Analytics / Cloud Logging)
- Optional **Loki** per runtime (mandatory for local; opt-in for cloud parity)
- **Gateway-enforced tenancy** for multi-tenant backends (X-Scope-OrgID header injection)

**Dashboards**
- **Grafana** (managed in cloud, OSS locally)
- **One organization per project** for isolation
- Cockpit provides deep links; no custom charting in Cockpit

---

## 8. GitOps & Fleet Management

**Argo CD (Cluster Admin)**
- Admins own **root app** (app-of-apps) installing platform components
- Define **AppProjects** per tenant with namespace-scoped permissions
- Enforce drift detection and rollback

**Argo CD (Project Teams)**
- Owners/Members manage app Helm charts via PRs in project Git
- Cockpit exposes safe UX for app-level syncs

**Fleet Management (Multi-Cluster)**
- Rancher manages clusters centrally with consistent labels
- Use for **Day-0 bootstrap** only (install Argo CD, Kyverno, ESO, ingress, Cockpit)
- All **ongoing changes via Argo CD** to avoid control loop conflicts

---

## 9. Capacity Planning & Quotas

**Cluster Sizing**
- Baseline: 5–20 projects × 3 envs × (2 DXP pods + DB + search + aux) per env
- Start with general-purpose + dedicated system/ingress pools
- Scale out additional pools for heavier tenants

**Quotas & Limits**
- Per-project `ResourceQuota` and `LimitRange`
- Request/limit templates in project bootstrap
- Storage quotas per namespace with enforced storage classes

---

## 10. Backup, Restore & DR

**Databases**
- Rely on managed service automated backups and snapshots
- Define RPO/RTO targets per environment
- **AWS:** RDS automated backups + PITR
- **Azure:** Azure Database for PostgreSQL backups
- **GCP:** Cloud SQL automated backups

**Search**
- Managed snapshots (OpenSearch/Elastic) to object storage
- Document restore procedures

**Object Storage**
- Versioning + lifecycle policies on buckets/containers
- S3 (AWS), Blob Storage (Azure), Cloud Storage (GCP)

**Kubernetes State**
- **Git is source of truth** for all Kubernetes manifests, policies, and configurations
- **Velero is explicitly not used**: LC2 follows "Kubernetes stateless by default"—all stateful data lives in managed services (databases, object storage, search) with native backup mechanisms
- PVCs used only by exception; when needed, use provider/CSI volume snapshots (not Velero)
- Critical non-Git resources use application-specific export/import playbooks

---

## 11. Registries & Artifact Policy

**Provider Registries**
- **AWS:** ECR
- **Azure:** ACR
- **GCP:** Artifact Registry
- Optional: GHCR for internal projects

**Policy Enforcement**
- Enforce trusted registries via Kyverno
- Require **image signatures** and attestations (build provenance, SBOM)
- Verify signatures at admission (Cosign + Kyverno verifyImages)

---

## 12. DNS, Certificates & Domains

**Naming Convention**
- Per-project subdomains: `<project>.<env>.<region>.<company-domain>`
- Examples: `retail.prd.us-east-1.myliferaypaas.com`

**Certificate Management**
- **cert-manager** ClusterIssuer for Let's Encrypt
- DNS-01 challenge using Route 53/Azure DNS/Cloud DNS
- Support custom domains via delegation and per-project Issuers

---

## 13. Bootstrap & Automation (Admin Path)

**Step 0 — Choose Runtime**
- Provisioning CLI asks: **Local** or **Cloud (AWS/Azure/GCP)**
- Capture topology choices (multi-AZ policy, DB/search layouts, VM classes)

**Step 1 — Terraform Infrastructure**
- VPC/VNet, subnets, NAT, gateways, security groups/NSGs
- IAM/service principals
- Managed DB/Search (per chosen layout), object storage, DNS, registries
- EKS/AKS/GKE cluster and node pools

**Step 2 — Cluster Bootstrap**
- Install: CNI, storage CSI, ingress, cert-manager, ESO, Kyverno, Argo CD, **Cockpit**
- Create ClusterSecretStore, ClusterIssuer, AppProjects, baseline namespaces
- Install observability agents (Alloy) with remote_write/log sinks

**Step 3 — Project Enablement**
- Create project skeleton repo with Helm values, quotas, RBAC, sample apps
- Onboard Owners/Members with least-privilege

**All steps are idempotent and PR-driven where possible.**

---

## 14. Naming, Tagging & Cost Controls

**Global Tags (all clouds)**
- `lc2:project`, `lc2:env`, `lc2:owner`, `lc2:tenant`, `lc2:region`
- Enable cost allocation and ownership tracking

**Naming Convention**
- Keep cluster, node pool, and infra resource names short, deterministic, and human-readable

**Cost Controls**
- Budgets and alerts per project
- Public egress controls
- Registry mirroring to reduce costs
- Spot/preemptible pools (opt-in via tolerations)

---

## 15. Security & Compliance

**Baseline Hardening**
- PSS **Restricted** in production namespaces; **Baseline** in dev/uat
- Default-deny **NetworkPolicies** with curated allowlists
- Sign all release images; verify on admission
- Rotate credentials and keys regularly
- Use workload identity/federation where available

**Audit**
- Control plane and RBAC audit logs enabled
- Git history for config changes
- Argo events/sync history retained per compliance baseline

---

## 16. Acceptance Checklist (Per Cluster)

- [ ] Terraform plan/apply completed for selected topology and sizes
- [ ] CSI drivers installed and storage classes configured
- [ ] Ingress, cert-manager, ESO, Kyverno, Argo CD, Cockpit installed and healthy
- [ ] ClusterSecretStore and ClusterIssuer configured; test cert issued
- [ ] Managed Prometheus remote_write operational; logs visible in provider console
- [ ] AppProjects and RBAC scoped for first project; quotas/limits set
- [ ] Sample app deployed in dev/uat/prd; failover/PDBs validated
- [ ] Backup retention and snapshot policies confirmed for DB/Search
- [ ] Cost tags and budgets in place

---

## 17. Open Questions & Refinements

- Thresholds for promoting projects from shared dev/uat DB/Search to dedicated instances
- Standard node SKUs per region to balance cost/performance for DXP
- Loki per-project in cloud vs vendor logs only (finalize by cost model)
- Optional service mesh (mTLS/traffic shaping) — only if justified by use cases

---

**One-liner:** *LC2 infrastructure provides cloud-agnostic foundation across Local/AWS/Azure/GCP with consistent GitOps, enforced isolation, managed services for durability, workload identity for security, and KISS principles throughout — all provisioned via Terraform and bootstrapped via Argo CD.*

