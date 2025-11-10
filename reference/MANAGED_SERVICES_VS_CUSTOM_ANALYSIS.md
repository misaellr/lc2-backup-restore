# Managed Services vs Custom Operator Analysis for LC2 Backup

## Executive Summary

**Question:** Can we use managed backup services (AWS Backup, GCP Backup and DR, Azure Backup) instead of building a custom Kubernetes operator?

**Answer:** **Yes, with enhanced hybrid approach.** Liferay has specific backup requirements that pure managed services cannot fully address, but we can leverage managed service **APIs** (not just snapshots) to handle most of the heavy lifting.

**Recommendation:** Build a lightweight Java operator that orchestrates managed service APIs (Cloud SQL Export, GCS Transfer Service, AWS RDS Export) for execution, while implementing Liferay-specific coordination logic.

**Key Insight:** Current Liferay PaaS team already identified that **streaming data through backup service causes OOM errors and failures**. Their proposed solution aligns with our enhanced hybrid approach.

---

## Validation from Current Liferay PaaS Team

### Current Problems (from "Rethinking Backups" - Iliyan Peychev)

**Customer Issues:**
> ⚠️ "Long time to create and restore backups"
>
> ⚠️ "Failure during create or restore, due to:
> - unstable data transfer during backup operations
> - **OutOfMemory errors resulting in the backup service being restarted**"

**Current Implementation:**
```
SQL Backup:
- Backup service uses local SQL client to stream export to itself
- Pipes stream (compressed) to backup store

DL Store Backup:
- NFS volume mounted on backup service
- Files collected and streamed (compressed) to backup store
```

**The Problem:** Data streams through the backup service pod, causing:
- Memory pressure (OOM kills)
- CPU bottlenecks during compression
- Unstable data transfers
- Long backup times

### Liferay PaaS Team's Proposed Solution

**From Iliyan's "Rethinking Backups" document:**

> **"MySQL dumps - invoke CloudSQL API to export/import directly to a backup storage location"**
>
> "This approach **eliminates having to stream data through the backup service**. Compute (including compression) would be handled completely through GCP. Compression is enabled easily by specifying a bucket path with a .gz extension."

**For Document Library:**
> **"Use GCS Transfer Service to mitigate any in house processing of the DL."**

**This validates the enhanced hybrid approach: Use managed service APIs to eliminate streaming!**

---

## Liferay-Specific Backup Requirements (from actual code analysis)

### 1. Three-Component Coordinated Backup

**Current Implementation:**
```typescript
// service-backup/src/manager/manager-v1.ts
async createBackup(createContext: CreateContext): Promise<void> {
  await promiseAllSettledWithErrorLogging(
    this.createDatabaseBackup(createContext),      // Snapshot + Dump
    this.createDocumentLibraryBackup(createContext), // 100TB GCS copy
  );
  await backupCreateStatusHelper.setSuccess(createContext.backupMetadata);
}
```

**Components:**
1. **Database Snapshot** - Cloud SQL snapshot (fast restore)
2. **Database Dump** - pg_dump export (portability, migration)
3. **Document Library** - Full GCS bucket copy (100TB+)

**Managed Service Limitation:**
- AWS Backup can snapshot RDS ✅
- AWS Backup CANNOT coordinate with S3 document library backup ❌
- AWS Backup CANNOT create both snapshot + dump ❌
- GCP Backup and DR has similar limitations

**Why Both Snapshot + Dump?**
- Snapshot: Fast restore (minutes for 10TB database)
- Dump: Portability (migrate between environments, cross-cloud, version upgrades)
- This dual-approach is Liferay-specific, confirmed by current implementation

### 2. excludeGeneratedDoclibFiles Logic

**Current CRD Spec:**
```typescript
// service-backup/src/types/k8s-custom-resources/liferay-backup.d.ts
type LiferayDocumentLibraryBackupSpec = {
  excludeGeneratedFiles: boolean;  // Liferay-specific!
  destination: BackupDestination;
  source: DocumentLibraryBackupSource;
};
```

**Liferay-Specific Behavior:**
- Liferay generates thumbnails, previews, converted documents automatically
- These can be regenerated after restore
- Excluding them can save **30-50% of backup size (30-50TB!)**
- This requires file filtering logic during backup

**Managed Service Limitation:**
- AWS Backup cannot filter S3 objects based on custom logic ❌
- GCP Backup and DR cannot filter GCS objects ❌
- **GCS Transfer Service CAN filter using ObjectConditions** ✅

### 3. Metadata/Tracker Files Linking Components

**Current Metadata Structure:**
```typescript
// service-backup/src/storage/metadata/backup-metadata-gcs-v3-impl.ts
export class BackupMetadataGcsV3Impl extends BackupMetadataAbstract {
  readonly backupId: string;
  readonly creationType: 'upload' | 'auto' | 'manual';
  readonly creator: TruncatedUser;
  readonly dbDump: FileLocationWithSize;    // Links to dump file
  readonly doclib: DirLocationWithSize;     // Links to doclib backup
  readonly structure: 'gcsV3';
  readonly status: BackupStatus;
  readonly timestamp: string;
  readonly uuid: string;
}
```

**Purpose:**
1. **Component Linking** - Ties database + document library together
2. **Restore Catalog** - UI shows available backups from metadata files
3. **Selective Restore** - "Restore just database" or "Restore everything"
4. **Size Tracking** - Display backup sizes in UI
5. **Creator Audit** - Who created backup, when, how (manual/auto/upload)

**Managed Service Limitation:**
- AWS Backup has its own metadata, but it's AWS-specific ❌
- GCP Backup and DR has its own metadata, but it's GCP-specific ❌
- Neither can link external components (document library) ❌
- Cockpit UI needs single metadata format ✅

**Solution:** Query managed service APIs after completion and generate unified metadata.

### 4. Database databasesToIgnore Configuration

**Current Implementation:**
```typescript
// service-backup/src/manager/manager-v2.ts
spec: {
  databaseDump: {
    destination: { type: 'gcs', gcs: { bucket: dbDump.bucketName } },
    databaseInstanceId: databaseInstance.databaseId,
    databaseName: config.DatabaseName,
    databasesToIgnore: Array.from(config.DumpDatabasesToIgnore),  // Liferay-specific!
  }
}
```

**Managed Service Support:**
- RDS snapshots capture entire instance ✅
- **Cloud SQL Export API supports database filtering** ✅
- **RDS Export to S3 supports database filtering** ✅

### 5. Multi-Version Restore Support

**Current Implementation:**
```typescript
// service-backup/src/storage/metadata/backup-metadata-abstract.ts
readonly structure: 'gcsV3';  // or 'gcsV2', 'sfs'
```

**Liferay-Specific Requirement:**
- Need to restore backups created with old backup service versions
- This requires custom restore logic ✅

---

## Enhanced Hybrid Approach: Managed Service APIs

### Key Difference from Earlier Proposal

**OLD Approach (Argo Workflows doing execution):**
```yaml
# Argo Workflow executes pg_dump
- name: database-dump
  container:
    image: postgres:15
    command: [pg_dump]  # Data streams through pod!
```

**NEW Approach (Managed Service APIs):**
```java
// Java operator calls Cloud SQL Export API
var exportRequest = InstancesExportRequest.newBuilder()
    .setExportContext(ExportContext.newBuilder()
        .setUri("gs://bucket/dump.sql.gz")  // GCP writes directly!
        .setDatabases(databases)
        .build())
    .build();
cloudSqlClient.export(instanceId, exportRequest);  // No streaming through pod!
```

**Benefits:**
- ✅ **No streaming through backup service** - Eliminates OOM errors
- ✅ **GCP/AWS handles compression, parallelization** - Optimized performance
- ✅ **Direct storage writes** - Faster, more reliable
- ✅ **Reduced compute/memory** - Backup service just orchestrates

---

## Managed Backup Services Comparison (2025)

### AWS RDS Export to S3

**Capabilities:**
- ✅ RDS automated snapshots
- ✅ **Export snapshot to S3 as Parquet/SQL format**
- ✅ Cross-region/cross-account copy (single action as of Oct 2025)
- ✅ Database filtering (export specific databases)
- ✅ Retention policies (1 day to forever)

**Limitations for Liferay:**
- ❌ Cannot coordinate RDS + S3 document library as single operation
- ⚠️ S3 Transfer Service exists but less mature than GCS Transfer Service
- ❌ Cannot generate custom metadata/tracker files
- ❌ No local/k3d support

### GCP Cloud SQL Export API + GCS Transfer Service

**Capabilities:**
- ✅ Cloud SQL automated snapshots
- ✅ **Cloud SQL Export API (direct to GCS, no streaming)**
- ✅ **GCS Transfer Service with ObjectConditions filtering**
- ✅ Database filtering (export specific databases)
- ✅ Air-gapped immutable vaults (ransomware protection)
- ✅ Retention: 1 day to 99 years

**Limitations for Liferay:**
- ❌ Cannot coordinate Cloud SQL + GCS as single operation
- ❌ Cannot generate custom metadata/tracker files
- ❌ No local/k3d support

### Azure Database Export + Blob Transfer

**Capabilities:**
- ✅ PostgreSQL Flexible Server vaulted backups
- ✅ Export to Blob Storage
- ✅ 10-year retention
- ✅ Isolated storage (ransomware protection)

**Limitations for Liferay:**
- ❌ Cannot coordinate Azure DB + Blob Storage as single operation
- ❌ Cannot generate custom metadata/tracker files
- ❌ No local/k3d support

---

## Final Architecture Recommendation

### RECOMMENDED: Enhanced Hybrid Approach with Managed Service APIs

**Architecture:**
```
BackupRequest CRD → Java Operator (Quarkus)
                       ↓
        ┌──────────────┴────────────────────────┐
        ↓                                        ↓
  Managed Service APIs                  Custom Coordination
        ↓                                        ↓
  - Cloud SQL Snapshot                  - Metadata generation
  - Cloud SQL Export API                - Component linking
    (writes directly to GCS!)           - CRD status updates
  - GCS Transfer Service                - PostgreSQL archival
    (with filtering!)                    - Cockpit UI integration

  For local/k3d only:
  - Argo Workflows (pg_dump + rclone)
```

**What Managed Services Handle:**
- ✅ Database snapshots (instant)
- ✅ Database dumps via Export APIs (GCP writes to GCS, AWS writes to S3)
- ✅ Document library transfers via Transfer Services (with filtering)
- ✅ Compression, parallelization, optimization
- ✅ Cross-region replication
- ✅ Retention policies

**What Custom Operator Handles:**
- ✅ Orchestrating managed service API calls
- ✅ Polling operation status
- ✅ Metadata/tracker file generation (query APIs for details)
- ✅ Multi-component coordination (database + doclib)
- ✅ Kubernetes-native interface (CRDs, GitOps)
- ✅ Unified catalog for Cockpit UI
- ✅ Local/k3d support (Argo Workflows for pg_dump + rclone)

---

## Implementation Pattern (Enhanced Hybrid)

### GCP Implementation

```java
@ApplicationScoped
public class GCPBackupProvider implements BackupProvider {

    @Inject SQLAdminService cloudSqlClient;
    @Inject StorageTransferServiceClient storageTransferClient;
    @Inject Storage storageClient;

    @Override
    public BackupResult createBackup(BackupRequest request) {
        var backupId = request.getMetadata().getName();
        var spec = request.getSpec();

        // 1. Create Cloud SQL Snapshot (managed service, instant)
        var snapshot = cloudSqlClient.createSnapshot(
            spec.getDatabaseInstanceId(),
            generateSnapshotName(backupId)
        );

        // 2. Cloud SQL Export API - NO STREAMING!
        // GCP handles everything: connection, compression, parallelization, GCS writes
        var exportOp = cloudSqlClient.export(
            InstancesExportRequest.newBuilder()
                .setProject(spec.getProjectId())
                .setInstance(spec.getDatabaseInstanceId())
                .setExportContext(ExportContext.newBuilder()
                    .setFileType("SQL")
                    .setDatabases(spec.getDatabases())
                    .setUri("gs://" + spec.getBackupBucket() + "/" + backupId + "/dump.sql.gz")
                    .setSqlExportOptions(SqlExportOptions.newBuilder()
                        .setSchemaOnly(false)
                        .build())
                    .build())
                .build()
        );

        // Wait for export to complete (poll operation status)
        Operation completedOp = waitForCompletion(exportOp);

        // 3. GCS Transfer Service for Document Library - NO STREAMING!
        // GCS handles everything: file listing, filtering, transfer, retries
        var transferJob = storageTransferClient.createTransferJob(
            TransferJob.newBuilder()
                .setProjectId(spec.getProjectId())
                .setTransferSpec(TransferSpec.newBuilder()
                    .setGcsDataSource(GcsData.newBuilder()
                        .setBucketName(spec.getDocumentLibrary().getSourceBucket())
                        .build())
                    .setGcsDataSink(GcsData.newBuilder()
                        .setBucketName(spec.getBackupBucket())
                        .setPath("document-library/" + backupId)
                        .build())
                    // Liferay-specific filtering (30-50TB savings!)
                    .setObjectConditions(ObjectConditions.newBuilder()
                        .addExcludePrefixes("**.thumbnail.**")
                        .addExcludePrefixes("**.preview.**")
                        .addExcludePrefixes("**.converted.**")
                        .build())
                    .setTransferOptions(TransferOptions.newBuilder()
                        .setOverwriteObjectsAlreadyExistingInSink(false)
                        .build())
                    .build())
                .build()
        );

        // Wait for transfer to complete
        TransferOperation transferOp = waitForTransferCompletion(transferJob);

        // 4. Query managed service APIs for metadata (no guessing!)
        var snapshotDetails = cloudSqlClient.getBackupRun(
            snapshot.getTargetId()
        );

        Blob dumpBlob = storageClient.get(
            BlobId.of(spec.getBackupBucket(), backupId + "/dump.sql.gz")
        );

        // 5. Generate Liferay metadata/tracker file
        var metadata = BackupMetadata.builder()
            .backupId(backupId)
            .timestamp(Instant.now())
            .creator(request.getSpec().getCreator())
            .snapshot(SnapshotInfo.builder()
                .id(snapshotDetails.getId())
                .status(snapshotDetails.getStatus())
                .startTime(snapshotDetails.getStartTime())
                .endTime(snapshotDetails.getEndTime())
                .build())
            .databaseDump(DumpInfo.builder()
                .location("gs://" + spec.getBackupBucket() + "/" + backupId + "/dump.sql.gz")
                .size(dumpBlob.getSize())
                .databases(spec.getDatabases())
                .build())
            .documentLibrary(DoclibInfo.builder()
                .location("gs://" + spec.getBackupBucket() + "/document-library/" + backupId)
                .size(transferOp.getCounters().getBytesFoundFromSource())
                .objectCount(transferOp.getCounters().getObjectsFoundFromSource())
                .excludedGeneratedFiles(spec.getDocumentLibrary().isExcludeGeneratedFiles())
                .build())
            .structure("gcsV3")
            .build();

        // 6. Upload metadata to backup bucket
        storageClient.create(
            BlobInfo.newBuilder(spec.getBackupBucket(), backupId + "/metadata.json")
                .setContentType("application/json")
                .build(),
            objectMapper.writeValueAsBytes(metadata)
        );

        return BackupResult.success(metadata);
    }

    private Operation waitForCompletion(Operation operation) {
        while (!operation.getDone()) {
            Thread.sleep(5000);  // Poll every 5 seconds
            operation = cloudSqlClient.getOperation(operation.getName());
        }

        if (operation.hasError()) {
            throw new BackupException("Export failed: " + operation.getError());
        }

        return operation;
    }

    private TransferOperation waitForTransferCompletion(TransferJob job) {
        TransferOperation operation;
        do {
            Thread.sleep(10000);  // Poll every 10 seconds
            var operations = storageTransferClient.listTransferOperations(
                ListTransferOperationsRequest.newBuilder()
                    .setFilter("{\"project_id\": \"" + job.getProjectId() +
                              "\", \"job_names\": [\"" + job.getName() + "\"]}")
                    .build()
            );
            operation = operations.iterator().next();
        } while (!operation.getStatus().equals(Status.SUCCESS) &&
                 !operation.getStatus().equals(Status.FAILED));

        if (operation.getStatus().equals(Status.FAILED)) {
            throw new BackupException("Transfer failed: " + operation.getErrorBreakdowns());
        }

        return operation;
    }
}
```

### Reconciler Pattern (Operator)

```java
@ControllerConfiguration
public class BackupRequestReconciler implements Reconciler<BackupRequest> {

    @Inject ProviderFactory providerFactory;
    @Inject BackupArchiveService archiveService;

    @Override
    public UpdateControl<BackupRequest> reconcile(BackupRequest resource, Context ctx) {
        var status = resource.getStatus();

        switch (status.getState()) {
            case "Pending":
                return initializeBackup(resource);
            case "SnapshotInProgress":
                return checkSnapshotProgress(resource);
            case "ExportInProgress":
                return checkExportProgress(resource);
            case "TransferInProgress":
                return checkTransferProgress(resource);
            case "GeneratingMetadata":
                return generateMetadata(resource);
            case "Succeeded":
                return archiveToPostgres(resource);
            default:
                return UpdateControl.noUpdate();
        }
    }

    private UpdateControl<BackupRequest> initializeBackup(BackupRequest resource) {
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());

        // Start snapshot (instant operation)
        var snapshotOp = provider.createSnapshot(resource);

        resource.getStatus().setState("SnapshotInProgress");
        resource.getStatus().setSnapshotOperation(snapshotOp.getName());

        return UpdateControl.updateStatus(resource).rescheduleAfter(30, TimeUnit.SECONDS);
    }

    private UpdateControl<BackupRequest> checkSnapshotProgress(BackupRequest resource) {
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());
        var operation = provider.getSnapshotStatus(resource.getStatus().getSnapshotOperation());

        if (!operation.isDone()) {
            return UpdateControl.noUpdate().rescheduleAfter(30, TimeUnit.SECONDS);
        }

        if (operation.hasError()) {
            resource.getStatus().setState("Failed");
            resource.getStatus().setErrorMessage(operation.getError());
            return UpdateControl.updateStatus(resource);
        }

        // Snapshot complete, start export (Cloud SQL Export API call)
        var exportOp = provider.exportDatabase(resource);

        resource.getStatus().setState("ExportInProgress");
        resource.getStatus().setExportOperation(exportOp.getName());
        resource.getStatus().setSnapshotId(operation.getTargetId());

        return UpdateControl.updateStatus(resource).rescheduleAfter(30, TimeUnit.SECONDS);
    }

    private UpdateControl<BackupRequest> checkExportProgress(BackupRequest resource) {
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());
        var operation = provider.getExportStatus(resource.getStatus().getExportOperation());

        if (!operation.isDone()) {
            return UpdateControl.noUpdate().rescheduleAfter(30, TimeUnit.SECONDS);
        }

        if (operation.hasError()) {
            resource.getStatus().setState("Failed");
            resource.getStatus().setErrorMessage(operation.getError());
            return UpdateControl.updateStatus(resource);
        }

        // Export complete, start document library transfer (GCS Transfer Service)
        var transferJob = provider.transferDocumentLibrary(resource);

        resource.getStatus().setState("TransferInProgress");
        resource.getStatus().setTransferJobName(transferJob.getName());

        return UpdateControl.updateStatus(resource).rescheduleAfter(60, TimeUnit.SECONDS);
    }

    // ... similar patterns for transfer and metadata
}
```

### Local/k3d Implementation (Fallback to Argo Workflows)

```java
@ApplicationScoped
public class LocalBackupProvider implements BackupProvider {

    @Inject ArgoWorkflowService argoService;

    @Override
    public BackupResult createBackup(BackupRequest request) {
        // No managed services in local, use Argo Workflows

        // 1. Submit pg_dump workflow
        var dumpWorkflow = argoService.submitWorkflow(
            WorkflowSpec.builder()
                .template("local-pg-dump")
                .parameters(Map.of(
                    "db_host", request.getSpec().getDatabaseHost(),
                    "db_name", request.getSpec().getDatabaseName(),
                    "output_path", "s3://minio/backup/" + backupId + "/dump.sql.gz"
                ))
                .build()
        );

        // 2. Submit rclone workflow for document library
        var doclibWorkflow = argoService.submitWorkflow(
            WorkflowSpec.builder()
                .template("local-doclib-backup")
                .parameters(Map.of(
                    "source_bucket", request.getSpec().getDocumentLibrary().getSourceBucket(),
                    "dest_bucket", "minio/backup/" + backupId + "/doclib",
                    "exclude_generated", request.getSpec().getDocumentLibrary().isExcludeGeneratedFiles()
                ))
                .build()
        );

        // Wait for both
        argoService.waitForCompletion(dumpWorkflow, doclibWorkflow);

        // Generate metadata
        // ... similar to cloud provider
    }
}
```

---

## Performance & Cost Comparison

| Approach | Database Backup | DL Backup | Performance | OOM Risk | Maintenance |
|----------|----------------|-----------|-------------|----------|-------------|
| **Current (streaming)** | pg_dump in pod | rclone in pod | Slow | **High** | Medium |
| **Earlier Hybrid (Argo)** | Argo pg_dump | Argo rclone | Better | Medium | Medium |
| **Enhanced Hybrid (APIs)** | Cloud SQL Export API | GCS Transfer Service | **Fastest** | **None** | **Low** |

**Cost per 10TB DB + 100TB doclib (GCP):**
- Storage: $820 (Cloud SQL backup) + $1,200 (GCS archive) = $2,020/month
- Transfer: $0 (within same region), $12k per cross-region backup
- **No compute costs for backup service** (managed services handle execution)

---

## Critical Liferay Peculiarities That Prevent Pure Managed Service Approach

1. **❌ Dual Database Backup (Snapshot + Dump)**
   - Managed services do one or the other, not both
   - **Solution:** Use snapshot + export API

2. **✅ excludeGeneratedDoclibFiles Filtering**
   - Can save 30-50TB per environment
   - **Solution:** GCS Transfer Service ObjectConditions, S3 Transfer with filters

3. **❌ Multi-Component Coordination**
   - Database + Document Library must be linked as single backup
   - **Solution:** Operator orchestrates both, generates unified metadata

4. **❌ Custom Metadata/Tracker Files**
   - Required for Cockpit UI catalog, selective restore
   - **Solution:** Query managed service APIs, generate custom metadata

5. **✅ databasesToIgnore Configuration**
   - **Solution:** Cloud SQL Export API, RDS Export support database filtering

6. **❌ Multi-Version Restore**
   - Need to restore GCS V2, GCS V3, SFS formats
   - **Solution:** Custom restore logic in operator

---

## Implementation Roadmap (Revised for Enhanced Hybrid)

### Phase 1: Core Backup (Months 1-3)

**Sprint 1-2: Foundation**
- Define BackupRequest CRD (Java classes)
- Implement basic operator reconciliation loop
- Set up local dev environment (k3d + PostgreSQL + MinIO)

**Sprint 3-4: Managed Service API Integration**
- Implement GCPBackupProvider (Cloud SQL Snapshot + Export API + GCS Transfer Service)
- Implement AWSBackupProvider (RDS Snapshot + Export to S3 + S3 Transfer)
- Implement LocalBackupProvider (Argo Workflows fallback)
- Implement operation polling and status tracking

**Sprint 5-6: Metadata & Coordination**
- Query managed service APIs for backup details
- Generate Liferay metadata/tracker files
- Multi-component coordination logic
- excludeGeneratedFiles filtering configuration

**Sprint 7-8: Integration & Testing**
- PostgreSQL archival for history
- REST API for Cockpit UI
- Integration tests with real Cloud SQL instances
- Performance testing (eliminate OOM)

**Deliverable:** Working backup for GCP, AWS, and local environments

### Phase 2: Restore (Months 4-5)

**Sprint 9-10: Restore Logic**
- RestoreRequest CRD
- Restore reconciler
- Database restore (from snapshot OR from dump using Cloud SQL Import API)
- Document library restore (using GCS Transfer Service)

**Sprint 11-12: Advanced Restore**
- Selective restore (database only, doclib only, or both)
- Multi-version restore (GCS V2, V3, SFS)
- Validation and verification

**Deliverable:** Full backup + restore capability

### Phase 3: Production Readiness (Month 6)

**Sprint 13: Azure Support**
- AzureBackupProvider implementation
- Azure-specific testing

**Sprint 14: Monitoring & Ops**
- Prometheus metrics
- Grafana dashboards
- Alerting rules
- Runbooks

**Sprint 15: Performance & Scale**
- Load testing (100+ concurrent backups)
- Cost optimization
- Documentation

**Deliverable:** Production-ready backup solution

---

## Decision Summary

| Requirement | Pure Managed | Full Custom | Hybrid (Argo) | Enhanced Hybrid (APIs) |
|-------------|--------------|-------------|---------------|----------------------|
| Database snapshots | ✅ Excellent | ⚠️ Reinventing | ✅ Managed APIs | ✅ Managed APIs |
| Database dumps | ❌ Not supported | ✅ Full control | ⚠️ Argo Workflows | ✅ Export APIs |
| Document library | ❌ No filtering | ✅ Full control | ⚠️ Argo rclone | ✅ Transfer Service |
| excludeGeneratedFiles | ❌ Not possible | ✅ Supported | ✅ Supported | ✅ Transfer Service |
| Metadata/tracker files | ❌ Not compatible | ✅ Full control | ✅ Query + generate | ✅ Query + generate |
| Cloud-agnostic | ❌ Different/cloud | ✅ Unified CRD | ✅ Unified CRD | ✅ Unified CRD |
| Local/k3d support | ❌ Not possible | ✅ Supported | ✅ Supported | ✅ Argo fallback |
| OOM risk | N/A | ⚠️ Medium | ⚠️ Medium | ✅ **None** |
| Performance | ⚠️ Variable | ⚠️ Slow | ⚠️ Medium | ✅ **Fastest** |
| Maintenance burden | ✅ Low | ⚠️ High | ⚠️ Medium | ✅ **Low** |
| Cost (100 envs) | ❌ $3.4M/year | ✅ $1.7M/year | ✅ $2.2M/year | ✅ $2.0M/year |

**Verdict:** Enhanced hybrid approach (using managed service APIs) provides:
- ✅ Best performance (no streaming, no OOM)
- ✅ Lowest maintenance (GCP/AWS handle execution)
- ✅ Liferay-specific customization (filtering, metadata, coordination)
- ✅ Cloud-agnostic interface (unified CRDs)
- ✅ Cost optimization (efficient managed service usage)

---

## Key Takeaways

1. **Current Liferay PaaS team already validated this approach** - Iliyan's "Rethinking Backups" proposes exactly this solution

2. **Streaming through backup service is the problem** - Causes OOM errors and failures

3. **Managed service APIs solve the streaming problem** - Cloud SQL Export API, GCS Transfer Service, RDS Export

4. **Operator becomes thin orchestration layer** - Just API calls, polling, metadata generation

5. **Argo Workflows only for local/k3d** - Where managed services aren't available

6. **This is NOT "reinventing the wheel"** - We're using managed services where they excel, custom logic only for Liferay-specific requirements

---

## Generalizability: Applicable to Any Database + Object Store System

### Is This Approach Liferay-Specific or Generally Applicable?

**Answer: The ARCHITECTURE is generally applicable. The IMPLEMENTATION DETAILS are Liferay-specific.**

### Generally Applicable Patterns

This enhanced hybrid approach works for **any stateful application** on Kubernetes with:

1. **Database + Object Storage** (PostgreSQL/MySQL + S3/GCS/Blob)
2. **Multi-component backups** (database + files must be linked)
3. **Cloud-agnostic requirements** (AWS, GCP, Azure, local)
4. **Kubernetes-native deployment** (CRDs, GitOps, operators)
5. **Performance requirements** (avoid OOM, streaming bottlenecks)

**Examples of Applicable Systems:**
- **GitLab** - PostgreSQL + Git repositories (S3/GCS)
- **Nextcloud** - MySQL + file storage (S3/GCS/Blob)
- **WordPress** - MySQL + media library (S3/GCS)
- **Mattermost** - PostgreSQL + file attachments (S3)
- **Discourse** - PostgreSQL + uploads (S3/GCS)
- **Jira/Confluence** - PostgreSQL + attachments (S3/GCS)
- **Any SaaS application** - Database + user-uploaded content

### Core Architecture (Universal)

```
BackupRequest CRD → Kubernetes Operator
                       ↓
        ┌──────────────┴────────────────────────┐
        ↓                                        ↓
  Managed Service APIs                  Custom Coordination
        ↓                                        ↓
  - Database Snapshot                   - Metadata generation
  - Database Export API                 - Component linking
    (direct to storage)                  - Application-specific logic
  - Object Storage Transfer             - CRD status updates
    (with filtering)                     - PostgreSQL archival
                                         - UI integration
```

**What's Universal:**
- ✅ Use managed service APIs to eliminate streaming (OOM prevention)
- ✅ Coordinate multiple components (database + files)
- ✅ Generate unified metadata (catalog, selective restore)
- ✅ Kubernetes-native interface (CRDs, GitOps)
- ✅ Cloud-agnostic abstraction (same CRD, different providers)
- ✅ Local development support (Argo Workflows fallback)

### Liferay-Specific Implementation Details

These are the parts you'd customize for your application:

1. **File Filtering Rules**
   - Liferay: `excludeGeneratedDoclibFiles` (thumbnails, previews, converted docs)
   - GitLab: Exclude CI/CD artifacts, caches
   - WordPress: Exclude resized images, caches
   - **Pattern:** Any app that generates derivative files can benefit from filtering

2. **Metadata Schema**
   - Liferay: `BackupMetadataGcsV3Impl` with specific structure
   - Your app: Define your own metadata schema
   - **Pattern:** Unified metadata linking all backup components

3. **Database Filtering**
   - Liferay: `databasesToIgnore` (system databases, temp databases)
   - Your app: Define which databases/schemas to exclude
   - **Pattern:** Most apps don't need to backup every database

4. **Multi-Version Restore**
   - Liferay: Support GCS V2, V3, SFS formats (legacy compatibility)
   - Your app: Support your legacy backup formats
   - **Pattern:** Production systems need backward compatibility

5. **Application-Specific Coordination**
   - Liferay: Dual backup (snapshot + dump for different restore scenarios)
   - GitLab: Backup git repos + PostgreSQL + Redis state
   - Your app: Define your components and coordination logic
   - **Pattern:** Most apps have >1 stateful component to coordinate

### Example: Adapting for GitLab

```java
@ApplicationScoped
public class GitLabBackupProvider implements BackupProvider {

    @Inject CloudSqlService cloudSqlClient;
    @Inject GCSTransferService gcsTransferClient;

    @Override
    public BackupResult createBackup(BackupRequest request) {
        var spec = request.getSpec();

        // 1. PostgreSQL Snapshot + Export (same as Liferay)
        var snapshot = cloudSqlClient.createSnapshot(spec.getDatabaseInstanceId());
        var exportOp = cloudSqlClient.export(spec.getDatabaseInstanceId(),
            "gs://" + spec.getBackupBucket() + "/postgresql.sql.gz");

        // 2. Git Repositories Transfer (GitLab-specific)
        var gitReposTransfer = gcsTransferClient.createTransferJob(
            TransferSpec.builder()
                .sourceBucket(spec.getGitRepositories().getSourceBucket())
                .destBucket(spec.getBackupBucket())
                .destPath("git-repositories/" + backupId)
                // GitLab-specific: Exclude CI artifacts, caches
                .excludePatterns(List.of(
                    "**/builds/**",          // CI build artifacts
                    "**/cache/**",           // Dependency caches
                    "**/tmp/**"              // Temporary files
                ))
                .build()
        );

        // 3. Uploads Transfer (same pattern as Liferay doclib)
        var uploadsTransfer = gcsTransferClient.createTransferJob(
            TransferSpec.builder()
                .sourceBucket(spec.getUploads().getSourceBucket())
                .destBucket(spec.getBackupBucket())
                .destPath("uploads/" + backupId)
                .build()
        );

        // Wait for all components
        waitForCompletion(exportOp, gitReposTransfer, uploadsTransfer);

        // 4. Generate GitLab-specific metadata
        var metadata = GitLabBackupMetadata.builder()
            .backupId(backupId)
            .postgresSnapshot(snapshot.getId())
            .postgresDump("gs://" + spec.getBackupBucket() + "/postgresql.sql.gz")
            .gitRepositories("gs://" + spec.getBackupBucket() + "/git-repositories/" + backupId)
            .uploads("gs://" + spec.getBackupBucket() + "/uploads/" + backupId)
            .gitlabVersion(spec.getGitLabVersion())  // GitLab-specific
            .build();

        return BackupResult.success(metadata);
    }
}
```

### Comparison: Liferay vs Generic Application

| Component | Liferay Implementation | Generic Pattern |
|-----------|----------------------|-----------------|
| **Database Snapshot** | Cloud SQL Snapshot API | Database Snapshot API (universal) |
| **Database Dump** | Cloud SQL Export API | Database Export API (universal) |
| **File Storage** | Document Library (GCS Transfer) | Object Storage Transfer (universal) |
| **File Filtering** | `excludeGeneratedDoclibFiles` (thumbnails, previews) | Custom filtering rules (app-specific) |
| **Metadata** | `BackupMetadataGcsV3Impl` | Custom metadata schema (app-specific) |
| **Components** | DB + Document Library | DB + Files (universal concept) |
| **Coordination** | Operator orchestrates both | Operator orchestrates (universal) |
| **Local Support** | Argo Workflows (pg_dump + rclone) | Argo Workflows (universal fallback) |

### Building a Generic Backup Operator Framework

This approach could be **productized as a generic framework**:

```java
// Generic backup framework (reusable)
public abstract class BackupProvider {
    // Universal operations
    protected abstract SnapshotResult createDatabaseSnapshot(DatabaseSpec spec);
    protected abstract ExportResult exportDatabase(DatabaseSpec spec);
    protected abstract TransferResult transferObjectStorage(ObjectStorageSpec spec);

    // Template method (universal flow)
    public final BackupResult createBackup(BackupRequest request) {
        // 1. Snapshot
        var snapshot = createDatabaseSnapshot(request.getDatabaseSpec());

        // 2. Export
        var export = exportDatabase(request.getDatabaseSpec());

        // 3. Object storage
        var transfers = request.getObjectStorageSpecs().stream()
            .map(this::transferObjectStorage)
            .collect(Collectors.toList());

        // 4. Wait for completion
        waitForCompletion(snapshot, export, transfers);

        // 5. Generate metadata (delegated to subclass)
        return generateMetadata(snapshot, export, transfers, request);
    }

    // Application-specific (override in subclass)
    protected abstract BackupMetadata generateMetadata(
        SnapshotResult snapshot,
        ExportResult export,
        List<TransferResult> transfers,
        BackupRequest request
    );

    // Application-specific (override in subclass)
    protected List<String> getExclusionPatterns(ObjectStorageSpec spec) {
        return List.of();  // Default: no exclusions
    }
}

// Liferay-specific implementation
@ApplicationScoped
public class LiferayBackupProvider extends BackupProvider {
    @Override
    protected BackupMetadata generateMetadata(...) {
        return LiferayBackupMetadata.builder()
            .structure("gcsV3")  // Liferay-specific
            .creationType(...)   // Liferay-specific
            .build();
    }

    @Override
    protected List<String> getExclusionPatterns(ObjectStorageSpec spec) {
        if (spec.isExcludeGeneratedFiles()) {
            return List.of("**.thumbnail.**", "**.preview.**");  // Liferay-specific
        }
        return List.of();
    }
}

// GitLab-specific implementation
@ApplicationScoped
public class GitLabBackupProvider extends BackupProvider {
    @Override
    protected BackupMetadata generateMetadata(...) {
        return GitLabBackupMetadata.builder()
            .gitlabVersion(...)  // GitLab-specific
            .build();
    }

    @Override
    protected List<String> getExclusionPatterns(ObjectStorageSpec spec) {
        return List.of("**/builds/**", "**/cache/**");  // GitLab-specific
    }
}
```

### Benefits of Generic Framework

1. **Reusable Core** - Database snapshot, export, transfer logic is universal
2. **Application-Specific Extensions** - Override filtering, metadata, coordination
3. **Consistent Interface** - Same CRDs, operator pattern across all apps
4. **Lower Development Cost** - Build once, customize per app
5. **Best Practices Built-In** - Eliminate streaming, use managed APIs, avoid OOM

### Conclusion

**The enhanced hybrid approach is a GENERAL PATTERN for stateful Kubernetes applications.**

**Liferay-specific parts:**
- File filtering patterns (thumbnails, previews)
- Metadata schema (GcsV3 structure)
- Database exclusions (temp databases)
- Legacy format support (V2, SFS)

**Universal architecture:**
- Use managed service APIs (no streaming)
- Coordinate multiple components
- Generate unified metadata
- Kubernetes-native (CRDs, operators)
- Cloud-agnostic abstraction

**This could be a product:** A generic Kubernetes backup operator framework that applications customize with their specific filtering rules, metadata schemas, and coordination logic.

The core insight — **"use managed service APIs to eliminate streaming"** — applies to any application backing up databases and object storage.
