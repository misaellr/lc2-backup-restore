# LC2 Backup and Restore Service - Next Steps

**Last Updated:** 2025-11-10
**Status:** Requirements phase complete, ready for implementation planning

---

## Recent Updates

### Drupal Configuration Management Enhancement (Commit 3a839d4)

**1. Database Standardization:**
- All applications now use **PostgreSQL 13+** (Liferay, Drupal, Bloomreach)
- Primary strategy: **Incremental backups** (reduces backup time and storage costs)
- Fallback strategy: **Snapshots** (for clouds that don't support incrementals)
- Dump enabled for cross-cloud portability

**2. Comprehensive Drupal Configuration Management:**
Added **dual-track approach** to Section 4.2:
- **Track 1:** Configuration backed up as part of PostgreSQL database
- **Track 2:** Automated `drush config:export` via pre-backup hooks
- **Rationale:**
  - Point-in-Time Recovery: Database backup provides complete state restoration
  - Version Control: YAML exports enable Git-based configuration tracking
  - Environment Parity: Exported configs can be deployed to dev/staging with different content
  - Selective Restore: Configuration-only restores without overwriting content

**3. Pre-Backup Hooks:**
```yaml
database:
  preBackupHooks:
    - name: export-drupal-config
      command: drush config:export --destination=/tmp/config-export
      storeArtifact: true
      artifactPath: /tmp/config-export
      timeout: 300
```

**4. What Gets Exported vs Excluded:**
- **Exported:** Site settings, content types, views, blocks, field definitions, taxonomy, roles, permissions
- **Excluded:** Content (nodes, users, media), state data (caches), `settings.php` (environment-specific)

---

## Requirements Clarification - All 12 Questions Answered

| # | Question | Answer | Impact on Requirements |
|---|----------|--------|------------------------|
| **1** | **Database types for all apps** | ✅ All use **PostgreSQL 13+** (Liferay, Drupal, Bloomreach) | Updated all app sections to specify PostgreSQL |
| **2** | **Elasticsearch backup strategy** | ✅ **Yes, needed long-term** - but **NOT part of MVP** | ⏸️ **Future:** Reference doc available at `reference/elasticsearch-backup-ideas.txt` |
| **3** | **Cross-cloud restore mechanics** | ✅ Always use **dump format** (snapshots are cloud-specific) | Documented in requirements |
| **4** | **Retention policy specifics** | ✅ **7 days default**, parameterizable per project | **TODO:** Update retention section with parameterization details |
| **5** | **Backup windows/availability** | ✅ **Zero downtime** - hot backups required | Already documented in NFR-1 |
| **6** | **MinIO in local environment** | ✅ **Both** - application storage + backup destination | **TODO:** Add MinIO configuration details |
| **7** | **Metadata store resilience** | ✅ **Yes** - metadata should be stored in file system for safety | **TODO:** Add metadata backup requirements |
| **8** | **Incremental backups** | ✅ **Primary strategy** for clouds that support them, snapshots as fallback | Updated all application sections |
| **9** | **Backup verification** | ✅ **Automatic checksum verification** only (no full restore testing) | Already documented in requirements |
| **10** | **Encryption key rotation** | ✅ **Manual re-encryption** during cross-cloud migration | Documented in Security Requirements |
| **11** | **Drupal configuration management** | ✅ **Dual-track**: Database backup + automated `drush config:export` | ✅ **COMPLETED** - Section 4.2 updated |
| **12** | **Monitoring alert thresholds** | ✅ **Run tests and define later** | Will be determined during implementation |

---

## Pending Requirements Updates (MVP Only)

Based on the questions answered, the following **3 sections** need updates for MVP:

### 1. ~~Elasticsearch/OpenSearch Backup Strategy~~ (Question #2) - **NOT MVP**
**Status:** ⏸️ **DEFERRED - Post-MVP enhancement**

**Reference:** Comprehensive strategy documented in `reference/elasticsearch-backup-ideas.txt`
- Covers snapshot-based backup using S3/GCS/MinIO repositories
- GitOps-friendly approach with BackupRequest/RestoreRequest CRD integration
- Supports local dev (MinIO), AWS (OpenSearch), GCP/Azure (ECK)
- Will be implemented in Phase 2+ after core database/storage backup is stable

**MVP Approach:** Elasticsearch indexes can be rebuilt from database if needed (acceptable for PoC phase)

### 2. Retention Policy Parameterization (Question #4)
**Location:** Section 4.1.2, 4.2.2, 4.3.2 - Update `retention` field

**What to add:**
- Document default retention: 7 days
- Show parameterization per environment
- Add weekly/monthly retention options
- Document compliance-driven retention (e.g., 7 years for financial data)

**Example:**
```yaml
retention:
  daily: 7        # Default: 7 days
  weekly: 4       # 4 weeks
  monthly: 12     # 12 months
  compliance:     # Optional: Industry-specific retention
    enabled: true
    period: 2555  # 7 years in days (GDPR, SOX, HIPAA)
```

### 3. MinIO Configuration Details (Question #6)
**Location:** Section 8 - Deployment Considerations (Local Environment)

**What to add:**
- MinIO as S3-compatible storage for local development
- Dual role: Application storage (Liferay doclib, Drupal files) + Backup destination
- Configuration examples for docker-compose
- Bucket naming conventions
- Access credentials management

**Example:**
```yaml
# docker-compose.yml
services:
  minio:
    image: minio/minio:latest
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Console

# Buckets:
# - liferay-doclib-dev       (Application storage)
# - drupal-files-dev         (Application storage)
# - lc2-backups-dev          (Backup destination)
```

### 4. Metadata Store Backup Requirements (Question #7)
**Location:** Section 5.1 - Functional Requirements (new FR-9)

**What to add:**
- PostgreSQL metadata store backup strategy
- File system export of backup metadata (JSON/YAML)
- Metadata recovery procedure
- Ability to rebuild metadata by scanning S3/GCS buckets

**Example:**
```yaml
FR-9: Metadata Store Resilience
- Backup operator shall export metadata to file system every hour
- Metadata export format: JSON (backup_metadata.json)
- Storage location: S3/GCS alongside backups (s3://backups/metadata/)
- Recovery: Operator can rebuild metadata by scanning backup buckets
- Metadata includes: backup ID, timestamp, size, status, application, components
```

---

## Implementation Planning - Next Phase

Once requirements updates are complete, proceed with implementation planning:

### Phase 1: Core Backup Operator

**1. Project Setup**
- [ ] Initialize Quarkus 3.x project (Java 17+)
- [ ] Add dependencies: Fabric8 Kubernetes Client, PostgreSQL driver, AWS SDK, GCP SDK
- [ ] Configure multi-module Maven/Gradle project structure
- [ ] Set up local development environment (k3d + MinIO + PostgreSQL)

**2. Custom Resource Definitions (CRDs)**
- [ ] Define `BackupRequest` CRD (v1alpha1)
- [ ] Define `RestoreRequest` CRD (v1alpha1)
- [ ] Generate Java models from CRD schemas
- [ ] Install CRDs in local k3d cluster

**3. Operator Core**
- [ ] Implement reconciliation loop (BackupRequestReconciler)
- [ ] Implement Provider Factory Pattern (AWS, GCP, Azure, Local)
- [ ] Implement metadata store (PostgreSQL repository)
- [ ] Add status updates to CRD instances

**4. PostgreSQL Backup - Local Provider**
- [ ] Implement `pg_basebackup` for incremental backups
- [ ] Implement `pg_dump` for portable dumps
- [ ] Implement upload to MinIO
- [ ] Add checksum verification

**5. Testing & Validation**
- [ ] Unit tests (JUnit 5)
- [ ] Integration tests (Testcontainers: PostgreSQL + MinIO)
- [ ] E2E test: Backup PostgreSQL → Verify in MinIO → Restore → Validate data

### Phase 2: Application-Specific Integrations

**6. Liferay Integration**
- [ ] Document library backup (S3/MinIO object transfer)
- [ ] `excludeGeneratedFiles` implementation
- [ ] Elasticsearch index backup/rebuild

**7. Drupal Integration**
- [ ] Pre-backup hook framework
- [ ] `drush config:export` automation
- [ ] Files directory backup
- [ ] Config versioning in S3/GCS

**8. Bloomreach Integration**
- [ ] API-based content model export
- [ ] Media asset backup
- [ ] JSON export to S3/GCS

### Phase 3: Multi-Cloud Support

**9. AWS Provider**
- [ ] RDS snapshot creation via AWS SDK
- [ ] RDS Export API integration (snapshot → S3)
- [ ] S3 object transfer
- [ ] KMS encryption integration

**10. GCP Provider**
- [ ] Cloud SQL snapshot creation
- [ ] Cloud SQL Export API integration
- [ ] GCS object transfer
- [ ] Cloud KMS integration

**11. Azure Provider**
- [ ] Azure Database for PostgreSQL snapshot
- [ ] Export to Blob Storage
- [ ] Azure Key Vault integration

### Phase 4: Observability & Operations

**12. Monitoring**
- [ ] Prometheus metrics (lc2_backup_requests_total, lc2_backup_duration_seconds, etc.)
- [ ] OpenTelemetry distributed tracing
- [ ] Grafana dashboards

**13. Security**
- [ ] RBAC roles and permissions
- [ ] Encryption at rest (AES-256)
- [ ] Audit logging
- [ ] Secret management (ExternalSecrets integration)

**14. Documentation**
- [ ] User guide (how to create backups, restores)
- [ ] Architecture Decision Records (ADRs)
- [ ] API reference (CRD schemas)
- [ ] Troubleshooting guide

---

## Open Questions for Next Session

1. **Elasticsearch backup strategy** - Do you want snapshot-based or rebuild-from-database strategy?
2. **Compliance requirements** - Any specific retention periods for regulatory compliance (GDPR, SOC 2, HIPAA)?
3. **Local development setup** - Should we create a docker-compose.yml for quick local testing?
4. **CI/CD integration** - GitHub Actions for automated testing and building?
5. **Helm chart** - Package operator as Helm chart for easy deployment?

---

## Repository Information

- **GitHub:** https://github.com/misaellr/lc2-backup-restore
- **Latest Commit:** 3a839d4 - "Update requirements: Enhanced Drupal configuration management"
- **Requirements Document:** `/home/misael/lc2/backuprestore/requirements.md` (2,489 lines)
- **Reference Docs:** `/home/misael/lc2/backuprestore/reference/` (10 files)

---

## Key Files Modified

1. **requirements.md** (Section 4.2 - Acquia Drupal)
   - Lines 334-342: Database component updated to PostgreSQL with incremental backups
   - Lines 353-372: Configuration component enhanced with dual-track approach
   - Lines 376-421: Backup API updated with pre-backup hooks and configuration management

2. **requirements.md** (Section 4.1 - Liferay DXP)
   - Lines 262-269: Database component updated to PostgreSQL with incremental backups
   - Lines 299-305: Backup API updated with incremental/snapshot strategy

---

## When You Return

1. **Review this document** to recall the current state
2. **Decide on pending updates** - Which of the 4 pending sections to tackle first?
3. **Begin implementation** - Or continue refining requirements?

**Recommended next step:** Add Elasticsearch backup strategy (#2) and MinIO configuration (#6) to requirements, then begin Phase 1 implementation planning.
