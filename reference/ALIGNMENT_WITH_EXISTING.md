# LC2 Backup - Alignment with Existing Liferay Cloud

## Purpose
This document analyzes how LC2 backup implementation aligns with existing Liferay Cloud (LCP) backup patterns and identifies migration paths.

---

## Current Liferay Cloud Backup Architecture

### GCP-Based Approach (Liferay PaaS)

#### Storage Strategies

**SFS (Simple File System)**:
- `project.metadata.documentLibraryStore = simpleFileSystem`
- Version 1: Timestamp-based folders
- Version 2: UUID-based with tracker files

**GCS (Google Cloud Storage)**:
- `project.metadata.documentLibraryStore = gcs`
- Version 1: Timestamp-based buckets
- Version 2: UUID-based with tracker files

#### Backup Components

1. **Document Library Backups**
   - SFS v1: `${bucket}/${env}/volume/*.tgz`
   - SFS v2: `${bucket}/document-library/${uuid}.tgz`
   - GCS v1: `backup-${env}-${timestamp}/`
   - GCS v2: `${bucket}/document-library/${uuid}/`

2. **Database Dumps**
   - Location: `${bucket}/database-dump/${uuid}.gz`
   - Format: Compressed SQL dumps
   - Tool: `pg_dump` or `mysqldump`

3. **Database Snapshots**
   - Service: Cloud SQL
   - Management: GCP Console > SQL > Backups
   - Distributed: Across database-1 and database-2 instances

4. **Tracker Files** (v2 only)
   - Location: `${bucket}/backup-metadata/${backupId}.json`
   - Format: JSON with backup metadata
   - Purpose: Index for backups, queryable metadata

---

### AWS-Based Approach (Cloud Native)

**Current CN Implementation**:
- Uses **AWS Backup Service** (managed service)
- Recovery points stored automatically by AWS
- Terraform modules for infrastructure
- Argo Workflows for restore orchestration
- IAM roles with assumed role pattern

**Infrastructure**:
```
cloud/terraform/aws/backup/
├── backup-vault/          # AWS Backup vault
├── backup-plan/           # Backup schedule & retention
└── backup-restore-workflow/  # Argo Workflows infrastructure
```

**Restore Workflow**:
- Argo Workflows ClusterWorkflowTemplate
- Git-based automation (commits to liferay-portal)
- Artifact storage in S3
- Shell scripts + official CLI images (terraform, kubectl, awscli)

---

## LC2 Backup Design Decisions

### ✅ Aligned with Existing Patterns

1. **Tracker Files → Manifest Files**
   - LCP: `backup-metadata/${backupId}.json`
   - LC2: `${projectId}/${env}/${timestamp}/manifest.json`
   - **Alignment**: Same concept, slightly different naming

2. **Separation of Concerns**
   - LCP: Separate folders for DL, DB dumps, DB snapshots
   - LC2: Same pattern - snapshots referenced by ARN, DL in prefix
   - **Alignment**: Perfect

3. **UUID/Timestamp-based Organization**
   - LCP: Uses UUIDs or timestamps
   - LC2: Uses timestamps for folders: `retail/prd/20251109/`
   - **Alignment**: Compatible

4. **Database Snapshots**
   - LCP: Cloud SQL snapshots
   - LC2: RDS snapshots
   - **Alignment**: Same pattern, different provider

5. **Cloud Provider Abstraction**
   - LCP: Supports GCP, AWS (via CN)
   - LC2: Provider interface for AWS/GCP/Azure
   - **Alignment**: LC2 is more abstract

---

### ⚠️ Differences to Consider

1. **Backup Service Architecture**
   - LCP: Per-customer Node.js service + AWS Backup Service (CN)
   - LC2: Centralized Kubernetes controller
   - **Impact**: LC2 is multi-tenant, more scalable

2. **Restore Orchestration**
   - LCP: Argo Workflows (CN only)
   - LC2: TBD (Phase 2) - could use Argo or custom controller
   - **Recommendation**: Consider Argo Workflows for LC2 restore

3. **Backup Metadata Storage**
   - LCP: Tracker files in GCS
   - LC2: PostgreSQL + S3 manifests (hybrid)
   - **Impact**: LC2 has richer querying (PostgreSQL)

4. **State Management**
   - LCP: Implicit (AWS Backup Service manages state)
   - LC2: Explicit (Kubernetes CRD status + PostgreSQL)
   - **Impact**: LC2 is more transparent

---

## Migration Path: GCP → AWS

### Phase 1: Backup Creation
**Goal**: Create LC2 backups that are compatible with LCP patterns

#### Manifest Format Alignment
Update LC2 manifest to include LCP-compatible fields:

```json
{
  "version": "2.0",
  "backupId": "retail-prd-backup-20251109-143000",
  "timestamp": "2025-11-09T14:30:00Z",
  "projectId": "retail",
  "environment": "prd",

  // LCP compatibility
  "lcp": {
    "trackerFileVersion": "v2",
    "storageStrategy": "gcs",  // or "sfs"
    "backupType": "automatic"   // or "manual"
  },

  // LC2 fields
  "database": {
    "snapshotId": "arn:aws:rds:...",
    "type": "snapshot",
    "engine": "postgres"
  },
  "documentLibrary": {
    "objectCount": 12453,
    "files": [...]
  }
}
```

#### Storage Path Alignment
```
LCP GCS:   gs://${bucket}/backup-metadata/${backupId}.json
           gs://${bucket}/document-library/${uuid}.tgz

LC2 S3:    s3://${bucket}/${projectId}/${env}/${timestamp}/manifest.json
           s3://${bucket}/${projectId}/${env}/${timestamp}/dl/file1.pdf
```

**Recommendation**: Add LCP-compatible path option:
```
s3://${bucket}/backup-metadata/${backupId}.json  (symlink to manifest)
```

---

### Phase 2: Restore (Future)

#### Option 1: Argo Workflows (Recommended)
**Pros**:
- Aligns with CN implementation
- Mature, battle-tested
- Excellent visualization
- Extensible (add custom steps)

**Cons**:
- Additional dependency (Argo operator)
- Learning curve for team

**Implementation**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: lc2-backup-restore
spec:
  templates:
  - name: restore-database
    script:
      image: amazon/aws-cli
      command: [sh]
      source: |
        # Restore RDS snapshot
        aws rds restore-db-instance-from-db-snapshot ...

  - name: restore-document-library
    script:
      image: amazon/aws-cli
      command: [sh]
      source: |
        # Copy S3 objects back
        aws s3 sync s3://backup-bucket/... s3://runtime-bucket/...
```

#### Option 2: Custom Controller (Alternative)
**Pros**:
- Consistent with backup approach
- No additional dependencies
- Full control

**Cons**:
- More code to maintain
- Re-inventing orchestration

---

## Recommended Changes to LC2 Implementation

### 1. Update Manifest Format ✅ (Minor)
Add LCP compatibility fields to `internal/provider/types.go`:

```go
type BackupManifest struct {
    Version     string
    BackupID    string
    Timestamp   time.Time
    ProjectID   string
    Environment string

    // LCP compatibility
    LCP *LCPCompatibility `json:"lcp,omitempty"`

    // LC2 fields
    Database        *DatabaseBackupManifest
    DocumentLibrary *DocumentLibraryBackupManifest
    Checksum        string
}

type LCPCompatibility struct {
    TrackerFileVersion string `json:"trackerFileVersion"`
    StorageStrategy    string `json:"storageStrategy"`
    BackupType         string `json:"backupType"`
}
```

### 2. Add Metadata Path Symlink (Nice-to-Have)
When uploading manifest, also create LCP-compatible path:
```go
// Upload to LC2 path
s3.Upload(bucket, "retail/prd/20251109/manifest.json", manifest)

// Also upload to LCP path for compatibility
s3.Upload(bucket, "backup-metadata/retail-prd-backup-20251109.json", manifest)
```

### 3. PostgreSQL Schema Enhancement ✅ (Already Done)
Our PostgreSQL schema already tracks all necessary metadata - no changes needed.

### 4. Consider Argo Workflows for Phase 2 Restore
Add to `implementation-plan-phase2.md`:
- Evaluate Argo Workflows vs custom controller
- Design RestoreRequest CRD (similar to BackupRequest)
- Implement restore orchestration

---

## API Compatibility

### LCP Backup API (Reference)
```
POST /projects/{projectId}/backups
GET  /projects/{projectId}/backups
GET  /projects/{projectId}/backups/{backupId}
POST /projects/{projectId}/restores
```

### LC2 Backup API (Kubernetes-Native)
```bash
# Create backup
kubectl apply -f backuprequest.yaml

# List backups
kubectl get backuprequests -n proj-retail-prd

# Get backup details
kubectl get backuprequest retail-prd-backup-20251109 -o yaml

# Future: Create restore
kubectl apply -f restorerequest.yaml
```

**Alignment**: LC2 is Kubernetes-native, LCP is REST API. Both are valid approaches for their respective architectures.

---

## Data Model Comparison

| Aspect | LCP (GCP) | LC2 (AWS) | Aligned? |
|--------|-----------|-----------|----------|
| Backup metadata | Tracker file (JSON) | Manifest + PostgreSQL | ✅ Yes |
| DB snapshots | Cloud SQL snapshots | RDS snapshots | ✅ Yes |
| DB dumps | pg_dump → GCS | pg_dump → S3 (Phase 2) | ✅ Yes |
| Document library | GCS bucket copy | S3 bucket copy | ✅ Yes |
| Retention | Managed by GCP | CRD field + cleanup job | ⚠️ Similar |
| Restore | Manual + Argo (CN) | TBD (Phase 2) | ⚠️ TBD |
| Multi-tenant | Per-customer service | Centralized controller | ❌ Different |

---

## Testing Migration Scenarios

### Scenario 1: Migrate Project from GCP to AWS
**Steps**:
1. Customer triggers final LCP backup on GCP
2. Export tracker file from GCS
3. Convert tracker file to LC2 manifest format
4. Upload manifest to S3 in LC2 format
5. Restore using LC2 restore workflow (Phase 2)

### Scenario 2: Parallel Running (Transition Period)
**Steps**:
1. Run LCP backups on GCP (legacy)
2. Run LC2 backups on AWS (new)
3. Keep both active during migration
4. Cutover after validation period

---

## Recommendations

### Immediate (Phase 1)
1. ✅ Keep current manifest format (already aligned)
2. ✅ Add LCP compatibility fields to manifest (optional)
3. ✅ Document migration path for GCP → AWS
4. ⚠️ Consider adding backup-metadata symlink path

### Future (Phase 2)
1. **Evaluate Argo Workflows** for restore orchestration
2. Design RestoreRequest CRD
3. Create GCP → AWS migration tooling
4. Add support for importing LCP tracker files

### Long-Term
1. Unify backup formats across LCP and LC2
2. Share backup tooling/libraries between teams
3. Support cross-cloud restore (GCP backup → AWS restore)

---

## Questions for Discussion

1. **Should we use Argo Workflows for LC2 restore?**
   - Pro: Aligns with CN implementation
   - Con: Additional dependency

2. **Should we support importing LCP tracker files?**
   - Pro: Easier migration
   - Con: Additional code complexity

3. **Should we create REST API wrapper around Kubernetes API?**
   - Pro: Familiar to LCP users
   - Con: Adds abstraction layer

4. **Should backup manifests be backward-compatible with tracker files?**
   - Pro: Migration simplicity
   - Con: Technical debt

---

## Conclusion

LC2 backup implementation is **well-aligned** with existing Liferay Cloud patterns:
- ✅ Tracker files → Manifests (same concept)
- ✅ Component separation (DB/DL)
- ✅ Cloud-native snapshots
- ✅ Metadata-driven architecture

**Key Differences**:
- Multi-tenant controller vs per-customer service
- Kubernetes CRD vs REST API
- Provider abstraction vs GCP-specific

**Next Steps**:
1. Minor manifest format tweaks for LCP compatibility
2. Plan Phase 2 restore using Argo Workflows
3. Design migration tooling for GCP → AWS

---

**Document Version**: 1.0
**Last Updated**: 2025-11-09
**Authors**: LC2 Backup Team
