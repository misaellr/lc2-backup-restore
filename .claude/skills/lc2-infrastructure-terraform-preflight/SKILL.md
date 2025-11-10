---
name: lc2-terraform-preflight
description: Preflight environment checks for LC2 cloud-agnostic deployments across Local, AWS, Azure, and GCP.
allowed-tools: Read, Grep, Glob
---

# LC2 Terraform Preflight

## Goal
Ensure the local environment is correctly configured before running Terraform or Helm for Local, AWS, Azure, or GCP.

## Checklist
- Verify core tools are installed and meet minimum versions:
  - Terraform ≥ 1.6
  - kubectl ≥ 1.28
  - Helm ≥ 3.13
  - AWS CLI / Azure CLI / gcloud (as applicable)
- Confirm you are authenticated to the chosen cloud provider.
- On GKE, verify the **gke-gcloud-auth-plugin** is installed and on PATH.
- Verify the target kubecontext explicitly before any Kubernetes/Helm operation.
- Confirm required provider APIs/quotas (e.g., EIP/NAT on AWS; APIs enabled on GCP).
- Read the relevant sections in **lc2-shared-references** → lessons-learned.md for cloud-specific quirks.

## Commands (Copy-Paste Friendly)
```bash
terraform -version
kubectl version --client
helm version
aws --version || true
az version || true
gcloud --version || true
which gke-gcloud-auth-plugin || echo "Install: https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke"
kubectl config current-context
```

## Failure Handling
- If any prerequisite is missing, stop and fix it. Document fixes in the changelog.
- Prefer small, testable changes and re-run the preflight after each change.

## References
- See **lc2-shared-references** → lessons-learned.md for the preflight rationale.
