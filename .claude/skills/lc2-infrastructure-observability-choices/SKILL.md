---
name: lc2-observability-choices
description: Adopt a portable local stack (Alloy/Loki) and managed backends in cloud, with consistent labeling and isolation.
allowed-tools: Read, Grep, Glob
---

# LC2 Observability Choices

## Goal
Keep observability portable from Local to Cloud while allowing managed services in production.

## Guidance
- Local: Alloy/Loki + Grafana for portability.
- Cloud: Prefer managed metrics backends (e.g., Managed Prometheus/Mimir) with agents shipping metrics/logs.
- Always expose deep links from Cockpit to the provider/stack dashboards instead of duplicating graphs.

## Validation
- Ensure all Kubernetes deployment groups have log coverage (label-based checks).
- Verify retention and isolation policies per tenant/project/environment.
