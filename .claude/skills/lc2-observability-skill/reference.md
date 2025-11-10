# Reference

- isolation: org-per-project in Grafana; backend tenant header `X-Scope-OrgID` for loki/mimir where applicable.
- collectors: grafana alloy daemonset (logs) and remote_write (metrics).
- backends (local): loki + mimir; (cloud): cloud logs + managed prometheus (amp/gmp/azure), optional dual-write to loki.
- kyverno: generate default-deny NetPol, enforce labels, deny disabling logging, and steer secrets via ESO.
- acceptance: ≥ 99% coverage of running pods producing logs within 60s; strict tenant separation.
