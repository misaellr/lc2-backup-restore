# Architecture (LC2)

## Purpose
Explain the LC2 architecture across four layers and the design choices that enable **portability (local ↔ cloud)**, **isolation (tenant safety)**, and the **KISS principle** (Keep it simple and straightforward).

## Principles
- Portability
- Isolation by default
- KISS — Keep it simple and straightforward
- GitOps‑first
- Cloud‑agnostic components

## Layered overview
1. Infrastructure (Local, AWS, Azure, GCP)
2. Kubernetes (K3D, GKE, AKS, EKS)
3. PaaS (Cockpit) — management plane
4. Project (environments and application)

## 1) Infrastructure
- Networking and identity (VPC/VNet, subnets, routing, OIDC/SSO)
- TLS endpoints (ACME or cloud certs consumed by cert-manager)
- Managed data planes: metrics (AMP/GMP/Azure), logs (CloudWatch/Cloud Logging/Log Analytics), object storage (S3/GCS/Blob)
- Local parity: Alloy + Loki/Mimir/Prometheus

## 2) Kubernetes
- Shared add‑ons: Argo CD (OSS), Kyverno, cert‑manager, ExternalDNS, External Secrets Operator, ingress controller
- Local: optional Loki/Mimir
- Cloud: managed metrics/logs with Alloy shipping
- Interfaces: up to infrastructure (LBs, certs, DNS, secret stores); down to PaaS and project

## 3) PaaS (Cockpit)
- Provisions projects (namespaces, quotas, AppProjects), wires SecretStores/ExternalSecrets, deep‑links to Argo and Grafana
- GitOps only; no direct kubectl
- Grafana (OSS) shared with organization per project; data sources scoped

## 4) Project (environments and application)
- Namespaces per env: `<project>-dev|uat|prd`, quotas/LimitRanges, default‑deny network policies, RBAC
- Workloads (Liferay DXP and others); configs; ExternalSecret → Secret
- Grafana Alloy agents per node/pod; telemetry routing to the chosen backends

## Shared vs per‑project matrix
- Argo CD: shared; isolation via AppProject + RBAC
- Kyverno: controller shared; policies apply per project
- Grafana: shared; organization per project
- Loki: shared backend (optional); isolation via X-Scope-OrgID
- Managed metrics/logs: shared services; project scoping via credentials/labels
- Namespaces, quotas, RBAC, apps: per project

## Acceptance criteria
- AppProjects prevent cross‑namespace drift
- Kyverno baseline enforced (PSS, quotas/limits, default‑deny, registry allow‑list, verify images)
- Grafana shows only tenant data within its organization
- Local ↔ cloud diffs limited to data‑source targets
