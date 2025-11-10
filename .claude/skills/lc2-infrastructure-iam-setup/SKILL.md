---
name: lc2-iam-setup
description: Apply least-privilege IAM mappings for AWS, Azure, and GCP using the LC2 IAM matrix and separate deploy/runtime identities.
allowed-tools: Read, Grep, Glob
---

# LC2 IAM Setup

## Goal
Enforce least-privilege by separating **deployment** and **runtime** identities per cloud.

## Instructions
1. Open **lc2-shared-references** → iam-permissions-matrix.md.
2. Identify the minimal roles for: networking, cluster (EKS/AKS/GKE), secrets, and database.
3. Create distinct accounts/identities for deployment (Terraform) vs runtime (workload identity / IRSA / MSI).
4. Store all credentials in the appropriate secret manager; do not print to logs or outputs.
5. Document the final mapping in the repo (no screenshots).

## Validation
- Run quick self-tests: whoami, can I create/read the minimal resources, do I fail on disallowed actions?
- Ensure IAM costs are zero and only created resources incur charges.

## Notes
- Avoid Owner/Administrator blanket roles.
- Prefer Workload Identity (GKE), IRSA (EKS), and Managed Identity (AKS) for in-cluster access.
