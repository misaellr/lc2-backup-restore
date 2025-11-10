---
name: lc2-cluster-bootstrap
description: Obtain kube credentials, verify context, and install baseline add-ons (cert-manager, ingress) using idempotent Helm.
allowed-tools: Read, Grep, Glob
---

# LC2 Cluster Bootstrap

## Goal
After Terraform creates the cluster, prepare access and bootstrap baseline add-ons consistently across clouds.

## Tasks
- Fetch cluster credentials:
  - **EKS**: `aws eks update-kubeconfig --name <cluster> --region <region>`
  - **AKS**: `az aks get-credentials --name <cluster> --resource-group <rg> --overwrite-existing`
  - **GKE**: `gcloud container clusters get-credentials <cluster> --region <region>`
- Verify kubecontext: `kubectl config current-context` (must match target)
- Use `helm upgrade --install` for all add-ons.
- Install cert-manager and create a ClusterIssuer for Let's Encrypt (staging→prod).

## Add-Ons (Minimal)
- cert-manager (ClusterIssuer)
- Ingress controller (cloud-appropriate)
- External Secrets (optional, if used)

## Notes
- Keep this Skill conservative; do not embed org-specific charts. Link to your platform Helm registry instead.
