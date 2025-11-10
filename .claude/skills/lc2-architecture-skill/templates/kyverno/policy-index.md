# Kyverno Policy Index (architecture view)

## Cluster guardrails (admin-owned)
- Pod Security Standards: Restricted (enforce)
- Default‑deny NetworkPolicy (generate + backfill existing)
- ResourceQuota and LimitRange per namespace
- Supply chain: registry allow‑list; disallow :latest; verifyImages with signatures/attestations
- Service exposure: forbid NodePort; control LoadBalancer via annotation
- PolicyException: narrow, time‑boxed, namespaced

## Tenant policies (project-owned)
- Labels/annotations contract (required keys)
- Egress allow‑lists (deny wildcard 0.0.0.0/0)
- Secrets discipline (prefer ExternalSecret; allow TLS kubernetes.io/tls)
- Image pull policy by environment

Implementation YAML should live in your kyverno repo; this index is the architectural contract.
