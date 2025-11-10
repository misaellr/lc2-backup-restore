# LC2 Backup and Restore Service - Requirements Document

**Version:** 1.0
**Date:** 2025-11-10
**Status:** Draft

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Business Context](#2-business-context)
3. [Technical Overview](#3-technical-overview)
4. [Application-Specific Requirements](#4-application-specific-requirements)
5. [System Requirements](#5-system-requirements)
6. [Security Requirements](#6-security-requirements)
7. [API Specifications](#7-api-specifications)
8. [Deployment Considerations](#8-deployment-considerations)
9. [Monitoring and Observability](#9-monitoring-and-observability)
10. [Testing and Validation](#10-testing-and-validation)
11. [Version Management and Upgrades](#11-version-management-and-upgrades)
12. [Cloud-Agnostic vs Cloud-Specific Design](#12-cloud-agnostic-vs-cloud-specific-design)
13. [Use Case Scenarios](#13-use-case-scenarios)
14. [Glossary](#14-glossary)
15. [References](#15-references)

---

## 1. Introduction

### 1.1 Purpose

The purpose of this document is to outline the requirements for the LC2 Backup and Restore Service, a critical component designed to ensure data integrity and availability for three Proof of Concept (PoC) applications: **Liferay DXP**, **Acquia Drupal**, and **Bloomreach**. This service aims to provide a robust backup and restore mechanism within a Kubernetes microservice architecture, addressing the needs of both local and AWS-based environments.

### 1.2 Scope

This document covers the definition, implementation, and management requirements for the LC2 Backup and Restore Service. It pertains to:

- **Local Environments:** Configured with k3d and Docker Compose using MinIO and PostgreSQL
- **Cloud Environments:** AWS-based deployments utilizing EKS, RDS, and S3
- **Key Innovation:** Enhanced Hybrid Approach leveraging managed service APIs to minimize operational issues such as Out-Of-Memory (OOM) errors
- **MVP Scope:** Database (PostgreSQL) and object storage (S3/MinIO) backup/restore
- **Post-MVP:** Elasticsearch/OpenSearch snapshot-based backup (see `reference/elasticsearch-backup-ideas.txt`)

The service will be deployed as a Kubernetes microservice and will support three distinct content management platforms with varying architectural requirements.

### 1.3 Stakeholders

The primary stakeholders include business and technical teams responsible for data management, IT infrastructure, and application delivery:

- **Business Analysts and Product Owners** who oversee data reliability and regulatory compliance
- **DevOps and Site Reliability Engineers** who manage infrastructure and deployment
- **Application Developers and Architects** focusing on integration with application ecosystems
- **IT Operations and Support Teams** responsible for maintaining system health and addressing incidents
- **Security and Compliance Officers** ensuring adherence to data protection regulations

### 1.4 Objectives

- Establish a reliable and efficient backup and restore framework compatible with the specified target applications
- Mitigate operational risks such as OOM errors via an Enhanced Hybrid Approach
- Enable seamless data management across both local and cloud environments
- Support business continuity through clearly defined Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO)
- Provide a cloud-agnostic abstraction layer while leveraging cloud-specific managed services for optimal performance

---

## 2. Business Context

### 2.1 Business Goals

The primary business goal of the LC2 Backup and Restore Service is to ensure uninterrupted access to critical business data across diverse application landscapes. Through this initiative, the organization aims to:

- **Enhance Data Resilience:** Protect intellectual property and customer data against loss, corruption, or accidental deletion
- **Improve Compliance:** Meet data protection regulations (GDPR, CCPA, SOC 2) through reliable backup and audit capabilities
- **Minimize Operational Disruptions:** Reduce downtime and data loss through efficient backup and restore operations
- **Support Multi-Cloud Strategy:** Enable flexibility in deployment environments while maintaining consistent backup capabilities
- **Optimize Resource Utilization:** Leverage managed cloud services to reduce operational overhead and infrastructure costs

By leveraging an innovative backup approach that eliminates data streaming through operator pods, the business seeks to maintain high availability and performance standards essential for enterprise-level operations.

### 2.2 Success Criteria

The success of the LC2 Backup and Restore Service will be measured against the following criteria:

#### Recovery Objectives

- **Recovery Time Objective (RTO):** 15 minutes for restoring critical application data, ensuring minimal operational downtime
- **Recovery Point Objective (RPO):** 5 minutes, limiting data loss exposure to recent transactions

#### Performance Metrics

- **Backup Completion Rate:** 99.9% successful backup completion rate across all environments
- **Restore Success Rate:** 100% successful restore operations with data integrity verification
- **Backup Window:** Complete backups within defined maintenance windows without impacting application performance
- **Storage Efficiency:** Achieve at least 3:1 compression ratio for database dumps

#### Operational Excellence

- **Stability:** Zero OOM errors or memory-related backup failures (improvement from current Liferay PaaS implementation)
- **Integration:** Successful integration with both local (k3d + Docker Compose) and AWS environments without data stream bottlenecks
- **Monitoring:** Complete observability with metrics, logs, and alerts for all backup operations
- **Security:** All backup data encrypted at rest and in transit, with audit logging for compliance

#### Application Coverage

- **Liferay DXP:** Support for coordinated database (snapshot + dump) and document library backups with selective file exclusion
- **Acquia Drupal:** PHP application-specific backup strategies including content and configuration separation
- **Bloomreach:** Headless CMS backup strategies ensuring API availability and content consistency

These parameters will guide the implementation and evaluation of the LC2 Backup and Restore Service, ensuring alignment with the overarching business goals.

---

## 3. Technical Overview

### 3.1 System Architecture

The LC2 Backup and Restore Service is built on a **Kubernetes Operator pattern** using **Quarkus** (Java) to orchestrate backup operations across multiple cloud providers and local environments. The architecture employs an **Enhanced Hybrid Approach** that leverages managed service APIs to eliminate data streaming through operator pods.

#### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (Any Cloud/Local)                      │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │         LC2 Backup Operator (Quarkus)                                  │ │
│  │                                                                         │ │
│  │  ┌──────────────┐    ┌──────────────────┐   ┌──────────────┐         │ │
│  │  │ Reconciler   │───▶│ Provider Factory │───│ PostgreSQL   │         │ │
│  │  │ (watches CRD)│    │ (Strategy)       │   │ (metadata)   │         │ │
│  │  └──────────────┘    └──────────────────┘   └──────────────┘         │ │
│  │         │                    │                                         │ │
│  │         │         ┌──────────┴──────────┬─────────┬─────────┐         │ │
│  │         │         ▼          ▼          ▼         ▼         ▼         │ │
│  │         │    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────────┐          │ │
│  │         │    │ AWS  │  │ GCP  │  │Azure │  │Local (MinIO) │          │ │
│  │         │    │(APIs)│  │(APIs)│  │(APIs)│  │              │          │ │
│  │         │    └───┬──┘  └───┬──┘  └───┬──┘  └──────────────┘          │ │
│  └─────────┼────────┼─────────┼─────────┼────────────────────────────────┘ │
│            │        │         │         │                                   │
│            ▼        │         │         │                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │               BackupRequest CRD                                      │  │
│  │  apiVersion: backup.lc2.io/v1alpha1                                 │  │
│  │  kind: BackupRequest                                                 │  │
│  │  spec:                                                                │  │
│  │    projectId: retail                                                 │  │
│  │    application: liferay | drupal | bloomreach                       │  │
│  │    provider: aws | gcp | azure | local                              │  │
│  │    database: {...}                                                   │  │
│  │    storage: {...}  # Document Library / File Storage                │  │
│  │  status:                                                             │  │
│  │    state: Pending | InProgress | Completed | Failed                 │  │
│  │    snapshot: {...}     # Managed service handles this!              │  │
│  │    exportOperation: {...}  # No streaming through operator!         │  │
│  │    transferJob: {...}  # Cloud Transfer Service handles this!       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Key Architectural Principles

1. **No Data Streaming Through Operator:** Managed service APIs handle data movement directly to object storage
2. **Provider Abstraction:** Strategy Pattern allows cloud-agnostic interface with cloud-specific implementations
3. **Dual State Management:** CRD for Kubernetes-native state + PostgreSQL for historical metadata and analytics
4. **Reconciliation Loop:** Kubernetes-native controller pattern for reliable orchestration
5. **Application-Specific Adapters:** Pluggable adapters for Liferay, Drupal, and Bloomreach-specific requirements

### 3.2 Backup Strategies

#### 3.2.1 Enhanced Hybrid Approach

**Problem Statement:** The current Liferay PaaS backup service experiences critical issues:
- OutOfMemory errors from streaming data through backup service pods
- Unstable data transfer due to memory pressure
- Long backup/restore times from inefficient processing

**Solution:** Use managed service APIs to eliminate data streaming through operator pods:

```
Traditional Approach (Current Liferay PaaS):
Database → Backup Pod (pg_dump) → Compress in Pod → Stream to Storage
                ↑ OOM Errors!

Enhanced Hybrid Approach (LC2):
Database → Managed Service API → Direct to Object Storage
                ↑ No pod involvement in data movement!
```

#### 3.2.2 Dual Backup Strategy

For database workloads, the service implements a dual strategy:

1. **Snapshot Backups**
   - **Purpose:** Fast restore capability (minutes for multi-TB databases)
   - **Implementation:** Cloud provider native snapshots (RDS snapshots, Cloud SQL snapshots, Azure Database snapshots)
   - **Limitations:** Cloud-specific, not portable across environments

2. **Dump Backups**
   - **Purpose:** Portability, cross-cloud migration, version upgrades
   - **Implementation:** Logical exports (pg_dump, mysqldump) via managed export APIs
   - **Benefits:** Cloud-agnostic, human-readable, selective restore capability

**Rationale:** This dual approach balances speed (snapshot) with portability (dump), as validated by current Liferay PaaS implementation.

#### 3.2.3 Application-Specific Filtering

The service supports application-specific backup filtering:

**Liferay DXP:**
- `excludeGeneratedFiles`: Skip auto-generated thumbnails, previews, and cache files
- Reduces backup size by 40-60% for typical Liferay installations
- Configurable via BackupRequest CRD

**Drupal:**
- Separate content and configuration backups
- Exclude cache tables and temporary files
- Configuration export via Drush integration

**Bloomreach:**
- API-based content export
- Separate content model and content instances
- Versioned backup format

### 3.3 Technology Stack

#### Core Framework
- **Quarkus 3.x:** Kubernetes-native Java framework with fast startup and low memory footprint
- **Java 17+:** Modern Java with performance improvements and language features
- **Fabric8 Kubernetes Client:** Java client for Kubernetes API interactions
- **Quarkus Operator SDK:** Framework for building Kubernetes operators

#### Data Management
- **PostgreSQL 15+:** Metadata storage, backup history, analytics
- **Flyway:** Database schema migrations
- **HikariCP:** Connection pooling

#### Cloud Provider SDKs
- **AWS SDK for Java:** RDS, S3, EKS integrations
- **Google Cloud Client Libraries:** Cloud SQL, GCS, GKE integrations
- **Azure SDK for Java:** Azure Database, Blob Storage, AKS integrations

#### Observability
- **Micrometer:** Metrics collection with Prometheus exposition
- **SLF4J + Logback:** Structured logging
- **OpenTelemetry:** Distributed tracing

#### Testing
- **JUnit 5:** Unit testing framework
- **Testcontainers:** Integration testing with containerized dependencies
- **Kubernetes Test Framework:** Operator testing

---

## 4. Application-Specific Requirements

### 4.1 Liferay DXP

**Application Type:** Java-based enterprise portal and digital experience platform

#### 4.1.1 Backup Components

1. **Database**
   - **Type:** PostgreSQL 13+
   - **Size:** Typically 10GB - 2TB
   - **Strategy:** Incremental backups (primary) + Snapshots (fallback)
   - **Special Considerations:**
     - Large blob fields (document metadata)
     - Complex schema with 400+ tables
     - Incremental backups significantly reduce backup time for large databases

2. **Document Library**
   - **Storage:** File system or object storage (GCS, S3, Azure Blob)
   - **Size:** 100GB - 100TB typical production deployments
   - **Strategy:** Object-to-object transfer via managed service APIs
   - **Special Considerations:**
     - `excludeGeneratedFiles` flag to skip auto-generated content:
       - Thumbnails (`/document_library/*/thumbnails/*`)
       - Previews (`/document_library/*/previews/*`)
       - Cache files (`/document_library/*/.cache/*`)
     - Reduces backup size by 40-60%

3. **Search Index** ⏸️ **NOT PART OF MVP**
   - **Type:** Elasticsearch 7.x or OpenSearch
   - **MVP Strategy:** Rebuild from database (acceptable for PoC phase)
   - **Post-MVP Strategy:** Snapshot-based backup using S3/GCS/MinIO repositories
   - **Reference:** Comprehensive strategy documented in `reference/elasticsearch-backup-ideas.txt`
   - **Consideration:** Rebuilding index may take hours for large datasets, but acceptable for MVP
   - **Rationale:** Core database and storage backup provides sufficient data protection for PoC phase
   - **Future Enhancement:** Full snapshot/restore strategy will be implemented in Phase 2+

#### 4.1.2 Backup API Requirements

```yaml
apiVersion: backup.lc2.io/v1alpha1
kind: BackupRequest
metadata:
  name: liferay-prod-backup
spec:
  application: liferay
  projectId: retail-prod
  provider: aws

  database:
    type: postgresql
    instance: liferay-prod-db
    incrementalEnabled: true     # Primary backup strategy
    snapshotEnabled: true        # Fallback strategy
    dumpEnabled: true            # For cross-cloud portability
    dumpFormat: custom           # PostgreSQL custom format

  storage:
    type: s3
    bucket: liferay-doclib-prod
    excludeGeneratedFiles: true
    excludePatterns:
      - "*/thumbnails/*"
      - "*/previews/*"
      - "*/.cache/*"

  retention:
    daily: 7
    weekly: 4
    monthly: 12
```

#### 4.1.3 Restore Requirements

**MVP Scope:**
- **Full Restore:** Database + Document Library
- **Partial Restore:** Database only (search index rebuilt from database - acceptable for PoC)
- **Point-in-Time Restore:** Use snapshot for specific timestamp
- **Cross-Environment Restore:** Use dump for dev/staging from production backup

**Post-MVP:**
- **Full Restore with Search Index:** Database + Document Library + Elasticsearch snapshot restore
- **Search Index Only Restore:** Restore Elasticsearch snapshot without affecting database/storage

### 4.2 Acquia Drupal

**Application Type:** PHP-based content management system

#### 4.2.1 Backup Components

1. **Database**
   - **Type:** PostgreSQL 13+
   - **Size:** Typically 1GB - 500GB
   - **Strategy:** Incremental backups (primary) + Snapshots (fallback)
   - **Special Considerations:**
     - Separate content tables from cache tables
     - Exclude cache_* and sessions tables from backups
     - Support for Drush database dump integration
     - Incremental backups reduce backup time and storage costs

2. **Files Directory**
   - **Storage:** S3, GCS, or local file system
   - **Size:** 10GB - 10TB
   - **Strategy:** Object-to-object transfer
   - **Special Considerations:**
     - Public files (`/sites/default/files/`)
     - Private files (if configured)
     - Temporary files (excluded)

3. **Configuration**
   - **Type:** YAML files exported via Drush
   - **Strategy:** Dual-track approach (Database backup + Separate config export)
   - **Special Considerations:**
     - **Database Backup:** Configuration stored in PostgreSQL is backed up as part of incremental/snapshot strategy
     - **Config Export:** Automated `drush config:export` executed before each backup via pre-backup hooks
     - **Rationale for Dual-Track:**
       - **Point-in-Time Recovery:** Database backup provides complete state restoration
       - **Version Control:** YAML exports enable Git-based configuration tracking
       - **Environment Parity:** Exported configs can be deployed to dev/staging with different content
       - **Selective Restore:** Configuration-only restores without overwriting content
     - **What Gets Exported:**
       - Site settings, content types, views, blocks, field definitions
       - Taxonomy vocabularies, user roles, permissions
       - Module configurations
     - **What's Excluded:**
       - Content (nodes, users, media) - handled by database backup
       - State data (caches, temporary data) - regenerated on restore
       - `settings.php` - environment-specific file
     - **Storage:** Exported YAML configs stored alongside database backups in S3/GCS

#### 4.2.2 Backup API Requirements

```yaml
apiVersion: backup.lc2.io/v1alpha1
kind: BackupRequest
metadata:
  name: drupal-prod-backup
spec:
  application: drupal
  projectId: marketing-prod
  provider: aws

  database:
    type: postgresql
    instance: drupal-prod-db
    incrementalEnabled: true     # Primary backup strategy
    snapshotEnabled: true        # Fallback strategy
    dumpEnabled: true            # For cross-cloud portability
    excludeTables:
      - cache_*
      - sessions
      - watchdog
    drushIntegration: true
    preBackupHooks:              # Execute before database backup
      - name: export-drupal-config
        command: |
          drush config:export --destination=/tmp/config-export
        storeArtifact: true      # Store exported YAML alongside database backup
        artifactPath: /tmp/config-export
        timeout: 300             # 5 minutes timeout

  storage:
    type: s3
    bucket: drupal-files-prod
    includePaths:
      - "sites/default/files/"
      - "sites/default/private/"
    excludePaths:
      - "sites/default/files/tmp/"
      - "sites/default/files/css/"
      - "sites/default/files/js/"

  configuration:
    enabled: true
    exportCommand: "drush config:export"
    storagePath: "config/"       # Store in separate S3 prefix
    versionControl: true         # Track configuration changes over time
```

#### 4.2.3 Restore Requirements

- **Full Restore:** Database + Files + Configuration
- **Content Only:** Database restore without configuration
- **Configuration Only:** Import configuration via Drush
- **Selective Restore:** Specific content types or file directories

### 4.3 Bloomreach

**Application Type:** Headless CMS with API-first architecture

#### 4.3.1 Backup Components

1. **Content Database**
   - **Type:** PostgreSQL 13+ or MongoDB (depending on deployment)
   - **Size:** Typically 5GB - 1TB
   - **Strategy:** Dual (snapshot + dump)
   - **Special Considerations:**
     - JSON document storage
     - Content versioning history
     - Media asset metadata

2. **Media Assets**
   - **Storage:** S3, GCS, or CDN-backed storage
   - **Size:** 50GB - 50TB
   - **Strategy:** Object-to-object transfer
   - **Special Considerations:**
     - High-volume assets
     - CDN cache invalidation after restore
     - Asset versioning

3. **Content Model**
   - **Type:** Schema definitions, content types, workflows
   - **Strategy:** API-based export to JSON
   - **Special Considerations:**
     - Separate from content instances
     - Version-controlled schema

#### 4.3.2 Backup API Requirements

```yaml
apiVersion: backup.lc2.io/v1alpha1
kind: BackupRequest
metadata:
  name: bloomreach-prod-backup
spec:
  application: bloomreach
  projectId: ecommerce-prod
  provider: aws

  database:
    type: postgresql
    instance: bloomreach-prod-db
    snapshotEnabled: true
    dumpEnabled: true
    dumpFormat: custom  # PostgreSQL custom format

  storage:
    type: s3
    bucket: bloomreach-media-prod
    parallelTransfer: true
    maxConcurrency: 10

  contentModel:
    enabled: true
    apiEndpoint: "https://api.bloomreach.example.com"
    exportFormat: json
    includeVersionHistory: false
```

#### 4.3.3 Restore Requirements

- **Full Restore:** Database + Media Assets + Content Model
- **Content Only:** Database restore (preserves existing schema)
- **Media Only:** Asset restore without database changes
- **Schema Migration:** Restore with content model version upgrade

---

## 5. System Requirements

### 5.1 Functional Requirements

#### 5.1.1 Backup Management

**FR-1: Create Backup**
- System shall support on-demand backup creation via BackupRequest CRD
- System shall support scheduled backups via cron expressions
- System shall validate backup requests before execution
- System shall support application-specific backup parameters

**FR-2: List Backups**
- System shall provide API to list all backups for a project
- System shall support filtering by date range, status, and application
- System shall return backup metadata including size, duration, and status

**FR-3: Delete Backup**
- System shall support manual backup deletion
- System shall enforce retention policies automatically
- System shall maintain audit log of deleted backups
- System shall support soft-delete with grace period

**FR-4: Backup Scheduling**
- System shall support cron-based backup scheduling
- System shall prevent overlapping backup operations
- System shall support backup windows (time-based execution constraints)

#### 5.1.2 Restore Capabilities

**FR-5: Full Restore**
- System shall restore all components (database + storage) to specified timestamp
- System shall validate backup integrity before restore
- System shall support dry-run mode for restore validation

**FR-6: Partial Restore**
- System shall support database-only restore
- System shall support storage-only restore
- System shall support selective file/directory restore

**FR-7: Point-in-Time Restore**
- System shall support restore to specific timestamp using snapshots
- System shall list available restore points
- System shall validate target environment compatibility

**FR-8: Cross-Environment Restore**
- System shall support restore from production to dev/staging
- System shall support data sanitization during cross-environment restore
- System shall handle environment-specific configuration differences

### 5.2 Non-Functional Requirements

#### 5.2.1 Performance

**NFR-1: Backup Performance**
- Backup operations shall complete within defined SLAs:
  - Database snapshot: < 5 minutes (regardless of size)
  - Database dump: < 2 hours for 1TB database
  - Object storage transfer: > 100 MB/s aggregate throughput

**NFR-2: Restore Performance**
- Restore operations shall meet RTO requirements:
  - Snapshot restore: < 15 minutes
  - Full restore (dump + storage): < 2 hours for typical production workload

**NFR-3: Resource Efficiency**
- Operator pod shall consume < 512MB memory during steady state
- Operator shall not stream data through pod memory
- Backup operations shall have minimal impact on application performance (< 5% CPU/IO)

#### 5.2.2 Scalability

**NFR-4: Concurrent Operations**
- System shall support up to 10 concurrent backup operations per cluster
- System shall support up to 5 concurrent restore operations per cluster
- System shall implement queuing for operations exceeding concurrency limits

**NFR-5: Multi-Tenancy**
- System shall support multiple projects within a single cluster
- System shall enforce project-level isolation for backups and restores
- System shall support separate retention policies per project

#### 5.2.3 Security

**NFR-6: Encryption**
- All backup data shall be encrypted at rest using AES-256
- All backup data shall be encrypted in transit using TLS 1.3
- Encryption keys shall be managed via cloud provider KMS or HashiCorp Vault

**NFR-7: Access Control**
- System shall integrate with Kubernetes RBAC for access control
- System shall enforce least-privilege access to backup operations
- System shall support service accounts for automated operations

**NFR-8: Audit Logging**
- System shall log all backup and restore operations with timestamps
- System shall log all access to backup data
- Audit logs shall be immutable and stored for minimum 90 days

#### 5.2.4 Reliability

**NFR-9: Availability**
- Operator shall have 99.9% uptime SLA
- System shall implement automatic retry with exponential backoff for transient failures
- System shall support graceful degradation during cloud provider outages

**NFR-10: Data Integrity**
- System shall verify backup integrity using checksums (SHA-256)
- System shall validate restore completeness
- System shall detect and report corruption during backup/restore operations

**NFR-11: Observability**
- System shall expose Prometheus metrics for all operations
- System shall emit structured logs in JSON format
- System shall support distributed tracing for debugging

---

## 6. Security Requirements

### 6.1 Data Protection

#### 6.1.1 Encryption at Rest

**SR-1: Backup Data Encryption**
- All backup data (database dumps, snapshots, file storage) shall be encrypted at rest using AES-256 encryption
- Encryption keys shall be managed via cloud provider Key Management Services (KMS):
  - AWS: AWS KMS with automatic key rotation enabled
  - GCP: Cloud KMS with HSM-backed keys
  - Azure: Azure Key Vault with managed HSM
  - Local: HashiCorp Vault with auto-unsealing
- Separate encryption keys shall be used per project/environment
- Encryption key access shall be logged and auditable

**SR-2: Metadata Encryption**
- PostgreSQL database containing backup metadata shall use Transparent Data Encryption (TDE)
- Database connection strings and credentials shall be stored in Kubernetes Secrets with encryption at rest enabled
- Sensitive fields in CRDs (passwords, tokens) shall be referenced via Secret references, not stored directly

#### 6.1.2 Encryption in Transit

**SR-3: Network Encryption**
- All communication between operator and cloud provider APIs shall use TLS 1.3
- All communication between operator and metadata database shall use TLS 1.2+
- Certificate validation shall be enforced (no self-signed certificates in production)
- Mutual TLS (mTLS) shall be supported for service-to-service communication

**SR-4: Data Transfer Security**
- Backup data transfers shall use encrypted protocols:
  - S3: Server-side encryption with SSE-KMS
  - GCS: Customer-managed encryption keys (CMEK)
  - Azure Blob: Customer-managed keys in Azure Key Vault
- Local development shall use MinIO with TLS enabled

### 6.2 Access Control

#### 6.2.1 Authentication

**SR-5: Operator Authentication**
- Operator shall authenticate to cloud providers using:
  - AWS: IAM Roles for Service Accounts (IRSA) via OIDC
  - GCP: Workload Identity Federation
  - Azure: Managed Identity for AKS
- No long-lived credentials shall be stored in the cluster
- Service account tokens shall be automatically rotated

**SR-6: API Authentication**
- REST API endpoints shall require authentication via:
  - Kubernetes service account tokens (for in-cluster access)
  - OAuth 2.0 / OIDC tokens (for external access)
  - API keys (for automated systems, with rotation policy)
- Anonymous access shall be disabled
- Failed authentication attempts shall be rate-limited and logged

#### 6.2.2 Authorization

**SR-7: Role-Based Access Control (RBAC)**
- Kubernetes RBAC shall enforce least-privilege access:
  - **backup-admin:** Full access to all backup operations
  - **backup-operator:** Create, list, and restore backups
  - **backup-viewer:** Read-only access to backup metadata
- Project-level isolation shall be enforced (users cannot access backups from other projects)
- Cross-namespace access shall be explicitly denied

**SR-8: Cloud Provider IAM**
- Operator IAM roles/service accounts shall follow least-privilege principle:
  - Read/write access only to designated backup buckets
  - Snapshot/export permissions only for managed databases
  - No admin or wildcard permissions
- IAM policies shall be reviewed quarterly
- Unused permissions shall be removed

Example AWS IAM Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBSnapshot",
        "rds:DescribeDBSnapshots",
        "rds:StartExportTask"
      ],
      "Resource": "arn:aws:rds:*:*:db:lc2-*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::lc2-backups-prod/*"
    }
  ]
}
```

### 6.3 Compliance and Audit

#### 6.3.1 Audit Logging

**SR-9: Comprehensive Audit Trail**
- All backup and restore operations shall be logged with:
  - User identity (service account or user)
  - Timestamp (ISO 8601 with timezone)
  - Operation type (create, restore, delete)
  - Resource identifiers (backup ID, project ID)
  - Success/failure status
  - Client IP address (if applicable)
- Audit logs shall be immutable (write-once)
- Audit logs shall be retained for minimum 365 days
- Audit logs shall be exported to centralized logging system (CloudWatch, Stackdriver, Azure Monitor)

**SR-10: Compliance Reporting**
- System shall generate compliance reports for:
  - GDPR: Data retention, right to erasure, data portability
  - SOC 2: Access controls, encryption, monitoring
  - HIPAA: Encryption, audit logging, access controls (if applicable)
- Reports shall be generated on-demand and scheduled monthly
- Reports shall include backup coverage, restore tests, and security incidents

#### 6.3.2 Data Retention and Deletion

**SR-11: Secure Deletion**
- Backup deletion shall follow secure deletion practices:
  - Cloud provider: Use object versioning with lifecycle policies to prevent accidental deletion
  - Multi-step deletion: Soft delete (30-day grace period) → Hard delete
  - Verification: Confirm deletion via cloud provider APIs
- Encryption keys for deleted backups shall be destroyed after retention period
- Audit log entries for deleted backups shall be preserved

**SR-12: Data Residency**
- Backups shall be stored in the same geographic region as source data (compliance with GDPR, data sovereignty laws)
- Cross-region replication shall require explicit configuration
- Data residency policies shall be enforceable via CRD validation webhooks

### 6.4 Vulnerability Management

**SR-13: Security Scanning**
- Container images shall be scanned for vulnerabilities using:
  - Trivy, Clair, or equivalent scanner
  - Scanning integrated into CI/CD pipeline
  - Critical vulnerabilities shall block deployment
- Dependencies (Java libraries, npm packages) shall be scanned for known CVEs
- Security patches shall be applied within 30 days of disclosure

**SR-14: Secrets Management**
- Kubernetes Secrets shall be encrypted at rest (etcd encryption enabled)
- External secrets management shall be supported:
  - AWS Secrets Manager
  - GCP Secret Manager
  - Azure Key Vault
  - HashiCorp Vault
- Secrets shall be rotated every 90 days
- Secrets shall never be logged or exposed in error messages

### 6.5 Network Security

**SR-15: Network Segmentation**
- Operator shall run in dedicated namespace (`lc2-system`) with network policies
- Network policies shall restrict:
  - Ingress: Only from API Gateway / Ingress Controller
  - Egress: Only to cloud provider APIs, metadata database, and Kubernetes API server
- Pod-to-pod communication shall be denied by default

**SR-16: Private Endpoints**
- Cloud provider APIs shall be accessed via private endpoints (VPC endpoints, Private Service Connect) where available
- Public internet access shall be minimized
- Backup data shall never traverse public internet (use VPN or Direct Connect/ExpressRoute for hybrid scenarios)

---

## 7. API Specifications

### 6.1 Custom Resource Definitions (CRDs)

#### 6.1.1 BackupRequest CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backuprequests.backup.lc2.io
spec:
  group: backup.lc2.io
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [application, projectId, provider]
              properties:
                application:
                  type: string
                  enum: [liferay, drupal, bloomreach]
                projectId:
                  type: string
                  pattern: '^[a-z0-9-]+$'
                provider:
                  type: string
                  enum: [aws, gcp, azure, local]
                database:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: [mysql, postgresql, mariadb, mongodb]
                    instance:
                      type: string
                    snapshotEnabled:
                      type: boolean
                      default: true
                    dumpEnabled:
                      type: boolean
                      default: true
                    excludeTables:
                      type: array
                      items:
                        type: string
                storage:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: [s3, gcs, azure-blob, minio]
                    bucket:
                      type: string
                    excludeGeneratedFiles:
                      type: boolean
                      default: false
                    excludePatterns:
                      type: array
                      items:
                        type: string
                retention:
                  type: object
                  properties:
                    daily:
                      type: integer
                      minimum: 1
                      default: 7
                    weekly:
                      type: integer
                      minimum: 0
                      default: 4
                    monthly:
                      type: integer
                      minimum: 0
                      default: 12
            status:
              type: object
              properties:
                state:
                  type: string
                  enum: [Pending, SnapshotInProgress, ExportInProgress,
                         TransferInProgress, GeneratingMetadata,
                         Completed, Failed]
                startTime:
                  type: string
                  format: date-time
                completionTime:
                  type: string
                  format: date-time
                snapshot:
                  type: object
                  properties:
                    id:
                      type: string
                    status:
                      type: string
                exportOperation:
                  type: object
                  properties:
                    operationId:
                      type: string
                    status:
                      type: string
                transferJob:
                  type: object
                  properties:
                    jobId:
                      type: string
                    status:
                      type: string
                metadata:
                  type: object
                  properties:
                    backupId:
                      type: string
                    size:
                      type: integer
                    checksum:
                      type: string
                error:
                  type: string
```

#### 6.1.2 RestoreRequest CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: restorerequests.backup.lc2.io
spec:
  group: backup.lc2.io
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [backupId, targetEnvironment]
              properties:
                backupId:
                  type: string
                targetEnvironment:
                  type: string
                restoreType:
                  type: string
                  enum: [full, database-only, storage-only, selective]
                  default: full
                dryRun:
                  type: boolean
                  default: false
                sanitize:
                  type: boolean
                  default: false
                  description: "Apply data sanitization for cross-env restore"
            status:
              type: object
              properties:
                state:
                  type: string
                  enum: [Pending, ValidatingBackup, RestoringSnapshot,
                         RestoringDump, RestoringStorage,
                         Completed, Failed]
                progress:
                  type: integer
                  minimum: 0
                  maximum: 100
                error:
                  type: string
```

### 6.2 REST API Endpoints

The operator will expose REST API endpoints for external integrations:

#### 6.2.1 Backup Operations

**Create Backup**
```
POST /api/v1/projects/{projectId}/backups
Content-Type: application/json

{
  "application": "liferay",
  "provider": "aws",
  "database": { ... },
  "storage": { ... }
}

Response: 201 Created
{
  "backupId": "backup-20251110-abc123",
  "status": "Pending",
  "requestId": "req-xyz789"
}
```

**List Backups**
```
GET /api/v1/projects/{projectId}/backups?application=liferay&startDate=2025-11-01

Response: 200 OK
{
  "backups": [
    {
      "backupId": "backup-20251110-abc123",
      "application": "liferay",
      "status": "Completed",
      "size": 524288000,
      "createdAt": "2025-11-10T10:30:00Z"
    }
  ],
  "total": 42
}
```

**Get Backup Details**
```
GET /api/v1/projects/{projectId}/backups/{backupId}

Response: 200 OK
{
  "backupId": "backup-20251110-abc123",
  "application": "liferay",
  "status": "Completed",
  "components": {
    "database": {
      "snapshot": "snapshot-abc123",
      "dump": "s3://backups/liferay/dump-abc123.sql.gz"
    },
    "storage": {
      "location": "s3://backups/liferay/doclib-abc123/"
    }
  }
}
```

**Delete Backup**
```
DELETE /api/v1/projects/{projectId}/backups/{backupId}

Response: 204 No Content
```

#### 6.2.2 Restore Operations

**Create Restore**
```
POST /api/v1/projects/{projectId}/restores
Content-Type: application/json

{
  "backupId": "backup-20251110-abc123",
  "targetEnvironment": "staging",
  "restoreType": "full",
  "sanitize": true
}

Response: 201 Created
{
  "restoreId": "restore-20251110-xyz789",
  "status": "Pending"
}
```

**Get Restore Status**
```
GET /api/v1/projects/{projectId}/restores/{restoreId}

Response: 200 OK
{
  "restoreId": "restore-20251110-xyz789",
  "status": "RestoringStorage",
  "progress": 65,
  "estimatedCompletion": "2025-11-10T11:45:00Z"
}
```

### 6.3 Managed Service API Integrations

#### 6.3.1 AWS Provider

**RDS Snapshot API**
```java
// Create snapshot
CreateDBSnapshotRequest request = CreateDBSnapshotRequest.builder()
    .dbInstanceIdentifier("liferay-prod-db")
    .dbSnapshotIdentifier("backup-20251110-abc123")
    .tags(Tag.builder().key("BackupId").value("backup-20251110-abc123").build())
    .build();

// Export to S3
StartExportTaskRequest exportRequest = StartExportTaskRequest.builder()
    .sourceArn(snapshotArn)
    .s3BucketName("lc2-backups-prod")
    .s3Prefix("backups/liferay/backup-20251110-abc123/")
    .iamRoleArn("arn:aws:iam::123456789:role/RDSExportRole")
    .kmsKeyId("arn:aws:kms:us-east-1:123456789:key/abc-123")
    .build();
```

**S3 Transfer**
```java
// No operator involvement - S3 handles replication
// Operator only monitors via S3 API
```

#### 6.3.2 GCP Provider

**Cloud SQL Export API**
```java
// Export database directly to GCS
InstancesExportRequest exportRequest = new InstancesExportRequest();
exportRequest.setExportContext(new ExportContext()
    .setFileType("SQL")
    .setUri("gs://lc2-backups-prod/backups/liferay/dump.sql.gz")
    .setDatabases(List.of("lportal")));

sqlAdmin.instances().export("project-id", "instance-id", exportRequest).execute();
```

**GCS Transfer Service**
```java
// Transfer between buckets without streaming through operator
TransferJob transferJob = new TransferJob()
    .setDescription("Liferay Document Library Backup")
    .setProjectId("project-id")
    .setTransferSpec(new TransferSpec()
        .setGcsDataSource(new GcsData().setBucketName("liferay-doclib-prod"))
        .setGcsDataSink(new GcsData().setBucketName("lc2-backups-prod"))
        .setObjectConditions(new ObjectConditions()
            .setExcludePrefixes(List.of("thumbnails/", "previews/", ".cache/"))));
```

---

## 7. Deployment Considerations

### 7.1 Local Environment (k3d + Docker Compose)

#### 7.1.1 Architecture

Local development uses k3d (lightweight Kubernetes) with Docker Compose for supporting services:

```
┌─────────────────────────────────────────────┐
│           k3d Cluster (Local)               │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │   LC2 Backup Operator               │   │
│  │   (LocalProvider implementation)    │   │
│  └─────────────────────────────────────┘   │
│                │                            │
│                ├──────────────────┐         │
│                ▼                  ▼         │
│  ┌──────────────────┐   ┌────────────────┐ │
│  │  PostgreSQL Pod  │   │  Liferay Pod   │ │
│  └──────────────────┘   └────────────────┘ │
└─────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│        Docker Compose Services                          │
│                                                         │
│  ┌──────────┐  ┌──────────────┐  ┌──────────┐          │
│  │  MinIO   │  │ OpenSearch ⏸ │  │PostgreSQL│          │
│  │ (S3 API) │  │ (Post-MVP)   │  │(Metadata)│          │
│  └──────────┘  └──────────────┘  └──────────┘          │
└─────────────────────────────────────────────────────────┘
```

#### 7.1.2 Configuration

**docker-compose.yml**
```yaml
version: '3.8'
services:
  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password
    command: server /data --console-address ":9001"
    volumes:
      - minio-data:/data

  opensearch:  # ⏸️ NOT PART OF MVP - For post-MVP Elasticsearch backup testing
    image: opensearchproject/opensearch:2.11.0
    environment:
      - discovery.type=single-node
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"

  postgres-metadata:
    image: postgres:15
    environment:
      POSTGRES_DB: lc2_backups
      POSTGRES_USER: lc2
      POSTGRES_PASSWORD: lc2pass
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  minio-data:
  postgres-data:
```

**Operator Configuration for Local**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-operator-config
  namespace: lc2-system
data:
  provider: local
  minio.endpoint: http://minio:9000
  minio.accessKey: admin
  minio.secretKey: password
  postgres.host: postgres-metadata
  postgres.database: lc2_backups
```

#### 7.1.3 Local Provider Implementation

The local provider uses pg_dump/pg_restore instead of managed APIs:

```java
public class LocalBackupProvider implements BackupProvider {
    @Override
    public BackupResult createBackup(BackupRequest request) {
        // For local: run pg_dump in pod, upload to MinIO
        String dumpCommand = String.format(
            "pg_dump -h %s -U %s -d %s -Fc | gzip > /tmp/dump.sql.gz",
            dbHost, dbUser, dbName
        );

        // Execute in database pod
        execInPod(databasePod, dumpCommand);

        // Upload to MinIO
        minioClient.uploadObject(UploadObjectArgs.builder()
            .bucket("lc2-backups")
            .object("backups/" + backupId + "/dump.sql.gz")
            .filename("/tmp/dump.sql.gz")
            .build());
    }
}
```

### 7.2 Cloud Environment (AWS)

#### 7.2.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Account                              │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              EKS Cluster                               │ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │   LC2 Backup Operator (AWSProvider)             │  │ │
│  │  │   IAM Role: BackupOperatorRole                  │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────┘ │
│                        │                                    │
│         ┌──────────────┼──────────────┐                    │
│         ▼              ▼              ▼                    │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐              │
│  │   RDS    │   │    S3    │   │    RDS   │              │
│  │ (MySQL)  │   │ Backups  │   │(Postgres)│              │
│  │          │   │  Bucket  │   │Metadata  │              │
│  └──────────┘   └──────────┘   └──────────┘              │
│       │              ▲              │                      │
│       └──── API ─────┘              │                      │
│      (Export directly               │                      │
│       to S3, no pod                 │                      │
│       involvement!)                 │                      │
│                                     │                      │
│  ┌──────────────────────────────────┘                     │
│  │                                                         │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐          │
│  └─▶│   KMS    │   │   IAM    │   │CloudWatch│          │
│     │(Encrypt) │   │ (Roles)  │   │(Metrics) │          │
│     └──────────┘   └──────────┘   └──────────┘          │
└─────────────────────────────────────────────────────────────┘
```

#### 7.2.2 IAM Configuration

**Operator Service Account**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-operator
  namespace: lc2-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/BackupOperatorRole
```

**IAM Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:CreateDBSnapshot",
        "rds:DescribeDBSnapshots",
        "rds:DeleteDBSnapshot",
        "rds:StartExportTask",
        "rds:DescribeExportTasks"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::lc2-backups-prod",
        "arn:aws:s3:::lc2-backups-prod/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789:key/*"
    }
  ]
}
```

#### 7.2.3 Network Configuration

**VPC Endpoints** (for private connectivity):
- S3 VPC Endpoint (Gateway)
- RDS VPC Endpoint (Interface)
- KMS VPC Endpoint (Interface)

**Security Groups**:
```yaml
# Operator Security Group
ingress: []  # No inbound traffic
egress:
  - protocol: tcp
    port: 443
    description: "HTTPS to AWS APIs"
  - protocol: tcp
    port: 5432
    description: "PostgreSQL metadata DB"

# RDS Security Group
ingress:
  - protocol: tcp
    port: 3306
    source: operator-sg
    description: "MySQL from operator"
```

#### 7.2.4 Terraform Infrastructure

```hcl
# RDS Instance for Application
resource "aws_db_instance" "liferay" {
  identifier           = "liferay-prod"
  engine              = "mysql"
  engine_version      = "8.0"
  instance_class      = "db.r6g.xlarge"
  allocated_storage   = 1000
  storage_encrypted   = true
  kms_key_id         = aws_kms_key.backup.arn

  backup_retention_period = 7
  backup_window          = "03:00-04:00"

  enabled_cloudwatch_logs_exports = ["error", "slowquery"]

  tags = {
    Application = "liferay"
    Environment = "production"
  }
}

# S3 Bucket for Backups
resource "aws_s3_bucket" "backups" {
  bucket = "lc2-backups-prod"

  tags = {
    Purpose = "LC2 Backup Storage"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "backups" {
  bucket = aws_s3_bucket.backups.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.backup.arn
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "backups" {
  bucket = aws_s3_bucket.backups.id

  rule {
    id     = "daily-backups"
    status = "Enabled"

    expiration {
      days = 7
    }

    filter {
      prefix = "backups/daily/"
    }
  }

  rule {
    id     = "monthly-backups"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "GLACIER"
    }

    filter {
      prefix = "backups/monthly/"
    }
  }
}

# KMS Key for Encryption
resource "aws_kms_key" "backup" {
  description             = "LC2 Backup Encryption Key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

# IAM Role for Operator
resource "aws_iam_role" "backup_operator" {
  name = "BackupOperatorRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${aws_iam_openid_connect_provider.eks.url}:sub" = "system:serviceaccount:lc2-system:backup-operator"
        }
      }
    }]
  })
}
```

---

## 8. Deployment Considerations

(Content already exists above - keep as-is)

---

## 9. Monitoring and Observability

### 9.1 Metrics

#### 9.1.1 Prometheus Metrics

The operator shall expose Prometheus metrics on `/metrics` endpoint:

**Backup Metrics:**
```
# Backup operation metrics
lc2_backup_requests_total{project="retail",application="liferay",status="success|failed"}
lc2_backup_duration_seconds{project="retail",application="liferay",phase="snapshot|export|transfer"}
lc2_backup_size_bytes{project="retail",application="liferay",component="database|storage"}
lc2_backup_active_operations{project="retail",application="liferay"}

# Restore operation metrics
lc2_restore_requests_total{project="retail",application="liferay",status="success|failed"}
lc2_restore_duration_seconds{project="retail",application="liferay",phase="validation|snapshot|dump|storage"}
lc2_restore_active_operations{project="retail",application="liferay"}

# Resource utilization
lc2_operator_cpu_usage_seconds_total
lc2_operator_memory_usage_bytes
lc2_operator_reconcile_loop_duration_seconds
```

**Business Metrics:**
```
# Coverage metrics
lc2_backup_coverage_ratio{project="retail"}  # % of applications with recent backups
lc2_backup_age_seconds{project="retail",backup_id="xyz"}  # Age of most recent backup

# Compliance metrics
lc2_backup_retention_compliance{project="retail"}  # % backups meeting retention policy
lc2_backup_encryption_compliance{project="retail"}  # % backups encrypted
```

#### 9.1.2 Grafana Dashboards

Pre-built Grafana dashboards shall be provided:

1. **Operations Dashboard**
   - Active backup/restore operations
   - Success/failure rates
   - Operation durations (P50, P95, P99)
   - Queue depth and wait times

2. **Business Dashboard**
   - Backup coverage by project/application
   - RTO/RPO compliance
   - Storage utilization trends
   - Cost analysis (cloud provider costs)

3. **Troubleshooting Dashboard**
   - Error rates by type
   - Retry attempts and backoff behavior
   - Cloud provider API throttling
   - Resource utilization (CPU, memory, network)

### 9.2 Logging

#### 9.2.1 Structured Logging

All logs shall be emitted in JSON format with standard fields:

```json
{
  "timestamp": "2025-11-10T10:30:00.123Z",
  "level": "INFO",
  "logger": "io.lc2.backup.BackupReconciler",
  "message": "Backup operation completed successfully",
  "project_id": "retail",
  "backup_id": "backup-20251110-abc123",
  "application": "liferay",
  "provider": "aws",
  "duration_ms": 245000,
  "trace_id": "a1b2c3d4e5f6",
  "span_id": "1234567890"
}
```

#### 9.2.2 Log Levels

- **ERROR:** Operation failures, cloud provider API errors, data corruption detected
- **WARN:** Retry attempts, performance degradation, approaching quota limits
- **INFO:** Operation lifecycle events (started, in-progress, completed)
- **DEBUG:** Detailed operation steps, API request/response details
- **TRACE:** Internal state transitions, reconciliation loop iterations

#### 9.2.3 Log Aggregation

Logs shall be exported to:
- **Local:** stdout (captured by container runtime)
- **AWS:** CloudWatch Logs with log group per application
- **GCP:** Cloud Logging with structured log entries
- **Azure:** Azure Monitor Logs

Log retention: Minimum 30 days, configurable up to 365 days

### 9.3 Distributed Tracing

#### 9.3.1 OpenTelemetry Integration

The operator shall instrument backup/restore operations with OpenTelemetry spans:

```
Trace: Backup Operation (backup-20251110-abc123)
├── Span: Validate Request (5ms)
├── Span: Create Database Snapshot (120s)
│   ├── Span: Call RDS CreateDBSnapshot API (2s)
│   └── Span: Poll Snapshot Status (118s)
├── Span: Export Snapshot to S3 (3600s)
│   ├── Span: Call RDS StartExportTask API (1s)
│   └── Span: Poll Export Status (3599s)
├── Span: Transfer Document Library (7200s)
│   ├── Span: List Source Objects (30s)
│   └── Span: Monitor S3 Replication (7170s)
└── Span: Generate Metadata (10s)
```

Traces shall be exported to Jaeger, Zipkin, or cloud-native tracing (AWS X-Ray, GCP Cloud Trace, Azure Application Insights)

### 9.4 Alerting

#### 9.4.1 Critical Alerts

**Backup Failure Alert:**
```yaml
alert: BackupOperationFailed
expr: rate(lc2_backup_requests_total{status="failed"}[5m]) > 0
severity: critical
summary: "Backup operation failed for {{$labels.project}}/{{$labels.application}}"
action: "Investigate backup logs and cloud provider status"
```

**RTO Violation Alert:**
```yaml
alert: RestoreExceedsRTO
expr: lc2_restore_duration_seconds > 900  # 15 minutes RTO
severity: critical
summary: "Restore operation exceeding RTO for {{$labels.project}}"
```

**Backup Coverage Alert:**
```yaml
alert: BackupCoverageInsufficient
expr: lc2_backup_coverage_ratio < 0.95
severity: warning
summary: "Backup coverage below 95% for {{$labels.project}}"
```

#### 9.4.2 Warning Alerts

- Backup operation duration exceeding P95 baseline
- Cloud provider API throttling detected
- Storage utilization exceeding 80% of quota
- Encryption key expiration within 30 days

### 9.5 Health Checks

#### 9.5.1 Liveness Probe

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

Returns 200 OK if:
- JVM is responsive
- No deadlocks detected
- Critical threads are running

#### 9.5.2 Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

Returns 200 OK if:
- Metadata database is reachable
- Cloud provider APIs are accessible
- Kubernetes API server is reachable
- No critical configuration errors

---

## 10. Testing and Validation

### 10.1 Unit Testing

**Coverage Requirements:**
- Minimum 80% code coverage for core logic
- 100% coverage for provider implementations (AWS, GCP, Azure, Local)
- Test frameworks: JUnit 5, Mockito, AssertJ

**Test Categories:**
```java
@Test
void testBackupRequestValidation() {
    // Validate CRD spec validation logic
}

@Test
void testProviderSelection() {
    // Verify correct provider selected based on configuration
}

@Test
void testRetryLogic() {
    // Verify exponential backoff and max retries
}
```

### 10.2 Integration Testing

**Testcontainers Strategy:**
```java
@Testcontainers
class BackupIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Container
    static GenericContainer<?> minio = new GenericContainer<>("minio/minio:latest")
        .withExposedPorts(9000);

    @Container
    static GenericContainer<?> operator = new GenericContainer<>("lc2/backup-operator:latest")
        .dependsOn(postgres, minio);

    @Test
    void testFullBackupRestoreCycle() {
        // Create backup → Verify in MinIO → Restore → Verify data integrity
    }
}
```

**Test Scenarios:**
- Full backup and restore cycle (database + storage)
- Partial restore (database only, storage only)
- Concurrent backup operations
- Failure scenarios (cloud API errors, network timeouts)
- Retry and recovery logic

### 10.3 End-to-End Testing

**E2E Test Environment:**
- Real Kubernetes cluster (k3d for local, EKS for staging)
- Real cloud provider resources (separate test accounts)
- All three applications deployed (Liferay, Drupal, Bloomreach)

**Test Scenarios:**
1. **Happy Path:**
   - Schedule backup → Wait for completion → Verify backup exists → Restore to new environment → Verify application functionality

2. **Disaster Recovery:**
   - Simulate database corruption → Restore from most recent backup → Verify RTO/RPO compliance

3. **Cross-Cloud Migration:**
   - Backup on GCP → Restore on AWS → Verify application compatibility

4. **Large Dataset:**
   - Backup 1TB database + 10TB storage → Verify no OOM errors → Measure performance

### 10.4 Disaster Recovery Testing

**DR Test Schedule:**
- Monthly DR drills for all production projects
- Quarterly full datacenter failover tests
- Annual cross-cloud migration tests

**DR Test Checklist:**
```markdown
- [ ] Identify restore point (timestamp or backup ID)
- [ ] Validate backup integrity (checksum verification)
- [ ] Provision target environment
- [ ] Execute restore operation
- [ ] Verify application functionality (smoke tests)
- [ ] Measure RTO (time to restore)
- [ ] Measure RPO (data loss, if any)
- [ ] Document lessons learned
```

**DR Test Metrics:**
- Actual RTO vs target (15 minutes)
- Actual RPO vs target (5 minutes)
- Restore success rate
- Application functionality verification (% of features working)

### 10.5 Performance Testing

**Benchmark Scenarios:**
```
Scenario 1: Small Database
- Database: 10GB PostgreSQL
- Storage: 100GB files
- Expected Backup Time: < 5 minutes
- Expected Restore Time: < 10 minutes

Scenario 2: Medium Database
- Database: 500GB MySQL
- Storage: 5TB files
- Expected Backup Time: < 30 minutes
- Expected Restore Time: < 1 hour

Scenario 3: Large Database
- Database: 2TB PostgreSQL
- Storage: 50TB files
- Expected Backup Time: < 2 hours
- Expected Restore Time: < 4 hours
```

**Load Testing:**
- 10 concurrent backup operations
- 5 concurrent restore operations
- Verify no resource exhaustion
- Verify no cloud provider API throttling

### 10.6 Chaos Engineering

**Chaos Experiments:**
```
Experiment 1: Operator Pod Restart
- Trigger: Kill operator pod during backup operation
- Expected: Backup resumes from last checkpoint

Experiment 2: Network Partition
- Trigger: Block egress to cloud provider APIs
- Expected: Exponential backoff and retry, eventual success

Experiment 3: Cloud Provider Outage
- Trigger: Simulate S3/GCS unavailability
- Expected: Queue operations, retry when service recovers

Experiment 4: Metadata Database Failure
- Trigger: Stop PostgreSQL
- Expected: Operator becomes unready, no new operations accepted
```

---

## 11. Version Management and Upgrades

### 11.1 Semantic Versioning

The service shall follow Semantic Versioning (SemVer):
- **Major (X.0.0):** Breaking changes to CRDs, APIs, or behavior
- **Minor (x.Y.0):** New features, backward-compatible
- **Patch (x.y.Z):** Bug fixes, security patches

### 11.2 CRD Versioning

**CRD Version Strategy:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backuprequests.backup.lc2.io
spec:
  group: backup.lc2.io
  versions:
    - name: v1alpha1
      served: true
      storage: true
      # Current version
    - name: v1beta1
      served: true
      storage: false
      # Preview of next version
```

**Deprecation Policy:**
- Alpha versions (v1alpha1): May change without notice
- Beta versions (v1beta1): Backward compatible for 6 months
- Stable versions (v1): Backward compatible for 12 months minimum

### 11.3 Upgrade Procedures

#### 11.3.1 Operator Upgrade

**Blue-Green Deployment:**
```
1. Deploy new version alongside old version
2. Monitor new version for 24 hours (canary deployment)
3. Route 10% of traffic to new version
4. Gradually increase to 100% over 1 week
5. Decommission old version
```

**Rollback Procedure:**
```
1. Scale down new version to 0 replicas
2. Scale up old version
3. Investigate failure
4. Fix issue in new version
5. Retry upgrade
```

#### 11.3.2 Database Migration

**Flyway Migration Strategy:**
```sql
-- V1__initial_schema.sql
CREATE TABLE backup_requests (...)

-- V2__add_encryption_metadata.sql
ALTER TABLE backup_requests ADD COLUMN encryption_key_id VARCHAR(256);

-- V3__add_compliance_fields.sql
ALTER TABLE backup_requests ADD COLUMN compliance_tags JSONB;
```

**Zero-Downtime Migration:**
- Use backward-compatible schema changes
- Add new columns as nullable
- Populate data in background job
- Make columns non-nullable in next version

### 11.4 Backward Compatibility

**API Compatibility Matrix:**
| Operator Version | v1alpha1 CRD | v1beta1 CRD | REST API v1 |
|-----------------|-------------|------------|-------------|
| 1.0.x           | ✓           | ✗          | ✓           |
| 1.1.x           | ✓           | ✓          | ✓           |
| 2.0.x           | ✗           | ✓          | ✓           |

**Compatibility Testing:**
- Test new operator version with old CRD versions
- Test old clients with new operator version
- Automated compatibility tests in CI/CD pipeline

### 11.5 Configuration Management

**Immutable Configuration:**
- CRD specs are immutable after creation
- Updates create new backup/restore requests
- Historical configurations preserved for audit

**Configuration Migration:**
```java
public class ConfigMigration {
    public void migrateV1AlphaToV1Beta(BackupRequestV1Alpha old) {
        BackupRequestV1Beta newRequest = new BackupRequestV1Beta();
        // Map fields
        newRequest.setSpec(convertSpec(old.getSpec()));
        // Apply defaults for new fields
        newRequest.getSpec().setComplianceTags(Map.of("migrated", "true"));
    }
}
```

---

## 12. Cloud-Agnostic vs Cloud-Specific Design

### 12.1 Cloud-Agnostic Elements

The service maintains cloud-agnostic abstractions while leveraging cloud-specific implementations:

#### 12.1.1 Provider Interface

```java
public interface BackupProvider {
    // Cloud-agnostic interface
    CompletionStage<BackupResult> createBackup(BackupRequest request);
    CompletionStage<RestoreResult> restoreBackup(RestoreRequest request);
    CompletionStage<List<Backup>> listBackups(String projectId);
    CompletionStage<Void> deleteBackup(String backupId);
}

// Implementations: AWSBackupProvider, GCPBackupProvider, AzureBackupProvider, LocalBackupProvider
```

#### 12.1.2 Common Data Models

```java
// Cloud-agnostic backup metadata
public class BackupMetadata {
    private String backupId;
    private String projectId;
    private Application application;
    private Instant createdAt;
    private BackupStatus status;
    private long sizeBytes;
    private String checksum;

    // Provider-specific details stored as JSON
    private Map<String, Object> providerDetails;
}
```

#### 12.1.3 Unified API

All BackupRequest and RestoreRequest CRDs are cloud-agnostic. Provider-specific details are abstracted:

```yaml
# Same CRD works across all clouds
apiVersion: backup.lc2.io/v1alpha1
kind: BackupRequest
spec:
  application: liferay
  provider: aws  # or gcp, azure, local
  # Rest of spec is identical
```

### 12.2 Cloud-Specific Implementation

#### 12.2.1 Provider Factory Pattern

```java
@ApplicationScoped
public class BackupProviderFactory {
    @Inject
    AWSBackupProvider awsProvider;

    @Inject
    GCPBackupProvider gcpProvider;

    @Inject
    AzureBackupProvider azureProvider;

    @Inject
    LocalBackupProvider localProvider;

    public BackupProvider getProvider(CloudProvider provider) {
        return switch (provider) {
            case AWS -> awsProvider;
            case GCP -> gcpProvider;
            case AZURE -> azureProvider;
            case LOCAL -> localProvider;
        };
    }
}
```

#### 12.2.2 AWS-Specific Optimizations

**RDS Export API** (eliminates streaming):
```java
public class AWSBackupProvider implements BackupProvider {
    @Override
    public CompletionStage<BackupResult> createBackup(BackupRequest request) {
        // 1. Create RDS snapshot (fast)
        String snapshotId = rdsClient.createDBSnapshot(...).dbSnapshotIdentifier();

        // 2. Export snapshot to S3 (managed by AWS, no pod involvement)
        String exportId = rdsClient.startExportTask(StartExportTaskRequest.builder()
            .sourceArn(getSnapshotArn(snapshotId))
            .s3BucketName(config.backupBucket())
            .iamRoleArn(config.exportRoleArn())
            .kmsKeyId(config.kmsKeyId())
            .build()).exportTaskIdentifier();

        // 3. Poll export status asynchronously
        return pollExportCompletion(exportId);
    }
}
```

**S3 Lifecycle Policies** (automated retention):
- Managed via Terraform/CloudFormation
- No operator involvement in deletion

#### 12.2.3 GCP-Specific Optimizations

**Cloud SQL Export API**:
```java
public class GCPBackupProvider implements BackupProvider {
    @Override
    public CompletionStage<BackupResult> createBackup(BackupRequest request) {
        // Export database directly to GCS (no streaming)
        Operation operation = sqlAdmin.instances().export(
            projectId,
            instanceId,
            new InstancesExportRequest()
                .setExportContext(new ExportContext()
                    .setFileType("SQL")
                    .setUri("gs://" + bucket + "/backups/" + backupId + "/dump.sql.gz"))
        ).execute();

        return pollOperationCompletion(operation);
    }
}
```

**GCS Transfer Service** (for document library):
```java
TransferJob transferJob = storagetransfer.transferJobs().create(
    new TransferJob()
        .setTransferSpec(new TransferSpec()
            .setGcsDataSource(new GcsData().setBucketName(sourceBucket))
            .setGcsDataSink(new GcsData().setBucketName(backupBucket))
            .setObjectConditions(new ObjectConditions()
                .setExcludePrefixes(excludePatterns)))
).execute();
```

#### 12.2.4 Local Environment Fallback

For local development (no managed APIs available):

```java
public class LocalBackupProvider implements BackupProvider {
    @Override
    public CompletionStage<BackupResult> createBackup(BackupRequest request) {
        // Execute pg_dump in database pod
        String dumpCommand = buildDumpCommand(request);
        String output = kubernetesClient.pods()
            .inNamespace(namespace)
            .withName(databasePod)
            .inContainer("postgres")
            .exec("sh", "-c", dumpCommand)
            .getOutput();

        // Upload to MinIO
        minioClient.uploadObject(...);

        return CompletableFuture.completedFuture(result);
    }
}
```

### 12.3 Design Principles

1. **Interface Segregation:** Define minimal, focused interfaces (BackupProvider, StorageProvider, DatabaseProvider)
2. **Dependency Inversion:** Depend on abstractions (interfaces), not concrete implementations
3. **Strategy Pattern:** Runtime selection of provider implementation
4. **Configuration-Driven:** Provider selection via environment variables/ConfigMaps
5. **Graceful Degradation:** Fall back to basic implementations when managed services unavailable

---

## 13. Use Case Scenarios

### 13.1 Daily Scheduled Backup

**Scenario:** Liferay production environment requires daily backups at 2 AM UTC

**Steps:**
1. Create BackupRequest CR with cron schedule:
```yaml
apiVersion: backup.lc2.io/v1alpha1
kind: BackupRequest
metadata:
  name: liferay-prod-daily
spec:
  application: liferay
  projectId: retail-prod
  provider: aws
  schedule: "0 2 * * *"  # Daily at 2 AM UTC
  database:
    snapshotEnabled: true
    dumpEnabled: true
  storage:
    excludeGeneratedFiles: true
  retention:
    daily: 7
    weekly: 4
    monthly: 12
```

2. Operator creates snapshot at 2:00 AM
3. Operator initiates RDS export to S3 (direct, no streaming)
4. Operator triggers S3 replication for document library
5. Operator generates metadata and marks backup as completed
6. Old backups automatically deleted per retention policy

**Expected Outcome:**
- Backup completed within 30 minutes
- Zero impact on application performance
- No OOM errors
- Backup encrypted at rest in S3

### 13.2 Disaster Recovery - Database Corruption

**Scenario:** Production Drupal database corrupted at 10:30 AM, need to restore from most recent backup

**Steps:**
1. Identify most recent backup:
```bash
kubectl get backuprequests -n lc2-system \
  -l project=marketing-prod,application=drupal \
  --sort-by=.status.completionTime
```

2. Create RestoreRequest:
```yaml
apiVersion: backup.lc2.io/v1alpha1
kind: RestoreRequest
metadata:
  name: drupal-dr-20251110
spec:
  backupId: backup-20251110-020000
  targetEnvironment: marketing-prod
  restoreType: database-only  # Only restore database
  dryRun: false
```

3. Operator validates backup integrity
4. Operator restores RDS snapshot (5 minutes)
5. Application reconnects to restored database
6. Team verifies data integrity

**Expected Outcome:**
- RTO: 10 minutes (within 15-minute target)
- RPO: 8 hours (last backup was at 2 AM)
- Application back online
- Zero data loss since last backup

### 13.3 Cross-Environment Restore (Prod → Staging)

**Scenario:** Refresh staging environment with production data for testing

**Steps:**
1. Create RestoreRequest with sanitization:
```yaml
apiVersion: backup.lc2.io/v1alpha1
kind: RestoreRequest
metadata:
  name: staging-refresh
spec:
  backupId: backup-20251110-020000
  targetEnvironment: bloomreach-staging
  restoreType: full
  sanitize: true  # Apply data sanitization
```

2. Operator provisions new RDS instance in staging VPC
3. Operator restores snapshot
4. Operator applies data sanitization:
   - Remove PII (personal identifiable information)
   - Anonymize email addresses
   - Reset passwords
5. Operator restores object storage
6. Staging environment ready for testing

**Expected Outcome:**
- Staging environment matches production structure
- Sensitive data sanitized
- Testing can proceed with realistic data

### 13.4 Cross-Cloud Migration (GCP → AWS)

**Scenario:** Migrate Liferay from GCP to AWS

**Steps:**
1. Create backup on GCP:
```yaml
spec:
  application: liferay
  projectId: retail
  provider: gcp
  database:
    dumpEnabled: true  # Use dump for portability
```

2. Download backup metadata:
```bash
gsutil cp gs://lc2-backups-gcp/backups/retail/backup-xyz/metadata.json .
```

3. Upload backup to AWS S3:
```bash
aws s3 sync gs://lc2-backups-gcp/backups/retail/backup-xyz/ \
  s3://lc2-backups-aws/backups/retail/backup-xyz/
```

4. Create RestoreRequest on AWS:
```yaml
spec:
  backupId: backup-xyz
  targetEnvironment: retail-aws
  provider: aws
```

5. Operator provisions RDS on AWS
6. Operator restores from dump (portable format)
7. Operator restores S3 objects
8. Application migrated to AWS

**Expected Outcome:**
- Successful migration between clouds
- Data integrity preserved
- Application functional on new cloud provider

### 13.5 Compliance Audit

**Scenario:** Generate compliance report for SOC 2 audit

**Steps:**
1. Query backup coverage:
```bash
curl -X GET https://lc2-api/v1/compliance/reports \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"period": "2025-Q4", "standard": "SOC2"}'
```

2. System generates report:
```json
{
  "period": "2025-Q4",
  "projects": [
    {
      "projectId": "retail-prod",
      "backupCoverage": 100,
      "encryptionCompliance": 100,
      "retentionCompliance": 100,
      "rtoCompliance": 99.9,
      "drTestsPassed": 12,
      "drTestsFailed": 0
    }
  ],
  "auditLog": [
    {
      "timestamp": "2025-11-10T10:30:00Z",
      "user": "admin@company.com",
      "operation": "restore",
      "resource": "backup-20251110-abc123",
      "status": "success"
    }
  ]
}
```

3. Export audit logs:
```bash
kubectl logs -n lc2-system backup-operator \
  --since=90d > audit-logs-q4.json
```

**Expected Outcome:**
- Compliance report demonstrates 100% backup coverage
- All backups encrypted
- Audit trail complete and immutable
- SOC 2 audit passes

### 13.6 Large-Scale Backup (100TB)

**Scenario:** Backup enterprise Liferay with 2TB database + 100TB document library

**Steps:**
1. Create BackupRequest:
```yaml
spec:
  application: liferay
  projectId: enterprise
  provider: gcp
  database:
    type: postgresql
    instance: liferay-enterprise
    snapshotEnabled: true
    dumpEnabled: true
  storage:
    type: gcs
    bucket: liferay-doclib-enterprise
    excludeGeneratedFiles: true
    parallelTransfer: true
    maxConcurrency: 50  # High concurrency for large transfer
```

2. Operator creates database snapshot (5 minutes)
3. Operator triggers Cloud SQL export (1 hour, direct to GCS)
4. Operator initiates GCS Transfer Service (8 hours, parallel transfer)
5. Backup completes, metadata generated

**Expected Outcome:**
- Total time: ~9 hours
- No OOM errors (no data streaming through operator)
- Operator memory usage: < 512MB throughout
- Cloud provider handles all data movement
- Backup size reduced 50% by excluding generated files

### 13.7 Automated Retention Policy Enforcement

**Scenario:** Automatically delete old backups per retention policy

**Steps:**
1. Retention policy defined in BackupRequest:
```yaml
retention:
  daily: 7    # Keep daily backups for 7 days
  weekly: 4   # Keep weekly backups for 4 weeks
  monthly: 12 # Keep monthly backups for 12 months
```

2. Operator runs daily cleanup job at midnight
3. Operator identifies backups older than retention period
4. Operator soft-deletes expired backups (30-day grace period)
5. After 30 days, operator hard-deletes backups:
   - Deletes S3/GCS objects
   - Destroys encryption keys
   - Preserves audit log entries

**Expected Outcome:**
- Storage costs optimized
- Compliance with data retention policies
- Audit trail maintained
- Accidental deletions prevented (30-day grace period)

---

## 14. Glossary

**Backup Window:** Time period during which backup operations are permitted to run

**CRD (Custom Resource Definition):** Kubernetes extension mechanism for defining custom resource types

**Document Library:** File storage system used by Liferay DXP for user-uploaded content

**Dual Backup Strategy:** Approach using both snapshots (fast restore) and dumps (portability)

**Enhanced Hybrid Approach:** Architecture leveraging managed service APIs to eliminate data streaming through operator pods

**Managed Service API:** Cloud provider API for database/storage operations (e.g., RDS Export API, Cloud SQL Export API)

**OOM (Out Of Memory):** Error condition when application exceeds available memory

**Operator Pattern:** Kubernetes pattern for encoding operational knowledge in software

**Provider Abstraction:** Interface-based design allowing cloud-agnostic implementation with cloud-specific optimizations

**Reconciliation Loop:** Kubernetes controller pattern for continuously converging desired and actual state

**Recovery Point Objective (RPO):** Maximum acceptable data loss measured in time

**Recovery Time Objective (RTO):** Maximum acceptable downtime for recovery operations

**Snapshot:** Point-in-time copy of database/storage (cloud-provider specific, fast but not portable)

**Transfer Service:** Managed service for copying data between storage locations (e.g., GCS Transfer Service, S3 Transfer Acceleration)

---

## 15. References

### Internal Documentation

- `reference/FINAL_ARCHITECTURE_JAVA.md` - Detailed Java-based architecture specification
- `reference/MANAGED_SERVICES_VS_CUSTOM_ANALYSIS.md` - Analysis of managed service approach vs custom implementation
- `reference/backup-requirements.md` - Original backup requirements document
- `reference/synthesis-all-learnings.md` - Consolidated learnings from research phase

### External Resources

#### AWS
- [RDS Snapshots](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html)
- [Exporting DB Snapshot Data to S3](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ExportSnapshot.html)
- [AWS Backup](https://docs.aws.amazon.com/aws-backup/)

#### GCP
- [Cloud SQL Export](https://cloud.google.com/sql/docs/mysql/import-export/export-import-sql-csv)
- [GCS Transfer Service](https://cloud.google.com/storage-transfer-service/docs)
- [Cloud SQL Backups](https://cloud.google.com/sql/docs/mysql/backup-recovery/backups)

#### Kubernetes
- [Kubernetes Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Custom Resource Definitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Fabric8 Kubernetes Client](https://github.com/fabric8io/kubernetes-client)

#### Quarkus
- [Quarkus Kubernetes](https://quarkus.io/guides/deploying-to-kubernetes)
- [Quarkus Operator SDK](https://github.com/quarkiverse/quarkus-operator-sdk)

#### Application-Specific
- [Liferay DXP Backup Documentation](https://learn.liferay.com/dxp/latest/en/installation-and-upgrades/maintaining-a-liferay-installation/backing-up.html)
- [Drupal Backup and Migrate](https://www.drupal.org/docs/administering-a-drupal-site/backing-up-and-restoring-your-site)
- [Bloomreach Documentation](https://documentation.bloomreach.com/)

---

**Document Version History**

| Version | Date       | Author | Changes                          |
|---------|------------|--------|----------------------------------|
| 1.0     | 2025-11-10 | LC2    | Initial comprehensive requirements document |

