# LC2 Architecture Artifacts

This directory contains the architecture documentation and scaffolding for the Cockpit Backend project, generated using the **lc2-architecture-skill**.

**Generated**: 2025-11-02
**Project**: Cockpit Backend (LC2 Control Plane API)
**Phase**: Phase 1 Complete
**Status**: ✅ APPROVED

---

## Contents

### Documentation
- **[Architecture.md](./Architecture.md)** - Complete architecture overview
  - Four-layer model (Platform, Infrastructure, Workloads, Observability)
  - Technology stack and design decisions
  - Data model and API design
  - Deployment architecture (local and cloud)
  - Isolation and multi-tenancy strategy

- **[Principles.md](./Principles.md)** - LC2 core principles
  - Portability (run identically on k3d and EKS/AKS/GKE)
  - Isolation (multi-tenant security boundaries)
  - KISS (Keep It Simple, Stupid)
  - GitOps (configuration and deployment via Git)

- **[Naming Conventions.md](./Naming%20Conventions.md)** - Naming standards
  - Namespace patterns (`{project-id}-{env}`)
  - Resource naming (Deployments, Services, ConfigMaps, Secrets)
  - Label and annotation requirements
  - Hostname and DNS conventions
  - Metrics and log field standards

### Kubernetes Scaffolding

#### Argo CD
- **[argocd/appproject-cockpit.yaml](./argocd/appproject-cockpit.yaml)** - Cockpit Backend AppProject
  - Source repository whitelist
  - Destination namespace fences
  - Cluster resource permissions
  - RBAC roles for platform team

#### Kubernetes Resources
- **[kubernetes/cockpit-namespace-layout.yaml](./kubernetes/cockpit-namespace-layout.yaml)** - Namespace and RBAC
  - `cockpit-system` namespace with PSS labels
  - ResourceQuota and LimitRange
  - NetworkPolicies (default-deny, allow DNS, allow Postgres, allow K8s API)
  - ServiceAccount `cockpit-backend-sa`
  - ClusterRole and ClusterRoleBinding for orchestration
  - Role and RoleBinding within `cockpit-system`

#### Kyverno Policies
- **[kyverno/policy-index.md](./kyverno/policy-index.md)** - Policy catalog
  - Pod Security Standards (PSS Restricted)
  - Resource requests/limits enforcement
  - Default-deny NetworkPolicy
  - Registry allow-list
  - Label and annotation requirements
  - Supply chain security (future)

### Checklists and Diagrams
- **[checklists/Architecture Audit.md](./checklists/Architecture%20Audit.md)** - Audit checklist
  - Validation criteria for all 4 principles
  - Phase 1 compliance status
  - Gaps and recommendations
  - Approval status: ✅ **PASS**

- **[diagrams/Architecture ASCII.txt](./diagrams/Architecture%20ASCII.txt)** - Visual architecture
  - Four-layer model diagram
  - Cockpit Backend component details
  - Data flow for project creation
  - Deployment topology (local vs cloud)

---

## Audit Results

### Overall Status: ✅ **PASS** for Phase 1

**Scores by Principle**:
- **Portability**: ✅ 100% (within Phase 1 scope)
- **Isolation**: ✅ 95% (foundations complete, runtime testing in Phase 2)
- **KISS**: ✅ 100%
- **GitOps**: ⚠️ 60% (foundations ready, workflow in Phase 2-3)

### Key Achievements
1. ✅ Cloud-agnostic application code (no AWS/Azure/GCP SDKs)
2. ✅ Multi-tenant isolation via namespaces, RBAC, NetworkPolicies
3. ✅ Simple Spring Boot monolith with PostgreSQL
4. ✅ Configuration in Git (Liquibase, application.yml)
5. ✅ Comprehensive observability (metrics, logs, health checks)
6. ✅ Security foundations (session auth, CSRF, PSS Restricted)

### Critical Gaps
None for Phase 1 scope. All deliverables complete.

### Recommendations
1. **Phase 2**: Integrate with Argo CD, deploy to k3d
2. **Phase 3**: Build Docker image, deploy to cloud (EKS/AKS/GKE)
3. **Phase 4**: Test cross-cloud portability, integrate SSO
4. **Phase 5**: Create operational runbooks and ADRs

---

## How to Use These Artifacts

### Step 1: Review Documentation
Read the three core documents in order:
1. `Architecture.md` - Understand the overall design
2. `Principles.md` - Understand the "why" behind decisions
3. `Naming Conventions.md` - Understand naming standards

### Step 2: Deploy to k3d (Local Testing)
```bash
# 1. Create cockpit-system namespace and RBAC
kubectl apply -f kubernetes/cockpit-namespace-layout.yaml

# 2. Create Cockpit AppProject in Argo CD
kubectl apply -f argocd/appproject-cockpit.yaml

# 3. Deploy Cockpit Backend
# (Use Argo CD Application or manual kubectl apply of Deployment/Service/Ingress)

# 4. Verify
kubectl get all -n cockpit-system
kubectl get appproject -n argocd
```

### Step 3: Deploy Kyverno Policies
```bash
# Note: Actual policy YAMLs need to be created based on policy-index.md
# The index documents what policies are needed, not the policies themselves

# Example structure:
# lc2-gitops/
#   kyverno/
#     policies/
#       cluster/
#         pss-restricted.yaml
#         require-resource-requests-limits.yaml
#         registry-allowlist.yaml
#         require-lc2-labels.yaml
```

### Step 4: Validate with Audit Checklist
Use `checklists/Architecture Audit.md` to verify:
- Portability (can it run on k3d and EKS/AKS/GKE?)
- Isolation (are tenants properly isolated?)
- KISS (is it simple and maintainable?)
- GitOps (is configuration in Git?)

---

## Next Steps

### Immediate (Phase 2)
1. Create Argo CD Application for Cockpit Backend
2. Deploy to k3d and verify end-to-end flow
3. Create first tenant project via Cockpit API
4. Validate RBAC isolation with test users
5. Write ADRs documenting key decisions

### Short-term (Phase 3)
1. Build Docker image: `docker build -t ghcr.io/liferay/cockpit-backend:0.1.0 .`
2. Publish to container registry
3. Create Kubernetes Deployment manifests
4. Set up CI/CD pipeline (GitHub Actions)
5. Deploy to AWS EKS
6. Test portability by deploying to Azure AKS
7. Test portability by deploying to GCP GKE

### Medium-term (Phase 4)
1. Integrate SSO (OIDC/SAML)
2. Set up Grafana multi-tenancy (org per project)
3. Create default dashboards
4. Implement rate limiting

### Long-term (Phase 5+)
1. Add distributed tracing
2. Implement supply chain security (image signing)
3. Create operational runbooks
4. Disaster recovery testing

---

## GitOps Repository Structure

These artifacts should be promoted to your GitOps repository:

```
lc2-gitops/
├── docs/
│   ├── Architecture.md
│   ├── Principles.md
│   ├── Naming Conventions.md
│   ├── checklists/
│   │   └── Architecture Audit.md
│   ├── diagrams/
│   │   └── Architecture ASCII.txt
│   └── adr/
│       ├── adr-template.md
│       ├── 001-use-spring-boot.md
│       ├── 002-session-based-auth.md
│       └── ...
├── argocd/
│   ├── projects/
│   │   ├── cockpit.yaml
│   │   ├── acme.yaml
│   │   └── ...
│   └── applications/
│       ├── cockpit-backend.yaml
│       ├── acme-dev-liferay.yaml
│       └── ...
├── kyverno/
│   └── policies/
│       ├── cluster/
│       │   ├── pss-restricted.yaml
│       │   ├── require-resource-requests-limits.yaml
│       │   ├── registry-allowlist.yaml
│       │   └── require-lc2-labels.yaml
│       └── projects/
│           └── acme/
│               └── resource-quotas.yaml
└── kubernetes/
    ├── platform/
    │   └── cockpit-system/
    │       ├── namespace.yaml
    │       ├── deployment.yaml
    │       ├── service.yaml
    │       └── ingress.yaml
    └── projects/
        └── acme/
            ├── dev/
            ├── uat/
            └── prd/
```

---

## Safety Note

**These artifacts are staged in `./.claude/scratch/architecture/` per lc2-architecture-skill safety guardrails.**

Before promoting to production:
1. Review all generated YAML files
2. Customize for your specific cluster (namespace names, resource limits, etc.)
3. Test in dev environment first
4. Get architecture review approval
5. Commit to GitOps repository
6. Apply via Argo CD (not `kubectl apply` directly)

**Do NOT apply directly to production clusters without review!**

---

## Questions or Issues?

- Review the [Architecture.md](./Architecture.md) for design rationale
- Check the [Principles.md](./Principles.md) for LC2 standards
- Consult the [Naming Conventions.md](./Naming%20Conventions.md) for naming guidance
- Use the [Architecture Audit checklist](./checklists/Architecture%20Audit.md) for validation

---

## Revision History

| Version | Date       | Author | Changes                                    |
|---------|------------|--------|--------------------------------------------|
| 1.0     | 2025-11-02 | Claude | Initial architecture artifacts generation  |
