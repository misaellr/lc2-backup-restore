# Guardrails Matrix

| Clause | Summary | Applies to | Evidence artifacts | Owner |
| --- | --- | --- | --- | --- |
| 1 | GitOps-only change paths | All repos and runtime ops | PRs, controller activity logs, release notes | Platform governance |
| 2 | Project and environment isolation | Projects, namespaces, RBAC | Access tests, permissions reports | Platform governance |
| 3 | Stateless by default | Architecture, workloads | Exceptions registry for PVCs, storage mappings | Platform governance |
| 4 | Local ↔ cloud parity | Tooling, workflows | Parity tests, side-by-side runbooks | Platform governance |
| 5 | Minimal Cockpit surface | Product UX | Deep links, CLI guidance, no custom charts | Product + UX |
| 6 | Prefer managed services | Cloud deployments | Service bindings via external secrets | Architecture |
| 7 | Secrets handling | All envs | Redaction checks, rotation logs | Security |
| 8 | Auditability | Ops and changes | Audit log with PR links and correlation IDs | SRE |
| 9 | Access boundaries | Access control | Role mappings, membership rules | Security |
| 10 | No cross-project observability for admins | Observability | Scope checks, org boundaries | SRE |
| 11 | Security baseline | Security posture | Posture reports, benchmarks | Security |
| 12 | Dependency policy | Engineering | Reviews, SBOM, rollback plans | Architecture |
| 13 | Amending clauses | Governance | Design review records | Platform governance |
| 14 | Acceptance checks | Program | Scorecards, audits | Platform governance |
| 15 | AI attribution | Docs and Git | Author records and contribution logs | Platform governance |
