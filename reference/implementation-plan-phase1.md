# Backup Controller - Phase 1 Implementation Plan

## Overview
**Duration**: Weeks 1-4 (4 weeks)
**Team Size**: 2 engineers
**Goal**: AWS-only MVP with basic backup capability

## Phase 1 Deliverables
1. AWS provider adapter (RDS snapshots + S3 operations)
2. Enhanced BackupRequest CRD
3. Multi-tenant resource isolation (basic)
4. PostgreSQL backup history archival
5. Backup manifest generation
6. LocalStack test environment

---

## Week 1: Foundation & Setup

### Task 1.1: Development Environment Setup
**Owner**: Both engineers
**Duration**: 2 days
**Files to create/modify**:
- `docker-compose.yml` (LocalStack + PostgreSQL)
- `Makefile` (new targets for local dev)
- `.envrc` or `.env.example` (local config)

**Acceptance Criteria**:
- [ ] LocalStack running with S3 + RDS endpoints
- [ ] PostgreSQL database for backup history
- [ ] k3d cluster with backup-controller installed
- [ ] AWS SDK configured for LocalStack
- [ ] `make local-up` starts entire environment

**Implementation Notes**:
```yaml
# docker-compose.yml snippet
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,rds
      - AWS_DEFAULT_REGION=us-east-1

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=backup_history
      - POSTGRES_USER=backup
      - POSTGRES_PASSWORD=backup
```

---

### Task 1.2: Database Schema for Backup History
**Owner**: Engineer 1
**Duration**: 1 day
**Files to create**:
- `internal/database/schema.sql`
- `internal/database/migrations/001_initial.sql`
- `internal/database/client.go`

**Acceptance Criteria**:
- [ ] SQL schema defined for backup_requests table
- [ ] Migration tooling integrated (golang-migrate or similar)
- [ ] Database client with connection pooling
- [ ] Health check for PostgreSQL connectivity

**Schema Design**:
```sql
CREATE TABLE backup_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cr_name VARCHAR(255) NOT NULL,
    cr_namespace VARCHAR(255) NOT NULL,
    project_id VARCHAR(100) NOT NULL,
    environment VARCHAR(50) NOT NULL,

    -- Backup metadata
    state VARCHAR(50) NOT NULL,
    start_time TIMESTAMPTZ,
    completion_time TIMESTAMPTZ,

    -- Database backup info
    db_instance_id VARCHAR(255),
    db_snapshot_id VARCHAR(255),
    db_snapshot_size_bytes BIGINT,

    -- Document library backup info
    dl_source_bucket VARCHAR(255),
    dl_destination_bucket VARCHAR(255),
    dl_destination_prefix VARCHAR(500),
    dl_object_count BIGINT,
    dl_total_size_bytes BIGINT,
    dl_manifest_path VARCHAR(500),

    -- Tracking
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    archived_at TIMESTAMPTZ,

    CONSTRAINT unique_cr UNIQUE (cr_namespace, cr_name)
);

CREATE INDEX idx_project_env ON backup_requests(project_id, environment);
CREATE INDEX idx_state ON backup_requests(state);
CREATE INDEX idx_created_at ON backup_requests(created_at DESC);
```

---

### Task 1.3: Enhanced CRD Definition
**Owner**: Engineer 2
**Duration**: 2 days
**Files to modify**:
- `api/backup/v1alpha1/backuprequest_types.go` (NEW)
- `config/crd/bases/backup.liferay.cloud_backuprequests.yaml` (NEW)
- `api/backup/v1alpha1/common_types.go` (enhance)

**Acceptance Criteria**:
- [ ] BackupRequest CRD supports AWS + GCP
- [ ] Provider field in spec (aws|gcp)
- [ ] Validation webhooks for required fields
- [ ] Status conditions follow Kubernetes standards
- [ ] `make manifests` generates valid CRD YAML

**CRD Structure**:
```go
// api/backup/v1alpha1/backuprequest_types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// BackupRequestSpec defines the desired state of BackupRequest
type BackupRequestSpec struct {
    // +kubebuilder:validation:Required
    ProjectID string `json:"projectId"`

    // +kubebuilder:validation:Required
    Environment string `json:"environment"`

    // +kubebuilder:validation:Enum=aws;gcp
    // +kubebuilder:default=aws
    Provider string `json:"provider,omitempty"`

    // +kubebuilder:validation:Required
    Database DatabaseBackupSpec `json:"database"`

    // +kubebuilder:validation:Required
    DocumentLibrary DocumentLibraryBackupSpec `json:"documentLibrary"`
}

type DatabaseBackupSpec struct {
    // +kubebuilder:validation:Enum=snapshot;dump
    // +kubebuilder:default=snapshot
    Type string `json:"type,omitempty"`

    // +kubebuilder:validation:Required
    InstanceID string `json:"instanceId"`

    // +kubebuilder:validation:Pattern=`^\d+[dhm]$`
    // +kubebuilder:default="7d"
    Retention string `json:"retention,omitempty"`
}

type DocumentLibraryBackupSpec struct {
    // +kubebuilder:validation:Required
    SourceBucket string `json:"sourceBucket"`

    // +kubebuilder:validation:Required
    DestinationBucket string `json:"destinationBucket"`

    DestinationPrefix string `json:"destinationPrefix,omitempty"`

    ExcludeGeneratedFiles bool `json:"excludeGeneratedFiles,omitempty"`
}

// BackupRequestStatus defines the observed state of BackupRequest
type BackupRequestStatus struct {
    // +kubebuilder:validation:Enum=Pending;InProgress;Succeeded;Failed
    State string `json:"state,omitempty"`

    StartTime *metav1.Time `json:"startTime,omitempty"`
    CompletionTime *metav1.Time `json:"completionTime,omitempty"`

    Database *DatabaseBackupStatus `json:"database,omitempty"`
    DocumentLibrary *DocumentLibraryBackupStatus `json:"documentLibrary,omitempty"`

    // Conditions follow standard Kubernetes condition pattern
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

type DatabaseBackupStatus struct {
    SnapshotID string `json:"snapshotId,omitempty"`
    Size string `json:"size,omitempty"`
}

type DocumentLibraryBackupStatus struct {
    ObjectCount int64 `json:"objectCount,omitempty"`
    TotalSize string `json:"totalSize,omitempty"`
    ManifestPath string `json:"manifestPath,omitempty"`
}
```

---

## Week 2: AWS Provider Implementation

### Task 2.1: Provider Interface Definition
**Owner**: Engineer 1
**Duration**: 1 day
**Files to create**:
- `internal/provider/interface.go`
- `internal/provider/types.go`

**Acceptance Criteria**:
- [ ] Provider interface defined
- [ ] Request/Response types for all operations
- [ ] Error types for provider failures
- [ ] Context support for cancellation

**Interface Design**:
```go
// internal/provider/interface.go
package provider

import (
    "context"
)

type Provider interface {
    // Database operations
    CreateDatabaseSnapshot(ctx context.Context, req DatabaseSnapshotRequest) (*DatabaseSnapshotResult, error)
    GetDatabaseSnapshotStatus(ctx context.Context, snapshotID string) (*SnapshotStatus, error)

    // Object storage operations
    CopyObjectStorage(ctx context.Context, req ObjectStorageCopyRequest) (*ObjectStorageCopyResult, error)

    // Validation
    ValidateCredentials(ctx context.Context) error
}

type DatabaseSnapshotRequest struct {
    InstanceID string
    SnapshotID string  // User-provided or generated
    Tags map[string]string
}

type DatabaseSnapshotResult struct {
    SnapshotID string
    Status string  // creating|available|failed
    SizeBytes int64
    StartTime string
}

type ObjectStorageCopyRequest struct {
    SourceBucket string
    SourcePrefix string
    DestinationBucket string
    DestinationPrefix string
    ExcludePatterns []string
    ManifestPath string  // Where to write manifest
}

type ObjectStorageCopyResult struct {
    ObjectCount int64
    TotalSizeBytes int64
    ManifestPath string
}
```

---

### Task 2.2: AWS Provider Implementation
**Owner**: Engineer 2
**Duration**: 3 days
**Files to create**:
- `internal/provider/aws/provider.go`
- `internal/provider/aws/rds.go`
- `internal/provider/aws/s3.go`
- `internal/provider/aws/config.go`

**Acceptance Criteria**:
- [ ] AWS SDK v2 integrated
- [ ] RDS snapshot creation with tagging
- [ ] S3 object copy with manifest generation
- [ ] Error handling with retries
- [ ] Unit tests with mocked AWS clients
- [ ] LocalStack integration tests

**Implementation Notes**:
```go
// internal/provider/aws/provider.go
package aws

import (
    "context"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/rds"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

type AWSProvider struct {
    rdsClient *rds.Client
    s3Client  *s3.Client
    region    string
}

func NewProvider(ctx context.Context, region string) (*AWSProvider, error) {
    cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion(region))
    if err != nil {
        return nil, err
    }

    return &AWSProvider{
        rdsClient: rds.NewFromConfig(cfg),
        s3Client:  s3.NewFromConfig(cfg),
        region:    region,
    }, nil
}
```

---

### Task 2.3: S3 Manifest Generator
**Owner**: Engineer 1
**Duration**: 2 days
**Files to create**:
- `internal/manifest/generator.go`
- `internal/manifest/types.go`

**Acceptance Criteria**:
- [ ] JSON manifest with file listing
- [ ] SHA256 checksums for verification
- [ ] Metadata (timestamp, project, environment)
- [ ] Compressed manifest support (gzip)

**Manifest Format**:
```json
{
  "version": "1.0",
  "timestamp": "2025-11-09T14:30:00Z",
  "project": "retail",
  "environment": "prd",
  "backup_id": "retail-prd-backup-20251109-143000",
  "database": {
    "snapshot_id": "rds:us-east-1:retail-prd-20251109-143000",
    "instance_id": "retail-prd-db",
    "size_bytes": 48586496000,
    "engine": "postgres",
    "engine_version": "15.4"
  },
  "document_library": {
    "source_bucket": "liferay-retail-prd-dl",
    "destination_bucket": "liferay-backups",
    "destination_prefix": "retail/prd/20251109-143000/dl",
    "object_count": 12453,
    "total_size_bytes": 135239680000,
    "files": [
      {
        "key": "document_library/2024/01/12345.pdf",
        "size_bytes": 1048576,
        "etag": "abc123...",
        "last_modified": "2025-11-08T10:15:00Z"
      }
    ]
  },
  "checksum": "sha256:a1b2c3..."
}
```

---

## Week 3: Controller Logic & Reconciliation

### Task 3.1: BackupRequest Controller
**Owner**: Both engineers
**Duration**: 3 days
**Files to create**:
- `internal/controller/backuprequest_controller.go`
- `internal/controller/backuprequest_controller_test.go`

**Acceptance Criteria**:
- [ ] Reconcile loop handles BackupRequest CRD
- [ ] State machine: Pending → InProgress → Succeeded/Failed
- [ ] Provider selection based on spec.provider
- [ ] Database snapshot → Document library copy (sequential)
- [ ] Status updates with conditions
- [ ] Event recording for user visibility

**State Machine**:
```
Pending
   ├─> InProgress (Database)
   │      ├─> InProgress (Document Library)
   │      │      └─> Succeeded
   │      └─> Failed (Database snapshot failed)
   └─> Failed (Validation failed)
```

**Controller Logic**:
```go
// internal/controller/backuprequest_controller.go
func (r *BackupRequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    var backupReq backupv1alpha1.BackupRequest
    if err := r.Get(ctx, req.NamespacedName, &backupReq); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // State machine
    switch backupReq.Status.State {
    case "":
        // Initialize
        return r.initializeBackup(ctx, &backupReq)
    case "Pending":
        // Start database backup
        return r.startDatabaseBackup(ctx, &backupReq)
    case "InProgress":
        // Check if database backup complete, then start DL backup
        return r.processDatabaseBackup(ctx, &backupReq)
    case "DatabaseComplete":
        // Start document library backup
        return r.startDocumentLibraryBackup(ctx, &backupReq)
    case "Succeeded", "Failed":
        // Archive to PostgreSQL if not already done
        return r.archiveBackup(ctx, &backupReq)
    }

    return ctrl.Result{}, nil
}
```

---

### Task 3.2: Multi-Tenant Isolation (Semaphore)
**Owner**: Engineer 1
**Duration**: 2 days
**Files to create**:
- `internal/semaphore/semaphore.go`
- `internal/semaphore/semaphore_test.go`

**Acceptance Criteria**:
- [ ] In-memory semaphore per project
- [ ] Max 2 concurrent backups per project
- [ ] Requeue with backoff if semaphore full
- [ ] Metrics for queue depth per project

**Implementation**:
```go
// internal/semaphore/semaphore.go
package semaphore

import (
    "context"
    "sync"
)

type ProjectSemaphore struct {
    mu sync.Mutex
    semaphores map[string]chan struct{}
    maxConcurrent int
}

func New(maxConcurrent int) *ProjectSemaphore {
    return &ProjectSemaphore{
        semaphores: make(map[string]chan struct{}),
        maxConcurrent: maxConcurrent,
    }
}

func (ps *ProjectSemaphore) Acquire(ctx context.Context, projectID string) error {
    ps.mu.Lock()
    sem, ok := ps.semaphores[projectID]
    if !ok {
        sem = make(chan struct{}, ps.maxConcurrent)
        ps.semaphores[projectID] = sem
    }
    ps.mu.Unlock()

    select {
    case sem <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (ps *ProjectSemaphore) Release(projectID string) {
    ps.mu.Lock()
    sem, ok := ps.semaphores[projectID]
    ps.mu.Unlock()

    if ok {
        <-sem
    }
}
```

---

### Task 3.3: PostgreSQL Archival Logic
**Owner**: Engineer 2
**Duration**: 1 day
**Files to create**:
- `internal/archiver/archiver.go`

**Acceptance Criteria**:
- [ ] Async archival after backup completes
- [ ] Upsert based on CR namespace+name
- [ ] Retry on database connection failures
- [ ] Metrics for archival lag

---

## Week 4: Testing & Integration

### Task 4.1: End-to-End Tests
**Owner**: Both engineers
**Duration**: 2 days
**Files to create**:
- `test/e2e/backup_test.go`
- `test/e2e/suite_test.go`

**Acceptance Criteria**:
- [ ] Create BackupRequest → Verify RDS snapshot created
- [ ] Create BackupRequest → Verify S3 objects copied
- [ ] Create BackupRequest → Verify manifest generated
- [ ] Create BackupRequest → Verify PostgreSQL record
- [ ] Test concurrent backups (semaphore enforcement)
- [ ] Test backup failure scenarios

**Test Structure**:
```go
// test/e2e/backup_test.go
var _ = Describe("Backup Controller", func() {
    Context("When creating a BackupRequest", func() {
        It("Should create RDS snapshot and copy S3 objects", func() {
            ctx := context.Background()

            // Create test BackupRequest
            backupReq := &backupv1alpha1.BackupRequest{
                ObjectMeta: metav1.ObjectMeta{
                    Name: "test-backup",
                    Namespace: "test-ns",
                },
                Spec: backupv1alpha1.BackupRequestSpec{
                    ProjectID: "test-project",
                    Environment: "dev",
                    Provider: "aws",
                    Database: backupv1alpha1.DatabaseBackupSpec{
                        Type: "snapshot",
                        InstanceID: "test-db-instance",
                    },
                    DocumentLibrary: backupv1alpha1.DocumentLibraryBackupSpec{
                        SourceBucket: "test-source",
                        DestinationBucket: "test-dest",
                    },
                },
            }

            Expect(k8sClient.Create(ctx, backupReq)).To(Succeed())

            // Wait for backup to complete
            Eventually(func() string {
                var updated backupv1alpha1.BackupRequest
                k8sClient.Get(ctx, client.ObjectKeyFromObject(backupReq), &updated)
                return updated.Status.State
            }, timeout, interval).Should(Equal("Succeeded"))

            // Verify RDS snapshot
            // Verify S3 objects
            // Verify manifest
            // Verify PostgreSQL record
        })
    })
})
```

---

### Task 4.2: Documentation
**Owner**: Engineer 1
**Duration**: 1 day
**Files to create/modify**:
- `docs/phase1-architecture.md`
- `docs/aws-provider.md`
- `docs/local-development.md`
- `README.md` (update)

**Acceptance Criteria**:
- [ ] Architecture diagram (Mermaid or ASCII)
- [ ] AWS provider configuration guide
- [ ] Local development setup guide
- [ ] Example BackupRequest manifests

---

### Task 4.3: Monitoring & Metrics
**Owner**: Engineer 2
**Duration**: 1 day
**Files to create**:
- `internal/metrics/metrics.go`
- `config/prometheus/servicemonitor.yaml`

**Acceptance Criteria**:
- [ ] Prometheus metrics exported
- [ ] Metrics: backup_requests_total (by state, project)
- [ ] Metrics: backup_duration_seconds (histogram)
- [ ] Metrics: semaphore_queue_depth (gauge)
- [ ] ServiceMonitor for Prometheus scraping

**Metrics**:
```go
// internal/metrics/metrics.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    BackupRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "backup_requests_total",
            Help: "Total number of backup requests by state and project",
        },
        []string{"project_id", "environment", "state"},
    )

    BackupDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "backup_duration_seconds",
            Help: "Backup duration in seconds",
            Buckets: prometheus.ExponentialBuckets(60, 2, 10), // 1m to ~17h
        },
        []string{"project_id", "environment"},
    )

    SemaphoreQueueDepth = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "backup_semaphore_queue_depth",
            Help: "Current queue depth per project",
        },
        []string{"project_id"},
    )
)

func init() {
    metrics.Registry.MustRegister(
        BackupRequestsTotal,
        BackupDuration,
        SemaphoreQueueDepth,
    )
}
```

---

## Phase 1 Acceptance Criteria

### Functional Requirements
- [ ] BackupRequest CRD can be created for AWS provider
- [ ] RDS snapshot created with correct tags
- [ ] S3 objects copied from source to destination bucket
- [ ] Manifest JSON generated and uploaded to S3
- [ ] Backup history archived to PostgreSQL
- [ ] Status conditions updated correctly
- [ ] Max 2 concurrent backups per project enforced

### Non-Functional Requirements
- [ ] LocalStack environment fully functional
- [ ] Unit test coverage > 80%
- [ ] E2E tests pass consistently
- [ ] Prometheus metrics exported
- [ ] Documentation complete

### Performance Targets
- [ ] Backup 1GB database in < 5 minutes
- [ ] Backup 100GB document library in < 30 minutes
- [ ] Controller reconciliation latency < 5 seconds

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| AWS SDK LocalStack compatibility issues | Medium | Test early, use official LocalStack images |
| S3 copy performance for large files | High | Implement pagination, consider multipart upload |
| PostgreSQL connection pool exhaustion | Medium | Set max connections, implement health checks |
| Semaphore not persisted across restarts | Low | Document as known limitation, add in Phase 2 |
| CRD validation webhook complexity | Medium | Start with basic validation, enhance incrementally |

---

## Daily Standup Format

**What did you do yesterday?**
**What will you do today?**
**Any blockers?**

---

## Definition of Done

A task is considered done when:
1. Code is written and follows Go best practices
2. Unit tests written (coverage > 80%)
3. Code reviewed and approved
4. E2E tests updated (if applicable)
5. Documentation updated
6. Metrics added (if applicable)
7. Merged to main branch

---

## File Structure (Expected by End of Phase 1)

```
backup-controller/
├── api/
│   └── backup/
│       └── v1alpha1/
│           ├── backuprequest_types.go  (NEW)
│           └── common_types.go         (MODIFIED)
├── internal/
│   ├── controller/
│   │   └── backuprequest_controller.go (NEW)
│   ├── provider/
│   │   ├── interface.go                (NEW)
│   │   ├── types.go                    (NEW)
│   │   └── aws/
│   │       ├── provider.go             (NEW)
│   │       ├── rds.go                  (NEW)
│   │       └── s3.go                   (NEW)
│   ├── database/
│   │   ├── schema.sql                  (NEW)
│   │   └── client.go                   (NEW)
│   ├── manifest/
│   │   └── generator.go                (NEW)
│   ├── semaphore/
│   │   └── semaphore.go                (NEW)
│   ├── archiver/
│   │   └── archiver.go                 (NEW)
│   └── metrics/
│       └── metrics.go                  (NEW)
├── test/
│   └── e2e/
│       └── backup_test.go              (NEW)
├── docs/
│   ├── phase1-architecture.md          (NEW)
│   ├── aws-provider.md                 (NEW)
│   └── local-development.md            (NEW)
├── docker-compose.yml                  (NEW)
└── Makefile                            (MODIFIED)
```

---

## Next Steps (Post Phase 1)

After Phase 1 completion, proceed to:
- **Phase 2**: Restore capability + Multi-tenant ResourceQuotas
- **Phase 3**: Operational excellence (DR, monitoring, alerting)

---

## Questions for User Review

1. Should we support GCP in Phase 1 or strictly AWS-only?
2. PostgreSQL archival - should we support external PostgreSQL or bundled?
3. Manifest format - any additional fields needed?
4. LocalStack sufficient for testing or need real AWS dev account?
