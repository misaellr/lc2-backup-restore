# Entrenchment Clauses — Liferay Cloud 2.0

**Status:** Binding (Non‑negotiable)
**Applies to:** Product design, implementation, operations, and governance

---

## 0) Purpose
Protect the core values of Liferay Cloud 2.0: Isolation first, GitOps always, KISS, Kubernetes stateless by default, and local ↔ cloud parity. These clauses constrain decisions across the product.

> Simplicity and KISS. Simple things should be simple; complex things possible. We follow KISS — Keep it simple and straightforward — and avoid unnecessary complexity.

## 1) GitOps‑Only Change Paths
What
- Declarative changes (manifests, Helm values, policies) must be performed via Git pull requests.
- Runtime operations (sync/refresh/rollback) are executed server‑side by Cockpit‑mediated controllers.

Prohibitions
- Owners and Members (project personas) cannot use kubectl (including `kubectl logs`).
- End‑user CLIs/browsers must not hold cluster‑admin credentials.

Evidence
- PR links and change logs referencing the change; controller activity with correlation IDs.

## 2) Isolation Above All (Projects and Environments)
What
- Each project is a tenant; each environment (dev/uat/prd) maps to a dedicated namespace: `proj-<id>-<env>`.
- Under no circumstances may an Owner/Member mutate resources outside their project’s namespaces.

Evidence
- Scoped permissions, per‑project namespaces, and isolation tests showing no cross‑boundary mutations.

## 3) Kubernetes Should Be as Stateless as Possible (Local and Cloud)
What
- All primary durable data (databases, file store, search) lives outside Kubernetes on managed services (e.g., RDS/Azure PG, S3/Blob, OpenSearch/Elastic). Pods are disposable.
- PVCs are exceptions, not baseline. Any PVC usage must be explicitly justified, documented, and protected (snapshot/restore policy).

Why
- Simpler scaling, faster recovery, fewer moving parts, and consistent parity between local and cloud.

Evidence
- Catalog of durable systems bound through service endpoints and external storage, plus an exceptions log for PVCs.

## 4) Local ↔ Cloud Parity
What
- Maintain as much feature parity as practical between local (k3d/kind) and cloud (EKS/AKS/GKE).
- Core experiences (GitOps flows, isolation semantics, observability access) must behave consistently.

How
- Keep provider choices pluggable; keep workflows and guardrails identical.

Evidence
- Side‑by‑side runbooks and parity tests demonstrating equivalent workflows.

## 5) Minimal Cockpit Surface Area
What
- Prefer links to managed consoles over building custom UI.

Rules
- Observability is UI‑first: open managed Grafana (cloud) or Grafana OSS (local). No custom charts in Cockpit.
- Logs are CLI‑first: LC2 CLI tails/queries provider logging planes with strict scoping.
- Backups are externalized: DB PITR (RDS/Azure PG); file store via S3/Blob versioning; search snapshots/reindex. No PVCs by default.

Evidence
- Cockpit provides deep links and CLI pathways; no custom visualizations duplicating vendor UIs.

## 6) Prefer Cloud‑Managed Services (When on EKS/AKS/GKE)
What
- Prefer managed databases, search, and object storage on each cloud (e.g., Amazon RDS, Azure Database for PostgreSQL, OpenSearch/Elastic Cloud, S3/Blob).

How
- Bind applications via external secret synchronization; secret values never transit Cockpit.

Evidence
- Service bindings via secret references; no plaintext secrets in PRs or logs.

## 7) Secrets Handling (No Plaintext Exposure)
What
- Secrets are managed by external secret managers and synchronized by the platform; Cockpit shows metadata/status only.
- PRs/diffs/logs must redact sensitive values.

Rotation
- Initiated from Cockpit and fulfilled by the provider or synchronization controller.

Evidence
- Redaction verified by audits; rotation events logged with correlation IDs.

## 8) Auditability
What
- Every change/operation generates an immutable audit: who/what/when/where/scope/result, with correlation ID and links to PRs and runtime operations.

Evidence
- Central audit log with cross‑references to Git and runtime controllers.

## 9) Access and Operational Boundaries
What
- No Owner/Member kubeconfigs. All write/read operations are mediated by Cockpit (UI/API/CLI) and executed by controllers with scoped service accounts.
- Owners and Members may belong to multiple projects; permissions are project‑scoped. Members cannot remove Owners.

Evidence
- Access reviews and tests confirming boundaries.

## 10) No Cross‑Project Observability for Admins
What
- Admins (cluster) stand up plumbing and guardrails; they do not need cross‑project dashboards.

Why
- Reduces scope and risk; keeps responsibility with project teams.

Evidence
- Admin dashboards limited to platform health; tenant data is scoped to tenant organizations.

## 11) Security Baseline
What
- TLS everywhere; network stance is default‑deny with curated egress; hardened pod posture; image policy enforcement; resource quotas/limits; signed images where available.
- Drift detection and optional self‑heal are controller‑managed and auditable.
- No cross‑tenant data exposure; logs/dashboards scoped per project/environment.

Evidence
- Security benchmarks, posture reports, and policy conformance checks.

## 12) Dependency Policy
What
- Choose stable, widely adopted open source or cloud‑managed equivalents.
- New dependencies require: security review, maintenance plan, parity assessment (local vs cloud), and rollback plan.

Remove/replace dependencies that increase blast radius or violate parity/simplicity.

Evidence
- Dependency review logs, SBOMs, and deprecation plans.

## 13) Amending These Clauses
What
- Changes require platform governance approval and a recorded design review.
- Clauses 1 (GitOps‑Only) and 2 (Isolation) are foundational and cannot be relaxed without a major‑version decision and explicit stakeholder communication.

Evidence
- Design review artifacts and governance sign‑offs.

## 14) Acceptance Checks (Must Always Hold)
- 100% of config changes originate from PRs; 100% of runtime ops are mediated by platform controllers.
- No Owner/Member user or service account (except platform controllers) can mutate resources directly.
- Statelessness default holds: no PVCs in projects unless a documented exception exists.
- Local builds demonstrate the same workflows as cloud (within practical limits).
- Audit sampling confirms end‑to‑end traceability and redaction.

## 15) AI Contribution Attribution
What
AI tools may assist, but must not include themselves as named documentation contributors or Git commit authors.
