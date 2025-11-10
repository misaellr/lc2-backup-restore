---
name: kyverno-guardrails
description: >
  Generate and maintain Kyverno policy packs for LC2 with strong tenant isolation and KISS guardrails.
  Use this when we need cluster-wide non-negotiables (PSS, default-deny, quotas, supply chain)
  and per-project conventions (labels, ESO-first secrets, egress allow-lists), plus docs and tests.
# allowed-tools omitted so Claude Code can write files when you approve changes.
---

# LC2 Kyverno Guardrails

## goals
- Isolation by default for each project (dev/uat/prd) while keeping developer freedom.
- KISS: a small set of cluster-wide **non‑negotiables** + project‑scoped conventions.
- Portability: policies work on local (K3D) and managed clouds (EKS/AKS/GKE).
- GitOps‑friendly: predictable diffs with Argo CD; avoid policy/manifest “tug‑of‑war”.

## inputs i will look for (read-only)
- `Architecture.md`, `Observability.md` (principles and wiring)
- `Kyverno-Policy-Simple.md` (prior decisions)
- Repo layout (e.g., `kyverno/`, `argocd/`, `charts/`)
- Project list (optional): `cockpit.projects.json` or folders under `projects/`

If something is missing, I will propose a minimal layout before writing anything.

## outputs i will create (with your confirmation)
- `kyverno/cluster/*.yaml` — ClusterPolicies (non‑negotiables)
- `kyverno/tenant/*.yaml` — namespaced Policies (templated per project)
- `kyverno/README.md` — rationale, test commands, rollout (audit → enforce → tighten)
- `argocd/appprojects/<project>.yaml` (optional scaffold) and `argocd/rbac/argocd-rbac-cm.yaml` tips
- A short **policy matrix** (who can do what) and **exception workflow**

## cluster guardrails (generated under kyverno/cluster)
1) **Pod Security Standards — Restricted (Enforce)**  
   Single-rule PSS Restricted profile cluster-wide (runAsNonRoot, no privilege escalation, seccomp RuntimeDefault, minimal caps).  
   _Reference: Kubernetes PSS; Kyverno PSS policy samples._

2) **Default‑deny NetworkPolicy (Generate + Backfill)**  
   Generate a `default-deny` NetworkPolicy for every Namespace on creation, and backfill existing Namespaces (`generateExisting: true`).  
   _Reference: Kyverno generate NetworkPolicy (existing/new)._

3) **Resource Governance**  
   Auto-generate `ResourceQuota` and `LimitRange`. Require explicit `resources.requests/limits` on all containers.

4) **Supply Chain Hygiene**  
   - Restrict image **registries** to an allow‑list.  
   - Disallow `:latest` tag.  
   - Verify image **signatures**/attestations (Cosign) with `verifyImages`; prefer **digests** in UAT/PRD.

5) **Service Exposure & Ingress**  
   Forbid `NodePort`; default `ClusterIP`. Allow `LoadBalancer` only with an explicit allow annotation. Enforce TLS in UAT/PRD.

6) **Policy Exceptions**  
   Enable **namespaced, time‑boxed** `PolicyException` with tight match/exclude selectors and owner approval notes.

7) **Argo CD friendliness** (notes in README)  
   Recommend `syncOptions: [Replace=true]` for Kyverno CRDs and `ignoreDifferences` for aggregated roles / Kyverno-managed fields.

## tenant conventions (generated under kyverno/tenant)
1) **Labels & Annotations Contract (mutate/validate)**  
   Ensure required keys exist and are normalized:  
   `app.kubernetes.io/{name,component,instance}` and `cockpit.liferay.io/{project,env,app}`.

2) **Egress allow‑list**  
   Validate that egress is explicitly allowed to known endpoints (DB/SaaS); deny wildcards like `0.0.0.0/0`.

3) **Secrets discipline (ESO‑first)**  
   Encourage `ExternalSecret` over raw `Secret` (Audit → Enforce). Deny cross-namespace secret references. Prefer per‑namespace `SecretStore` and limit `ClusterSecretStore` usage.

4) **Image pull policy by environment**  
   DEV uses `IfNotPresent` (faster); UAT/PRD require digests and/or strict tag rules.

5) **Ingress TLS by environment**  
   DEV may allow HTTP; UAT/PRD require TLS and approved host patterns.

## file layout i will create
```
kyverno/
  cluster/
    pss-restricted.yaml
    netpol-default-deny-generate.yaml
    netpol-default-deny-existing.yaml
    quota-generate.yaml
    limitrange-generate.yaml
    restrict-registries.yaml
    disallow-latest-tag.yaml
    verifyimages-signatures.yaml
    service-exposure-validate.yaml
    policyexception-guardrails.yaml

  tenant/
    labels-contract.yaml
    egress-allowlist-validate.yaml
    steer-secrets-to-eso.yaml
    imagepullpolicy-by-env.yaml
    ingress-tls-by-env.yaml

kyverno/README.md
```

## templates i will start from (snippets)
> **PSS Restricted (ClusterPolicy)**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: pss-restricted
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: pss-restricted
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        podSecurity:
          level: restricted
          version: latest
```
> **Default‑deny for new namespaces (Generate)**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-networkpolicy-default-deny
spec:
  background: true
  rules:
    - name: add-default-deny
      match:
        any:
          - resources:
              kinds: ["Namespace"]
      generate:
        kind: NetworkPolicy
        apiVersion: networking.k8s.io/v1
        name: default-deny
        namespace: "{{ request.object.metadata.name }}"
        data:
          spec:
            podSelector: {}
            policyTypes: ["Ingress", "Egress"]
```
> **Backfill default‑deny to existing namespaces**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-networkpolicy-default-deny-existing
spec:
  generateExisting: true
  rules:
    - name: add-existing-default-deny
      match:
        any:
          - resources:
              kinds: ["Namespace"]
      generate:
        kind: NetworkPolicy
        apiVersion: networking.k8s.io/v1
        name: default-deny
        namespace: "{{ request.object.metadata.name }}"
        data:
          spec:
            podSelector: {}
            policyTypes: ["Ingress", "Egress"]
```
> **Disallow :latest tag**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: no-latest
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Do not use the mutable :latest tag."
        pattern:
          spec:
            containers:
            - image: "!*:latest"
```
> **Verify images (Cosign)**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: verify-signed
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "ghcr.io/your-org/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      REPLACE_WITH_YOUR_KEY
                      -----END PUBLIC KEY-----
```

## rollout (phased)
1) **Audit**: enable background scans and PolicyReports; fix drift.
2) **Enforce**: flip cluster guardrails (PSS/default‑deny/quotas/supply‑chain) to Enforce.
3) **Tighten**: move tenant‑side policies (ESO‑first, egress) from Audit → Enforce after one clean sprint.

## argo cd & gitops tips
- Set `syncOptions: [Replace=true]` for Kyverno CRDs; consider `ignoreDifferences` for aggregated roles/generated fields.
- Keep one **shared Argo CD** per cluster; use **AppProjects** to fence repos/namespaces and **RBAC** to scope users/groups.
- Avoid “flip‑flop” with generated resources (`synchronize: true`) vs Argo deletions; prefer clear ownership and patterns per resource.

## examples (how to ask me)
- "Generate cluster guardrails and tenant policies for project **acme** (dev/uat/prd)."
- "Add verifyImages for ghcr.io/liferay/** with our Cosign public key."
- "Switch secrets to ESO-first and warn on raw Secrets in UAT/PRD."
- "Backfill default-deny to all existing namespaces; keep audit only for now."

## safety & caveats
- NetworkPolicies require an enforcing CNI (e.g., Calico).
- verifyImages needs reachable signature stores and properly distributed keys.
- PolicyException must be narrowly scoped and time‑boxed.
- Test policies with **kyverno CLI** before enforcing.

## minimal tests i will write to README
```bash
# dry-run validate workloads in a namespace
kyverno apply kyverno/cluster/*.yaml -n acme-uat --resource manifests/deploy-acme.yaml

# background scan existing resources
kyverno apply kyverno/cluster/*.yaml --policy-report --audit-warn

# generate preview only
kyverno apply kyverno/cluster/netpol-default-deny-generate.yaml --resource ns.yaml --dry-run
```
