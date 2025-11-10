# LC2 Architecture Principles

This document defines the four core principles that guide all architectural decisions in Liferay Cloud 2.0 (LC2), including the Cockpit Backend control plane.

## The Four Principles

### 1. Portability

**Definition**: Applications and infrastructure must run identically across local (k3d) and cloud (EKS/AKS/GKE) environments without code changes.

**Rationale**: Developers should be able to test the complete stack locally, ensuring that what works on their workstation will work in production. This eliminates "works on my machine" problems and accelerates development cycles.

**Implementation in Cockpit Backend**:
- Cloud-agnostic Kubernetes API usage via Fabric8 client
- PostgreSQL used everywhere (Docker locally, RDS/Azure Database/Cloud SQL in cloud)
- ExternalSecrets Operator abstracts AWS Secrets Manager, Azure Key Vault, GCP Secret Manager
- Standard Kubernetes Ingress with nginx controller (no cloud-specific LoadBalancer annotations)
- Prometheus metrics format (self-hosted or managed)
- Structured JSON logs to stdout (consumed by Loki, CloudWatch, or Stackdriver)

**Configuration Differences**:
Only environment variables change between local and cloud:
```yaml
# Local
DB_URL=jdbc:postgresql://localhost:5432/cockpit
K8S_NAMESPACE=default
kubernetes.client.enabled=false

# Cloud (dev/uat/prd)
DB_URL=jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
K8S_NAMESPACE=cockpit-system
kubernetes.client.enabled=true
```

**Anti-Patterns to Avoid**:
- Cloud-specific SDKs in application code (AWS SDK, Azure SDK, GCP SDK)
- Hard-coded cloud provider endpoints
- Conditional logic based on cloud provider (`if (cloud == "aws") { ... }`)
- Different manifest structures for local vs cloud

**Validation**:
- Application JAR is identical across all environments
- Kubernetes manifests differ only in namespace, resource limits, and external service endpoints
- Integration tests run with Testcontainers (PostgreSQL, mock Kubernetes API) without cloud access

---

### 2. Isolation

**Definition**: Multi-tenant projects must have strong boundaries preventing cross-project data leakage, resource interference, and security breaches.

**Rationale**: In a multi-tenant PaaS, one project's actions or misconfigurations should never impact another project. Isolation enables safe multi-tenancy and builds customer trust.

**Implementation in Cockpit Backend**:

#### Namespace Isolation
Each project gets dedicated namespaces per environment:
- `{project-id}-dev`
- `{project-id}-uat`
- `{project-id}-prd`

Control plane namespace:
- `cockpit-system` (Cockpit Backend, supporting resources)

#### Database Isolation
- **Logical Isolation**: All multi-tenant tables have `projectId` foreign key
- **Service Layer Checks**: `RbacService.requireReadAccess(projectId, userEmail)` enforced before queries
- **Audit Trail**: All cross-project queries logged as potential security events

#### Kubernetes RBAC Isolation
- Cockpit Backend ServiceAccount can list/read all namespaces (for orchestration)
- Project members get RBAC limited to their `{project-id}-*` namespaces only
- No `ClusterRole` bindings for tenant users

#### Argo CD Isolation
Each project gets an `AppProject` with strict fencing:
```yaml
sourceRepos:
  - https://github.com/acme-corp/gitops-repo  # Tenant repo only
destinations:
  - namespace: acme-dev
    server: https://kubernetes.default.svc
  # Cannot deploy to other projects' namespaces
clusterResourceWhitelist: []  # No cluster-wide resources
```

#### Network Isolation
- Default-deny NetworkPolicy in all project namespaces
- Explicit allow-list for:
  - DNS (kube-dns/coredns)
  - Egress to managed databases
  - Ingress from ingress-nginx only

#### Grafana Isolation
- One Grafana **Organization** per project
- Provisioned data sources scoped to project namespaces (e.g., Prometheus queries filtered by `namespace=~"acme-.*"`)
- RBAC: Project members can only access their org

#### Resource Quota Isolation
Each project namespace gets ResourceQuotas:
```yaml
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    persistentvolumeclaims: "10"
```

**Anti-Patterns to Avoid**:
- Shared namespaces for multiple projects
- Database queries without `WHERE projectId = ?`
- Cluster-wide RBAC for tenant users
- Argo CD ApplicationSet targeting multiple projects
- Shared Grafana dashboards without project filtering

**Validation**:
- Integration tests verify `RbacService.AccessDeniedException` thrown when user queries other project
- Kyverno policy blocks pods without `lc2.liferay.com/project-id` label
- Argo CD sync fails if Application targets namespace outside AppProject fence
- Grafana API returns 403 when user queries other org's dashboards

---

### 3. KISS (Keep It Simple, Stupid)

**Definition**: Choose simple, well-understood solutions over complex, cutting-edge technologies. Optimize for maintainability and debuggability, not cleverness.

**Rationale**: Complexity is the enemy of reliability. Simple architectures are easier to understand, debug, and extend. Prefer boring technology that works over exciting technology that might work.

**Implementation in Cockpit Backend**:

#### Technology Choices
- **PostgreSQL**: Industry-standard relational database (not NoSQL unless justified)
- **Spring Boot**: Mainstream Java framework with excellent documentation
- **Session-based Auth**: HTTP-only cookies + CSRF (defer JWT/OAuth until SSO required)
- **REST APIs**: JSON over HTTP (no GraphQL, gRPC unless proven necessary)
- **Liquibase**: Declarative SQL migrations (not hand-written migration scripts)
- **Fabric8 Kubernetes Client**: Official Java client (not custom kubectl wrappers)

#### Architectural Simplicity
- **Monolithic Backend** (Phase 1-6): Single Spring Boot application, not microservices
- **Synchronous APIs**: Immediate responses, not async/message queues (until scale requires it)
- **Direct Database Access**: Spring Data JPA repositories, not event sourcing or CQRS
- **Explicit Service Layer**: Business logic in service classes, not distributed across controllers/repositories

#### Configuration Simplicity
- **Spring Profiles**: `local`, `dev`, `uat`, `prd` (not per-engineer custom profiles)
- **Environment Variables**: Standard 12-factor config (not complex config servers)
- **Liquibase Changesets**: Versioned SQL in YAML (not code-based schema generation)

#### Deployment Simplicity
- **Single Container**: One Spring Boot JAR in one Docker image
- **Standard Kubernetes Deployment**: No Helm charts with 100+ values (defer until Phase 7)
- **Blue-Green via Argo CD**: Native rollback, not custom deployment orchestration

**When to Add Complexity**:
Only when you have evidence (metrics, user complaints) that simplicity is insufficient:
- Event-driven architecture: When sync APIs become too slow (>1s p95)
- Microservices: When monolith deployment size causes >30min rollouts
- GraphQL: When mobile app requires complex aggregations that REST can't serve efficiently
- Custom auth: When SSO compliance (SAML, OIDC) is mandated by customers

**Anti-Patterns to Avoid**:
- Introducing Kafka/RabbitMQ before you have 10,000+ events/sec
- Building custom Kubernetes operators before evaluating off-the-shelf solutions
- Splitting into microservices before monolith crosses 100k LOC or 10+ devs
- Using NoSQL databases "for scale" without benchmarking PostgreSQL first

**Validation**:
- New team members can run the full stack locally within 30 minutes
- README troubleshooting section covers 90% of issues
- No custom bash scripts >50 lines (use standard tools like kubectl, helm, argocd)
- Codebase has <5 third-party libraries that aren't Spring ecosystem

---

### 4. GitOps

**Definition**: All configuration and deployment changes are declared in Git and applied via automated reconciliation (Argo CD), never via `kubectl apply` or manual edits.

**Rationale**: Git provides audit trails, rollback capabilities, and a single source of truth. Manual changes are error-prone, undocumented, and impossible to replicate.

**Implementation in Cockpit Backend**:

#### Infrastructure as Code
- **Terraform**: Managed resources (RDS, VPCs, IAM roles) declared in HCL, state in S3/Blob
- **Kubernetes Manifests**: Deployments, Services, Ingress in YAML committed to Git
- **Argo CD Applications**: App-of-apps pattern, all applications tracked in Git

#### Deployment Workflow
1. Developer pushes code changes to `main` branch
2. CI pipeline builds Docker image, tags with git SHA
3. CI updates Kubernetes manifest in GitOps repo with new image tag
4. Argo CD detects Git change, syncs to cluster
5. Rollback = revert Git commit + Argo sync

#### Configuration Management
- **Application Config**: Spring profiles in `application.yml` (checked into Git)
- **Secrets**: ExternalSecrets YAML in Git (syncs values from vault, never commit plaintext)
- **Database Schema**: Liquibase changesets in Git (applied by Spring Boot on startup)

#### Audit Trail
- Git commit log shows **who** changed **what** and **when**
- Argo CD sync history shows **which** Git commit was deployed
- Cockpit audit events log **why** (e.g., "User alice@example.com triggered rollback")

#### Break-Glass Access
For emergencies, admins can use `kubectl` directly:
```bash
kubectl delete pod broken-pod-xyz -n acme-dev
```
**Requirements**:
1. Must be logged in Cockpit `AuditEvent` table (action="kubectl.break_glass")
2. Must be followed by Git commit reflecting change (e.g., pod restart policy)
3. Incident review required if break-glass used >1x per month

**Anti-Patterns to Avoid**:
- `kubectl apply -f local-file.yaml` without committing to Git first
- Editing ConfigMaps/Secrets in cluster with `kubectl edit`
- CI pipeline that runs `kubectl set image` instead of updating Git
- "Temporary" manual changes that are never documented

**Validation**:
- Argo CD is the only process with write access to application namespaces (RBAC enforced)
- Drift detection: Argo CD shows `OutOfSync` if manual changes are made
- Audit queries: `SELECT COUNT(*) FROM audit_events WHERE action LIKE 'kubectl.%' AND timestamp > NOW() - INTERVAL '30 days'` should be <10

---

## Applying the Principles

### Decision Framework
When evaluating any architectural decision, ask:

1. **Portability**: Can this run on k3d and EKS/AKS/GKE without code changes?
2. **Isolation**: Can one project's usage impact another project?
3. **KISS**: Is this the simplest solution that solves the problem?
4. **GitOps**: Can I roll back this change by reverting a Git commit?

If the answer to any is "No", reconsider the approach.

### Trade-offs
Sometimes principles conflict. Prioritization:
1. **Isolation** > Everything (security and tenant safety are non-negotiable)
2. **KISS** > Portability (prefer simple cloud-specific solutions over complex portable ones, if portability isn't critical)
3. **GitOps** > KISS (accept Git workflow complexity to gain audit trails and rollback)
4. **Portability** > Bleeding-edge features (defer cloud-native services that aren't portable yet)

### Exceptions Process
To violate a principle:
1. Document the exception in an Architecture Decision Record (ADR)
2. Explain the problem that can't be solved within the principle
3. Propose mitigation (e.g., "Use managed Redis for session storage, but abstract behind Spring Session interface")
4. Get approval from architecture review

---

## Examples from Cockpit Backend

### Good: Portability
**Scenario**: Need to store database credentials securely.

**Bad Approach** (not portable):
```java
// AWS SDK in application code
SecretsManagerClient client = SecretsManagerClient.create();
String password = client.getSecretValue(r -> r.secretId("db-password")).secretString();
```

**Good Approach** (portable):
```yaml
# ExternalSecret in Git
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: cockpit-db-credentials
spec:
  secretStoreRef:
    name: aws-secrets-manager  # Swap for azure-keyvault or gcp-sm
  target:
    name: cockpit-db-secret
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /lc2/cockpit/db-password
```
```java
// Application code just reads environment variable
@Value("${spring.datasource.password}")
private String dbPassword;  // Injected from Kubernetes Secret
```

### Good: Isolation
**Scenario**: List all pods in a namespace.

**Bad Approach** (no isolation check):
```java
@GetMapping("/api/pods")
public List<Pod> listPods(@RequestParam String namespace) {
    return k8sService.listPods(namespace);  // ⚠️ No project ownership check!
}
```

**Good Approach** (enforced isolation):
```java
@GetMapping("/api/pods")
public List<Pod> listPods(@RequestParam String namespace,
                          @RequestParam UUID projectId,
                          Authentication auth) {
    String userEmail = auth.getName();
    rbacService.requireReadAccess(projectId, userEmail);  // ✅ Enforced

    // Verify namespace belongs to project
    if (!namespace.startsWith(projectId.toString())) {
        throw new AccessDeniedException("Namespace does not belong to project");
    }

    return k8sService.listPods(namespace);
}
```

### Good: KISS
**Scenario**: Need to track project creation metrics.

**Bad Approach** (over-engineered):
```java
// Publish event to Kafka, consumed by metrics aggregator, stored in time-series DB
eventPublisher.publish(new ProjectCreatedEvent(projectId, timestamp));
```

**Good Approach** (simple):
```java
// Micrometer counter directly in service
metricsService.recordProjectCreated(project.getId());

// In MetricsService:
Counter.builder("cockpit_projects_created_total")
    .tag("environment", environment)
    .register(meterRegistry)
    .increment();
```

### Good: GitOps
**Scenario**: Deploy new version of Cockpit Backend.

**Bad Approach** (manual):
```bash
kubectl set image deployment/cockpit-backend \
  cockpit-backend=cockpit:v2.0.0 -n cockpit-system
```

**Good Approach** (GitOps):
```bash
# 1. Update Git
cd gitops-repo
vim cockpit-system/cockpit-backend/deployment.yaml
# Change image: cockpit:v1.9.0 -> cockpit:v2.0.0
git commit -m "chore: upgrade cockpit-backend to v2.0.0"
git push

# 2. Argo CD syncs automatically (or manual sync)
argocd app sync cockpit-backend

# 3. Rollback = revert commit
git revert HEAD
git push
argocd app sync cockpit-backend
```

---

## Revision History

| Version | Date       | Author | Changes                           |
|---------|------------|--------|-----------------------------------|
| 1.0     | 2025-11-02 | Claude | Initial principles documentation  |
