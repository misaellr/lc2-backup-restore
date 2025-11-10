---
name: lc2-database-optional
description: Provision managed Postgres only when requested; otherwise keep local/dev light. Output secret references, not credentials.
allowed-tools: Read, Grep, Glob
---

# LC2 Optional Database Provisioning

## Goal
Make database provisioning optional and environment-aware, outputting only connection coordinates and secret references.

## Approach
- Enable managed Postgres per cloud (RDS, Azure Database for PostgreSQL, Cloud SQL) behind a uniform module interface.
- For Local/Dev, support a lightweight DB (e.g., SQLite or ephemeral Postgres).
- Surface only non-sensitive outputs; store credentials in the secret manager.

## Validation
- Smoke test connectivity from a throwaway pod.
- Confirm secrets are referenced, not printed, in Terraform outputs.

## References
- See **lc2-shared-references** → lessons-learned.md for DB trade-offs and testing notes.
