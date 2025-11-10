# Architecture Audit Checklist

- [ ] AppProject limits destinations to tenant namespaces only
- [ ] AppProject restricts source repos to approved orgs
- [ ] Kyverno PSS Restricted enforced on all tenant namespaces
- [ ] Default‑deny NetworkPolicy generated with backfill
- [ ] ResourceQuota and LimitRange present per namespace
- [ ] Registry allow‑list and `:latest` disallowed
- [ ] Image signatures/attestations verified (where configured)
- [ ] ESO in place; no raw Secrets except TLS
- [ ] Ingress/TLS/DNS standardized; NodePort forbidden
- [ ] Grafana OSS shared with organization per project
- [ ] Loki optional; if enabled, X-Scope-OrgID enforced at gateway
- [ ] Local ↔ cloud parity verified (agents, data sources)
