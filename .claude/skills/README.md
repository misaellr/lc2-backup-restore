
# LC2 Claude Code Skills (Cloud-Agnostic Deployment)

This package provides **Agent Skills** for Claude Code to standardize and accelerate **Liferay DXP (LC2)** cloud-agnostic deployments across **Local, AWS, Azure, and GCP**.

## Installation
- Copy the `.claude/skills/` directory to the root of your repository **or** to your personal skills folder: `~/.claude/skills/`.
- Restart Claude Code to load new Skills.
- Skills will be auto-discovered and used when relevant to your prompts.

## Contents
- lc2-shared-references — canonical docs (Lessons Learned, IAM Matrix, Requirements, Changelog)
- lc2-terraform-preflight — environment verification before any plan/apply
- lc2-state-ops — safe state backups, restores, and surgical fixes
- lc2-iam-setup — least-privilege patterns and deployment vs runtime identities
- lc2-adapter-selection — pick stack + tfvars, keep module contracts neutral
- lc2-cluster-bootstrap — kubeconfig, context checks, cert-manager + ingress
- lc2-dns-tls-ingress — Let's Encrypt via cert-manager; uniform ingress
- lc2-database-optional — optional managed DBs; secret refs only
- lc2-observability-choices — portable local stack, managed cloud backends

## Notes
- **Skill names in frontmatter must be lowercase with hyphens** (Claude requirement). Headings and prose use normal casing.
- Keep Skills focused; avoid bundling too many capabilities into one.
- Update shared references by replacing files under `.claude/skills/lc2-shared-references/resources/`.

## Last Updated
2025-11-01
