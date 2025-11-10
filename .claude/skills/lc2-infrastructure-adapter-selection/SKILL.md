---
name: lc2-adapter-selection
description: Pick the correct stack and TFVars for the target cloud while keeping the module contracts cloud-neutral.
allowed-tools: Read, Grep, Glob
---

# LC2 Cloud Adapter & Stack Selection

## Goal
Choose the correct Terraform stack and adapter (Local, AWS, Azure, GCP) and bind the right environment variables/TFVars.

## Steps
1. Select the stack directory (e.g., `terraform/stacks/aws`).
2. Select the environment TFVars (e.g., `environments/dev/aws.tfvars`).
3. Confirm remote state backend matches the target cloud.
4. Ensure module interfaces are cloud-neutral; only adapters use provider specifics.
5. Document the selection in the PR description (stack + tfvars + commit SHA).

## Apply Pattern
```bash
terraform -chdir=terraform/stacks/<cloud> init
terraform -chdir=terraform/stacks/<cloud> plan -var-file=../../../environments/<env>/<cloud>.tfvars
terraform -chdir=terraform/stacks/<cloud> apply -var-file=../../../environments/<env>/<cloud>.tfvars
```

## References
- See **lc2-shared-references** → requirements.md for the normalized module contracts.
