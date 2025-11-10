---
name: lc2-state-ops
description: Back up, restore, and surgically adjust Terraform state safely across AWS, Azure, and GCP backends.
allowed-tools: Read, Grep, Glob
---

# LC2 Terraform State Operations

## Goal
Treat Terraform state as a first-class artifact with safe backups, restores, and surgical repairs across clouds and regions.

## Practices
- Use remote backends per cloud (S3+DynamoDB, Azure Storage, GCS) and enable versioning/locks.
- Create timestamped snapshots before risky operations (migrations, refactors).
- Prefer `terraform import`/`state rm` for drift or resource type changes.
- Separate Infra (Terraform) from App (Helm/Argo) for easier rollbacks.

## Procedures
- **Snapshot**: Export the backend objects with a timestamp. Document the location.
- **Restore**: Validate the snapshot integrity, then restore and run `terraform plan` to verify.
- **Surgical Fix**: Use `terraform state mv`/`rm` and `import` to realign resources without tear-downs.

## Guardrails
- Never output secrets from state; use secret managers (AWS SM, Azure Key Vault, GCP Secret Manager).
- Prefer plan-validate-execute: draft changes, validate with a dry run, then apply.

## References
- See **lc2-shared-references** → lessons-learned.md (state and teardown guidance).
