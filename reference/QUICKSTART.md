# Backup Controller - Quick Start Guide

Get up and running with the LC2 Backup Controller in 5 minutes.

---

## Prerequisites

- Docker & Docker Compose
- Go 1.21+
- kubectl
- k3d (optional, for local Kubernetes)

---

## 1. Clone and Setup (1 minute)

```bash
cd /home/misael/lc2-backup/backup-controller

# Install Go dependencies
go mod download

# Install development tools
make install-tools
```

---

## 2. Start Local Environment (2 minutes)

```bash
# Start LocalStack (AWS emulator) + PostgreSQL
make local-up

# Verify services are healthy
make local-status
```

**Expected Output**:
```
NAME                              STATUS   PORTS
backup-controller-localstack      Up       0.0.0.0:4566->4566/tcp
backup-controller-postgres        Up       0.0.0.0:5432->5432/tcp
backup-controller-localstack-init Exit 0
```

---

## 3. Configure AWS CLI for LocalStack (30 seconds)

```bash
# Set environment variables
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# Verify S3 buckets were created
aws s3 ls
```

**Expected Output**:
```
2025-11-09 14:30:00 liferay-backups
2025-11-09 14:30:00 liferay-test-dl
```

---

## 4. Generate CRDs (30 seconds)

```bash
# Generate Kubernetes manifests
make manifests

# Verify BackupRequest CRD was generated
ls -la config/crd/bases/backup.liferay.cloud_backuprequests.yaml
```

---

## 5. Run Controller Locally (30 seconds)

```bash
# Option 1: Run directly (no k8s cluster needed for testing)
make run

# Option 2: With local k3d cluster
k3d cluster create backup-test
make install  # Install CRDs
make run
```

---

## 6. Create a Test Backup (1 minute)

```bash
# Apply a test BackupRequest
kubectl apply -f - <<EOF
apiVersion: backup.liferay.cloud/v1alpha1
kind: BackupRequest
metadata:
  name: test-backup
  namespace: default
spec:
  projectId: test-project
  environment: dev
  provider: aws
  database:
    type: snapshot
    instanceId: test-db-instance
  documentLibrary:
    sourceBucket: liferay-test-dl
    destinationBucket: liferay-backups
    destinationPrefix: test/dev/$(date +%Y%m%d)/
    excludeGeneratedFiles: true
  retentionDays: 7
EOF

# Watch the backup progress
kubectl get backuprequests -w
```

**Expected Output**:
```
NAME          PROJECT        ENVIRONMENT   STATE       AGE
test-backup   test-project   dev           Pending     1s
test-backup   test-project   dev           InProgress  5s
test-backup   test-project   dev           Succeeded   45s
```

---

## 7. Verify Backup Results

### Check RDS Snapshot (LocalStack)
```bash
aws rds describe-db-snapshots \
  --db-snapshot-identifier test-backup-snapshot
```

### Check S3 Objects
```bash
# List copied objects
aws s3 ls s3://liferay-backups/test/dev/$(date +%Y%m%d)/ --recursive

# Download manifest
aws s3 cp s3://liferay-backups/test/dev/$(date +%Y%m%d)/manifest.json -
```

### Check PostgreSQL History
```bash
# Connect to database
make db-shell

# Query backup history
SELECT project_id, environment, state, db_snapshot_id, dl_object_count
FROM backup_requests
WHERE project_id = 'test-project';
```

---

## 8. View Logs and Metrics

### Controller Logs
```bash
# If running with `make run`
# Logs appear in terminal

# If running in k8s cluster
kubectl logs -f deployment/backup-controller-manager -n backup-system
```

### LocalStack Logs
```bash
make local-logs
```

### Database Queries
```bash
# Recent successful backups
make db-shell
SELECT * FROM recent_successful_backups LIMIT 10;

# Backup summary by project
SELECT * FROM backup_summary_by_project;
```

---

## Common Tasks

### Run Tests
```bash
# Unit tests
make test

# Specific package
go test ./internal/provider/aws/... -v

# With coverage
make test-coverage
```

### Regenerate Code
```bash
# After modifying CRD types
make generate-all
```

### Clean Up
```bash
# Stop local services
make local-down

# Remove all data
make local-clean

# Delete k3d cluster (if created)
k3d cluster delete backup-test
```

---

## Troubleshooting

### LocalStack Not Starting
```bash
# Check Docker is running
docker ps

# View LocalStack logs
docker-compose logs localstack

# Restart LocalStack
make local-down && make local-up
```

### PostgreSQL Connection Issues
```bash
# Check PostgreSQL is healthy
docker-compose exec postgres pg_isready -U backup

# View PostgreSQL logs
docker-compose logs postgres

# Reset database
make local-clean && make local-up
```

### Controller Not Reconciling
```bash
# Check CRD is installed
kubectl get crd | grep backuprequest

# Check controller logs
kubectl logs -f deployment/backup-controller-manager

# Describe the BackupRequest
kubectl describe backuprequest test-backup
```

### AWS CLI Commands Failing
```bash
# Verify environment variables
echo $AWS_ENDPOINT_URL
echo $AWS_ACCESS_KEY_ID

# Test LocalStack connectivity
curl http://localhost:4566/_localstack/health

# Re-export variables
source <(make aws-local)
```

---

## Development Workflow

### 1. Make Code Changes
```bash
# Edit files in internal/, api/, etc.
vim internal/provider/aws/rds.go
```

### 2. Run Code Generation
```bash
make generate-all
```

### 3. Run Tests
```bash
make test
```

### 4. Run Locally
```bash
make run
```

### 5. Test with Real CR
```bash
kubectl apply -f config/samples/backup_v1alpha1_backuprequest.yaml
```

---

## Example Workflows

### Backup a Postgres Database
```yaml
apiVersion: backup.liferay.cloud/v1alpha1
kind: BackupRequest
metadata:
  name: prod-db-backup
spec:
  projectId: ecommerce
  environment: prd
  provider: aws
  database:
    type: snapshot
    instanceId: ecommerce-prd-postgres
  documentLibrary:
    sourceBucket: ecommerce-prd-liferay-dl
    destinationBucket: ecommerce-backups
    destinationPrefix: prd/20251109/
  retentionDays: 30
```

### Backup with Excluded Files
```yaml
apiVersion: backup.liferay.cloud/v1alpha1
kind: BackupRequest
metadata:
  name: dev-backup-no-thumbs
spec:
  projectId: blog
  environment: dev
  database:
    type: snapshot
    instanceId: blog-dev-db
  documentLibrary:
    sourceBucket: blog-dev-dl
    destinationBucket: blog-backups
    destinationPrefix: dev/backup-$(date +%Y%m%d)/
    excludeGeneratedFiles: true  # Excludes thumbnails, previews, etc.
```

---

## Next Steps

1. **Read the full implementation plan**: `implementation-plan-phase1.md`
2. **Review requirements**: `backup-requirements-refined.md`
3. **Check implementation status**: `IMPLEMENTATION_STATUS.md`
4. **Implement the controller**: Start with `internal/controller/backuprequest_controller.go`

---

## Useful Commands

```bash
# List all make targets
make help

# Watch BackupRequests
kubectl get br -w

# Get BackupRequest details
kubectl get br test-backup -o yaml

# Delete BackupRequest
kubectl delete br test-backup

# View all CRDs
kubectl get crd | grep backup.liferay.cloud

# Describe BackupRequest CRD
kubectl explain backuprequest.spec
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │          Backup Controller (Reconciler)              │   │
│  │                                                       │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌──────────┐ │   │
│  │  │ Semaphore   │    │  Archiver   │    │ Metrics  │ │   │
│  │  │ (max 2/proj)│    │ (PostgreSQL)│    │(Prometheus)│ │
│  │  └─────────────┘    └─────────────┘    └──────────┘ │   │
│  │                                                       │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │         Provider Interface                     │  │   │
│  │  │  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   │  │   │
│  │  │  │ AWS  │   │ GCP  │   │Azure │   │Local │   │  │   │
│  │  │  └──────┘   └──────┘   └──────┘   └──────┘   │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              BackupRequest CRD                        │   │
│  │  metadata:                                            │   │
│  │    name: test-backup                                  │   │
│  │  spec:                                                │   │
│  │    projectId: test                                    │   │
│  │    database: {...}                                    │   │
│  │    documentLibrary: {...}                             │   │
│  │  status:                                              │   │
│  │    state: Succeeded                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
          │                              │
          ▼                              ▼
   ┌──────────────┐             ┌────────────────┐
   │ LocalStack   │             │  PostgreSQL    │
   │ (AWS Mock)   │             │  (History DB)  │
   │  - RDS       │             │                │
   │  - S3        │             │ backup_requests│
   └──────────────┘             └────────────────┘
```

---

**Ready to contribute? Start coding!**

For questions, check:
- Implementation plan: `implementation-plan-phase1.md`
- Requirements: `backup-requirements-refined.md`
- Status: `IMPLEMENTATION_STATUS.md`
