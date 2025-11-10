# LC2 Backup - Complete Synthesis of Learnings

## Overview
This document synthesizes learnings from:
1. Original LC2 requirements (requirements.md)
2. Initial backup requirements analysis (backup-requirements.md)
3. OpenAI debate outcomes (backup-requirements-refined.md)
4. Current Liferay Cloud implementations (LCP GCP + CN AWS)

---

## Core Requirements (Non-Negotiable)

### LC2 Platform Principles
1. **GitOps-First**: Git as source of truth, Argo CD for reconciliation
2. **Kubernetes Stateless**: No PVCs, external managed services
3. **Multi-Tenant Hard Isolation**: Namespace boundaries, RBAC
4. **Local↔Cloud Parity**: Same workflows everywhere
5. **KISS**: Keep It Simple and Straightforward
6. **Cloud-Agnostic**: Must work on AWS, GCP, Azure, and local

### Backup-Specific Requirements
1. **Multi-Tenant Service**: NOT per-customer deployment
2. **Three-Component Backup**:
   - Database (MySQL/PostgreSQL) - 10GB to 10TB
   - Document Library (Object Storage) - 100GB to 100TB
   - Search Indexes (Optional) - Can reindex post-restore
3. **Retention Policies**: 7-365 days configurable
4. **Consistency**: Database-first, eventual consistency acceptable
5. **Multi-Cloud**: AWS, GCP, Azure, Local support

---

## Current Implementation Analysis

### Liferay PaaS (GCP-Based)

**Architecture**:
- Per-customer Node.js service (sits idle 95% of time)
- GCP-specific (Cloud SQL, GCS)
- Two storage strategies: SFS (Simple File System) and GCS

**Storage Pattern**:
```
gs://${bucket}/
├── backup-metadata/${backupId}.json    # Tracker file
├── document-library/${uuid}.tgz        # DL backup
└── database-dump/${uuid}.gz            # DB dump

Cloud SQL:
- Snapshots managed separately
- Listed via GCP Console
```

**Tracker File Format** (v2):
```json
{
  "backupId": "...",
  "timestamp": "...",
  "documentLibrary": {
    "uuid": "...",
    "path": "..."
  },
  "databaseDump": {
    "uuid": "...",
    "path": "..."
  }
}
```

**Key Insights**:
- ✅ Tracker files are a good pattern (metadata + index)
- ✅ Separation of concerns (DB, DL, metadata)
- ❌ Per-customer service is wasteful
- ❌ GCP-only, not portable

---

### Cloud Native (AWS-Based)

**Architecture**:
- **AWS Backup Service** (managed service, not custom code)
- Argo Workflows for restore orchestration
- Terraform modules for infrastructure
- IAM with assumed role pattern

**Components**:
```
Terraform Modules:
├── backup-vault/           # AWS Backup vault
├── backup-plan/            # Backup schedule & retention
└── backup-restore-workflow/  # Argo infrastructure

Argo Workflows:
- ClusterWorkflowTemplate for restore
- Git-based automation
- Artifact repository in S3
- Uses official images (terraform, kubectl, awscli)
```

**Restore Workflow**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: liferay-aws-backup-restore
spec:
  templates:
  - name: restore-database
  - name: restore-document-library
  - name: verify-restore
```

**Key Insights**:
- ✅ Argo Workflows are powerful and extensible
- ✅ Declarative, transparent (shell scripts + YAML)
- ✅ Managed service (AWS Backup) reduces code
- ❌ AWS-specific, not portable
- ❌ Restore-only (backup is AWS Backup Service)

---

## Previous Debate Outcomes (Go-Based Approach)

### Decisions Made
1. **Language**: Go (not Java) - for Kubernetes-native
2. **State**: Hybrid (CRD + PostgreSQL)
3. **Provider**: Simple interface (not plugins)
4. **Isolation**: ResourceQuota + Semaphore
5. **Backup**: RDS snapshots + full S3 copy
6. **Search**: Skip backups, reindex post-restore
7. **Scheduling**: Kubernetes CronJob
8. **Scope**: AWS-only MVP, add GCP/Azure later
9. **Timeline**: 12 weeks (3 phases)

### Architecture (Go)
```
backup-controller (Go + Kubebuilder)
├── AWS Provider Adapter
├── Multi-tenant Isolation
├── Backup Manifest to S3
├── PostgreSQL Archival
└── CRD Reconciliation
```

**Key Insights**:
- ✅ Kubernetes-native with controller-runtime
- ✅ Extensible provider abstraction
- ✅ Multi-tenant by design
- ❌ Go, not Java (user now wants Java)
- ❌ Custom controller, not managed service

---

## Liferay-Specific Nuances

### Document Library Challenges
1. **Size**: 100GB to 100TB (millions of files)
2. **Generated Files**: Thumbnails, previews (can be excluded)
3. **Structure**:
   ```
   document_library/
   ├── 2024/01/12345.pdf
   ├── 2024/01/12346.jpg
   └── .thumbs/          # Generated, can exclude
   ```

### Database Challenges
1. **Size**: 10GB to 10TB
2. **Engines**: MySQL and PostgreSQL
3. **Consistency**: No cross-service transactions
4. **Point-in-Time**: Desired but not MVP

### Search Index
- OpenSearch/Elasticsearch
- Reindex from database post-restore
- Not backed up (agreed in debate)

---

## Key Patterns Identified

### 1. Tracker Files / Manifests
**Pattern**: JSON metadata file indexing backup contents
**Used by**: LCP GCP (tracker files), our approach (manifests)
**Benefits**: Queryable, portable, version-able

### 2. Component Separation
**Pattern**: DB, DL, and metadata stored separately
**Used by**: All approaches
**Benefits**: Independent restore, partial backups

### 3. Managed Service vs Custom Code
**Option A**: Use cloud provider's backup service (AWS Backup, Cloud SQL Backups)
- ✅ Less code, managed, reliable
- ❌ Cloud-specific, vendor lock-in

**Option B**: Custom controller/service
- ✅ Cloud-agnostic, full control
- ❌ More code, more maintenance

### 4. Orchestration
**Pattern**: Workflow engine for multi-step restore
**Used by**: CN (Argo Workflows)
**Benefits**: Declarative, extensible, visual

---

## New Requirement: Java-Based

### Why Java?
- ✅ Liferay's primary language (team expertise)
- ✅ Rich ecosystem (Spring Boot, Quarkus)
- ✅ Cloud SDK support (AWS SDK, GCP SDK, Azure SDK)
- ✅ Kubernetes client (Fabric8)
- ❌ More verbose than Go
- ❌ Slower cold starts than Go

### Java Kubernetes Options
1. **Fabric8 Kubernetes Client**: Java client for Kubernetes
2. **Java Operator SDK**: Framework for building operators
3. **Quarkus Operator SDK**: Cloud-native Java operators
4. **Spring Cloud Kubernetes**: Spring integration

---

## Cloud-Agnostic Design Challenges

### Database Snapshots
- **AWS**: RDS CreateDBSnapshot
- **GCP**: Cloud SQL Backups API
- **Azure**: Azure Database Backups
- **Local**: pg_dump/mysqldump only

**Abstraction**:
```java
interface DatabaseBackupProvider {
    SnapshotResult createSnapshot(SnapshotRequest req);
    SnapshotStatus getSnapshotStatus(String snapshotId);
    void deleteSnapshot(String snapshotId);
}
```

### Object Storage
- **AWS**: S3 CopyObject
- **GCP**: GCS Objects.copy
- **Azure**: Blob Storage Copy
- **Local**: MinIO or filesystem

**Abstraction**:
```java
interface ObjectStorageProvider {
    CopyResult copyObjects(CopyRequest req);
    void uploadManifest(String bucket, String key, Manifest manifest);
    Manifest downloadManifest(String bucket, String key);
}
```

---

## Synthesis: What Works Across All Approaches

### ✅ Universal Patterns
1. **Metadata Files**: Tracker files (LCP) / Manifests (our approach)
2. **Component Separation**: DB, DL, metadata separate
3. **Eventual Consistency**: DB-first, DL follows
4. **Retention Policies**: Configurable, automated cleanup
5. **Kubernetes CRD**: Declarative backup requests
6. **Provider Abstraction**: Interface for multi-cloud

### ✅ Proven Technologies
1. **Argo Workflows**: Restore orchestration (CN proven)
2. **PostgreSQL**: Rich querying for backup history
3. **Cloud SDKs**: Official AWS/GCP/Azure SDKs
4. **Kubernetes**: Platform for multi-tenant isolation

### ❌ Approaches to Avoid
1. **Per-Customer Services**: Wasteful, doesn't scale
2. **Cloud-Specific Code**: Hard to migrate, vendor lock-in
3. **Complex Plugins**: Over-engineering for MVP
4. **Two-Phase Commit**: Impossible across DB + Object Storage

---

## Open Questions for Debate

### 1. Architecture: Microservice vs Operator?
**Option A: Java Microservice** (Spring Boot / Quarkus)
- REST API for backup operations
- Calls cloud provider SDKs
- Stores state in PostgreSQL
- Scheduled backups via Quartz/Spring Scheduler

**Option B: Java Kubernetes Operator** (Operator SDK)
- Reconciles BackupRequest CRDs
- Kubernetes-native (like Go approach)
- State in CRD status + PostgreSQL
- Scheduled via CronJob CRDs

### 2. Backup Strategy: Snapshots vs Dumps?
**Snapshots**:
- ✅ Fast, cheap, provider-managed
- ❌ Cloud-specific (RDS snapshots != Cloud SQL backups)

**Dumps**:
- ✅ Portable (pg_dump works everywhere)
- ❌ Slow for large databases
- ❌ Requires compute (where to run pg_dump?)

**Hybrid**:
- Use snapshots where available
- Fall back to dumps for local/unsupported

### 3. Restore: Argo Workflows vs Custom?
**Argo Workflows**:
- ✅ Proven by CN team
- ✅ Declarative, extensible
- ❌ Additional dependency

**Custom Java Restore**:
- ✅ Consistent with backup
- ✅ No additional dependencies
- ❌ Re-inventing orchestration

### 4. State Management: CRD vs PostgreSQL vs Hybrid?
**CRD Only**:
- ✅ Kubernetes-native
- ❌ Limited querying (no SQL)

**PostgreSQL Only**:
- ✅ Rich querying
- ❌ Not Kubernetes-native

**Hybrid** (our previous choice):
- ✅ Best of both worlds
- ❌ Complexity of two systems

### 5. Multi-Tenancy: How to Isolate Projects?
**Options**:
- Namespace-per-project (heavy)
- ResourceQuota + Semaphore (our choice)
- Priority queues
- Rate limiting

---

## Constraints & Trade-offs

### Technical Constraints
1. **No Cross-Service Transactions**: DB + Object Storage can't be atomic
2. **Large Data Volumes**: 100TB DL requires streaming, not in-memory
3. **Multi-Cloud APIs**: Each provider has different semantics
4. **Kubernetes Limits**: etcd size limits for CRD status

### Business Constraints
1. **Team Expertise**: Java preferred over Go
2. **Timeline**: 3 months for MVP
3. **Team Size**: 2 engineers
4. **Cost**: Customer pays for storage (not blocker)

### Trade-offs
| Aspect | Option A | Option B |
|--------|----------|----------|
| Language | Java | Go |
| Architecture | Microservice | Operator |
| Backup | Managed Service | Custom Code |
| Restore | Argo Workflows | Custom |
| State | PostgreSQL | CRD + PostgreSQL |

---

## Success Criteria (Unchanged)

### Functional
- Multi-cloud support (AWS, GCP, Azure, Local)
- Multi-tenant isolation
- Scheduled backups
- Manual on-demand backups
- Retention policy enforcement
- Backup verification
- Restore capability (Phase 2)

### Non-Functional
- Handle 100TB document libraries
- Handle 10TB databases
- Max 2 concurrent backups per project
- <5min to backup 1GB database
- <30min to backup 100GB DL
- 99.9% backup success rate

---

## Next Step: Debate with OpenAI

### Focus Questions
1. **Java Microservice vs Java Operator**: Which fits LC2 better?
2. **Cloud Abstraction**: Best way to abstract AWS/GCP/Azure in Java?
3. **Argo Workflows**: Should we adopt for restore orchestration?
4. **State Management**: Stick with hybrid or go CRD-only?
5. **Backup Strategy**: Snapshots + dumps or snapshots-only MVP?

### Debate Goal
Design a **cloud-agnostic, Java-based** backup solution that:
- Leverages proven patterns (tracker files, Argo Workflows)
- Works on AWS, GCP, Azure, and local
- Scales to multi-tenant (100+ projects)
- Aligns with LC2 principles (GitOps, Kubernetes-native)
- Can be built in 3 months with 2 engineers

---

**Document Version**: 1.0
**Created**: 2025-11-09
**Purpose**: Input for OpenAI debate on Java-based cloud-agnostic solution
