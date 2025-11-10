---
name: lc2-shared-references
description: Shared references for LC2 cloud-agnostic deployment. Use whenever LC2 Skills need authoritative context (lessons, IAM matrix, requirements, changelog).
allowed-tools: Read, Grep, Glob
---

# LC2 Shared References

## Purpose
Centralize authoritative LC2 materials so other Skills can reference them without duplicating content.

## Files
- [resources/lessons-learned.md](resources/lessons-learned.md)
- [resources/iam-permissions-matrix.md](resources/iam-permissions-matrix.md)
- [resources/requirements.md](resources/requirements.md)
- [resources/changelog.md](resources/changelog.md)

## How To Use
- When a Skill asks for LC2 context, first open the relevant file(s) from **resources/**.
- Prefer reading specific sections with a quick grep before loading entire files.
- Do not quote large chunks verbatim; summarize the relevant points.
- Treat these files as the single source of truth for LC2 guardrails and conventions.

## Notes
Keep this Skill read-only. Any edits to canonical documents should be made in the main repo and then synced here.
