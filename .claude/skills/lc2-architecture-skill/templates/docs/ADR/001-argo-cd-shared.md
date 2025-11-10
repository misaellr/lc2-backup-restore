# Use Argo CD OSS shared with AppProjects

- **Status:** accepted
- **Context:** need GitOps control plane with tenant isolation
- **Decision:** single Argo CD instance per cluster; isolation via AppProjects and RBAC
- **Consequences:** simpler ops; tenants cannot affect cluster scope
- **Alternatives:** per‑project Argo CD instances (more moving parts)
