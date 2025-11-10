# Cockpit Backend Architecture

## Overview

The Cockpit Backend is the **control plane API** for Liferay Cloud 2.0 (LC2). It serves as the central orchestration layer that enables project provisioning, multi-tenant isolation, environment management, and GitOps workflows across local (k3d) and cloud (EKS/AKS/GKE) Kubernetes clusters.

**Project ID**: `cockpit`
**Purpose**: PaaS Cockpit control plane API
**Environments**: dev, uat, prd
**Runtime**: local (k3d), AWS (EKS), Azure (AKS), GCP (GKE)
**Current Phase**: Phase 1 - Backend Foundation and Data Layer (Complete)

## Architecture Principles

This architecture adheres to the four core LC2 principles:

1. **Portability** - Run identically across local (k3d) and cloud (EKS/AKS/GKE)
2. **Isolation** - Multi-tenant with project-scoped namespaces and RBAC
3. **KISS** - Simple, maintainable, no over-engineering
4. **GitOps** - Configuration and deployment via Git with Argo CD

See [Principles.md](./Principles.md) for detailed guidelines.

## Four-Layer Model

The Cockpit Backend operates within LC2's four-layer architecture:

### Layer 1: Shared Platform Services
**Scope**: Cluster-wide, shared infrastructure
**Ownership**: Platform team
**Namespace**: `cockpit-system`, `argocd`, `kyverno`, `cert-manager`, `ingress-nginx`, `external-secrets-system`

Components deployed by Cockpit:
- **Argo CD** (OSS) - Shared GitOps engine with per-project AppProjects
- **Kyverno** - Policy enforcement for guardrails (PSS, quotas, default-deny)
- **Cert-Manager** - TLS certificate management
- **Ingress-NGINX** - Shared ingress controller
- **External Secrets Operator** - Secret synchronization from cloud vaults
- **Grafana** (OSS) - Shared observability UI with org-per-project isolation

### Layer 2: Per-Project Infrastructure
**Scope**: Project-scoped resources
**Ownership**: Project owners via Cockpit API
**Namespace Pattern**: `{project-id}-{env}` (e.g., `acme-dev`, `acme-uat`, `acme-prd`)

Resources managed by Cockpit:
- **Namespaces** - Isolated per project/environment
- **ResourceQuotas** - CPU, memory, storage limits
- **NetworkPolicies** - Default-deny with allowed egress
- **RBAC** - ServiceAccounts, Roles, RoleBindings for project members
- **Argo CD AppProject** - Limits repos, namespaces, and resource kinds
- **Grafana Organization** - Per-project observability isolation
- **External Secrets** - Project-scoped secret references (RDS credentials, API keys)

### Layer 3: Application Workloads
**Scope**: Tenant applications (Liferay DXP, custom apps)
**Ownership**: Project members via GitOps
**Namespace**: Same as Layer 2 (`{project-id}-{env}`)

Resources:
- **Deployments** - Liferay DXP pods, custom microservices
- **Services** - Internal service endpoints
- **Ingress** - External HTTPS endpoints with TLS
- **ConfigMaps/Secrets** - Application configuration
- **PersistentVolumeClaims** - Stateful storage

### Layer 4: Observability
**Scope**: Logs, metrics, traces
**Ownership**: Shared platform + per-project isolation
**Namespace**: `grafana`, `loki` (optional), `prometheus` (managed)

Implementation:
- **Metrics**: Spring Boot Actuator + Micrometer Prometheus
  - Custom metrics: `cockpit_projects_created_total`, `cockpit_audit_events_total`, `cockpit_kubernetes_api_calls_total`
  - Common tags: `application`, `environment`, `service`
  - Endpoint: `/actuator/prometheus`
- **Logs**: Structured JSON logs via Logback → Loki (optional) or CloudWatch/Stackdriver
- **Health Checks**: `/actuator/health` with custom indicators (DB, Kubernetes)
- **Grafana**: Shared OSS instance with per-project organizations and provisioned dashboards

## Technology Stack

### Backend Core
- **Java 17** - Programming language
- **Spring Boot 3.2.1** - Application framework
- **Spring Security** - Session-based authentication with CSRF protection
- **Spring Data JPA** - Data access with Hibernate 6.4
- **PostgreSQL 15** - Relational database (RDS/Azure Database in cloud, Docker locally)
- **Liquibase** - Database schema migrations

### Kubernetes Integration
- **Fabric8 Kubernetes Client 6.9.2** - Java client for K8s API
- **Conditional Configuration** - KubernetesService is optional (`@ConditionalOnBean`)
- **In-Cluster Auth** - ServiceAccount RBAC for production
- **Kubeconfig** - Local development with `~/.kube/config`

### Observability
- **Micrometer** - Metrics abstraction layer
- **Prometheus** - Metrics export format
- **Spring Boot Actuator** - Health checks and management endpoints
- **Custom Health Indicators** - DatabaseHealthIndicator, KubernetesHealthIndicator

### Testing
- **JUnit 5** - Unit testing framework
- **Mockito** - Mocking for unit tests
- **Testcontainers** - Integration tests with ephemeral PostgreSQL
- **Spring Boot Test** - Integration test support

## Data Model

### Core Entities

#### Project
- `id` (UUID) - Primary key
- `name` (String) - Display name
- `description` (String) - Purpose
- `ownerId` (UUID) - Reference to User
- `status` (Enum) - ACTIVE, SUSPENDED, ARCHIVED
- `createdAt`, `updatedAt` (Timestamp)

#### Environment
- `id` (UUID) - Primary key
- `projectId` (UUID) - Foreign key to Project
- `name` (String) - dev, uat, prd
- `namespace` (String) - Kubernetes namespace (e.g., `acme-dev`)
- `clusterName` (String) - Target cluster
- `readinessGates` (JSON) - Health status checks

#### Member
- `id` (UUID) - Primary key
- `projectId` (UUID) - Foreign key to Project
- `userEmail` (String) - User identifier
- `role` (Enum) - OWNER, DEVELOPER, VIEWER
- `invitedAt`, `acceptedAt` (Timestamp)

#### AuditEvent
- `id` (UUID) - Primary key
- `projectId` (UUID) - Foreign key to Project (nullable for cluster-wide events)
- `actor` (String) - User email
- `action` (String) - Action type (e.g., `project.create`, `member.remove`)
- `resourceType`, `resourceId` (String) - Target resource
- `outcome` (Enum) - SUCCESS, FAILURE
- `timestamp` (Timestamp)
- `metadata` (JSON) - Additional context

#### ArgoApplication
- `id` (UUID) - Primary key
- `environmentId` (UUID) - Foreign key to Environment
- `name` (String) - Argo CD Application name
- `gitRepo` (String) - Source repository URL
- `gitPath` (String) - Path within repository
- `syncStatus` (Enum) - SYNCED, OUT_OF_SYNC, UNKNOWN
- `healthStatus` (Enum) - HEALTHY, DEGRADED, PROGRESSING

## API Design

### REST Endpoints

#### Authentication
- `POST /api/auth/login` - Create session
- `POST /api/auth/logout` - Invalidate session
- `GET /api/me` - Current user profile

#### Admin Project Management
- `GET /api/admin/projects` - List all projects (ADMIN only)
- `POST /api/admin/projects` - Create new project (ADMIN only)
- `GET /api/admin/projects/{projectId}` - Get project details
- `PUT /api/admin/projects/{projectId}` - Update project
- `DELETE /api/admin/projects/{projectId}` - Archive/suspend project

#### Environment Management
- `GET /api/projects/{projectId}/environments` - List environments
- `GET /api/projects/{projectId}/environments/{env}` - Get environment details
- `GET /api/projects/{projectId}/environments/{env}/readiness` - Readiness gates

#### Member Management
- `GET /api/projects/{projectId}/members` - List project members
- `POST /api/projects/{projectId}/members` - Invite member (OWNER only)
- `PUT /api/projects/{projectId}/members/{memberId}` - Update member role
- `DELETE /api/projects/{projectId}/members/{memberId}` - Remove member

#### Kubernetes Operations
- `GET /api/pods?namespace={ns}&projectId={id}` - List pods
- `GET /api/pods/{namespace}/{name}` - Get pod details
- `GET /api/pods/logs?namespace={ns}&podName={name}` - Get pod logs
- `DELETE /api/pods/{namespace}/{name}` - Delete pod

#### Audit Logging
- `GET /api/audit?projectId={id}` - Get audit events
- `GET /api/audit/actions` - List audit action types

#### Health and Metrics
- `GET /actuator/health` - Spring Boot health check
- `GET /actuator/prometheus` - Prometheus metrics
- `GET /api/health` - Custom health endpoint with K8s status

### Security Model

#### Authentication
- Session-based with HTTP-only cookies (JSESSIONID)
- CSRF protection enabled for state-changing operations
- Rate limiting on login endpoint (future enhancement)
- Temporary admin user for MVP (SSO deferred)

#### Authorization (RBAC)
Three role hierarchy:
1. **ADMIN** - Cluster-wide operations, create/suspend projects
2. **OWNER** - Full access within project, invite/remove members
3. **DEVELOPER** - Read/write access to environments and applications
4. **VIEWER** - Read-only access to project resources

Enforcement:
- `RbacService.requireReadAccess(projectId, userEmail)`
- `RbacService.requireModifyAccess(projectId, userEmail)`
- `RbacService.requireAdminAccess(userEmail)`
- `@PreAuthorize` annotations (future enhancement)

## Deployment Architecture

### Local (k3d)
```
┌─────────────────────────────────────────────┐
│ Developer Workstation                       │
│ ┌─────────────────────────────────────────┐ │
│ │ k3d Cluster (local)                     │ │
│ │ ┌─────────────┐  ┌──────────────────┐  │ │
│ │ │ cockpit-    │  │ Project          │  │ │
│ │ │ system      │  │ Namespaces       │  │ │
│ │ │ - Backend   │  │ - acme-dev       │  │ │
│ │ │ - Postgres  │  │ - beta-dev       │  │ │
│ │ └─────────────┘  └──────────────────┘  │ │
│ │ ┌─────────────────────────────────────┐ │ │
│ │ │ Shared Platform                     │ │ │
│ │ │ - Argo CD                           │ │ │
│ │ │ - Kyverno                           │ │ │
│ │ │ - Grafana OSS                       │ │ │
│ │ └─────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

**Configuration**:
- Database: `localhost:5432` (Docker PostgreSQL)
- Kubernetes: `~/.kube/config` with k3d context
- Profile: `spring.profiles.active=local`
- K8s Client: Disabled by default (`kubernetes.client.enabled=false`)

### Cloud (EKS/AKS/GKE)
```
┌───────────────────────────────────────────────────────────┐
│ Kubernetes Cluster (EKS/AKS/GKE)                          │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Namespace: cockpit-system                             │ │
│ │ ┌──────────────────┐  ┌──────────────────────────┐   │ │
│ │ │ Cockpit Backend  │  │ Managed PostgreSQL       │   │ │
│ │ │ - Deployment     │  │ (RDS/Azure DB)           │   │ │
│ │ │ - Service        │  │ External to cluster      │   │ │
│ │ │ - Ingress        │  └──────────────────────────┘   │ │
│ │ │ - ServiceAccount │                                  │ │
│ │ └──────────────────┘                                  │ │
│ └───────────────────────────────────────────────────────┘ │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Project Namespaces (per tenant)                       │ │
│ │ - acme-dev, acme-uat, acme-prd                        │ │
│ │ - beta-dev, beta-uat, beta-prd                        │ │
│ │ Each with: Quotas, NetworkPolicies, RBAC             │ │
│ └───────────────────────────────────────────────────────┘ │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Shared Platform (argocd, kyverno, grafana, etc.)     │ │
│ └───────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

**Configuration**:
- Database: Managed PostgreSQL (RDS/Azure Database) via ExternalSecret
- Kubernetes: In-cluster ServiceAccount with RBAC
- Profile: `spring.profiles.active=dev|uat|prd`
- K8s Client: Enabled (`kubernetes.client.enabled=true`)
- Ingress: TLS termination with cert-manager

## Configuration Management

### Environment Profiles

#### Local Profile (`local`)
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/cockpit
    username: cockpit
    password: cockpit
kubernetes:
  client:
    enabled: false
```

#### Dev/UAT/Prd Profiles
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}  # From ExternalSecret
kubernetes:
  client:
    enabled: true
    namespace: ${K8S_NAMESPACE:cockpit}
    master-url: https://kubernetes.default.svc
```

### Secret Management
- **Local**: Plain environment variables or `application-local.yml`
- **Cloud**: ExternalSecrets Operator syncing from:
  - AWS Secrets Manager (EKS)
  - Azure Key Vault (AKS)
  - GCP Secret Manager (GKE)

Example ExternalSecret:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cockpit-db-credentials
  namespace: cockpit-system
spec:
  secretStoreRef:
    name: aws-secrets-manager  # or azure-keyvault, gcp-sm
    kind: SecretStore
  target:
    name: cockpit-db-secret
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /lc2/cockpit/db-password
```

## Isolation and Multi-Tenancy

### Namespace Strategy
Each project gets dedicated namespaces per environment:
- `{project-id}-dev`
- `{project-id}-uat`
- `{project-id}-prd`

Control plane namespace:
- `cockpit-system` (Cockpit Backend and supporting resources)

### Argo CD Isolation
Each project gets an AppProject with strict fencing:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: project-acme
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/acme-corp/gitops-repo  # Tenant repo only
  destinations:
    - namespace: acme-dev
      server: https://kubernetes.default.svc
    - namespace: acme-uat
      server: https://kubernetes.default.svc
    - namespace: acme-prd
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []  # No cluster-wide resources
  namespaceResourceWhitelist:
    - group: '*'
      kind: Deployment
    - group: '*'
      kind: Service
    # ... allowed kinds only
```

### Kyverno Guardrails
Policies enforced cluster-wide:
1. **Pod Security Standards** - Restricted profile (no privileged, no host mounts)
2. **Resource Quotas** - CPU/memory limits per namespace
3. **Default Deny NetworkPolicy** - Explicit allow-list for egress
4. **Registry Allow-List** - Only approved container registries
5. **Require Labels** - `app.kubernetes.io/name`, `lc2.liferay.com/project-id`, `lc2.liferay.com/environment`

### Grafana Multi-Tenancy
- Shared Grafana OSS instance
- One **Organization** per project
- Provisioned data sources scoped to project namespaces
- RBAC: Project members get Viewer/Editor roles in their org only

### Database Isolation
- Logical isolation: `projectId` foreign key on all multi-tenant tables
- Row-level checks in service layer (`RbacService.requireReadAccess`)
- Audit trail for all cross-project queries (logged as potential breach)

## Portability Guarantees

### Cloud-Agnostic Abstractions
1. **Database**: PostgreSQL (RDS, Azure Database, Cloud SQL, or Docker)
2. **Secrets**: ExternalSecrets Operator with pluggable backends
3. **Ingress**: Standard Kubernetes Ingress with nginx controller
4. **Metrics**: Prometheus format (managed or self-hosted)
5. **Logs**: Structured JSON to stdout → Loki/CloudWatch/Stackdriver

### No Cloud-Specific APIs in Code
- No AWS SDK, Azure SDK, or GCP SDK in application code
- Provisioning via Terraform (separate job orchestration)
- Cloud resources referenced via ExternalSecrets

### Configuration Differences
Only environment variables differ:
- `DB_HOST`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`
- `K8S_NAMESPACE`
- `ARGOCD_SERVER_URL`
- `OIDC_ISSUER_URI` (for SSO, future)

Application code remains **100% portable**.

## Future Enhancements (Post-Phase 1)

1. **Argo CD Integration** (Phase 2)
   - Create/sync Argo CD Applications via API
   - Poll sync status and health
   - Deep links to Argo CD UI

2. **Terraform Provisioner Jobs** (Phase 3)
   - Orchestrate Kubernetes Job to run Terraform
   - Provision managed PostgreSQL, object storage, DNS
   - Track provisioning state and logs

3. **ExternalSecrets Polling** (Phase 3)
   - Monitor ExternalSecret `status.conditions`
   - Alert on sync failures

4. **SSO Integration** (Phase 4)
   - OIDC/SAML with regional identity provider
   - Replace temporary admin user

5. **Real-Time Updates** (Phase 5)
   - Server-Sent Events (SSE) for live pod status, logs
   - WebSocket fallback

6. **Advanced RBAC** (Phase 6)
   - Fine-grained permissions (e.g., read-only on production)
   - Service accounts for CI/CD

## References

- [Principles.md](./Principles.md) - LC2 core principles
- [Naming Conventions.md](./Naming%20Conventions.md) - Namespace, label, annotation standards
- [Architecture Audit Checklist](./Checklists/Architecture%20Audit.md) - Validation checklist
- [Kyverno Policy Index](../kyverno/policy-index.md) - Required guardrails
- [Cockpit Backend README](../../cockpit-backend/README.md) - Development setup

## Revision History

| Version | Date       | Author | Changes                           |
|---------|------------|--------|-----------------------------------|
| 1.0     | 2025-11-02 | Claude | Initial architecture documentation after Phase 1 completion |
