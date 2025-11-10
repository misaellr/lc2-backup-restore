# Kyverno Policy Index for LC2

This document catalogs the required Kyverno policies for Liferay Cloud 2.0 (LC2) to enforce security, isolation, and resource management guardrails.

## Policy Categories

1. **Pod Security Standards** - Enforce PSS Restricted profile
2. **Resource Management** - Quotas, limits, and requests
3. **Network Isolation** - Default-deny and allowed egress
4. **Registry Control** - Allow-list for container images
5. **Label and Annotation Requirements** - Standard metadata
6. **Supply Chain Security** - Image verification (future)

---

## 1. Pod Security Standards (PSS)

### Policy: `pss-restricted`
**Purpose**: Enforce Kubernetes Pod Security Standards at Restricted level
**Scope**: Cluster-wide
**Validation Mode**: Enforce
**Background Mode**: true

**What it enforces**:
- No privileged containers
- No host path mounts
- No host network, host PID, host IPC
- Drop all capabilities, allow only NET_BIND_SERVICE
- Run as non-root user
- Read-only root filesystem
- No privilege escalation
- Seccomp profile required (RuntimeDefault or Localhost)

**Example violation**:
```yaml
# ❌ This will be rejected
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        privileged: true  # ❌ Not allowed
```

**Compliant example**:
```yaml
# ✅ This will be accepted
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: nginx
      image: nginx
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

**Policy Location**: `policies/cluster/pss-restricted.yaml`

---

## 2. Resource Management

### Policy: `require-resource-requests-limits`
**Purpose**: Ensure all pods have CPU and memory requests/limits
**Scope**: Cluster-wide (excluding kube-system, kyverno, argocd)
**Validation Mode**: Enforce
**Background Mode**: true

**What it enforces**:
- Every container must have `resources.requests.cpu`
- Every container must have `resources.requests.memory`
- Every container must have `resources.limits.cpu`
- Every container must have `resources.limits.memory`

**Example violation**:
```yaml
# ❌ This will be rejected
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          # ❌ Missing resources
```

**Compliant example**:
```yaml
# ✅ This will be accepted
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

**Policy Location**: `policies/cluster/require-resource-requests-limits.yaml`

---

### Policy: `enforce-resource-quotas`
**Purpose**: Ensure every project namespace has a ResourceQuota
**Scope**: Namespaces matching `*-dev`, `*-uat`, `*-prd`
**Validation Mode**: Audit (warn only, cockpit creates quotas)
**Background Mode**: true

**What it validates**:
- Namespace must have at least one ResourceQuota object
- Quota must define limits for CPU, memory, and PVCs

**Policy Location**: `policies/cluster/enforce-resource-quotas.yaml`

---

## 3. Network Isolation

### Policy: `default-deny-network-policy`
**Purpose**: Automatically create default-deny NetworkPolicy in new namespaces
**Scope**: All namespaces except kube-system, ingress-nginx
**Validation Mode**: Generate (creates resource)
**Background Mode**: true

**What it does**:
When a new namespace is created, automatically generate:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <new-namespace>
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Policy Location**: `policies/cluster/default-deny-network-policy.yaml`

---

### Policy: `require-explicit-egress`
**Purpose**: Ensure all egress is explicitly allowed via NetworkPolicy
**Scope**: Project namespaces (`*-dev`, `*-uat`, `*-prd`)
**Validation Mode**: Audit (warn only)
**Background Mode**: true

**What it validates**:
- Namespace must have at least one NetworkPolicy allowing egress
- Common egress destinations:
  - DNS (kube-dns/coredns on port 53)
  - Managed databases (port 5432, 3306, 1433)
  - HTTPS endpoints (port 443)

**Policy Location**: `policies/cluster/require-explicit-egress.yaml`

---

## 4. Registry Control

### Policy: `registry-allowlist`
**Purpose**: Only allow container images from approved registries
**Scope**: Cluster-wide (excluding kube-system, argocd, kyverno)
**Validation Mode**: Enforce
**Background Mode**: true

**Allowed registries**:
- `docker.io/liferay/*` - Official Liferay images
- `ghcr.io/liferay/*` - Liferay GitHub Container Registry
- `*.dkr.ecr.*.amazonaws.com/*` - AWS ECR (tenant-specific)
- `*.azurecr.io/*` - Azure Container Registry
- `*.gcr.io/*` - Google Container Registry
- `gcr.io/*` - Google public registry
- `registry.k8s.io/*` - Kubernetes official images
- `quay.io/*` - Red Hat Quay

**Example violation**:
```yaml
# ❌ This will be rejected
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: random-hub.io/untrusted/image:latest  # ❌ Not in allowlist
```

**Compliant example**:
```yaml
# ✅ This will be accepted
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: ghcr.io/liferay/cockpit-backend:0.1.0  # ✅ Allowed
```

**Policy Location**: `policies/cluster/registry-allowlist.yaml`

---

## 5. Label and Annotation Requirements

### Policy: `require-lc2-labels`
**Purpose**: Enforce standard LC2 labels on all resources
**Scope**: Project namespaces (`*-dev`, `*-uat`, `*-prd`)
**Validation Mode**: Enforce
**Background Mode**: true

**Required labels**:
```yaml
metadata:
  labels:
    app.kubernetes.io/name: "?*"        # Must exist
    app.kubernetes.io/instance: "?*"    # Must exist
    app.kubernetes.io/version: "?*"     # Must exist
    lc2.liferay.com/project-id: "?*"    # Must exist
    lc2.liferay.com/environment: "dev|uat|prd"  # Must match namespace
```

**Applies to resource kinds**:
- Deployment
- StatefulSet
- Service
- Ingress
- ConfigMap
- Secret

**Example violation**:
```yaml
# ❌ This will be rejected
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: acme-prd
  labels:
    app: myapp  # ❌ Missing standard labels
```

**Compliant example**:
```yaml
# ✅ This will be accepted
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: acme-prd
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: acme-myapp-prd
    app.kubernetes.io/version: 1.0.0
    lc2.liferay.com/project-id: acme
    lc2.liferay.com/environment: prd
```

**Policy Location**: `policies/cluster/require-lc2-labels.yaml`

---

### Policy: `require-owner-annotations`
**Purpose**: Ensure ownership and audit metadata on tenant resources
**Scope**: Project namespaces
**Validation Mode**: Enforce
**Background Mode**: true

**Required annotations**:
```yaml
metadata:
  annotations:
    lc2.liferay.com/owner: "<email>"         # Must be valid email
    lc2.liferay.com/created-by: "<email>"    # Must be valid email
    lc2.liferay.com/created-at: "<iso8601>"  # ISO 8601 timestamp
```

**Policy Location**: `policies/cluster/require-owner-annotations.yaml`

---

## 6. Supply Chain Security (Future)

### Policy: `verify-image-signatures`
**Purpose**: Require signed container images with Sigstore/Cosign
**Scope**: Production namespaces (`*-prd`)
**Validation Mode**: Enforce
**Background Mode**: false
**Status**: Planned for Phase 8

**What it will enforce**:
- All production images must be signed with Cosign
- Public key for verification provided via ConfigMap
- Reject unsigned or untrusted images

**Policy Location**: `policies/cluster/verify-image-signatures.yaml` (not yet implemented)

---

## Policy Deployment

### Installation
All policies are stored in the `lc2-gitops` repository under `kyverno/policies/`.

```bash
# Apply cluster-wide policies
kubectl apply -f kyverno/policies/cluster/

# Apply tenant-specific policies (created by Cockpit)
kubectl apply -f kyverno/policies/projects/{project-id}/
```

### Argo CD Management
Policies are managed via Argo CD Application:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno-policies
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/liferay/lc2-gitops
    path: kyverno/policies
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Policy Monitoring

### Metrics
Kyverno exports Prometheus metrics for policy violations:
```
kyverno_policy_results_total{policy="pss-restricted",result="fail"} 0
kyverno_policy_results_total{policy="registry-allowlist",result="fail"} 0
```

### Alerts
Grafana alerts configured for:
- **Critical**: PSS violations in production namespaces
- **Warning**: Missing resource requests/limits
- **Info**: Unapproved registries in dev namespaces

### Policy Reports
Query policy violations:
```bash
# List all policy violations
kubectl get policyreport -A

# Get violations in specific namespace
kubectl get policyreport -n acme-prd -o yaml
```

---

## Exemptions

Some resources may need to be exempted from policies:

### Example: Exempt monitoring tools from PSS
```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: pss-restricted
spec:
  rules:
    - name: restricted
      exclude:
        any:
          - resources:
              namespaces:
                - prometheus
                - grafana
              names:
                - "*-exporter"  # Prometheus exporters may need host access
```

### Exemption Request Process
1. Create ADR documenting why exemption is needed
2. Submit PR to `lc2-gitops` repository with policy update
3. Architecture review and approval required
4. Exemption logged in audit trail

---

## Testing Policies

### Dry-Run Mode
Test policies without enforcement:
```bash
# Set policy to Audit mode
kubectl patch clusterpolicy pss-restricted \
  -p '{"spec":{"validationFailureAction":"Audit"}}'

# Review violations
kubectl get policyreport -A | grep -E 'FAIL|WARN'
```

### Local Testing with Kyverno CLI
```bash
# Install kyverno CLI
kyverno version

# Test policy against manifest
kyverno apply policies/cluster/pss-restricted.yaml \
  --resource test-deployment.yaml

# Expected output: PASS/FAIL with violations
```

---

## Policy Lifecycle

### Creating New Policies
1. Draft policy YAML in `kyverno/policies/drafts/`
2. Test locally with kyverno CLI
3. Deploy to dev cluster in Audit mode
4. Monitor violations for 1 week
5. Adjust policy based on feedback
6. Promote to Enforce mode in dev
7. Deploy to uat → prd with approval

### Updating Existing Policies
1. Create ADR documenting change rationale
2. Update policy in Git with new version annotation
3. Test in dev first, monitor for regressions
4. Roll out to uat → prd

### Deprecating Policies
1. Document deprecation reason in ADR
2. Set policy to Audit mode for 30 days
3. Remove policy from cluster
4. Archive in `kyverno/policies/archived/`

---

## Reference

- [Kyverno Documentation](https://kyverno.io/docs/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Naming Conventions](../Naming%20Conventions.md)
- [Architecture Principles](../Principles.md)

---

## Revision History

| Version | Date       | Author | Changes                           |
|---------|------------|--------|-----------------------------------|
| 1.0     | 2025-11-02 | Claude | Initial Kyverno policy index      |
