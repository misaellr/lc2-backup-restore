# LC2 Backup Service - Final Java-Based Architecture

## Executive Summary

After comprehensive analysis and debate, this document defines the **definitive architecture** for the LC2 Backup Service:

- **Language**: Java (Quarkus)
- **Pattern**: Kubernetes Operator + **Managed Service APIs**
- **State**: Hybrid (CRD + PostgreSQL)
- **Multi-Cloud**: Strategy Pattern with provider abstraction
- **Timeline**: 3 months, 2 engineers
- **Validation**: Confirmed by Liferay PaaS team's "Rethinking Backups" analysis

### Why Managed Service APIs?

The current Liferay PaaS backup service suffers from critical issues (documented in "Rethinking Backups" by Iliyan Peychev):
- **OutOfMemory errors** from streaming data through backup service pods
- **Unstable data transfer** due to memory pressure
- **Long backup/restore times** from inefficient processing

**Solution**: Use managed service APIs (Cloud SQL Export API, GCS Transfer Service) to **eliminate data streaming through operator pods**. Managed services handle compression, parallelization, and direct storage writes.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (Any Cloud)                            │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │         LC2 Backup Operator (Quarkus)                                  │ │
│  │                                                                         │ │
│  │  ┌──────────────┐    ┌──────────────────┐   ┌──────────────┐         │ │
│  │  │ Reconciler   │───▶│ Provider Factory │───│ PostgreSQL   │         │ │
│  │  │ (watches CRD)│    │ (Strategy)       │   │ (archives)   │         │ │
│  │  └──────────────┘    └──────────────────┘   └──────────────┘         │ │
│  │         │                    │                                         │ │
│  │         │         ┌──────────┴──────────┬─────────┬─────────┐         │ │
│  │         │         ▼          ▼          ▼         ▼         ▼         │ │
│  │         │    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────────┐          │ │
│  │         │    │ AWS  │  │ GCP  │  │Azure │  │Local (Argo)  │          │ │
│  │         │    │(APIs)│  │(APIs)│  │(APIs)│  │(Fallback)    │          │ │
│  │         │    └───┬──┘  └───┬──┘  └───┬──┘  └──────────────┘          │ │
│  └─────────┼────────┼─────────┼─────────┼────────────────────────────────┘ │
│            │        │         │         │                                   │
│            ▼        │         │         │                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │               BackupRequest CRD                                      │  │
│  │  apiVersion: backup.liferay.cloud/v1alpha1                          │  │
│  │  kind: BackupRequest                                                 │  │
│  │  spec:                                                                │  │
│  │    projectId: retail                                                 │  │
│  │    provider: gcp                                                     │  │
│  │    database: {...}                                                   │  │
│  │    documentLibrary: {...}                                            │  │
│  │  status:                                                             │  │
│  │    state: InProgress                                                 │  │
│  │    snapshot: {...}     # Managed service handles this!              │  │
│  │    exportOperation: {...}  # No streaming through operator!         │  │
│  │    transferJob: {...}  # GCS Transfer Service handles this!         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
           │                │                 │
           ▼                ▼                 ▼
  ┌─────────────────┐  ┌──────────────┐  ┌──────────────────┐
  │ Cloud SQL API   │  │GCS Transfer  │  │   PostgreSQL     │
  │                 │  │Service API   │  │ (Backup History) │
  │ - Snapshot API  │  │              │  │                  │
  │ - Export API    │  │- Filters     │  │ - backup_requests│
  │   (direct to    │  │- Excludes    │  │ - metadata       │
  │    GCS!)        │  │- No streaming│  │ - metrics        │
  └─────────────────┘  └──────────────┘  └──────────────────┘
           │                    │
           └────────────────────┘
                     │
                     ▼
            ┌────────────────┐
            │ GCS Bucket     │
            │                │
            │ /backups/      │
            │   snapshot.id  │
            │   dump.sql.gz  │ ◀── Written directly by Cloud SQL!
            │   doclib/      │ ◀── Copied by GCS Transfer Service!
            │   metadata.json│
            └────────────────┘

Key Difference: NO DATA STREAMS THROUGH OPERATOR!
- Cloud SQL Export writes directly to GCS
- GCS Transfer Service copies between buckets
- Operator only orchestrates via APIs
```

---

## Technology Stack

### Core Framework
**Quarkus 3.x** (Kubernetes-native Java)

**Why Quarkus over Spring Boot?**
- ✅ Kubernetes-native (container-first design)
- ✅ Fast startup (~50ms vs 3-5s Spring Boot)
- ✅ Low memory footprint (30MB vs 150MB Spring Boot)
- ✅ Built-in Kubernetes Operator SDK extension
- ✅ Designed for cloud-native microservices
- ✅ GraalVM native compilation support (optional)
- ⚠️ Learning curve (but worth it for long-term gains)

### Dependencies

```xml
<dependencies>
    <!-- Quarkus Core -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-resteasy-reactive</artifactId>
    </dependency>

    <!-- Kubernetes Operator SDK -->
    <dependency>
        <groupId>io.quarkiverse.operatorsdk</groupId>
        <artifactId>quarkus-operator-sdk</artifactId>
        <version>6.x</version>
    </dependency>

    <!-- Kubernetes Client -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-kubernetes-client</artifactId>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-hibernate-orm-panache</artifactId>
    </dependency>
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-jdbc-postgresql</artifactId>
    </dependency>

    <!-- Cloud Provider SDKs (Managed Service APIs) -->

    <!-- AWS: RDS, RDS Export to S3, S3 Transfer -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>rds</artifactId>
        <version>2.20.x</version>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>2.20.x</version>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3-transfer-manager</artifactId>
        <version>2.20.x</version>
    </dependency>

    <!-- GCP: Cloud SQL Admin API, GCS Transfer Service, Storage -->
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>google-cloud-sql</artifactId>
        <version>2.x</version>
    </dependency>
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>google-cloud-storage</artifactId>
        <version>2.x</version>
    </dependency>
    <dependency>
        <groupId>com.google.cloud</groupId>
        <artifactId>google-cloud-storage-transfer</artifactId>
        <version>1.x</version>
    </dependency>

    <!-- Azure: Database Flexible Server, Blob Storage, Blob Transfer -->
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-resourcemanager-postgresql</artifactId>
        <version>1.x</version>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>
        <version>12.x</version>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob-batch</artifactId>
        <version>12.x</version>
    </dependency>

    <!-- Argo Workflows (Local/k3d fallback only) -->
    <dependency>
        <groupId>io.argoproj.workflow</groupId>
        <artifactId>argo-client-java</artifactId>
        <version>3.5.x</version>
        <scope>provided</scope>
    </dependency>

    <!-- Observability -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-logging-json</artifactId>
    </dependency>
</dependencies>
```

---

## Core Components

### 1. BackupRequest CRD

**File**: `src/main/java/com/liferay/backup/crd/BackupRequest.java`

```java
package com.liferay.backup.crd;

import io.fabric8.kubernetes.api.model.Namespaced;
import io.fabric8.kubernetes.client.CustomResource;
import io.fabric8.kubernetes.model.annotation.Group;
import io.fabric8.kubernetes.model.annotation.Version;

@Group("backup.liferay.cloud")
@Version("v1alpha1")
public class BackupRequest extends CustomResource<BackupRequestSpec, BackupRequestStatus>
    implements Namespaced {
}
```

**Spec**:
```java
public class BackupRequestSpec {
    private String projectId;
    private String environment;
    private String provider; // aws|gcp|azure|local
    private DatabaseBackupSpec database;
    private DocumentLibraryBackupSpec documentLibrary;
    private Integer retentionDays;

    // Getters/setters
}

public class DatabaseBackupSpec {
    private String type; // snapshot|dump
    private String instanceId;
    private String databaseName; // Postgres only
    private List<String> databasesToIgnore; // MySQL only
}

public class DocumentLibraryBackupSpec {
    private String sourceBucket;
    private String sourcePrefix;
    private String destinationBucket;
    private String destinationPrefix;
    private Boolean excludeGeneratedFiles;
}
```

**Status**:
```java
public class BackupRequestStatus {
    private String state; // Pending|SnapshotInProgress|ExportInProgress|TransferInProgress|GeneratingMetadata|Succeeded|Failed
    private ZonedDateTime startTime;
    private ZonedDateTime completionTime;

    // Managed service operation tracking
    private SnapshotStatus snapshot; // Cloud SQL/RDS snapshot operation
    private ExportStatus exportOperation; // Cloud SQL/RDS export operation
    private TransferStatus transferJob; // GCS/S3 Transfer Service job

    // Final results
    private DatabaseBackupStatus database;
    private DocumentLibraryBackupStatus documentLibrary;
    private String metadataLocation; // gs://bucket/backupId/metadata.json
    private String message;

    // Getters/setters
}

public class SnapshotStatus {
    private String snapshotId;
    private String status; // creating|available|error
    private Long sizeBytes;
}

public class ExportStatus {
    private String operationName; // GCP operation resource name
    private String exportTaskId; // AWS export task ID
    private String status; // running|succeeded|failed
    private String location; // gs://bucket/dump.sql.gz
    private Long sizeBytes;
}

public class TransferStatus {
    private String jobName; // GCS Transfer Job name
    private String status; // running|succeeded|failed
    private Long objectsTransferred;
    private Long bytesTransferred;
}
```

---

### 2. Backup Reconciler (Kubernetes Operator)

**File**: `src/main/java/com/liferay/backup/reconciler/BackupRequestReconciler.java`

**Key Principle**: Operator orchestrates via managed service APIs; **NO data streams through operator pods**.

```java
package com.liferay.backup.reconciler;

import io.javaoperatorsdk.operator.api.reconciler.*;
import io.javaoperatorsdk.operator.api.reconciler.Context;
import jakarta.inject.Inject;

@ControllerConfiguration
public class BackupRequestReconciler
    implements Reconciler<BackupRequest>, ErrorStatusHandler<BackupRequest> {

    @Inject
    BackupArchiveService archiveService;

    @Inject
    ProviderFactory providerFactory;

    @Override
    public UpdateControl<BackupRequest> reconcile(
        BackupRequest resource,
        Context<BackupRequest> context
    ) {
        var spec = resource.getSpec();
        var status = resource.getStatus();

        // State machine for managed service API orchestration
        switch (status.getState()) {
            case "":
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
                return archiveBackup(resource);

            case "Failed":
                return UpdateControl.noUpdate(); // Already failed

            default:
                return UpdateControl.noUpdate();
        }
    }

    private UpdateControl<BackupRequest> initializeBackup(BackupRequest resource) {
        // 1. Validate spec
        var validation = validateBackupRequest(resource);
        if (!validation.isValid()) {
            return failBackup(resource, validation.getError());
        }

        // 2. Get provider
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());

        // 3. Initiate snapshot via managed service API (NO DATA STREAMING!)
        try {
            var snapshotResult = provider.createDatabaseSnapshot(
                resource.getSpec().getDatabase()
            );

            // 4. Update status
            var snapshot = new SnapshotStatus();
            snapshot.setSnapshotId(snapshotResult.getSnapshotId());
            snapshot.setStatus("creating");

            resource.getStatus().setState("SnapshotInProgress");
            resource.getStatus().setSnapshot(snapshot);
            resource.getStatus().setStartTime(ZonedDateTime.now());

            return UpdateControl.updateStatus(resource);
        } catch (Exception e) {
            return failBackup(resource, "Failed to initiate snapshot: " + e.getMessage());
        }
    }

    private UpdateControl<BackupRequest> checkSnapshotProgress(BackupRequest resource) {
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());
        var snapshotId = resource.getStatus().getSnapshot().getSnapshotId();

        // Poll snapshot status from managed service
        var snapshotStatus = provider.getSnapshotStatus(snapshotId);

        if (snapshotStatus.isCompleted()) {
            // Snapshot complete! Now initiate export (NO DATA STREAMING!)
            try {
                var exportResult = provider.exportDatabaseToStorage(
                    resource.getSpec().getDatabase(),
                    snapshotId,
                    generateBackupPath(resource)
                );

                var exportStatus = new ExportStatus();
                exportStatus.setOperationName(exportResult.getOperationName());
                exportStatus.setStatus("running");
                exportStatus.setLocation(exportResult.getDestination());

                resource.getStatus().getSnapshot().setStatus("available");
                resource.getStatus().getSnapshot().setSizeBytes(snapshotStatus.getSizeBytes());
                resource.getStatus().setState("ExportInProgress");
                resource.getStatus().setExportOperation(exportStatus);

                return UpdateControl.updateStatus(resource);
            } catch (Exception e) {
                return failBackup(resource, "Failed to initiate export: " + e.getMessage());
            }
        }

        if (snapshotStatus.isFailed()) {
            return failBackup(resource, "Snapshot failed: " + snapshotStatus.getError());
        }

        // Still creating, requeue
        return UpdateControl.noUpdate();
    }

    private UpdateControl<BackupRequest> checkExportProgress(BackupRequest resource) {
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());
        var operationName = resource.getStatus().getExportOperation().getOperationName();

        // Poll export operation status from managed service
        var exportStatus = provider.getExportStatus(operationName);

        if (exportStatus.isCompleted()) {
            // Export complete! Now initiate transfer (NO DATA STREAMING!)
            try {
                var transferResult = provider.transferObjectStorage(
                    resource.getSpec().getDocumentLibrary()
                );

                var transferStatus = new TransferStatus();
                transferStatus.setJobName(transferResult.getJobName());
                transferStatus.setStatus("running");

                resource.getStatus().getExportOperation().setStatus("succeeded");
                resource.getStatus().getExportOperation().setSizeBytes(exportStatus.getSizeBytes());
                resource.getStatus().setState("TransferInProgress");
                resource.getStatus().setTransferJob(transferStatus);

                return UpdateControl.updateStatus(resource);
            } catch (Exception e) {
                return failBackup(resource, "Failed to initiate transfer: " + e.getMessage());
            }
        }

        if (exportStatus.isFailed()) {
            return failBackup(resource, "Export failed: " + exportStatus.getError());
        }

        // Still running, requeue
        return UpdateControl.noUpdate();
    }

    private UpdateControl<BackupRequest> checkTransferProgress(BackupRequest resource) {
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());
        var jobName = resource.getStatus().getTransferJob().getJobName();

        // Poll transfer job status from managed service
        var transferStatus = provider.getTransferStatus(jobName);

        if (transferStatus.isCompleted()) {
            // Transfer complete! Now generate metadata
            resource.getStatus().getTransferJob().setStatus("succeeded");
            resource.getStatus().getTransferJob().setObjectsTransferred(transferStatus.getObjectCount());
            resource.getStatus().getTransferJob().setBytesTransferred(transferStatus.getBytesTransferred());
            resource.getStatus().setState("GeneratingMetadata");

            return UpdateControl.updateStatus(resource);
        }

        if (transferStatus.isFailed()) {
            return failBackup(resource, "Transfer failed: " + transferStatus.getError());
        }

        // Still running, requeue
        return UpdateControl.noUpdate();
    }

    private UpdateControl<BackupRequest> generateMetadata(BackupRequest resource) {
        var provider = providerFactory.getProvider(resource.getSpec().getProvider());

        try {
            // Generate Liferay-specific metadata/tracker file
            var metadata = BackupMetadata.builder()
                .backupId(generateBackupId(resource))
                .timestamp(resource.getStatus().getStartTime())
                .creator(resource.getSpec().getCreator())
                .snapshot(resource.getStatus().getSnapshot())
                .databaseExport(resource.getStatus().getExportOperation())
                .documentLibrary(resource.getStatus().getTransferJob())
                .structure("gcsV3")
                .build();

            // Upload metadata to bucket
            var metadataLocation = provider.uploadMetadata(
                resource.getSpec().getDocumentLibrary().getDestinationBucket(),
                generateBackupPath(resource),
                metadata
            );

            resource.getStatus().setMetadataLocation(metadataLocation);
            resource.getStatus().setState("Succeeded");
            resource.getStatus().setCompletionTime(ZonedDateTime.now());

            return UpdateControl.updateStatus(resource);
        } catch (Exception e) {
            return failBackup(resource, "Failed to generate metadata: " + e.getMessage());
        }
    }

    private UpdateControl<BackupRequest> archiveBackup(BackupRequest resource) {
        // Archive to PostgreSQL for history/analytics
        archiveService.archiveBackupRequest(resource);

        // No more updates needed
        return UpdateControl.noUpdate();
    }

    private UpdateControl<BackupRequest> failBackup(BackupRequest resource, String error) {
        resource.getStatus().setState("Failed");
        resource.getStatus().setMessage(error);
        resource.getStatus().setCompletionTime(ZonedDateTime.now());

        // Still archive failures for analytics
        archiveService.archiveBackupRequest(resource);

        return UpdateControl.updateStatus(resource);
    }

    @Override
    public ErrorStatusUpdateControl<BackupRequest> updateErrorStatus(
        BackupRequest resource,
        Context<BackupRequest> context,
        Exception e
    ) {
        resource.getStatus().setState("Failed");
        resource.getStatus().setMessage(e.getMessage());
        resource.getStatus().setCompletionTime(ZonedDateTime.now());
        return ErrorStatusUpdateControl.updateStatus(resource);
    }
}
```

---

### 3. Provider Abstraction (Strategy Pattern)

**Interface**: `src/main/java/com/liferay/backup/provider/BackupProvider.java`

```java
package com.liferay.backup.provider;

/**
 * Provider abstraction for cloud-agnostic backup operations.
 *
 * Key principle: Use managed service APIs to eliminate data streaming through operator.
 * - Database exports write directly to object storage
 * - Object transfers use native cloud transfer services
 * - Operator only orchestrates and polls status
 */
public interface BackupProvider {

    /**
     * Get the provider type (aws|gcp|azure|local)
     */
    ProviderType getProviderType();

    /**
     * Create a database snapshot using managed service API.
     * Examples:
     * - AWS: rdsClient.createDBSnapshot()
     * - GCP: sqladminClient.insert().execute()
     * - Azure: serversClient.backup()
     */
    DatabaseSnapshotResult createDatabaseSnapshot(DatabaseBackupSpec spec);

    /**
     * Get database snapshot status from managed service.
     * Used for polling until snapshot completes.
     */
    SnapshotStatusResult getSnapshotStatus(String snapshotId);

    /**
     * Export database to object storage using managed service API.
     * This is the KEY method that eliminates data streaming!
     *
     * Examples:
     * - AWS: rdsClient.startExportTask() - writes directly to S3
     * - GCP: sqladminClient.export() - writes directly to GCS
     * - Azure: serversClient.export() - writes directly to Blob Storage
     *
     * @param spec Database spec with export settings
     * @param snapshotId Snapshot to export from (optional for some providers)
     * @param destination Object storage destination (gs://bucket/path or s3://bucket/path)
     * @return Export operation handle for polling
     */
    ExportOperationResult exportDatabaseToStorage(
        DatabaseBackupSpec spec,
        String snapshotId,
        String destination
    );

    /**
     * Get database export operation status from managed service.
     * Used for polling until export completes.
     */
    ExportStatusResult getExportStatus(String operationName);

    /**
     * Transfer object storage using managed transfer service.
     * This is the KEY method for document library backup!
     *
     * Examples:
     * - AWS: s3TransferManager.copy() or AWS DataSync
     * - GCP: storageTransferClient.createTransferJob() with ObjectConditions
     * - Azure: blobBatchClient.setBlobAccessTier() or AzCopy
     *
     * Supports filtering (excludeGeneratedFiles for Liferay):
     * - GCP: ObjectConditions with excludePrefixes
     * - AWS: S3 Batch Operations with filters
     *
     * @param spec Document library spec with source, dest, filtering
     * @return Transfer job handle for polling
     */
    TransferOperationResult transferObjectStorage(DocumentLibraryBackupSpec spec);

    /**
     * Get object storage transfer status from managed service.
     * Used for polling until transfer completes.
     */
    TransferStatusResult getTransferStatus(String jobName);

    /**
     * Upload backup metadata/tracker file to object storage.
     * This is a small JSON file, so direct upload is acceptable.
     */
    String uploadMetadata(String bucket, String prefix, BackupMetadata metadata);

    /**
     * Download backup metadata/tracker file from object storage.
     */
    BackupMetadata downloadMetadata(String bucket, String key);
}
```

**GCP Provider** (Reference Implementation): `src/main/java/com/liferay/backup/provider/gcp/GCPBackupProvider.java`

```java
package com.liferay.backup.provider.gcp;

import com.google.api.services.sqladmin.SQLAdmin;
import com.google.cloud.storage.Storage;
import com.google.cloud.storagetransfer.v1.proto.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

/**
 * GCP provider using managed service APIs.
 *
 * Key APIs used:
 * 1. Cloud SQL Admin API - for snapshots and exports
 * 2. GCS Transfer Service - for document library with filtering
 * 3. GCS Storage API - for metadata upload
 */
@ApplicationScoped
public class GCPBackupProvider implements BackupProvider {

    @Inject
    SQLAdmin cloudSqlClient;

    @Inject
    StorageTransferServiceClient storageTransferClient;

    @Inject
    Storage storageClient;

    @Override
    public ProviderType getProviderType() {
        return ProviderType.GCP;
    }

    @Override
    public DatabaseSnapshotResult createDatabaseSnapshot(DatabaseBackupSpec spec) {
        // Create Cloud SQL backup run (snapshot)
        var backupRun = new BackupRun()
            .setInstance(spec.getInstanceId())
            .setDescription("Liferay backup - " + spec.getBackupId());

        var insert = cloudSqlClient.backupRuns()
            .insert(spec.getProjectId(), spec.getInstanceId(), backupRun);

        var response = insert.execute();

        return DatabaseSnapshotResult.builder()
            .snapshotId(response.getId().toString())
            .status("RUNNING")
            .build();
    }

    @Override
    public SnapshotStatusResult getSnapshotStatus(String snapshotId) {
        // Poll Cloud SQL backup run status
        var backupRun = cloudSqlClient.backupRuns()
            .get(projectId, instanceId, Long.parseLong(snapshotId))
            .execute();

        return SnapshotStatusResult.builder()
            .completed("SUCCESSFUL".equals(backupRun.getStatus()))
            .failed("FAILED".equals(backupRun.getStatus()))
            .sizeBytes(backupRun.getDiskEncryptionStatus().getKmsKeyVersionName())
            .build();
    }

    @Override
    public ExportOperationResult exportDatabaseToStorage(
        DatabaseBackupSpec spec,
        String snapshotId,
        String destination
    ) {
        // KEY METHOD: Cloud SQL Export API writes directly to GCS!
        // NO DATA STREAMS THROUGH OPERATOR POD!

        var exportContext = new ExportContext()
            .setFileType("SQL")
            .setUri(destination) // gs://bucket/backupId/dump.sql.gz
            .setDatabases(spec.getDatabases())
            .setSqlExportOptions(new SqlExportOptions()
                .setSchemaOnly(false)
                .setMysqlExportOptions(new MySqlExportOptions()
                    .setMasterData(1)));

        var exportRequest = new InstancesExportRequest()
            .setExportContext(exportContext);

        var operation = cloudSqlClient.instances()
            .export(spec.getProjectId(), spec.getInstanceId(), exportRequest)
            .execute();

        return ExportOperationResult.builder()
            .operationName(operation.getName())
            .destination(destination)
            .build();
    }

    @Override
    public ExportStatusResult getExportStatus(String operationName) {
        // Poll Cloud SQL operation status
        var operation = cloudSqlClient.operations()
            .get(projectId, operationName)
            .execute();

        var completed = "DONE".equals(operation.getStatus());
        var failed = operation.getError() != null;

        // Query GCS for export file size if completed
        Long sizeBytes = null;
        if (completed) {
            var exportUri = operation.getExportContext().getUri();
            var blob = parseGcsUri(exportUri);
            var blobInfo = storageClient.get(blob.bucket, blob.object);
            sizeBytes = blobInfo.getSize();
        }

        return ExportStatusResult.builder()
            .completed(completed)
            .failed(failed)
            .sizeBytes(sizeBytes)
            .error(failed ? operation.getError().getMessage() : null)
            .build();
    }

    @Override
    public TransferOperationResult transferObjectStorage(DocumentLibraryBackupSpec spec) {
        // KEY METHOD: GCS Transfer Service copies between buckets!
        // NO DATA STREAMS THROUGH OPERATOR POD!

        var transferSpec = TransferSpec.newBuilder()
            .setGcsDataSource(GcsData.newBuilder()
                .setBucketName(spec.getSourceBucket())
                .setPath(spec.getSourcePrefix())
                .build())
            .setGcsDataSink(GcsData.newBuilder()
                .setBucketName(spec.getDestinationBucket())
                .setPath(spec.getDestinationPrefix())
                .build())
            .setTransferOptions(TransferOptions.newBuilder()
                .setOverwriteObjectsAlreadyExistingInSink(false)
                .build());

        // LIFERAY-SPECIFIC: Exclude generated files (thumbnails, previews)
        // This saves 30-50TB per environment!
        if (spec.getExcludeGeneratedFiles()) {
            transferSpec.setObjectConditions(ObjectConditions.newBuilder()
                .addExcludePrefixes("**.thumbnail.**")
                .addExcludePrefixes("**.preview.**")
                .addExcludePrefixes("**.converted.**")
                .build());
        }

        var transferJob = TransferJob.newBuilder()
            .setProjectId(spec.getProjectId())
            .setTransferSpec(transferSpec.build())
            .setStatus(TransferJob.Status.ENABLED)
            .build();

        var createdJob = storageTransferClient.createTransferJob(
            CreateTransferJobRequest.newBuilder()
                .setTransferJob(transferJob)
                .build()
        );

        return TransferOperationResult.builder()
            .jobName(createdJob.getName())
            .build();
    }

    @Override
    public TransferStatusResult getTransferStatus(String jobName) {
        // Poll GCS Transfer Service job status
        var job = storageTransferClient.getTransferJob(
            GetTransferJobRequest.newBuilder()
                .setJobName(jobName)
                .setProjectId(projectId)
                .build()
        );

        // Get latest operation for this job
        var operations = storageTransferClient.listTransferOperations(
            TransferProto.ListTransferOperationsRequest.newBuilder()
                .setFilter("{\"project_id\": \"" + projectId + "\", " +
                          "\"job_names\": [\"" + jobName + "\"]}")
                .build()
        );

        var latestOp = operations.getOperationsList().stream()
            .findFirst()
            .orElse(null);

        if (latestOp == null) {
            return TransferStatusResult.running();
        }

        var metadata = latestOp.getMetadata()
            .unpack(TransferOperation.class);

        var counters = metadata.getCounters();

        return TransferStatusResult.builder()
            .completed(latestOp.getDone() && metadata.getStatus() == TransferOperation.Status.SUCCESS)
            .failed(latestOp.getDone() && metadata.getStatus() == TransferOperation.Status.FAILED)
            .objectCount(counters.getObjectsFoundFromSource())
            .bytesTransferred(counters.getBytesFoundFromSource())
            .build();
    }

    @Override
    public String uploadMetadata(String bucket, String prefix, BackupMetadata metadata) {
        // Upload small metadata JSON to GCS
        var blobId = BlobId.of(bucket, prefix + "/metadata.json");
        var blobInfo = BlobInfo.newBuilder(blobId)
            .setContentType("application/json")
            .build();

        storageClient.create(blobInfo,
            objectMapper.writeValueAsBytes(metadata));

        return "gs://" + bucket + "/" + prefix + "/metadata.json";
    }

    @Override
    public BackupMetadata downloadMetadata(String bucket, String key) {
        var blob = storageClient.get(BlobId.of(bucket, key));
        var content = blob.getContent();
        return objectMapper.readValue(content, BackupMetadata.class);
    }
}
```

**Factory**: `src/main/java/com/liferay/backup/provider/ProviderFactory.java`

```java
package com.liferay.backup.provider;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

@ApplicationScoped
public class ProviderFactory {

    @Inject
    Instance<BackupProvider> providers;

    public BackupProvider getProvider(String providerType) {
        return providers.stream()
            .filter(p -> p.getProviderType().name().equalsIgnoreCase(providerType))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "Unknown provider: " + providerType));
    }
}
```

---

### 4. Local Provider (Argo Workflows Fallback)

**NOTE**: Argo Workflows is ONLY used for local/k3d environments where managed service APIs are unavailable. For cloud environments (AWS/GCP/Azure), managed service APIs are used instead.

**Service**: `src/main/java/com/liferay/backup/provider/local/LocalBackupProvider.java`

```java
package com.liferay.backup.provider.local;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

/**
 * Local provider for k3d/development environments.
 *
 * Uses Argo Workflows to execute pg_dump and rclone since:
 * - No managed service APIs available locally
 * - MinIO provides S3-compatible storage
 * - PostgreSQL/MySQL run as pods
 *
 * NOTE: This is a FALLBACK approach. In cloud environments, managed
 * service APIs are preferred to avoid data streaming through pods.
 */
@ApplicationScoped
public class LocalBackupProvider implements BackupProvider {

    @Inject
    KubernetesClient k8sClient;

    @Inject
    ArgoWorkflowService argoService;

    @Override
    public ProviderType getProviderType() {
        return ProviderType.LOCAL;
    }

    @Override
    public DatabaseSnapshotResult createDatabaseSnapshot(DatabaseBackupSpec spec) {
        // For local, we submit Argo Workflow that runs pg_dump
        // This WILL stream data through pods (acceptable for local/dev only)
        var workflowName = argoService.submitDatabaseDumpWorkflow(spec);

        return DatabaseSnapshotResult.builder()
            .snapshotId(workflowName) // Use workflow name as snapshot ID
            .status("RUNNING")
            .build();
    }

    @Override
    public SnapshotStatusResult getSnapshotStatus(String snapshotId) {
        // Poll Argo Workflow status
        var workflowStatus = argoService.getWorkflowStatus(snapshotId);

        return SnapshotStatusResult.builder()
            .completed(workflowStatus.isSucceeded())
            .failed(workflowStatus.isFailed())
            .build();
    }

    @Override
    public ExportOperationResult exportDatabaseToStorage(
        DatabaseBackupSpec spec,
        String snapshotId,
        String destination
    ) {
        // For local, snapshot and export are the same operation
        // Return completed immediately
        return ExportOperationResult.builder()
            .operationName(snapshotId)
            .destination(destination)
            .build();
    }

    @Override
    public TransferOperationResult transferObjectStorage(DocumentLibraryBackupSpec spec) {
        // Submit Argo Workflow that runs rclone
        var workflowName = argoService.submitDocumentLibraryTransferWorkflow(spec);

        return TransferOperationResult.builder()
            .jobName(workflowName)
            .build();
    }

    // Other methods delegating to Argo Workflows...
}
```

**Argo Workflows** (used only by LocalBackupProvider):

---

### 5. PostgreSQL Archival Service

(Argo Workflow Template details omitted for brevity - see k8s/argo-workflows/ directory for local/k3d workflows)

---

### 6. PostgreSQL Archival Service

**File**: `src/main/java/com/liferay/backup/archive/BackupArchiveService.java`

```java
package com.liferay.backup.archive;

import io.quarkus.hibernate.orm.panache.PanacheRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class BackupArchiveService implements PanacheRepository<BackupRecord> {

    @Transactional
    public void archiveBackupRequest(BackupRequest cr) {
        var record = new BackupRecord();
        record.crName = cr.getMetadata().getName();
        record.crNamespace = cr.getMetadata().getNamespace();
        record.crUid = cr.getMetadata().getUid();
        record.projectId = cr.getSpec().getProjectId();
        record.environment = cr.getSpec().getEnvironment();
        record.provider = cr.getSpec().getProvider();
        record.state = cr.getStatus().getState();
        record.startTime = cr.getStatus().getStartTime();
        record.completionTime = cr.getStatus().getCompletionTime();

        // Database info
        if (cr.getStatus().getDatabase() != null) {
            record.dbSnapshotId = cr.getStatus().getDatabase().getSnapshotId();
            record.dbSnapshotSize = cr.getStatus().getDatabase().getSizeBytes();
            record.dbEngine = cr.getStatus().getDatabase().getEngine();
            record.dbEngineVersion = cr.getStatus().getDatabase().getEngineVersion();
        }

        // Document library info
        if (cr.getStatus().getDocumentLibrary() != null) {
            record.dlSourceBucket = cr.getSpec().getDocumentLibrary().getSourceBucket();
            record.dlDestBucket = cr.getSpec().getDocumentLibrary().getDestinationBucket();
            record.dlDestPrefix = cr.getSpec().getDocumentLibrary().getDestinationPrefix();
            record.dlObjectCount = cr.getStatus().getDocumentLibrary().getObjectCount();
            record.dlTotalSize = cr.getStatus().getDocumentLibrary().getTotalSizeBytes();
            record.dlManifestPath = cr.getStatus().getDocumentLibrary().getManifestPath();
        }

        record.retentionDays = cr.getSpec().getRetentionDays();
        record.archivedAt = ZonedDateTime.now();

        // Upsert (insert or update)
        persist(record);
    }

    public List<BackupRecord> findByProject(String projectId, String environment, int limit) {
        return find("projectId = ?1 and environment = ?2 order by createdAt desc",
                    projectId, environment)
            .page(0, limit)
            .list();
    }
}
```

**Entity**: `src/main/java/com/liferay/backup/archive/BackupRecord.java`

```java
package com.liferay.backup.archive;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.ZonedDateTime;
import java.util.UUID;

@Entity
@Table(name = "backup_requests")
public class BackupRecord extends PanacheEntityBase {

    @Id
    @GeneratedValue
    public UUID id;

    @Column(name = "cr_name", nullable = false)
    public String crName;

    @Column(name = "cr_namespace", nullable = false)
    public String crNamespace;

    @Column(name = "cr_uid")
    public String crUid;

    @Column(name = "project_id", nullable = false)
    public String projectId;

    @Column(name = "environment", nullable = false)
    public String environment;

    @Column(name = "provider", nullable = false)
    public String provider;

    @Column(name = "state", nullable = false)
    public String state;

    @Column(name = "start_time")
    public ZonedDateTime startTime;

    @Column(name = "completion_time")
    public ZonedDateTime completionTime;

    @Column(name = "db_snapshot_id")
    public String dbSnapshotId;

    @Column(name = "db_snapshot_size_bytes")
    public Long dbSnapshotSize;

    @Column(name = "db_engine")
    public String dbEngine;

    @Column(name = "db_engine_version")
    public String dbEngineVersion;

    @Column(name = "dl_source_bucket")
    public String dlSourceBucket;

    @Column(name = "dl_destination_bucket")
    public String dlDestBucket;

    @Column(name = "dl_destination_prefix")
    public String dlDestPrefix;

    @Column(name = "dl_object_count")
    public Long dlObjectCount;

    @Column(name = "dl_total_size_bytes")
    public Long dlTotalSize;

    @Column(name = "dl_manifest_path")
    public String dlManifestPath;

    @Column(name = "retention_days")
    public Integer retentionDays;

    @Column(name = "archived_at")
    public ZonedDateTime archivedAt;

    @Column(name = "created_at", nullable = false, updatable = false)
    public ZonedDateTime createdAt = ZonedDateTime.now();

    @Column(name = "updated_at", nullable = false)
    public ZonedDateTime updatedAt = ZonedDateTime.now();
}
```

---

### 7. REST API (for Cockpit UI)

**File**: `src/main/java/com/liferay/backup/api/BackupResource.java`

```java
package com.liferay.backup.api;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@Path("/api/v1/backups")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class BackupResource {

    @Inject
    BackupArchiveService archiveService;

    @Inject
    KubernetesClient k8sClient;

    /**
     * List backups for a project
     */
    @GET
    public Response listBackups(
        @QueryParam("projectId") String projectId,
        @QueryParam("environment") String environment,
        @QueryParam("limit") @DefaultValue("50") int limit
    ) {
        var backups = archiveService.findByProject(projectId, environment, limit);
        return Response.ok(backups).build();
    }

    /**
     * Get specific backup
     */
    @GET
    @Path("/{id}")
    public Response getBackup(@PathParam("id") String id) {
        var backup = archiveService.findById(UUID.fromString(id));
        if (backup == null) {
            return Response.status(404).build();
        }
        return Response.ok(backup).build();
    }

    /**
     * Create backup (creates BackupRequest CRD)
     */
    @POST
    public Response createBackup(BackupRequestDTO dto) {
        var backupRequest = new BackupRequest();
        backupRequest.getMetadata().setGenerateName(
            dto.getProjectId() + "-" + dto.getEnvironment() + "-"
        );
        backupRequest.getMetadata().setNamespace(
            "proj-" + dto.getProjectId() + "-" + dto.getEnvironment()
        );
        backupRequest.setSpec(dto.toSpec());

        var created = k8sClient
            .resources(BackupRequest.class)
            .inNamespace(backupRequest.getMetadata().getNamespace())
            .create(backupRequest);

        return Response.status(201)
            .entity(created)
            .build();
    }
}
```

---

## Project Structure

```
lc2-backup-operator/
├── src/
│   ├── main/
│   │   ├── java/com/liferay/backup/
│   │   │   ├── crd/
│   │   │   │   ├── BackupRequest.java
│   │   │   │   ├── BackupRequestSpec.java
│   │   │   │   ├── BackupRequestStatus.java
│   │   │   │   └── ...
│   │   │   ├── reconciler/
│   │   │   │   └── BackupRequestReconciler.java
│   │   │   ├── provider/
│   │   │   │   ├── BackupProvider.java (interface)
│   │   │   │   ├── ProviderFactory.java
│   │   │   │   ├── aws/
│   │   │   │   │   └── AWSBackupProvider.java
│   │   │   │   ├── gcp/
│   │   │   │   │   └── GCPBackupProvider.java
│   │   │   │   ├── azure/
│   │   │   │   │   └── AzureBackupProvider.java
│   │   │   │   └── local/
│   │   │   │       └── LocalBackupProvider.java
│   │   │   ├── argo/
│   │   │   │   ├── ArgoWorkflowService.java
│   │   │   │   └── WorkflowStatus.java
│   │   │   ├── archive/
│   │   │   │   ├── BackupArchiveService.java
│   │   │   │   └── BackupRecord.java
│   │   │   ├── api/
│   │   │   │   └── BackupResource.java
│   │   │   └── manifest/
│   │   │       ├── ManifestGenerator.java
│   │   │       └── BackupManifest.java
│   │   └── resources/
│   │       ├── application.properties
│   │       └── db/migration/
│   │           └── V1__initial_schema.sql
│   └── test/
│       └── java/com/liferay/backup/
│           ├── provider/
│           │   └── AWSBackupProviderTest.java
│           └── reconciler/
│               └── BackupRequestReconcilerTest.java
├── k8s/
│   ├── crd/
│   │   └── backup-request-crd.yaml
│   ├── argo-workflows/
│   │   ├── unified-backup-workflow.yaml
│   │   └── restore-workflow.yaml (Phase 2)
│   ├── deployment/
│   │   ├── operator-deployment.yaml
│   │   ├── postgres-deployment.yaml
│   │   └── service-account.yaml
│   └── samples/
│       └── sample-backup-request.yaml
├── pom.xml
├── Dockerfile
├── docker-compose.yml (local dev)
└── README.md
```

---

## Configuration

**File**: `src/main/resources/application.properties`

```properties
# Quarkus
quarkus.application.name=lc2-backup-operator
quarkus.http.port=8080
quarkus.log.level=INFO
quarkus.log.category."com.liferay".level=DEBUG

# Kubernetes Operator SDK
quarkus.operator-sdk.crd.apply=true
quarkus.operator-sdk.crd.validate=true

# Kubernetes Client
quarkus.kubernetes-client.trust-certs=true

# Database
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=backup
quarkus.datasource.password=${DB_PASSWORD:backup}
quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:backup_history}
quarkus.hibernate-orm.database.generation=validate
quarkus.flyway.migrate-at-start=true

# AWS
aws.region=${AWS_REGION:us-east-1}
aws.endpoint-override=${AWS_ENDPOINT_URL:}

# GCP
gcp.project-id=${GCP_PROJECT:}

# Azure
azure.subscription-id=${AZURE_SUBSCRIPTION_ID:}

# Argo Workflows
argo.namespace=${ARGO_NAMESPACE:argo}

# Metrics
quarkus.micrometer.export.prometheus.enabled=true
```

---

## Deployment

### 1. Build

```bash
# JVM mode
./mvnw clean package

# Native mode (optional, for production)
./mvnw clean package -Pnative

# Docker image
docker build -t liferay/lc2-backup-operator:latest .
```

### 2. Deploy to Kubernetes

```bash
# Apply CRD
kubectl apply -f k8s/crd/backup-request-crd.yaml

# Deploy PostgreSQL
kubectl apply -f k8s/deployment/postgres-deployment.yaml

# Deploy Operator
kubectl apply -f k8s/deployment/operator-deployment.yaml

# Install Argo Workflows (if not already installed)
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.0/install.yaml

# Deploy workflow templates
kubectl apply -f k8s/argo-workflows/unified-backup-workflow.yaml
```

---

## Usage

### Create a Backup (kubectl)

```bash
kubectl apply -f - <<EOF
apiVersion: backup.liferay.cloud/v1alpha1
kind: BackupRequest
metadata:
  name: retail-prd-backup-$(date +%Y%m%d-%H%M%S)
  namespace: proj-retail-prd
spec:
  projectId: retail
  environment: prd
  provider: aws
  database:
    type: snapshot
    instanceId: retail-prd-db
  documentLibrary:
    sourceBucket: retail-prd-liferay-dl
    destinationBucket: liferay-backups
    destinationPrefix: retail/prd/$(date +%Y%m%d)/
    excludeGeneratedFiles: true
  retentionDays: 30
EOF
```

### Create a Backup (REST API)

```bash
curl -X POST http://backup-operator.liferay-system.svc.cluster.local:8080/api/v1/backups \
  -H "Content-Type: application/json" \
  -d '{
    "projectId": "retail",
    "environment": "prd",
    "provider": "aws",
    "database": {
      "type": "snapshot",
      "instanceId": "retail-prd-db"
    },
    "documentLibrary": {
      "sourceBucket": "retail-prd-liferay-dl",
      "destinationBucket": "liferay-backups",
      "destinationPrefix": "retail/prd/20251109/",
      "excludeGeneratedFiles": true
    },
    "retentionDays": 30
  }'
```

### List Backups

```bash
# kubectl
kubectl get backuprequests -n proj-retail-prd

# REST API
curl http://backup-operator.liferay-system.svc.cluster.local:8080/api/v1/backups?projectId=retail&environment=prd
```

---

## Testing Strategy

### Unit Tests
- Test each provider implementation
- Mock AWS/GCP/Azure SDKs
- Test state machine logic

### Integration Tests
- LocalStack for AWS (RDS, S3)
- TestContainers for PostgreSQL
- Kubernetes test framework

### E2E Tests
- k3d cluster with Argo Workflows
- MinIO for S3-compatible storage
- Test all 4 providers (aws/gcp/azure/local)

---

## Timeline & Milestones

### Week 1-4: Foundation
- ✅ Set up Quarkus project
- ✅ Define BackupRequest CRD with managed service status tracking
- ✅ Implement reconciler state machine (Pending → Snapshot → Export → Transfer → Metadata → Succeeded)
- ✅ PostgreSQL schema + archival service
- ✅ Strategy pattern for providers

### Week 5-8: GCP Provider (Reference Implementation)
- Implement GCP provider using managed service APIs:
  - Cloud SQL Snapshot API
  - Cloud SQL Export API (writes directly to GCS!)
  - GCS Transfer Service (with ObjectConditions filtering)
- State polling logic for async operations
- Testing with GCP test project

### Week 9-12: Multi-Cloud + Polish
- Implement AWS provider (RDS Snapshot, RDS Export to S3, S3 Transfer)
- Implement Azure provider (Flexible Server Backup, Blob Transfer)
- Implement Local provider (Argo Workflows fallback for k3d)
- REST API for Cockpit UI
- E2E testing with all 4 providers
- Documentation
- Production deployment

---

## Success Criteria

**Architecture**:
- [x] Cloud-agnostic (works on AWS, GCP, Azure, local)
- [x] Java-based (Quarkus)
- [x] Kubernetes-native (CRD + Operator)
- [x] GitOps-friendly (declarative)
- [x] Multi-tenant isolation
- [x] Hybrid state (CRD + PostgreSQL)
- [x] **Enhanced Hybrid Approach** using managed service APIs

**Critical Requirements (validated by Liferay PaaS team)**:
- [x] **NO data streaming through operator pods** (eliminates OOM errors)
- [x] Database exports write directly to object storage (Cloud SQL Export API, RDS Export to S3)
- [x] Object transfers use native cloud services (GCS Transfer Service, S3 Transfer Manager)
- [x] Liferay-specific filtering (excludeGeneratedFiles saves 30-50TB)
- [x] Dual backup (snapshot for fast restore + dump for portability)
- [x] Metadata/tracker file generation

**Delivery**:
- [ ] 3-month delivery timeline
- [ ] Production-ready
- [ ] Validated against current Liferay PaaS pain points

---

## Open Questions

1. **Native Compilation**: Should we target GraalVM native? (Can defer to Phase 2)
2. **Restore Workflow**: Use managed service Import APIs or custom streaming? (Prefer managed Import APIs for consistency)
3. **Monitoring**: Prometheus + Grafana sufficient? (Yes - track operation durations, failure rates)
4. **Cost Attribution**: How to tag backups for chargeback? (Add to metadata + cloud resource tags)
5. **Rate Limiting**: How to handle managed service API quotas? (Implement exponential backoff, respect quotas)

---

## References

- **Liferay PaaS "Rethinking Backups"** (Iliyan Peychev): Validated that streaming through backup service causes OOM errors; recommended Cloud SQL Export API and GCS Transfer Service
- **GCP Cloud SQL Export API**: https://cloud.google.com/sql/docs/mysql/import-export/import-export-sql
- **GCS Transfer Service**: https://cloud.google.com/storage-transfer-service
- **AWS RDS Export to S3**: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ExportSnapshot.html
- **Quarkus Operator SDK**: https://github.com/quarkiverse/quarkus-operator-sdk

---

**Document Version**: 2.0 (Enhanced Hybrid Approach)
**Last Updated**: 2025-11-10
**Status**: Ready for Implementation
**Next Step**: Kickoff with team, set up Quarkus project, implement GCP provider first
**Key Change from v1.0**: Replaced Argo Workflows execution with managed service APIs to eliminate data streaming
