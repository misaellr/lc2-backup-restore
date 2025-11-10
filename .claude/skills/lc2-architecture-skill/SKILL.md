---
name: lc2-architecture
description: Scaffold or audit the LC2 architecture (infrastructure, Kubernetes, PaaS Cockpit, project). Produces Architecture.md, ADRs, Argo CD app-of-apps skeleton, Kyverno policy index, naming conventions, and checklists. Uses normal casing throughout. Designed for local (k3d) and cloud (EKS/AKS/GKE).
allowed-tools: Read, Grep, Glob, Edit, Write, Bash(git:*), Bash(kubectl:*), Bash(helm:*), Bash(kustomize:*), Bash(argocd:*), Bash(jq:*), Bash(yq:*)
---

# LC2 Architecture Skill

## intent
Provide a repeatable workflow to **bootstrap or audit** LC2’s four-layer architecture with **portability, isolation, and KISS**. This skill writes documentation and safe scaffolding (Git-first) and can optionally create Argo CD objects when `apply=true`.

## inputs (collect or infer)
- project id (e.g., `acme`)
- environments (default: `dev, uat, prd`)
- runtime: `local | aws | azure | gcp`
- cluster name and region (when cloud)
- shared vs per-project toggles (Argo CD shared, Grafana OSS shared with org-per-project; Loki optional)
- managed services preferences (managed Prometheus/logs vs self-hosted)
- namespace and label conventions

## outputs (what this skill creates)
- `templates/docs/Architecture.md` (generated from your choices, normal casing)
- `templates/docs/Principles.md` (Portability, Isolation, KISS, GitOps)
- `templates/docs/Naming Conventions.md` (namespaces, labels, annotations, hostnames)
- `templates/docs/Checklists/Architecture Audit.md` (coverage checklist)
- `templates/docs/ADR/adr-template.md` and example ADRs
- `templates/argocd/app-of-apps.yaml` and `templates/argocd/appproject-<project>.yaml`
- `templates/kyverno/policy-index.md` (policy catalog + links; YAML kept separate per your policy repo)
- `templates/kubernetes/project-layout.yaml` (namespaces, quotas, rbac scaffolding)
- `templates/diagrams/Architecture ASCII.txt` (layered overview)

## safety guardrails
1. Never write secrets to Git. When needed, reference External Secrets (ESO).
2. Stage under `./.claude/scratch/architecture/` and show diff before promoting to your repo.
3. Only run `kubectl/helm/argocd` when `apply=true` is present.
4. Do not write outside the project folder.

## steps (new architecture init)
1. Detect runtime (`local | aws | azure | gcp`) and confirm four-layer model.
2. Generate **Architecture.md** and **Principles.md** reflecting your shared/per-project decisions.
3. Scaffold **Argo CD** app-of-apps and per-tenant **AppProject**.
4. Produce **Kyverno policy index** that lists mandatory cluster guardrails and tenant policies.
5. Create **Kubernetes project layout** (namespaces per env, quotas, RBAC placeholders).
6. Emit **ADR template** and starter ADRs (e.g., “Use managed Prometheus in cloud”, “Grafana OSS only”). 
7. Write an **audit checklist** to validate isolation and portability.
8. Optionally apply GitOps objects when `apply=true`.

## steps (audit mode)
- Verify the presence and correctness of files above.
- Check Argo CD **AppProject** fences (repos, namespaces, allowed kinds).
- Confirm Kyverno baseline is declared and enforced (PSS, quotas, default-deny, supply chain).
- Validate namespace and labeling conventions.
- Report gaps and propose diffs.

## acceptance checks (automatic)
- Git contains Architecture.md and Principles.md with four-layer sections.
- Argo CD project limits destinations to the tenant namespaces.
- Kyverno policy index includes PSS Restricted, default-deny, quotas/limits, registry allow-list, and verify images.
- Namespaces exist for each environment or are declared in Git for Argo to reconcile.
- All docs use **normal casing** (no camel case only).

## commands (only when apply=true)
- `kubectl apply -f templates/argocd/*.yaml`
- `argocd app create/sync` (if Argo CD API is reachable)
- `kubectl apply -f templates/kubernetes/project-layout.yaml` (for local PoC only; in cloud use GitOps PR)

## examples (triggers)
- "initialize lc2 architecture for project acme on eks with managed prometheus and cloudwatch"
- "audit architecture for project beta on gke; list argo fences and kyverno gaps"

## notes
- Argo CD is shared (OSS) with **AppProjects** and RBAC for tenant isolation.
- Grafana is **OSS** shared; each project uses its own organization and provisioned data sources.
- Loki is optional; when used, isolation relies on the `X-Scope-OrgID` header at the gateway.
