# Principles

- Portability: same manifests and agents on local and cloud; dual‑write optional for logs parity.
- Isolation by default: namespaces, quotas, default‑deny networking, admission policies, per‑project Grafana org.
- KISS: keep the control plane small and predictable; prefer shared OSS add‑ons with clear boundaries.
- GitOps‑first: all changes flow through Git; Argo CD reconciles the cluster.
- Cloud‑agnostic: prefer provider‑neutral components; wrap managed backends behind consistent data sources.
