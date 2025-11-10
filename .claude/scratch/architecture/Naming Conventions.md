# LC2 Naming Conventions

This document defines the naming standards for namespaces, labels, annotations, and resources in Liferay Cloud 2.0 (LC2).

## Namespace Naming

### Pattern
```
{project-id}-{environment}
```

### Examples
- `acme-dev` - ACME Corp development environment
- `acme-uat` - ACME Corp user acceptance testing
- `acme-prd` - ACME Corp production
- `beta-dev` - Beta Inc development

### System Namespaces
These are reserved for platform components (not tenant projects):
- `cockpit-system` - Cockpit Backend control plane
- `argocd` - Argo CD GitOps engine
- `kyverno` - Policy enforcement
- `cert-manager` - TLS certificate automation
- `ingress-nginx` - Ingress controller
- `external-secrets-system` - Secret synchronization
- `grafana` - Shared Grafana OSS observability
- `loki` - Optional Loki log aggregation
- `prometheus` - Prometheus metrics (if self-hosted)

### Rules
1. **Project ID**: Lowercase alphanumeric + hyphens only (regex: `[a-z0-9-]+`)
2. **Environment**: One of `dev`, `uat`, `prd`
3. **Max Length**: 63 characters (Kubernetes limit)
4. **Reserved Prefixes**: `kube-*`, `default`, `kyverno`, `argocd`, `cockpit-*`

---

## Resource Naming

### Deployments
```
{app-name}
```

Examples:
- `cockpit-backend` - Cockpit API server
- `liferay-dxp` - Liferay Digital Experience Platform
- `postgres` - PostgreSQL database

### Services
Match deployment name:
```
{app-name}
```

Examples:
- `cockpit-backend` → Service exposes Deployment `cockpit-backend`
- `liferay-dxp` → Service exposes Deployment `liferay-dxp`

### Ingress
```
{app-name}-ingress
```

Examples:
- `cockpit-backend-ingress`
- `liferay-dxp-ingress`

### ConfigMaps and Secrets
```
{app-name}-{purpose}
```

Examples:
- `cockpit-backend-config` - Application configuration
- `cockpit-backend-db-secret` - Database credentials
- `liferay-dxp-license` - Liferay license key

### ServiceAccounts
```
{app-name}-sa
```

Examples:
- `cockpit-backend-sa`
- `liferay-dxp-sa`

---

## Label Standards

All resources MUST include these standard labels:

### Required Labels
```yaml
metadata:
  labels:
    app.kubernetes.io/name: <app-name>
    app.kubernetes.io/instance: <instance-id>
    app.kubernetes.io/version: <version>
    app.kubernetes.io/component: <component>
    app.kubernetes.io/part-of: <system>
    app.kubernetes.io/managed-by: <tool>
    lc2.liferay.com/project-id: <project-id>
    lc2.liferay.com/environment: <env>
```

### Label Definitions

| Label | Description | Example | Required |
|-------|-------------|---------|----------|
| `app.kubernetes.io/name` | Name of the application | `cockpit-backend`, `liferay-dxp` | Yes |
| `app.kubernetes.io/instance` | Unique identifier for this instance | `cockpit-backend-v1`, `acme-liferay` | Yes |
| `app.kubernetes.io/version` | Application version (SemVer) | `1.0.0`, `7.4.3.120` | Yes |
| `app.kubernetes.io/component` | Component role | `api`, `database`, `cache` | Yes |
| `app.kubernetes.io/part-of` | Logical application name | `cockpit`, `liferay-cloud` | Yes |
| `app.kubernetes.io/managed-by` | Tool managing this resource | `argocd`, `helm`, `kubectl` | Yes |
| `lc2.liferay.com/project-id` | Tenant project identifier | `acme`, `beta` | Yes (for tenant resources) |
| `lc2.liferay.com/environment` | Environment name | `dev`, `uat`, `prd` | Yes (for tenant resources) |

### Example: Cockpit Backend Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cockpit-backend
  namespace: cockpit-system
  labels:
    app.kubernetes.io/name: cockpit-backend
    app.kubernetes.io/instance: cockpit-backend-main
    app.kubernetes.io/version: 0.1.0
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: cockpit
    app.kubernetes.io/managed-by: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: cockpit-backend
      app.kubernetes.io/instance: cockpit-backend-main
  template:
    metadata:
      labels:
        app.kubernetes.io/name: cockpit-backend
        app.kubernetes.io/instance: cockpit-backend-main
        app.kubernetes.io/version: 0.1.0
        app.kubernetes.io/component: api
        app.kubernetes.io/part-of: cockpit
        app.kubernetes.io/managed-by: argocd
    spec:
      # ... pod spec
```

### Example: Tenant Liferay DXP Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liferay-dxp
  namespace: acme-prd
  labels:
    app.kubernetes.io/name: liferay-dxp
    app.kubernetes.io/instance: acme-liferay-prd
    app.kubernetes.io/version: 7.4.3.120
    app.kubernetes.io/component: application
    app.kubernetes.io/part-of: liferay-cloud
    app.kubernetes.io/managed-by: argocd
    lc2.liferay.com/project-id: acme
    lc2.liferay.com/environment: prd
spec:
  # ... deployment spec
```

---

## Annotation Standards

### Required Annotations
```yaml
metadata:
  annotations:
    lc2.liferay.com/owner: <owner-email>
    lc2.liferay.com/created-by: <creator-email>
    lc2.liferay.com/created-at: <iso8601-timestamp>
    argocd.argoproj.io/sync-wave: <wave-number>
```

### Annotation Definitions

| Annotation | Description | Example | Required |
|------------|-------------|---------|----------|
| `lc2.liferay.com/owner` | Current resource owner email | `alice@example.com` | Yes (for tenant resources) |
| `lc2.liferay.com/created-by` | Original creator email | `bob@example.com` | Yes (for tenant resources) |
| `lc2.liferay.com/created-at` | Creation timestamp | `2025-11-02T10:30:00Z` | Yes (for tenant resources) |
| `lc2.liferay.com/description` | Human-readable description | `Production Liferay DXP for ACME Corp` | No |
| `argocd.argoproj.io/sync-wave` | Argo CD sync order | `0`, `1`, `2` | Yes (for GitOps) |
| `argocd.argoproj.io/sync-options` | Argo CD sync behavior | `Prune=false` | No |

### Example: Namespace with Full Annotations
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: acme-prd
  labels:
    lc2.liferay.com/project-id: acme
    lc2.liferay.com/environment: prd
  annotations:
    lc2.liferay.com/owner: alice@acme-corp.com
    lc2.liferay.com/created-by: admin@liferay.com
    lc2.liferay.com/created-at: "2025-11-02T10:30:00Z"
    lc2.liferay.com/description: "ACME Corp production environment"
```

---

## Hostname and DNS Naming

### Pattern
```
{service}.{environment}.{project-domain}
```

### Examples
- `api.dev.acme-corp.lc2.liferay.cloud` - Cockpit API dev endpoint
- `portal.prd.acme-corp.lc2.liferay.cloud` - ACME Liferay production
- `grafana.prd.acme-corp.lc2.liferay.cloud` - ACME Grafana dashboard

### System Hostnames
```
{service}.{cluster-domain}
```

Examples:
- `cockpit-backend.lc2.liferay.cloud` - Cockpit Backend production
- `argocd.lc2.liferay.cloud` - Argo CD UI
- `grafana.lc2.liferay.cloud` - Shared Grafana instance

### TLS Certificate Naming
Ingress resources should request TLS certificates via cert-manager:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cockpit-backend-ingress
  namespace: cockpit-system
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - cockpit-backend.lc2.liferay.cloud
      secretName: cockpit-backend-tls
  rules:
    - host: cockpit-backend.lc2.liferay.cloud
      # ... routing rules
```

---

## Argo CD Naming

### AppProject Naming
```
project-{project-id}
```

Examples:
- `project-acme` - ACME Corp AppProject
- `project-beta` - Beta Inc AppProject

### Application Naming
```
{project-id}-{environment}-{app-name}
```

Examples:
- `acme-dev-liferay` - ACME Liferay dev environment
- `acme-prd-liferay` - ACME Liferay production
- `cockpit-backend` - Cockpit Backend (system app, no project prefix)

### App-of-Apps Naming
```
{project-id}-apps
```

Examples:
- `acme-apps` - ACME Corp app-of-apps
- `beta-apps` - Beta Inc app-of-apps

---

## Docker Image Naming

### Pattern
```
{registry}/{organization}/{repository}:{tag}
```

### Examples
- `ghcr.io/liferay/cockpit-backend:0.1.0`
- `docker.io/liferay/liferay-dxp:7.4.3.120-ga120`
- `123456789.dkr.ecr.us-west-1.amazonaws.com/acme/custom-app:v2.0.0`

### Tagging Strategy
- **SemVer**: `v1.2.3` (production releases)
- **Git SHA**: `sha-a1b2c3d` (for traceability)
- **Environment**: `dev-latest`, `uat-stable` (for testing)
- **Immutable Tags**: Never re-tag a version (e.g., don't change `v1.2.3` to point to a different image)

---

## Database Naming

### PostgreSQL Database Names
```
{project-id}_{environment}
```

Examples:
- `acme_dev` - ACME development database
- `acme_prd` - ACME production database
- `cockpit` - Cockpit Backend database

### PostgreSQL User Names
```
{project-id}_{environment}_user
```

Examples:
- `acme_dev_user`
- `acme_prd_user`
- `cockpit_user`

### PostgreSQL Connection Strings
```
jdbc:postgresql://{host}:{port}/{database}?sslmode=require
```

Example:
```
jdbc:postgresql://cockpit-db.us-west-1.rds.amazonaws.com:5432/cockpit?sslmode=require
```

---

## Secret Naming in Vault

### AWS Secrets Manager Path
```
/lc2/{project-id}/{environment}/{secret-name}
```

Examples:
- `/lc2/acme/prd/db-password`
- `/lc2/acme/prd/liferay-license-key`
- `/lc2/cockpit/prd/db-password`

### Azure Key Vault Secret Name
```
lc2-{project-id}-{environment}-{secret-name}
```

Examples (Azure doesn't support `/` in names):
- `lc2-acme-prd-db-password`
- `lc2-acme-prd-liferay-license-key`
- `lc2-cockpit-prd-db-password`

### GCP Secret Manager Secret Name
```
lc2_{project_id}_{environment}_{secret_name}
```

Examples:
- `lc2_acme_prd_db_password`
- `lc2_acme_prd_liferay_license_key`
- `lc2_cockpit_prd_db_password`

---

## Grafana Naming

### Organization Names
```
{project-name} ({project-id})
```

Examples:
- `ACME Corp (acme)`
- `Beta Inc (beta)`

### Dashboard Names
```
{project-id} - {environment} - {component}
```

Examples:
- `acme - prd - Liferay Overview`
- `acme - prd - Database Performance`
- `cockpit - prd - API Metrics`

### Data Source Names
```
{project-id}-{environment}-{type}
```

Examples:
- `acme-prd-prometheus` - Prometheus data source for ACME production
- `acme-prd-loki` - Loki logs data source for ACME production

---

## Kyverno Policy Naming

### ClusterPolicy Naming
```
{category}-{purpose}
```

Examples:
- `pss-restricted` - Pod Security Standards - Restricted profile
- `require-labels` - Require standard Kubernetes labels
- `default-deny-network` - Default-deny NetworkPolicy
- `registry-allowlist` - Only allow approved registries

### Policy Naming
Tenant-specific policies use project ID prefix:
```
{project-id}-{purpose}
```

Examples:
- `acme-resource-quotas` - ACME resource quota enforcement
- `acme-network-policy` - ACME network isolation

---

## Git Repository Naming

### Platform Repositories
```
lc2-{component}
```

Examples:
- `lc2-cockpit-backend` - Cockpit Backend source code
- `lc2-cockpit-frontend` - Cockpit Frontend source code
- `lc2-infrastructure` - Terraform infrastructure as code
- `lc2-gitops` - Argo CD application manifests

### Tenant Repositories
```
{project-id}-gitops
```

Examples:
- `acme-gitops` - ACME Corp GitOps repository
- `beta-gitops` - Beta Inc GitOps repository

---

## Metrics and Log Labels

### Prometheus Metric Names
```
{namespace}_{subsystem}_{metric_name}_{unit}
```

Examples (from Cockpit Backend):
- `cockpit_projects_created_total` - Total projects created (counter)
- `cockpit_audit_events_total` - Total audit events (counter)
- `cockpit_kubernetes_api_calls_total` - Total Kubernetes API calls (counter)
- `cockpit_kubernetes_api_errors_total` - Total Kubernetes API errors (counter)
- `cockpit_project_operations_seconds` - Project operation duration (histogram)

### Common Metric Labels
All metrics should include:
```
application="cockpit-backend"
environment="dev|uat|prd"
service="paas-cockpit"
```

Tenant-scoped metrics add:
```
project_id="acme"
```

### Log Fields (JSON Structured Logs)
```json
{
  "timestamp": "2025-11-02T10:30:00.123Z",
  "level": "INFO",
  "logger": "com.liferay.cloud.cockpit.service.ProjectService",
  "message": "Project created successfully",
  "project_id": "acme",
  "environment": "dev",
  "actor": "alice@example.com",
  "correlation_id": "req-123abc"
}
```

---

## Validation

### Kyverno Policy to Enforce Labels
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-lc2-labels
      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet
                - Service
              namespaceSelector:
                matchExpressions:
                  - key: lc2.liferay.com/project-id
                    operator: Exists
      validate:
        message: "Missing required LC2 labels"
        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              app.kubernetes.io/instance: "?*"
              app.kubernetes.io/version: "?*"
              lc2.liferay.com/project-id: "?*"
              lc2.liferay.com/environment: "dev|uat|prd"
```

---

## Revision History

| Version | Date       | Author | Changes                                |
|---------|------------|--------|----------------------------------------|
| 1.0     | 2025-11-02 | Claude | Initial naming conventions definition  |
