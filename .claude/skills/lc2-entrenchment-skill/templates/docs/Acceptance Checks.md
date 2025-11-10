# Acceptance Checks

- [ ] All config and policy changes originate from pull requests (no direct changes).
- [ ] Runtime operations are mediated by platform controllers; no direct kubectl access for tenants.
- [ ] No PVCs without a recorded exception and protection policy.
- [ ] Local ↔ cloud parity verified for GitOps flows and isolation semantics.
- [ ] Secrets never appear in PRs or logs; rotations are recorded with correlation IDs.
- [ ] Audit trail exists linking PR → controller action → deployment result.
- [ ] Admin dashboards contain no tenant data; tenant dashboards are scoped by project and environment.
