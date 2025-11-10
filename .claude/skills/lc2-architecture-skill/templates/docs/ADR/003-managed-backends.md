# Prefer managed metrics/log backends in cloud

- **Status:** accepted
- **Context:** reduce ops burden in cloud while keeping local parity
- **Decision:** AMP/GMP/Azure managed Prometheus; CloudWatch/Cloud Logging/Log Analytics for logs; optional Loki dual‑write
- **Consequences:** minimized ops; consistent dashboards via data source mapping
- **Alternatives:** self‑host everything
