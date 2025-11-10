---
name: lc2-entrenchment
description: Establish, publish, and audit the Entrenchment Clauses for Liferay Cloud 2.0. This skill creates executive-level documents, guardrail matrices, PR templates, and scorecards. It is tool-agnostic (not Kyverno/Argo specific) and never talks to a cluster. Uses normal casing throughout.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash(git:*)
---

# LC2 Entrenchment Skill

## intent
Codify the non-negotiable principles of Liferay Cloud 2.0 (Isolation first, GitOps always, KISS, Kubernetes stateless by default, local ↔ cloud parity) as binding clauses, and keep them visible and auditable across repos and teams.

## inputs (collect or infer)
- product name and scope (default: Liferay Cloud 2.0)
- governance owners (e.g., Platform PM, Staff Engineer, SRE Lead)
- exception approvers and SLA (e.g., 7 days)
- review cadence (e.g., quarterly)
- repository paths where PR templates and CODEOWNERS should live

## outputs (what this skill creates)
- templates/docs/Entrenchment Clauses.md (authoritative, executive-facing document)
- templates/docs/Guardrails Matrix.md (clause → applies to → evidence artifacts → owner)
- templates/docs/Acceptance Checks.md (must-always-hold checks)
- templates/docs/Exceptions.md (process + starter registry CSV)
- templates/docs/Decision Log.md (how to record approvals and amendments)
- templates/docs/Change Governance.md (GitOps-only change paths and code ownership)
- templates/Checklists/Quarterly Review.md (operational review script)
- .github/PULL_REQUEST_TEMPLATE.md (entrenchment checklist)
- CODEOWNERS (place in repo root; governance owners as code owners)

## safety guardrails
1. This skill never runs kubectl/helm/argocd or touches cluster state.
2. This skill writes only within the project and templates/ folders or .github/ subfolder.
3. Secrets are out of scope. Documents and templates only.
4. Use normal casing in all generated files.

## steps (new standardization)
1. Create the executive Entrenchment Clauses document.
2. Emit the Guardrails Matrix mapping each clause to “applies to”, “evidence artifacts”, “owners”.
3. Create Acceptance Checks and a Quarterly Review checklist.
4. Add Change Governance guidance and Decision Log format.
5. Stage a .github/PULL_REQUEST_TEMPLATE.md and a CODEOWNERS skeleton.
6. Commit staged files and open a PR (only if apply=true is provided).

## steps (audit mode)
- Parse recent PRs and release notes to find references to clauses.
- Flag repos lacking PR templates/CODEOWNERS.
- Build a scorecard against Acceptance Checks and produce remediation actions.

## acceptance checks (automatic)
- 100% of config and policy changes originate from PRs (GitOps-only).
- No project persona has direct cluster-admin or kubectl access.
- Default stateless posture holds (PVCs require approved exception).
- Local ↔ cloud parity preserved for core workflows.
- Audit trail present with PR link and decision record.

## examples (triggers)
- “create the lc2 entrenchment package and PR template for our repo (apply=false)”
- “audit repos for entrenchment compliance and produce a scorecard (dry run)”

## notes
- Entrenchment is tool-agnostic: it constrains product and operational decisions, regardless of the underlying controllers or vendors.
