# Liferay Cloud 2.0 (LC2) Backup Service Requirements

**Status:** Draft (Architecture)
**Date:** 2025-11-09
**Principles:** *GitOps-first • Kubernetes stateless by default • Cloud-agnostic • KISS • Centralized service model*

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Analysis](#2-current-state-analysis)
3. [Problem Statement](#3-problem-statement)
4. [Proposed Solution](#4-proposed-solution)
5. [Liferay-Specific Backup Nuances](#5-liferay-specific-backup-nuances)
6. [Architecture](#6-architecture)
7. [Cloud Provider Abstraction](#7-cloud-provider-abstraction)
8. [Data Model & API](#8-data-model--api)
9. [Technology Stack](#9-technology-stack)
10. [Implementation Phases](#10-implementation-phases)
11. [Acceptance Criteria](#11-acceptance-criteria)
12. [Non-Goals](#12-non-goals)

---

## 1) Executive Summary

**Objective**: Replace the current per-customer deployed backup service (Node.js) with a **centralized, cloud-agnostic Java-based microservice** that is natively integrated into the LC2 platform.

**Key Changes**:
- **Deployment Model**: From per-customer container image → centralized platform microservice
- **Technology**: Node.js/TypeScript → Java (Spring Boot)
- **Scope**: GCP-only → AWS, Azure, GCP, and local
- **Integration**: Standalone service → native LC2 Cockpit component
- **Resource Efficiency**: Always-running per-customer → shared, multi-tenant service

**Value Proposition**:
- Customers don't manage backup infrastructure
- Reduced operational overhead (no per-tenant deployments)
- Consistent backup/restore experience across all clouds
- Leverages provider-native services for durability and cost efficiency
- Better resource utilization through multi-tenancy

---

## 2) Current State Analysis

### 2.1 Existing Components

#### **service-backup/** (Node.js/TypeScript)
**Purpose**: HTTP API service for backup operations
**Technology**: Express.js, Node.js 22.17.1, TypeScript
**Deployment**: One container per customer project
**Cloud Support**: GCP only

**Key Features**:
- REST API endpoints for backup create/restore/list/delete
- Scheduled backup creation (cron-based)
- Integration with GCP services (Cloud Storage, Cloud SQL)
- Creates Kubernetes CRDs (`LiferayBackup`) for the operator to process
- Database activities tracking
- User authentication and authorization

**Limitations**:
- GCP-specific (`@google-cloud/storage`, `@googleapis/sqladmin`)
- Requires deployment per customer
- Sits idle most of the time (backups are typically nightly)
- No support for AWS or Azure
- Complex per-customer configuration management

#### **backup-controller/** (Go/Kubebuilder)
**Purpose**: Kubernetes operator that reconciles backup CRDs
**Technology**: Go, Kubebuilder (operator-sdk)
**Deployment**: Cluster-scoped operator
**Cloud Support**: GCP only

**Custom Resource Definitions (CRDs)**:
1. `LiferayBackup` - Orchestrates full backup (DB dump + snapshot + document library)
2. `LiferayDatabaseDump` - Logical database export (mysqldump/pg_dump)
3. `LiferayDatabaseSnapshot` - Cloud provider snapshot
4. `LiferayDocumentLibraryBackup` - Object storage copy
5. `LiferayDatabaseDumpOperation` - Tracks dump job execution
6. `LiferayDatabaseSnapshotOperation` - Tracks snapshot operation
7. `STSJobOperation` - Storage Transfer Service job tracking

**Reconciliation Logic**:
- Watches CRD creation/updates
- Calls GCP APIs (Cloud SQL Admin, Cloud Storage, Storage Transfer Service)
- Updates CRD status with operation results
- Handles retries and error conditions

**Limitations**:
- Tightly coupled to GCP APIs
- No abstraction layer for multi-cloud support
- Complex CRD hierarchy

---

## 3) Problem Statement

### 3.1 Operational Challenges

**Resource Waste**:
- Each customer runs a dedicated backup service container 24/7
- Backup operations are typically scheduled (e.g., nightly at 2 AM)
- Service sits idle ~95-99% of the time
- Memory and CPU reserved but unused

**Deployment Complexity**:
- Customers must manage backup service image versions
- Configuration drift across customers
- Difficult to roll out updates (requires coordination with all customers)
- Each deployment needs separate monitoring and troubleshooting

**Cloud Lock-In**:
- Current implementation is GCP-specific
- Cannot support LC2's multi-cloud strategy (AWS EKS, Azure AKS)
- Custom abstractions would be needed for each cloud

**Inconsistent Experience**:
- Different backup semantics across clouds
- No unified API or workflows
- Local development doesn't match cloud behavior

### 3.2 Liferay-Specific Challenges

**Three-Component Backup**:
Liferay DXP requires coordinated backup of:
1. **Database** (Liferay schema: `lportal` or custom)
2. **Document Library** (file storage)
3. **Search Indexes** (OpenSearch/Elasticsearch)

**Consistency Requirements**:
- Database and Document Library must be consistent at a point-in-time
- Inconsistent backups can cause:
  - Broken file references (DB points to files that don't exist)
  - Orphaned files (files exist but no DB records)
  - Search index mismatches

**Database Considerations**:
- **Size**: Can range from 10 GB to 10+ TB
- **Type**: MySQL or PostgreSQL
- **Operations**: Requires both logical dumps (portability) and snapshots (speed)
- **Downtime**: Production systems require online backups

**Document Library Considerations**:
- **Size**: Can be 100 GB to 100+ TB
- **Format**: Hierarchical directory structure or object storage
- **Features**: Supports generated files (thumbnails, previews) that can be excluded
- **Performance**: Large DL backups can saturate network or storage I/O

**Search Indexes**:
- **Strategy**: Typically regenerated from database post-restore
- **Alternative**: Snapshot if reindex time is prohibitive (large data sets)
- **Complexity**: Can be skipped for many use cases (restore + reindex workflow)

---

## 4) Proposed Solution

### 4.1 Architecture Overview

**Centralized Backup Microservice** (Java/Spring Boot)
- Single multi-tenant service deployed in the `liferay-cockpit` namespace
- Handles backup/restore operations for **all projects** in the cluster
- Exposes REST API consumed by Cockpit UI/API
- Implements cloud provider abstraction layer
- Integrates with LC2's audit and RBAC systems

**Key Principles**:
1. **Cloud-Agnostic Core**: Business logic independent of cloud provider
2. **Provider Adapters**: Pluggable implementations for AWS/Azure/GCP/Local
3. **Leverage Managed Services**: Use provider-native backup features
4. **Kubernetes Stateless**: No PVCs; all backup data in external storage
5. **GitOps Integration**: Backup schedules and policies defined in Git
6. **Audit Everything**: Full traceability of all backup/restore operations

### 4.2 Deployment Model

```
┌─────────────────────────────────────────────────────────┐
│ LC2 Cockpit Namespace (liferay-cockpit)                │
│                                                         │
│  ┌──────────────────┐      ┌─────────────────┐        │
│  │ Cockpit API      │─────>│ Backup Service  │        │
│  │ (Spring Boot)    │      │ (Spring Boot)   │        │
│  └──────────────────┘      └────────┬────────┘        │
│                                      │                  │
│                                      v                  │
│                            ┌─────────────────┐         │
│                            │ Provider Layer  │         │
│                            │ (Adapters)      │         │
│                            └────────┬────────┘         │
└─────────────────────────────────────┼──────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
         ┌──────────v────┐  ┌────────v──────┐  ┌──────v──────┐
         │ AWS Adapter   │  │ Azure Adapter │  │ GCP Adapter │
         │ - RDS         │  │ - Azure DB    │  │ - Cloud SQL │
         │ - S3          │  │ - Blob        │  │ - GCS       │
         │ - OpenSearch  │  │ - OpenSearch  │  │ - Firestore │
         └───────────────┘  └───────────────┘  └─────────────┘
```

**Benefits**:
- Single service instance per cluster (vs. per project)
- Shared connection pools, caches, and resources
- Centralized monitoring and logging
- Easy to update and rollback
- Consistent behavior across all projects

---

## 5) Liferay-Specific Backup Nuances

### 5.1 The Three-Component Backup

#### **Database Backup** (Two Methods)

**Method 1: Logical Dump** (Portability)
- **Tool**: `mysqldump` or `pg_dump`
- **Use Case**: Database migration, version upgrades, cross-cloud restore
- **Pros**: Portable, can edit/inspect, includes schema
- **Cons**: Slow for large databases (GB/hour), requires processing
- **Liferay Specifics**:
  - Dump only `lportal` schema (or customer-named schema)
  - Exclude MySQL system tables (`mysql`, `information_schema`, `performance_schema`)
  - Include stored procedures and triggers
  - Option to exclude generated tables (e.g., analytics aggregates)

**Method 2: Provider Snapshot** (Speed)
- **Tool**: RDS snapshots, Azure DB backups, Cloud SQL snapshots
- **Use Case**: Fast backup/restore, disaster recovery, same-cloud operations
- **Pros**: Very fast (minutes), efficient (incremental), automated retention
- **Cons**: Cloud-specific, cannot migrate between clouds, opaque format
- **Liferay Specifics**:
  - Capture point-in-time state
  - Supports PITR (Point-In-Time Recovery) for recent changes
  - No special Liferay configuration needed

**Recommended Strategy**:
- **Production**: Snapshots (daily) + Logical dumps (weekly)
- **Dev/UAT**: Logical dumps only (smaller, more portable)
- **DR Planning**: Snapshots for speed, dumps for migration

#### **Document Library Backup**

**Storage Types**:
1. **File System** (legacy, not recommended in LC2)
   - Mounted PVC in Kubernetes
   - Complex to backup (requires pod filesystem access)
   - LC2 discourages PVCs for stateful data

2. **Object Storage** (recommended)
   - S3, Azure Blob, Google Cloud Storage
   - Native versioning and lifecycle policies
   - No filesystem access needed

**Backup Strategies**:

**Strategy 1: Object Copy** (Cross-bucket)
```
Source Bucket: liferay-project-123-dl
Destination Bucket: liferay-project-123-dl-backups/2025-11-09/
```
- **Use Case**: Create immutable backup snapshots
- **Implementation**: Use provider APIs to copy objects
  - AWS: S3 Batch Operations, S3 Replication
  - Azure: AzCopy, Blob Batch
  - GCP: Storage Transfer Service
- **Pros**: Immutable, separate from production, supports retention policies
- **Cons**: Storage costs (full copies), time for large DLs

**Strategy 2: Object Versioning** (In-place)
```
Bucket: liferay-project-123-dl (versioning enabled)
All objects retain historical versions automatically
```
- **Use Case**: Protect against accidental deletion/modification
- **Implementation**: Enable bucket versioning
- **Pros**: No manual backup process, instant recovery
- **Cons**: Same bucket (not isolated), complex retention management

**Strategy 3: Incremental Sync** (Hybrid)
```
Daily incremental backup of changed files to backup bucket
Weekly full snapshot for validation
```
- **Use Case**: Large document libraries (>10 TB)
- **Implementation**: Track object metadata (etag, lastModified)
- **Pros**: Efficient for large data sets
- **Cons**: More complex logic, requires state tracking

**Liferay-Specific Optimizations**:

**Exclude Generated Files**:
Liferay generates derivative files that can be recreated:
- **Thumbnails**: `/document_library/thumbnails/`
- **Previews**: `/document_library/previews/`
- **Cache**: `/document_library/temp/`

**Configuration**: Add option `excludeGeneratedDoclibFiles: true`
**Benefit**: Reduce backup size by 30-60% in many cases
**Risk**: Longer restore time (regeneration needed)

**Folder Structure Preservation**:
- Liferay stores metadata (permissions, versions, workflow) in database
- Backup must preserve object keys/paths exactly
- Example: `document_library/12345/67890/filename.pdf`
  - `12345` = Company ID
  - `67890` = Repository ID

#### **Search Index Backup** (Optional)

**Default Strategy**: **Skip and Reindex**
- Search indexes are derived from database content
- Post-restore workflow: Database restore → Trigger full reindex
- Reindex time: ~1-10 GB/hour (depends on content volume)

**Alternative Strategy**: **Snapshot for Large Datasets**
- **When**: Reindex time exceeds acceptable RTO (e.g., >4 hours)
- **How**: OpenSearch/Elasticsearch snapshot API
  - AWS: OpenSearch snapshots to S3
  - Azure: Elasticsearch snapshots to Blob
  - GCP: Elasticsearch snapshots to GCS
- **Pros**: Faster restore (minutes vs hours)
- **Cons**: More complexity, storage costs, index version compatibility

**Recommendation**:
- **MVP**: Skip search backup, document reindex workflow
- **Future**: Add snapshot support for large-scale customers

---

### 5.2 Backup Consistency and Ordering

**Critical Requirement**: Database and Document Library must be consistent

**Problem Scenario**:
```
10:00 AM: User uploads document "contract.pdf"
10:01 AM: Database record created (ID=12345, path=/dl/contract.pdf)
10:02 AM: Document Library backup starts (captures contract.pdf)
10:05 AM: Database backup starts (missing ID=12345 record)
Result: Backup has orphaned file with no DB reference
```

**Solution Strategies**:

**Strategy 1: Quiesce Application** (Safest, Downtime)
1. Put Liferay in maintenance mode (read-only)
2. Backup database (snapshot or dump)
3. Backup document library (copy or snapshot)
4. Resume Liferay (read-write)

**Pros**: Perfect consistency
**Cons**: Requires downtime (unacceptable for 24/7 production)

**Strategy 2: Database-First** (Recommended)
1. Backup database (captures DB state at T0)
2. Immediately backup document library
3. Accept: DL may have files not in DB (uploaded after T0)
4. Result: Safe (extra files, no broken references)

**Pros**: No downtime, safe for restore
**Cons**: May have orphaned files (cleaned up post-restore)

**Strategy 3: Snapshot-Everything** (Cloud-Native)
1. Use provider-coordinated snapshots
   - AWS: RDS snapshot + S3 versioning (tag with same timestamp)
   - Azure: Database backup + Blob snapshot (same time)
   - GCP: Cloud SQL snapshot + GCS versioning
2. Rely on provider's consistency guarantees

**Pros**: Fast, no downtime, provider-managed
**Cons**: Cloud-specific, requires tagging/coordination

**LC2 Recommendation**: **Strategy 2 (Database-First)** for MVP
- No downtime required
- Simple to implement across clouds
- Acceptable consistency trade-offs (DL can be cleaned post-restore)

---

### 5.3 Backup Validation

**Problem**: Backups can be corrupt or incomplete

**Validation Checklist**:

**Database Dump Validation**:
- ✅ File size > 0 and matches expected range
- ✅ Gzip integrity check (`gzip -t`)
- ✅ SQL syntax check (parse first 100 lines)
- ✅ Contains expected schema markers (`CREATE TABLE DLFileEntry`)
- ✅ Row count spot-check (e.g., `SELECT COUNT(*) FROM User_`)

**Database Snapshot Validation**:
- ✅ Snapshot status = "available"
- ✅ Snapshot size matches database size
- ✅ Snapshot tagged with backup ID and timestamp

**Document Library Validation**:
- ✅ Object count matches expected range
- ✅ Total size matches source bucket (±threshold)
- ✅ Random sampling (verify 10 random objects exist)
- ✅ Metadata preservation (content-type, custom headers)

**Restoration Test** (Periodic):
- Monthly DR drill: Restore backup to isolated namespace
- Validate Liferay boots successfully
- Spot-check content (users, sites, documents)
- Document RTO/RPO metrics

---

### 5.4 Restore Workflows

**Use Case 1: Full Disaster Recovery**
```
Scenario: Production database corrupted, need to restore to last good state
Steps:
1. Create new namespace: proj-retail-prd-restore-20251109
2. Restore database snapshot to new RDS instance
3. Restore document library to new S3 bucket
4. Deploy Liferay pointing to restored resources
5. Validate data integrity
6. (Optional) Promote restored environment to production
7. Archive old environment
```

**Use Case 2: Point-In-Time Recovery (PITR)**
```
Scenario: Accidental data deletion at 2:30 PM, need to recover to 2:00 PM
Steps:
1. Identify PITR timestamp (2025-11-09T14:00:00Z)
2. Create new database from PITR
3. Restore document library from nearest backup before 2:00 PM
4. Deploy Liferay to test namespace
5. Export specific data (e.g., deleted articles)
6. Import data back to production
```

**Use Case 3: Partial Restore (DL Only)**
```
Scenario: User accidentally deleted important folder, DB is fine
Steps:
1. Identify folder path in backup
2. Restore specific objects from backup bucket
3. Copy to production DL bucket
4. No database restore needed
```

**Use Case 4: Cross-Environment Clone**
```
Scenario: Refresh UAT with production data
Steps:
1. Create backup of production
2. Restore to UAT namespace (separate DB/DL)
3. Run data sanitization scripts (anonymize PII)
4. Update Liferay configuration for UAT
5. Validate and notify team
```

**LC2 Guardrails**:
- **Owner/Member**: Can restore to **new namespace** within their project (safe clone)
- **Admin**: Required for **in-place restore** (overwrites existing data)
- **Approval Required**: Destructive operations (in-place restore to PRD)
- **Audit**: All restore operations logged (who/what/when/where/result)

---

## 6) Architecture

### 6.1 High-Level Design

```
┌─────────────────────────────────────────────────────────────────┐
│                        LC2 Cockpit                              │
│                                                                 │
│  ┌──────────────┐        ┌────────────────────────────┐        │
│  │ Cockpit UI   │───────>│ Cockpit API (Spring Boot)  │        │
│  │ (Next.js)    │ REST   │ /api/projects/{id}/backups │        │
│  └──────────────┘        └────────────┬───────────────┘        │
│                                       │                         │
│                                       │ HTTP/gRPC               │
│                                       v                         │
│                          ┌────────────────────────┐             │
│                          │ Backup Service         │             │
│                          │ (Spring Boot)          │             │
│                          │                        │             │
│                          │ ┌────────────────────┐ │             │
│                          │ │ Core Service Layer │ │             │
│                          │ │ - BackupOrchestrator│ │            │
│                          │ │ - RestoreOrchestrator│ │           │
│                          │ │ - ScheduleManager  │ │             │
│                          │ │ - ValidationService│ │             │
│                          │ └────────┬───────────┘ │             │
│                          │          │             │             │
│                          │ ┌────────v───────────┐ │             │
│                          │ │ Provider Abstraction│ │            │
│                          │ │ - BackupProvider   │ │             │
│                          │ │   interface        │ │             │
│                          │ └────────┬───────────┘ │             │
│                          └──────────┼─────────────┘             │
└───────────────────────────────────────┼───────────────────────────┘
                                        │
                ┌───────────────────────┼──────────────────────┐
                │                       │                      │
       ┌────────v─────────┐   ┌────────v────────┐   ┌────────v────────┐
       │ AWS Provider     │   │ Azure Provider  │   │  GCP Provider   │
       │                  │   │                 │   │                 │
       │ - RDS Adapter    │   │ - AzureDB       │   │ - CloudSQL      │
       │ - S3 Adapter     │   │   Adapter       │   │   Adapter       │
       │ - OpenSearch     │   │ - Blob Adapter  │   │ - GCS Adapter   │
       │   Adapter        │   │ - OpenSearch    │   │ - OpenSearch    │
       └────────┬─────────┘   │   Adapter       │   │   Adapter       │
                │              └────────┬────────┘   └────────┬────────┘
                │                       │                     │
       ┌────────v─────────┐   ┌────────v────────┐   ┌────────v────────┐
       │ AWS Services     │   │ Azure Services  │   │  GCP Services   │
       │ - RDS API        │   │ - Azure SDK     │   │ - Cloud APIs    │
       │ - S3 SDK         │   │                 │   │                 │
       └──────────────────┘   └─────────────────┘   └─────────────────┘
```

### 6.2 Component Responsibilities

#### **Cockpit API**
- **Responsibility**: Project-scoped REST API for backup operations
- **Functions**:
  - Authenticate and authorize users (RBAC)
  - Validate request parameters (project, environment, backup unit)
  - Generate backup/restore requests
  - Query backup status and history
  - Enforce namespace isolation
  - Emit audit records
- **Integration**: Calls Backup Service via HTTP or internal gRPC

#### **Backup Service (Core)**
- **Responsibility**: Business logic and orchestration
- **Functions**:
  - Coordinate multi-component backups (DB + DL + Search)
  - Manage backup schedules (cron-based or event-driven)
  - Implement retry logic and error handling
  - Validate backup integrity
  - Track operation state and progress
  - Implement backup retention policies
  - Support backup versioning and tagging

#### **Provider Abstraction Layer**
- **Responsibility**: Cloud-agnostic interface
- **Interface Definition**:
```java
public interface BackupProvider {
    // Database operations
    DatabaseBackupResult backupDatabase(DatabaseBackupRequest request);
    DatabaseRestoreResult restoreDatabase(DatabaseRestoreRequest request);
    List<DatabaseBackup> listDatabaseBackups(String instanceId);

    // Object storage operations
    ObjectStorageBackupResult backupObjectStorage(ObjectStorageBackupRequest request);
    ObjectStorageRestoreResult restoreObjectStorage(ObjectStorageRestoreRequest request);
    List<ObjectStorageBackup> listObjectStorageBackups(String bucketId);

    // Validation
    BackupValidationResult validateBackup(String backupId);

    // Cleanup
    void deleteBackup(String backupId);
}
```

#### **Cloud Provider Adapters**
Each adapter implements `BackupProvider` interface:

**AWS Adapter**:
- **Database**: AWS SDK for RDS (snapshots, PITR, automated backups)
- **Storage**: AWS SDK for S3 (copy, versioning, lifecycle)
- **Search**: AWS OpenSearch Service (snapshots)

**Azure Adapter**:
- **Database**: Azure SDK for PostgreSQL/MySQL (backups, PITR)
- **Storage**: Azure SDK for Blob Storage (copy, snapshots, lifecycle)
- **Search**: Azure OpenSearch/Elastic (snapshots)

**GCP Adapter**:
- **Database**: Cloud SQL Admin API (snapshots, backups, clones)
- **Storage**: Cloud Storage API (copy, Transfer Service)
- **Search**: Elasticsearch/Firestore (snapshots)

**Local Adapter**:
- **Database**: `pg_dump`/`mysqldump` via exec
- **Storage**: MinIO client or local filesystem copy
- **Search**: Elasticsearch snapshot API (local cluster)

---

### 6.3 Data Flow

#### **Backup Creation Flow**

```
1. User clicks "Create Backup" in Cockpit UI
   └─> POST /api/projects/retail/backups/pr
       Body: { env: "prd", unit: "all", retention: "7d" }

2. Cockpit API validates request
   - Check user has project:write permission
   - Verify environment exists
   - Validate retention policy

3. Cockpit API generates PR (GitOps)
   - Create BackupRequest YAML in Git
   - Commit to branch: backups/retail-prd-20251109-143000
   - Open PR with title: "Backup: retail-prd-20251109"

4. User/Admin approves and merges PR

5. Argo CD reconciles → creates BackupRequest CRD

6. Backup Service watches BackupRequest CRD
   - Reconcile loop triggered

7. Backup Service orchestrates backup
   a. Identify provider (AWS/Azure/GCP/Local)
   b. Call provider.backupDatabase()
      - AWS: Create RDS snapshot via AWS SDK
      - Update BackupRequest status: "DatabaseBackupInProgress"
   c. Call provider.backupObjectStorage()
      - AWS: Start S3 Batch copy job
      - Update BackupRequest status: "ObjectStorageBackupInProgress"
   d. (Optional) Call provider.backupSearch()
   e. Wait for all operations to complete (poll provider APIs)
   f. Validate backup integrity
   g. Update BackupRequest status: "Succeeded" or "Failed"
   h. Emit audit event

8. Cockpit UI polls BackupRequest status
   - Display progress (database: 100%, storage: 45%, search: queued)
   - Show completion time and backup ID
   - Enable "Restore" button when succeeded
```

#### **Restore Flow**

```
1. User clicks "Restore" in Cockpit UI
   └─> POST /api/projects/retail/restores/pr
       Body: {
         env: "prd",
         backupId: "bkp-20251109-143000",
         targetNamespace: "proj-retail-prd-restore",
         confirmDestructive: true
       }

2. Cockpit API validates request
   - Check user has project:write (or admin for in-place restore)
   - Verify backup exists and is complete
   - Validate target namespace is within project boundary

3. Cockpit API generates PR (GitOps)
   - Create RestoreRequest YAML in Git
   - Include safety checks (require approval if in-place)

4. Approval flow (if required)
   - In-place restore to PRD requires Admin approval
   - Restore to new namespace auto-approved for Owners

5. PR merged → Argo CD creates RestoreRequest CRD

6. Backup Service watches RestoreRequest CRD
   - Reconcile loop triggered

7. Backup Service orchestrates restore
   a. Identify provider and backup location
   b. Create target infrastructure (if new namespace)
      - Provision new RDS instance (or restore snapshot)
      - Create new S3 bucket (or restore to existing)
   c. Call provider.restoreDatabase()
      - AWS: Restore RDS snapshot to new instance
      - Update RestoreRequest status: "DatabaseRestoreInProgress"
   d. Call provider.restoreObjectStorage()
      - AWS: Copy objects from backup bucket to target
      - Update RestoreRequest status: "ObjectStorageRestoreInProgress"
   e. Wait for all operations to complete
   f. Validate restored data
   g. Update RestoreRequest status: "Succeeded" or "Failed"
   h. Emit audit event
   i. (Optional) Trigger Liferay reindex if search not restored

8. Cockpit displays restore result
   - Show new environment details (DB endpoint, bucket name)
   - Provide instructions for next steps
   - Link to Liferay admin console for validation
```

---

## 7) Cloud Provider Abstraction

### 7.1 Design Principles

**Principle 1: Abstraction Over Implementation**
- Core business logic must not reference provider-specific APIs
- Provider selection determined by configuration, not code paths

**Principle 2: Favor Provider Strengths**
- Use managed services over custom implementations
- RDS snapshots > manual dumps (for same-cloud restores)
- S3 versioning > custom copy logic (when available)

**Principle 3: Portability Escape Hatches**
- Always offer logical dumps (cross-cloud portability)
- Document migration paths (GCP → AWS, etc.)

**Principle 4: Local Development Parity**
- Local adapter mimics cloud behavior with lightweight tools
- Same workflows, different backends

---

### 7.2 Provider Capabilities Matrix

| Capability                | AWS                     | Azure                   | GCP                     | Local               |
|---------------------------|-------------------------|-------------------------|-------------------------|---------------------|
| **Database Snapshot**     | RDS Snapshots           | Azure DB Backups        | Cloud SQL Snapshots     | ❌ (dump only)      |
| **Database PITR**         | RDS PITR (5m window)    | Azure DB PITR           | Cloud SQL PITR          | ❌                  |
| **Database Dump**         | Custom (pg_dump/mysqldump via Lambda) | Custom (Azure Functions) | Custom (Cloud Run job) | ✅ (native tools)   |
| **Object Versioning**     | S3 Versioning           | Blob Versioning         | GCS Object Versioning   | MinIO versioning    |
| **Object Copy**           | S3 Batch Operations     | AzCopy / Blob Batch     | Storage Transfer Service| rsync / mc mirror   |
| **Object Lifecycle**      | S3 Lifecycle Policies   | Blob Lifecycle Mgmt     | GCS Lifecycle Rules     | Manual / cron       |
| **Search Snapshots**      | OpenSearch Snapshots→S3 | Elasticsearch Snaps→Blob| Elasticsearch Snaps→GCS | Elasticsearch local |
| **IAM / Auth**            | IAM Roles (IRSA)        | Managed Identity        | Workload Identity       | Static credentials  |
| **Encryption**            | KMS                     | Key Vault               | Cloud KMS               | ❌ (or local certs) |
| **Monitoring**            | CloudWatch              | Azure Monitor           | Cloud Monitoring        | Prometheus          |

---

### 7.3 Provider Adapter Implementation

#### **Interface Definition**

```java
public interface BackupProvider {

    /**
     * Provider identification
     */
    CloudProvider getProviderType(); // AWS, AZURE, GCP, LOCAL

    /**
     * Database backup operations
     */
    DatabaseBackupResult createDatabaseSnapshot(DatabaseSnapshotRequest request);
    DatabaseBackupResult createDatabaseDump(DatabaseDumpRequest request);
    DatabaseRestoreResult restoreDatabaseFromSnapshot(DatabaseSnapshotRestoreRequest request);
    DatabaseRestoreResult restoreDatabaseFromDump(DatabaseDumpRestoreRequest request);
    List<DatabaseBackup> listDatabaseBackups(DatabaseBackupFilter filter);
    BackupMetadata getDatabaseBackupMetadata(String backupId);
    void deleteDatabaseBackup(String backupId);

    /**
     * Object storage backup operations
     */
    ObjectStorageBackupResult createObjectStorageBackup(ObjectStorageBackupRequest request);
    ObjectStorageRestoreResult restoreObjectStorage(ObjectStorageRestoreRequest request);
    List<ObjectStorageBackup> listObjectStorageBackups(ObjectStorageBackupFilter filter);
    void deleteObjectStorageBackup(String backupId);

    /**
     * Search index operations (optional)
     */
    SearchBackupResult createSearchSnapshot(SearchSnapshotRequest request);
    SearchRestoreResult restoreSearchSnapshot(SearchSnapshotRestoreRequest request);

    /**
     * Backup validation
     */
    ValidationResult validateBackup(String backupId);

    /**
     * Operation status tracking
     */
    OperationStatus getOperationStatus(String operationId);
}
```

#### **Configuration-Based Provider Selection**

```yaml
# application.yaml
liferay:
  backup:
    provider: ${CLOUD_PROVIDER:local}  # aws | azure | gcp | local

    aws:
      region: ${AWS_REGION:us-east-1}
      rds:
        snapshotRetentionDays: 7
      s3:
        backupBucketPrefix: "liferay-backups"

    azure:
      location: ${AZURE_LOCATION:eastus}
      database:
        backupRetentionDays: 7
      blob:
        backupContainerPrefix: "liferay-backups"

    gcp:
      project: ${GCP_PROJECT_ID}
      region: ${GCP_REGION:us-central1}
      cloudsql:
        backupRetentionDays: 7
      gcs:
        backupBucketPrefix: "liferay-backups"

    local:
      backupPath: "/var/backups/liferay"
      postgres:
        host: ${POSTGRES_HOST:localhost}
        port: ${POSTGRES_PORT:5432}
      minio:
        endpoint: ${MINIO_ENDPOINT:http://minio:9000}
```

#### **Spring Boot Provider Factory**

```java
@Configuration
public class BackupProviderConfig {

    @Value("${liferay.backup.provider}")
    private String providerType;

    @Bean
    public BackupProvider backupProvider() {
        return switch (providerType.toLowerCase()) {
            case "aws" -> new AwsBackupProvider(awsConfig());
            case "azure" -> new AzureBackupProvider(azureConfig());
            case "gcp" -> new GcpBackupProvider(gcpConfig());
            case "local" -> new LocalBackupProvider(localConfig());
            default -> throw new IllegalArgumentException("Unknown provider: " + providerType);
        };
    }

    // ... provider-specific config beans
}
```

---

### 7.4 AWS Provider Implementation (Example)

```java
@Service
public class AwsBackupProvider implements BackupProvider {

    private final RdsClient rdsClient;
    private final S3Client s3Client;
    private final String region;

    public AwsBackupProvider(AwsBackupConfig config) {
        this.region = config.getRegion();
        this.rdsClient = RdsClient.builder()
            .region(Region.of(region))
            .credentialsProvider(/* IRSA or default */)
            .build();
        this.s3Client = S3Client.builder()
            .region(Region.of(region))
            .build();
    }

    @Override
    public DatabaseBackupResult createDatabaseSnapshot(DatabaseSnapshotRequest request) {
        String snapshotId = generateSnapshotId(request);

        CreateDbSnapshotRequest snapshotRequest = CreateDbSnapshotRequest.builder()
            .dbInstanceIdentifier(request.getInstanceId())
            .dbSnapshotIdentifier(snapshotId)
            .tags(buildTags(request))
            .build();

        CreateDbSnapshotResponse response = rdsClient.createDBSnapshot(snapshotRequest);

        return DatabaseBackupResult.builder()
            .backupId(snapshotId)
            .status(BackupStatus.IN_PROGRESS)
            .startTime(Instant.now())
            .providerOperationId(snapshotId)
            .build();
    }

    @Override
    public ObjectStorageBackupResult createObjectStorageBackup(ObjectStorageBackupRequest request) {
        String sourceBucket = request.getSourceBucket();
        String destBucket = request.getDestinationBucket();
        String destPrefix = request.getDestinationPrefix();

        // Strategy: S3 Batch Copy (for large buckets) or sync-based copy
        if (isLargeBucket(sourceBucket)) {
            return createBatchCopyJob(sourceBucket, destBucket, destPrefix);
        } else {
            return createSyncCopy(sourceBucket, destBucket, destPrefix);
        }
    }

    private ObjectStorageBackupResult createSyncCopy(String source, String dest, String prefix) {
        ListObjectsV2Request listRequest = ListObjectsV2Request.builder()
            .bucket(source)
            .build();

        ListObjectsV2Response listResponse = s3Client.listObjectsV2(listRequest);

        List<CompletableFuture<CopyObjectResponse>> copyFutures = new ArrayList<>();

        for (S3Object object : listResponse.contents()) {
            String sourceKey = object.key();
            String destKey = prefix + "/" + sourceKey;

            CopyObjectRequest copyRequest = CopyObjectRequest.builder()
                .sourceBucket(source)
                .sourceKey(sourceKey)
                .destinationBucket(dest)
                .destinationKey(destKey)
                .build();

            CompletableFuture<CopyObjectResponse> future =
                CompletableFuture.supplyAsync(() -> s3Client.copyObject(copyRequest));

            copyFutures.add(future);
        }

        // Wait for all copies to complete (or implement progress tracking)
        CompletableFuture.allOf(copyFutures.toArray(new CompletableFuture[0])).join();

        return ObjectStorageBackupResult.builder()
            .backupId(generateBackupId())
            .status(BackupStatus.SUCCEEDED)
            .objectCount(listResponse.contents().size())
            .build();
    }

    @Override
    public OperationStatus getOperationStatus(String operationId) {
        // Poll RDS snapshot status
        DescribeDbSnapshotsRequest request = DescribeDbSnapshotsRequest.builder()
            .dbSnapshotIdentifier(operationId)
            .build();

        DescribeDbSnapshotsResponse response = rdsClient.describeDBSnapshots(request);

        if (response.dbSnapshots().isEmpty()) {
            return OperationStatus.NOT_FOUND;
        }

        String status = response.dbSnapshots().get(0).status();

        return switch (status) {
            case "available" -> OperationStatus.SUCCEEDED;
            case "creating" -> OperationStatus.IN_PROGRESS;
            case "failed" -> OperationStatus.FAILED;
            default -> OperationStatus.UNKNOWN;
        };
    }
}
```

---

## 8) Data Model & API

### 8.1 Domain Model

#### **BackupRequest** (CRD or Database Entity)

```yaml
apiVersion: backup.liferay.cloud/v1alpha1
kind: BackupRequest
metadata:
  name: retail-prd-backup-20251109-143000
  namespace: proj-retail-prd
  labels:
    project: retail
    environment: prd
    backup.liferay.cloud/project-id: retail
    backup.liferay.cloud/environment: prd
spec:
  projectId: retail
  environment: prd
  units:
    - database
    - documentLibrary
    - search  # optional

  database:
    type: snapshot  # or "dump"
    instanceId: retail-prd-db
    retention: 7d

  documentLibrary:
    sourceBucket: liferay-retail-prd-dl
    destinationBucket: liferay-retail-prd-dl-backups
    destinationPrefix: "backups/20251109-143000"
    excludeGeneratedFiles: true

  search:
    enabled: false  # MVP: skip search backups

  schedule:
    cron: "0 2 * * *"  # 2 AM daily
    timezone: "America/New_York"

  retention:
    keepDaily: 7
    keepWeekly: 4
    keepMonthly: 12

status:
  state: InProgress  # Pending | InProgress | Succeeded | Failed
  startTime: "2025-11-09T14:30:00Z"
  completionTime: null

  database:
    status: Succeeded
    backupId: "snapshot-retail-prd-db-20251109-143000"
    providerOperationId: "rds:us-east-1:snapshot-123456"
    size: "45.2 GB"
    completionTime: "2025-11-09T14:35:00Z"

  documentLibrary:
    status: InProgress
    backupId: "dl-retail-prd-20251109-143000"
    objectCount: 12453
    totalSize: "125.8 GB"
    copiedSize: "67.3 GB"
    progress: 53

  search:
    status: Skipped

  conditions:
    - type: DatabaseBackupReady
      status: "True"
      reason: SnapshotSucceeded
      message: "Database snapshot completed successfully"
      lastTransitionTime: "2025-11-09T14:35:00Z"

    - type: DocumentLibraryBackupReady
      status: "False"
      reason: CopyInProgress
      message: "Copying 12453 objects (53% complete)"
      lastTransitionTime: "2025-11-09T14:40:00Z"
```

#### **RestoreRequest**

```yaml
apiVersion: backup.liferay.cloud/v1alpha1
kind: RestoreRequest
metadata:
  name: retail-prd-restore-20251109-150000
  namespace: proj-retail-prd
spec:
  projectId: retail
  sourceEnvironment: prd
  targetEnvironment: prd-restore  # or same env for in-place
  backupId: retail-prd-backup-20251109-143000

  units:
    - database
    - documentLibrary

  database:
    targetInstanceId: retail-prd-restore-db  # new or existing
    pointInTime: "2025-11-09T14:30:00Z"  # optional PITR

  documentLibrary:
    targetBucket: liferay-retail-prd-restore-dl

  approvals:
    required: true  # if in-place restore to prd
    approvedBy: admin@example.com
    approvedAt: "2025-11-09T15:00:00Z"

status:
  state: InProgress
  startTime: "2025-11-09T15:01:00Z"

  database:
    status: InProgress
    providerOperationId: "rds:us-east-1:restore-789012"
    progress: 34

  documentLibrary:
    status: Pending

  conditions:
    - type: DatabaseRestoreReady
      status: "False"
      reason: RestoreInProgress
      message: "Restoring from snapshot (34% complete)"
```

---

### 8.2 REST API Specification

#### **Base URL**
```
https://{cloud}-{region}.{cockpit-domain}/api
```

#### **Authentication**
```
Authorization: Bearer {session-token}
```

#### **Endpoints**

##### **1. Create Backup (Generate PR)**

```http
POST /projects/{projectId}/backups/pr
Content-Type: application/json

Request:
{
  "env": "prd",
  "units": ["database", "documentLibrary"],
  "database": {
    "type": "snapshot",
    "retention": "7d"
  },
  "documentLibrary": {
    "excludeGeneratedFiles": true
  },
  "notes": "Pre-deployment backup"
}

Response: 201 Created
{
  "prUrl": "https://github.com/org/repo/pull/123",
  "prNumber": 123,
  "backupId": "retail-prd-backup-20251109-143000",
  "estimatedDuration": "30m"
}
```

##### **2. List Backups**

```http
GET /projects/{projectId}/backups?env=prd&unit=database&page_size=50

Response: 200 OK
{
  "items": [
    {
      "backupId": "retail-prd-backup-20251109-143000",
      "projectId": "retail",
      "environment": "prd",
      "units": ["database", "documentLibrary"],
      "status": "Succeeded",
      "startTime": "2025-11-09T14:30:00Z",
      "completionTime": "2025-11-09T14:45:00Z",
      "size": "171 GB",
      "initiator": "alice@example.com",
      "retention": "7d"
    },
    ...
  ],
  "nextPageToken": "eyJ..."
}
```

##### **3. Get Backup Details**

```http
GET /projects/{projectId}/backups/{backupId}

Response: 200 OK
{
  "backupId": "retail-prd-backup-20251109-143000",
  "status": "Succeeded",
  "units": {
    "database": {
      "type": "snapshot",
      "snapshotId": "snapshot-retail-prd-db-20251109-143000",
      "size": "45.2 GB",
      "instanceId": "retail-prd-db",
      "completionTime": "2025-11-09T14:35:00Z"
    },
    "documentLibrary": {
      "backupId": "dl-retail-prd-20251109-143000",
      "sourceBucket": "liferay-retail-prd-dl",
      "destinationBucket": "liferay-retail-prd-dl-backups",
      "objectCount": 12453,
      "size": "125.8 GB",
      "completionTime": "2025-11-09T14:45:00Z"
    }
  },
  "audit": {
    "createdBy": "alice@example.com",
    "createdAt": "2025-11-09T14:30:00Z",
    "correlationId": "4f1a6c3b-..."
  }
}
```

##### **4. Create Restore (Generate PR)**

```http
POST /projects/{projectId}/restores/pr
Content-Type: application/json

Request:
{
  "env": "prd",
  "backupId": "retail-prd-backup-20251109-143000",
  "targetNamespace": "proj-retail-prd-restore",
  "units": ["database", "documentLibrary"],
  "confirmDestructive": false
}

Response: 201 Created
{
  "prUrl": "https://github.com/org/repo/pull/124",
  "prNumber": 124,
  "restoreId": "retail-prd-restore-20251109-150000",
  "requiresApproval": false,
  "estimatedDuration": "45m"
}
```

##### **5. Get Restore Status**

```http
GET /projects/{projectId}/restores/{restoreId}

Response: 200 OK
{
  "restoreId": "retail-prd-restore-20251109-150000",
  "status": "InProgress",
  "progress": 34,
  "startTime": "2025-11-09T15:01:00Z",
  "units": {
    "database": {
      "status": "InProgress",
      "progress": 34,
      "targetInstanceId": "retail-prd-restore-db"
    },
    "documentLibrary": {
      "status": "Pending"
    }
  }
}
```

##### **6. Delete Backup**

```http
DELETE /projects/{projectId}/backups/{backupId}

Response: 204 No Content
```

---

## 9) Technology Stack

### 9.1 Core Technologies

**Language & Framework**:
- **Java 21** (LTS)
- **Spring Boot 3.3+**
- **Spring Cloud** (for distributed configuration if needed)

**Why Java?**
- ✅ Strong typing and IDE support (vs. Node.js dynamic typing)
- ✅ Better performance for long-running operations
- ✅ Excellent cloud SDK support (AWS, Azure, GCP all have mature Java SDKs)
- ✅ Enterprise-grade libraries for scheduling, retry, observability
- ✅ Aligns with Cockpit API (also Java/Spring Boot)
- ✅ Strong Kubernetes client libraries
- ✅ Better memory management for large backup operations

### 9.2 Dependencies

#### **Cloud SDKs**

```xml
<!-- AWS SDK -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>rds</artifactId>
    <version>2.20.0</version>
</dependency>
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.20.0</version>
</dependency>
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>opensearch</artifactId>
    <version>2.20.0</version>
</dependency>

<!-- Azure SDK -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-resourcemanager-postgresql</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-storage-blob</artifactId>
    <version>12.20.0</version>
</dependency>

<!-- GCP SDK -->
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>google-cloud-storage</artifactId>
    <version>2.30.0</version>
</dependency>
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>google-cloud-sql</artifactId>
    <version>2.10.0</version>
</dependency>
```

#### **Kubernetes Integration**

```xml
<!-- Kubernetes Java Client -->
<dependency>
    <groupId>io.kubernetes</groupId>
    <artifactId>client-java</artifactId>
    <version>20.0.0</version>
</dependency>

<!-- Optional: Fabric8 Kubernetes Client (more ergonomic) -->
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>kubernetes-client</artifactId>
    <version>6.9.0</version>
</dependency>
```

#### **Scheduling & Async**

```xml
<!-- Spring Scheduling (built-in) -->
<!-- Already included in Spring Boot -->

<!-- Quartz (for advanced scheduling) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

#### **Observability**

```xml
<!-- Micrometer (metrics) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- OpenTelemetry (traces) - optional -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.32.0</version>
</dependency>
```

#### **Database (for state tracking)**

```xml
<!-- PostgreSQL (Cockpit's management DB) -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>

<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Flyway (schema migrations) -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

---

### 9.3 Project Structure

```
backup-service/
├── src/main/java/com/liferay/cloud/backup/
│   ├── BackupServiceApplication.java
│   │
│   ├── api/                           # REST API controllers
│   │   ├── BackupController.java
│   │   ├── RestoreController.java
│   │   └── dto/
│   │       ├── BackupRequest.java
│   │       ├── BackupResponse.java
│   │       ├── RestoreRequest.java
│   │       └── RestoreResponse.java
│   │
│   ├── service/                       # Business logic
│   │   ├── BackupOrchestrator.java
│   │   ├── RestoreOrchestrator.java
│   │   ├── ScheduleManager.java
│   │   ├── ValidationService.java
│   │   └── RetentionPolicyService.java
│   │
│   ├── provider/                      # Cloud abstraction
│   │   ├── BackupProvider.java        # Interface
│   │   ├── aws/
│   │   │   ├── AwsBackupProvider.java
│   │   │   ├── AwsRdsAdapter.java
│   │   │   ├── AwsS3Adapter.java
│   │   │   └── AwsOpenSearchAdapter.java
│   │   ├── azure/
│   │   │   ├── AzureBackupProvider.java
│   │   │   ├── AzureDatabaseAdapter.java
│   │   │   ├── AzureBlobAdapter.java
│   │   │   └── AzureOpenSearchAdapter.java
│   │   ├── gcp/
│   │   │   ├── GcpBackupProvider.java
│   │   │   ├── GcpCloudSqlAdapter.java
│   │   │   ├── GcpStorageAdapter.java
│   │   │   └── GcpElasticsearchAdapter.java
│   │   └── local/
│   │       ├── LocalBackupProvider.java
│   │       ├── LocalPostgresAdapter.java
│   │       ├── LocalMinioAdapter.java
│   │       └── LocalElasticsearchAdapter.java
│   │
│   ├── k8s/                           # Kubernetes integration
│   │   ├── BackupRequestReconciler.java
│   │   ├── RestoreRequestReconciler.java
│   │   └── crd/
│   │       ├── BackupRequest.java
│   │       └── RestoreRequest.java
│   │
│   ├── repository/                    # Database entities
│   │   ├── BackupRepository.java
│   │   ├── RestoreRepository.java
│   │   └── entity/
│   │       ├── BackupEntity.java
│   │       └── RestoreEntity.java
│   │
│   ├── config/                        # Configuration
│   │   ├── BackupProviderConfig.java
│   │   ├── SecurityConfig.java
│   │   └── ObservabilityConfig.java
│   │
│   └── util/                          # Utilities
│       ├── BackupIdGenerator.java
│       ├── SizeFormatter.java
│       └── RetryHelper.java
│
├── src/main/resources/
│   ├── application.yaml
│   ├── application-aws.yaml
│   ├── application-azure.yaml
│   ├── application-gcp.yaml
│   ├── application-local.yaml
│   └── db/migration/                  # Flyway migrations
│       └── V1__initial_schema.sql
│
├── src/test/java/
│   ├── integration/
│   └── unit/
│
├── pom.xml
└── Dockerfile
```

---

## 10) Implementation Phases

### Phase 1: Foundation (MVP)
**Goal**: Centralized backup service for AWS with database + document library

**Scope**:
- ✅ Java/Spring Boot service with REST API
- ✅ AWS provider implementation (RDS snapshots, S3 copy)
- ✅ BackupRequest/RestoreRequest CRDs
- ✅ Kubernetes reconciliation loop
- ✅ Basic scheduling (cron-based)
- ✅ Audit logging
- ✅ Integration with Cockpit API

**Deliverables**:
- Backup service Docker image
- Helm chart for deployment
- API documentation (OpenAPI spec)
- Integration tests (AWS LocalStack)

**Timeline**: 6-8 weeks

---

### Phase 2: Multi-Cloud (Azure + GCP)
**Goal**: Support all LC2 target clouds

**Scope**:
- ✅ Azure provider implementation
- ✅ GCP provider implementation
- ✅ Local provider implementation (for development)
- ✅ Provider capability matrix
- ✅ Cross-cloud restore documentation

**Deliverables**:
- Azure and GCP adapters
- Local development guide
- Migration playbooks (GCP→AWS, etc.)

**Timeline**: 4-6 weeks

---

### Phase 3: Advanced Features
**Goal**: Production-grade capabilities

**Scope**:
- ✅ Search index backups (OpenSearch snapshots)
- ✅ Incremental backups (document library)
- ✅ Backup validation (checksums, test restores)
- ✅ Retention policies (keep daily/weekly/monthly)
- ✅ Progress tracking (real-time updates)
- ✅ Parallel operations (multi-threaded copies)

**Deliverables**:
- Enhanced backup strategies
- Validation framework
- Performance benchmarks

**Timeline**: 4-6 weeks

---

### Phase 4: Operational Excellence
**Goal**: Enterprise-ready monitoring and DR

**Scope**:
- ✅ Prometheus metrics (backup duration, size, success rate)
- ✅ Alerting (backup failures, quota exceeded, slow operations)
- ✅ DR drills automation (scheduled test restores)
- ✅ Backup catalog UI (in Cockpit)
- ✅ Self-service restore (safe clones for Owners)

**Deliverables**:
- Grafana dashboards
- Alerting rules
- DR playbooks
- Enhanced Cockpit UI

**Timeline**: 4-6 weeks

---

## 11) Acceptance Criteria

### 11.1 Functional Requirements

**Backup Creation**:
- ✅ User can create on-demand backup via Cockpit UI
- ✅ Scheduled backups run at configured times
- ✅ Backup includes database (snapshot or dump) and document library
- ✅ Backup completes within SLA (e.g., 1 hour for typical workload)
- ✅ Backup is validated (integrity checks)
- ✅ Backup is tagged with metadata (project, env, timestamp, initiator)

**Backup Listing**:
- ✅ User can list all backups for their project/environment
- ✅ List supports filtering (date range, status, unit)
- ✅ List supports pagination
- ✅ List displays size, duration, and status

**Backup Restore**:
- ✅ User can restore to new namespace (safe clone)
- ✅ Admin can restore in-place (with approval)
- ✅ Restore creates new infrastructure (DB instance, bucket)
- ✅ Restore validates data integrity post-restore
- ✅ Restore is audited (who/what/when/where/result)

**Multi-Cloud Support**:
- ✅ Same workflows work on AWS, Azure, GCP, and local
- ✅ Provider-specific optimizations (RDS snapshots, S3 versioning)
- ✅ Migration path documented (cross-cloud restore)

**Isolation & Security**:
- ✅ Users can only access backups for their projects
- ✅ RBAC enforced (Owner/Member/Admin permissions)
- ✅ Secrets never exposed (database credentials, API keys)
- ✅ Audit trail for all operations

---

### 11.2 Non-Functional Requirements

**Performance**:
- ✅ Database snapshot: < 5 minutes for 50 GB database
- ✅ Document library copy: > 100 MB/s throughput
- ✅ API response time: P95 < 500ms for list operations
- ✅ Concurrent backups: Support 10+ projects simultaneously

**Reliability**:
- ✅ Backup success rate: > 99%
- ✅ Automatic retries on transient failures
- ✅ Idempotent operations (safe to retry)
- ✅ Graceful degradation (partial backups marked as failed)

**Scalability**:
- ✅ Support 50+ projects per cluster
- ✅ Support 100+ backups per project
- ✅ Support 10 TB+ document libraries

**Observability**:
- ✅ Metrics exported to Prometheus
- ✅ Logs structured (JSON) and queryable
- ✅ Traces for distributed operations (optional)
- ✅ Alerts for failures, quota issues, slow operations

---

## 12) Non-Goals

**Out of Scope for MVP**:
- ❌ Cross-cluster backups (backup in cluster A, restore in cluster B)
- ❌ Continuous replication (real-time sync)
- ❌ Application-level backups (e.g., Liferay export/import)
- ❌ Custom backup scripts (shell access)
- ❌ Backup encryption (rely on provider-managed encryption)
- ❌ Multi-region backups (single region per backup)
- ❌ Backup deduplication (rely on provider features)
- ❌ Backup compression (handled by provider or tools)
- ❌ Fine-grained restore (restore individual tables/files)

**Future Enhancements**:
- 🔮 Backup cost estimation and FinOps dashboards
- 🔮 AI-driven backup optimization (schedule during low-usage periods)
- 🔮 Compliance reporting (SOC2, GDPR data retention)
- 🔮 Backup marketplace (share backups between projects/orgs)

---

## Conclusion

This backup service design addresses the core requirements:

1. **Centralized Multi-Tenant Service**: Replaces per-customer deployments with a shared platform service
2. **Cloud-Agnostic**: Supports AWS, Azure, GCP, and local development
3. **Liferay-Aware**: Understands three-component backup, consistency requirements, and optimization opportunities
4. **Production-Ready**: Audit, RBAC, validation, retention policies, and observability
5. **GitOps-Integrated**: Backup/restore operations flow through PRs and Argo CD

**Next Steps**:
1. Review and approve this requirements document
2. Create detailed API specification (OpenAPI)
3. Design database schema (Flyway migrations)
4. Implement Phase 1 (AWS MVP)
5. Deploy to staging for validation
6. Iterate based on feedback

---

**Document Version**: 1.0
**Last Updated**: 2025-11-09
**Authors**: LC2 Backup Team
**Reviewers**: [TBD]
