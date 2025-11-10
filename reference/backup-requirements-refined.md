# Liferay Cloud 2.0 (LC2) Backup Service Requirements - REFINED

**Status:** Architecture (Post-Debate Refinement)
**Date:** 2025-11-09
**Principles:** *GitOps-first • Kubernetes-native • Multi-cloud ready • KISS • Pragmatic MVP focus*

---

## Executive Summary

**Goal**: Replace per-customer backup containers with a **centralized, cloud-agnostic backup service** natively integrated into LC2.

**Key Decision After Architecture Debate**:
- **MVP Focus**: AWS support, core backup/restore, 3-month delivery with 2 engineers
- **Technology**: Enhance existing **Go backup-controller** (not new Java service)
- **Approach**: Pragmatic, focused on critical path, defer optimization to later phases

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Liferay Backup Nuances](#2-liferay-backup-nuances)
3. [MVP Architecture (3-Month Target)](#3-mvp-architecture-3-month-target)
4. [Critical Decisions & Trade-offs](#4-critical-decisions--trade-offs)
5. [Implementation Roadmap](#5-implementation-roadmap)
6. [Post-MVP Enhancements](#6-post-mvp-enhancements)

---

## 1) Problem Statement

### Current State (Liferay PaaS v1)

**service-backup/** (Node.js/TypeScript):
- Deployed **per customer** (1 container per project)
- Sits idle ~95% of time (backups are nightly)
- GCP-only (Cloud SQL, Cloud Storage, Storage Transfer Service)
- Customer-managed image updates

**backup-controller/** (Go/Kubebuilder):
- Kubernetes operator with CRDs
- GCP-specific API calls
- Works well but limited to Google Cloud

### Problems

1. **Resource Waste**: Hundreds of idle containers reserving CPU/memory
2. **Cloud Lock-In**: Cannot support LC2's AWS/Azure/GCP strategy
3. **Operational Overhead**: Per-customer deployment, configuration drift, update coordination
4. **Inconsistent Experience**: Different backup semantics across clouds

### Solution Requirements

✅ **Centralized**: Single service for all projects (multi-tenant)
✅ **Cloud-Agnostic**: Support AWS, Azure, GCP, and local
✅ **Native to LC2**: Integrated with Cockpit, GitOps workflows, and RBAC
✅ **Efficient**: Leverage provider-managed services (RDS snapshots, S3, etc.)

---

## 2) Liferay Backup Nuances

### The Three-Component Challenge

Liferay DXP requires coordinated backup of:

1. **Database** (MySQL/PostgreSQL)
   - Size: 10 GB - 10+ TB
   - Schema: `lportal` or customer-named
   - Requirement: Consistent point-in-time snapshot

2. **Document Library** (Object Storage)
   - Size: 100 GB - 100+ TB
   - Format: Hierarchical structure (`/document_library/{companyId}/{repoId}/...`)
   - Optimization: Can exclude generated files (thumbnails, previews) → 30-60% size reduction

3. **Search Indexes** (OpenSearch/Elasticsearch)
   - **MVP Decision**: Skip backup, rely on post-restore reindex
   - Reindex time: ~1-10 GB/hour
   - **Future**: Add snapshot support for >10TB datasets

### Consistency Strategy (Debated & Decided)

**Problem**: Database and Document Library are separate systems with no distributed transactions.

**Rejected Approaches**:
- ❌ Two-phase commit (impossible across PostgreSQL + S3)
- ❌ Quiesce application (downtime unacceptable)
- ❌ Application-level coordination (Liferay is proprietary)

**Adopted Approach: Database-First + Eventual Consistency**
```
1. Backup Database at T0 (captures consistent DB state)
2. Backup Document Library immediately (may have T0+delta files)
3. Accept: DL may contain files not in DB (safe, no broken refs)
4. Post-Restore: Optional cleanup script removes orphaned files
```

**Rationale**:
- ✅ No downtime
- ✅ No broken references (DB won't point to missing files)
- ✅ Simple to implement
- ✅ Works across all cloud providers

### Backup Strategies Per Component

#### Database

**Option 1: Provider Snapshot** (MVP for production)
- **AWS**: RDS automated snapshots + manual snapshots
- **How**: Call RDS API `CreateDBSnapshot`
- **Pros**: Fast (minutes), incremental, cheap, PITR support
- **Cons**: Cloud-specific, can't migrate between providers
- **When**: Production, same-cloud restores, DR

**Option 2: Logical Dump** (Post-MVP for portability)
- **Tool**: `pg_dump` or `mysqldump`
- **How**: Kubernetes Job runs dump, streams to S3
- **Pros**: Portable, human-readable, cross-cloud migration
- **Cons**: Slow (GB/hour), requires processing time
- **When**: Cross-cloud migration, compliance archives

**MVP Decision**: Snapshot only (RDS for AWS)

#### Document Library

**Strategy: Full S3 Copy** (MVP)
```
Source: s3://liferay-project-retail-prd-dl/
Destination: s3://liferay-backups/retail/prd/20251109-143000/dl/
```

**Implementation**:
- Use AWS SDK S3 CopyObject API
- Parallelize with goroutines (10-50 concurrent copies)
- Track progress (objects copied / total objects)
- Option: Exclude generated files via prefix filter

**Post-MVP Optimizations**:
- Incremental backups (copy only changed files via ETag comparison)
- S3 Batch Operations (AWS-managed, scales to billions of objects)
- Intelligent Tiering (auto-transition to cheaper storage classes)

---

## 3) MVP Architecture (3-Month Target)

### High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│ LC2 Cluster (EKS)                                           │
│                                                             │
│  ┌──────────────────────────────────────┐                  │
│  │ liferay-cockpit namespace            │                  │
│  │                                       │                  │
│  │  ┌────────────────────────────────┐  │                  │
│  │  │ backup-controller (Go)         │  │                  │
│  │  │ - Watches BackupRequest CRDs   │  │                  │
│  │  │ - Multi-tenant reconciliation  │  │                  │
│  │  │ - AWS provider adapter         │  │                  │
│  │  │ - Status updates               │  │                  │
│  │  └───────────┬────────────────────┘  │                  │
│  │              │                        │                  │
│  │              │ Writes status          │                  │
│  │              v                        │                  │
│  │  ┌────────────────────────────────┐  │                  │
│  │  │ PostgreSQL (Cockpit DB)        │  │                  │
│  │  │ - Backup history archive       │  │                  │
│  │  │ - Queryable audit log          │  │                  │
│  │  └────────────────────────────────┘  │                  │
│  └──────────────────────────────────────┘                  │
│                                                             │
│  ┌──────────────────────────────────────┐                  │
│  │ Project Namespaces                   │                  │
│  │                                       │                  │
│  │  proj-retail-prd:                    │                  │
│  │    ┌──────────────────────────────┐  │                  │
│  │    │ BackupRequest CRD            │  │                  │
│  │    │ - spec: db + dl backup       │  │                  │
│  │    │ - status: InProgress → Done  │  │                  │
│  │    └──────────────────────────────┘  │                  │
│  └──────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
                       │
                       v
         ┌─────────────────────────────┐
         │ AWS Services                │
         │ - RDS (snapshots)           │
         │ - S3 (object copies)        │
         │ - IAM (IRSA)                │
         └─────────────────────────────┘
```

### Component Responsibilities

#### **backup-controller** (Go - Enhanced Existing)

**Current Capabilities** (already implemented):
- CRD reconciliation loop
- GCP API integration (Cloud SQL, Cloud Storage)
- Operation status tracking

**MVP Enhancements** (new work):
- ✅ Add AWS provider adapter (RDS, S3)
- ✅ Multi-tenant isolation (semaphore per project)
- ✅ Backup manifest generation (write to S3)
- ✅ Archive completed backups to PostgreSQL
- ✅ Improved error handling and retries

**Does NOT Handle** (future):
- REST API (Cockpit calls Kubernetes API directly)
- Advanced scheduling (use Kubernetes CronJob)
- Cross-cloud migration
- Incremental backups

#### **BackupRequest CRD** (Already Exists, Enhance)

```yaml
apiVersion: backup.liferay.cloud/v1alpha1
kind: BackupRequest
metadata:
  name: retail-prd-backup-20251109-143000
  namespace: proj-retail-prd
  labels:
    backup.liferay.cloud/project: retail
    backup.liferay.cloud/environment: prd
spec:
  projectId: retail
  environment: prd

  database:
    type: snapshot  # MVP: snapshot only
    instanceId: retail-prd-db
    retention: 7d

  documentLibrary:
    sourceBucket: liferay-retail-prd-dl
    destinationBucket: liferay-backups
    destinationPrefix: retail/prd/20251109-143000/dl
    excludeGeneratedFiles: true  # optional optimization

  # Search omitted in MVP (reindex post-restore)

status:
  state: Succeeded  # Pending | InProgress | Succeeded | Failed
  startTime: "2025-11-09T14:30:00Z"
  completionTime: "2025-11-09T14:45:00Z"

  database:
    snapshotId: rds:us-east-1:retail-prd-20251109-143000
    size: "45.2 GB"
    status: Succeeded

  documentLibrary:
    objectCount: 12453
    totalSize: "125.8 GB"
    status: Succeeded
    manifestPath: s3://liferay-backups/retail/prd/20251109-143000/manifest.json

  conditions:
    - type: Ready
      status: "True"
      reason: BackupCompleted
      message: "Database and document library backup completed"
```

#### **Backup Manifest** (Disaster Recovery)

Written to S3 alongside backup data for self-healing:

```json
{
  "backupId": "retail-prd-backup-20251109-143000",
  "projectId": "retail",
  "environment": "prd",
  "timestamp": "2025-11-09T14:30:00Z",
  "initiator": "alice@example.com",
  "database": {
    "type": "rds-snapshot",
    "snapshotId": "rds:us-east-1:retail-prd-20251109-143000",
    "instanceId": "retail-prd-db",
    "size": 48590020608,
    "checksum": "sha256:abc123..."
  },
  "documentLibrary": {
    "bucket": "liferay-backups",
    "prefix": "retail/prd/20251109-143000/dl",
    "objectCount": 12453,
    "totalSize": 135223918592,
    "excludedPatterns": ["*/thumbnails/*", "*/previews/*"]
  }
}
```

**Use Case**: If PostgreSQL is corrupted, rebuild backup history from S3 manifests.

---

## 4) Critical Decisions & Trade-offs

### Decision 1: Go (Not Java, Not Hybrid)

**Rationale**:
- ✅ Existing backup-controller is Go (leverage, don't replace)
- ✅ Kubernetes-native (client-go, controller-runtime)
- ✅ Team has Go expertise
- ✅ Simpler than Java + Go hybrid
- ✅ Smaller binaries, faster startup

**Trade-off**:
- ❌ Cockpit is Java (different language)
- **Mitigation**: Cockpit calls Kubernetes API directly (standard pattern)

### Decision 2: CRD + PostgreSQL Hybrid (Not Pure CRD)

**Rationale**:
- ✅ CRD for active operations (GitOps-native, survives pod restarts)
- ✅ PostgreSQL for completed backup history (rich queries, pagination)
- ✅ Async archival (eventual consistency acceptable)

**Trade-off**:
- ❌ Two sources of truth (sync complexity)
- **Mitigation**: CRD is authoritative, PostgreSQL is read-only archive

### Decision 3: Simple Provider Interface (Not Plugins)

**Rationale**:
- ✅ MVP = AWS only (YAGNI for plugin complexity)
- ✅ Can refactor to plugins after 2+ clouds working
- ✅ Simple interface sufficient for 3-4 providers

**Trade-off**:
- ❌ Less flexible for future clouds
- **Mitigation**: Design interface with extension in mind, refactor in phase 2

### Decision 4: ResourceQuota + Semaphore Isolation (Not Namespace Pods)

**Rationale**:
- ✅ Kubernetes ResourceQuota limits CPU/memory per project
- ✅ In-memory semaphore limits concurrent backups per project (max 2)
- ✅ Simpler than deploying worker pods per project
- ✅ Good enough for MVP

**Trade-off**:
- ❌ Weaker isolation than namespace pods
- **Mitigation**: Monitor for noisy neighbor issues, add pods if needed

### Decision 5: RDS Snapshots + Full S3 Copy (No Incremental)

**Rationale**:
- ✅ RDS snapshots are cheap, fast, managed (incremental automatically)
- ✅ S3 full copy is simple (no dedup logic)
- ✅ Customer pays for storage (not MVP blocker)

**Trade-off**:
- ❌ Higher storage costs
- **Mitigation**: Add incremental DL backups in phase 3 (cost optimization)

### Decision 6: Skip Search Backups (Reindex Post-Restore)

**Rationale**:
- ✅ Search indexes are derived from database (can regenerate)
- ✅ Reindex time acceptable for most customers (<4 hours)
- ✅ Simplifies backup (2 components instead of 3)

**Trade-off**:
- ❌ Longer RTO for large datasets
- **Mitigation**: Add search snapshots in phase 3 for enterprise tier

### Decision 7: Kubernetes CronJob Scheduling (Not Quartz)

**Rationale**:
- ✅ Declarative, GitOps-native
- ✅ No additional dependencies
- ✅ Works on local and cloud

**Trade-off**:
- ❌ Less flexible than Quartz (no dynamic schedules)
- **Mitigation**: CronJob schedules defined in Git, easy to change via PR

---

## 5) Implementation Roadmap

### Phase 1: MVP Foundation (Weeks 1-4)

**Goal**: Basic backup for AWS with single project

**Deliverables**:
- [ ] AWS provider adapter (RDS snapshots, S3 copy)
- [ ] Enhanced BackupRequest CRD (AWS-specific fields)
- [ ] Database backup reconciliation loop
- [ ] Document library backup reconciliation loop
- [ ] Backup manifest generation (write to S3)
- [ ] Basic status reporting
- [ ] Integration tests (LocalStack)

**Acceptance Criteria**:
- ✅ Create RDS snapshot via BackupRequest CRD
- ✅ Copy S3 objects to backup bucket
- ✅ Update CRD status with progress
- ✅ Write manifest.json to S3

### Phase 2: Multi-Tenant & Restore (Weeks 5-8)

**Goal**: Production-ready with restore capability

**Deliverables**:
- [ ] Multi-tenant isolation (semaphore per project)
- [ ] RestoreRequest CRD implementation
- [ ] Restore from RDS snapshot (to new instance)
- [ ] Restore from S3 backup (to new bucket)
- [ ] PostgreSQL archival (async job)
- [ ] Backup history API (query PostgreSQL)
- [ ] GitOps integration (Argo CD sync)

**Acceptance Criteria**:
- ✅ Multiple projects can backup concurrently without interference
- ✅ Restore to new namespace (safe clone)
- ✅ Query backup history via kubectl or PostgreSQL
- ✅ Argo CD can create BackupRequest from Git

### Phase 3: Operational Excellence (Weeks 9-12)

**Goal**: Production deployment and monitoring

**Deliverables**:
- [ ] Prometheus metrics (backup duration, size, success rate)
- [ ] Grafana dashboards
- [ ] Alert rules (backup failures, quota exceeded)
- [ ] Scheduled backup automation (CronJob)
- [ ] Retention policy enforcement (delete old backups)
- [ ] Documentation (runbooks, troubleshooting)
- [ ] DR drill automation

**Acceptance Criteria**:
- ✅ Deployed to production EKS cluster
- ✅ Scheduled backups running nightly
- ✅ Alerts firing for failures
- ✅ Backup retention policy enforced (7 days)
- ✅ DR drill successfully restores backup

---

## 6) Post-MVP Enhancements

### Phase 4: Azure & GCP Support (Months 4-5)

**Deliverables**:
- Azure provider adapter (Azure Database, Blob Storage)
- GCP provider adapter (Cloud SQL, Cloud Storage)
- Local provider adapter (pg_dump, MinIO)
- Provider abstraction refactoring (if needed)

### Phase 5: Cost Optimization (Month 6)

**Deliverables**:
- Incremental document library backups (ETag-based dedup)
- S3 Intelligent Tiering lifecycle policies
- Backup compression (optional)
- Storage cost reporting

### Phase 6: Advanced Features (Months 7-9)

**Deliverables**:
- Search index backups (OpenSearch snapshots)
- Logical database dumps (cross-cloud portability)
- In-place restore (with approval workflow)
- Backup validation (periodic test restores)
- REST API for Cockpit (if Kubernetes API insufficient)

### Phase 7: Enterprise Features (Months 10-12)

**Deliverables**:
- Cross-region replication (DR)
- Compliance reporting (SOC2, GDPR retention)
- Backup encryption (customer-managed keys)
- Fine-grained restore (table-level, file-level)
- FinOps dashboards (cost per project)

---

## Technology Stack

### Core

- **Language**: Go 1.21+
- **Framework**: Kubebuilder (controller-runtime)
- **Kubernetes**: client-go v0.29+

### Cloud SDKs

```go
// AWS
import (
    "github.com/aws/aws-sdk-go-v2/service/rds"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

// Future: Azure
import (
    "github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/postgresql/armpostgresqlflexibleservers"
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
)

// Future: GCP
import (
    "cloud.google.com/go/storage"
    sqladmin "google.golang.org/api/sqladmin/v1beta4"
)
```

### Database

- **PostgreSQL**: Cockpit management DB (backup history archive)
- **ORM**: pgx (native PostgreSQL driver)
- **Migrations**: golang-migrate

### Observability

- **Metrics**: Prometheus (controller-runtime built-in)
- **Logging**: Structured JSON (logr)
- **Tracing**: (Future) OpenTelemetry

---

## Acceptance Criteria (MVP)

### Functional

- ✅ Create AWS RDS snapshot via BackupRequest CRD
- ✅ Copy S3 objects to backup bucket (full copy)
- ✅ Restore RDS snapshot to new instance
- ✅ Restore S3 backup to new bucket
- ✅ Update CRD status with real-time progress
- ✅ Write backup manifest to S3 (disaster recovery)
- ✅ Archive completed backups to PostgreSQL
- ✅ Multi-tenant isolation (projects don't interfere)
- ✅ GitOps integration (Argo CD can create BackupRequests)

### Non-Functional

- ✅ Database snapshot: < 5 minutes for 50 GB
- ✅ Document library copy: > 100 MB/s throughput
- ✅ Support 10+ concurrent backups (across projects)
- ✅ Backup success rate: > 99%
- ✅ Automatic retries on transient failures
- ✅ Prometheus metrics exported
- ✅ Structured logs (JSON, queryable)

---

## What's NOT in MVP

- ❌ Azure/GCP support (phase 4)
- ❌ Search index backups (phase 6)
- ❌ Incremental backups (phase 5)
- ❌ Logical dumps (phase 6)
- ❌ In-place restore (phase 6)
- ❌ REST API (Cockpit uses Kubernetes API)
- ❌ Quartz scheduling (use CronJob)
- ❌ Event sourcing (simple audit in PostgreSQL)
- ❌ Backup validation automation (manual for MVP)
- ❌ Cross-region replication (phase 7)

---

## Conclusion

**What Changed After Debate?**

1. **Language**: Java → **Go** (leverage existing controller)
2. **Scope**: Multi-cloud MVP → **AWS-only MVP** (add clouds incrementally)
3. **Complexity**: Plugin architecture → **Simple interface** (refactor later)
4. **Isolation**: Namespace pods → **Semaphore + ResourceQuota** (simpler)
5. **Optimization**: Incremental backups → **Full copy** (optimize in phase 5)
6. **Timeline**: Ambitious 8 weeks → **Realistic 12 weeks** (3 months)

**Why These Changes?**

- ✅ **Pragmatism**: Focus on delivering working AWS backup in 3 months
- ✅ **Leverage existing**: Don't throw away working Go controller
- ✅ **Reduce risk**: Proven patterns, fewer unknowns
- ✅ **Team capacity**: 2 engineers, realistic timeline
- ✅ **Iterative**: MVP first, optimize later

**Core Value Delivered**:
- ✅ Centralized backup service (no more per-customer containers)
- ✅ AWS support (primary cloud for LC2)
- ✅ GitOps-native (fits LC2 architecture)
- ✅ Production-ready (monitoring, isolation, DR)

**Next Steps**:
1. Review and approve refined requirements
2. Create detailed implementation plan (GitHub issues)
3. Set up development environment (LocalStack for AWS)
4. Start Phase 1 (weeks 1-4)

---

**Document Version**: 2.0 (Post-Debate Refinement)
**Last Updated**: 2025-11-09
**Debate Participants**: LC2 Team + OpenAI GPT-4o
**Status**: Ready for Implementation
