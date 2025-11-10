---
name: lc2-observability
description: Configure or audit observability for Liferay Cloud 2.0 across local (k3d) and cloud (EKS/AKS/GKE). Use when the user asks to set up or verify logs, metrics, Grafana, or isolation for a project/runtime. Generates Alloy configs, wires Loki/Mimir or managed Prometheus, provisions Grafana org-per-project, creates Argo CD apps, and enforces Kyverno guardrails.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash(git:*), Bash(kubectl:*), Bash(helm:*), Bash(kustomize:*), Bash(argocd:*), Bash(jq:*), Bash(yq:*)
---

# LC2 Observability Skill

## intent
Provide a repeatable workflow to **bootstrap or audit** LC2 observability with strong **tenant isolation** and **KISS** defaults. Works for **local (k3d)** and **cloud (EKS/AKS/GKE)** run-times.

## inputs (collect or infer)
- project id: e.g., `acme`
- environments: `dev, uat, prd` (default)
- runtime: `local | aws | azure | gcp`
- cluster name / region
- log backend: `loki | cloudwatch | cloud-logging | log-analytics` (optional dual-write to loki)
- metrics backend: `mimir | cortex | managed-prometheus (amp/gmp/azure)`
- retention (days): dev=7, uat=14, prd=30 (defaults)
- per-tenant caps: streams, ingestion rate, query concurrency

## file layout this skill creates or edits (project repo)
- `observability/helm/`:
  - `alloy/values.yaml` (collector: logs/metrics; k8s metadata)
  - `loki/values.yaml` and `gateway/values.yaml` (when using OSS Loki)
  - `mimir/values.yaml` (when using OSS metrics backend)
  - `grafana/provisioning/{datasources,dashboards}/*`
- `argocd/apps/observability/*.yaml` (app-of-apps + appprojects)
- `kyverno/policies/observability/*.yaml` (labels, deny-disable-logging, default-deny netpol backfill)
- `overrides/<project>.yaml` (per-tenant limits + retention)

## safety guardrails
1. never write secrets to git. reference ESO secrets (external-secrets).
2. stage changes under `./.claude/scratch` first. show the diff. require confirmation to promote into `observability/`.
3. require an explicit `apply=true` flag before running any `kubectl/helm/argocd` commands.
4. block writes outside the repo folder.

## steps (new install)
1. detect runtime (`local|aws|azure|gcp`) and cluster identity.
2. scaffold Argo CD apps for observability (app-of-apps + tenant appprojects) using `templates/argocd-apps.yaml`.
3. generate Alloy values with namespace selectors `lc2.liferay.com/logging=enabled` and metadata enrichment for `project`, `env`, `app`.
4. choose backends:
   - **local:** loki (logs) and mimir or prometheus tsdb (metrics).
   - **cloud:** prefer cloud logs (cloudwatch / cloud logging / log analytics) and managed prometheus (amp/gmp/azure). optionally configure **dual-write** logs to loki for parity.
5. provision Grafana (OSS, shared per runtime) with **organization per project** and data sources scoped to the chosen backends.
6. create Kyverno policies:
   - generate default-deny `NetworkPolicy` for every project namespace (with `generateExisting`).
   - mutate/validate required labels; deny attempts to disable logging in project namespaces.
   - verify images (optional) and enforce https in uat/prd (optional).
7. configure the Loki or metrics **gateway** to inject tenant header (`X-Scope-OrgID=<project-id>`) and to **strip client-provided tenant headers**.
8. wire External Secrets for backend credentials (cloud endpoints, tokens).
9. commit staged files and open a PR.

## steps (audit mode)
- verify collectors present and discovering `/var/log/containers/*.log` for all project namespaces.
- verify Grafana org + data sources exist and are scoped.
- confirm gateway enforces header injection and blocks client-provided tenant headers.
- check per-tenant retention/limits and effective overrides.
- run the **coverage check**: count running pods in project namespaces and distinct log streams in backend for last 5 minutes. surface gaps.

## acceptance checks (automatic)
- ≥ 99% of running pods in project namespaces have active log streams (5‑minute window).
- tenant user from project **A** cannot query streams/series from project **B**.
- retention matches requested days per env; data older than TTL is absent.
- alloy agent scrape / remote_write is healthy (no backpressure) for the project.

## commands (only when apply=true)
- `helm upgrade -i` for alloy/loki/mimir/grafana charts with generated values.
- `kubectl apply -f` manifests for gateways, kyverno policies, and netpols.
- `argocd app create/sync` for app-of-apps and project appprojects.

## examples (triggers)
- "set up observability for project acme on eks (aws) with managed prom + cloudwatch"
- "audit observability for project beta on gke; show coverage and isolation results"
- "switch project acme logs to dual-write (cloudwatch + loki) and set prod retention to 90d"

## notes
- grafana must remain **OSS**; no enterprise features are assumed.
- prometheus server is not exposed to tenants; multi-tenant metrics are served by **mimir/cortex or managed prom** with remote_write.
- keep KISS: shared backends per runtime; project-level agents and orgs for isolation.
