# Data Model Suggestion 2: Event-Sourced / CQRS Architecture

> Project: Virtual Data Room (476) -- Generated: 2026-05-26

## Overview

This model stores every state change in the Virtual Data Room as an immutable, append-only event in a centralized event store. Current state is never mutated in place -- instead, it is reconstructed by replaying the ordered sequence of events from the beginning or from a snapshot. The Command Query Responsibility Segregation (CQRS) pattern separates the write path (commands that produce events) from the read path (materialized projections optimized for specific query patterns).

This architecture is a natural fit for a Virtual Data Room because the domain's core compliance requirements -- immutable audit trails for SEC 17a-4, SOC 2 Type II, and ISO 27001 -- demand exactly what event sourcing provides by default: a tamper-proof, chronologically ordered record of every action ever taken. In a traditional model, the audit log is a secondary artifact bolted alongside mutable state tables. In event sourcing, the audit log *is* the source of truth.

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event store | EventStoreDB 24.x or PostgreSQL with append-only tables | EventStoreDB provides built-in stream management, subscriptions, and projections; PostgreSQL offers operational familiarity |
| Command bus | MassTransit / Wolverine (.NET) or Axon Framework (Java) | Typed command dispatch, saga orchestration, idempotent retries |
| Read model DB | PostgreSQL 16+ | Relational projections for rooms, documents, permissions; familiar SQL querying |
| Analytics projection | ClickHouse or TimescaleDB | Columnar storage for high-volume engagement analytics and page-view heatmaps |
| Search projection | Elasticsearch / OpenSearch | Full-text search across document content and metadata |
| Snapshot store | PostgreSQL or S3 (JSON) | Periodic aggregate snapshots to avoid full replay on large streams |
| Message broker | Apache Kafka or NATS JetStream | Durable event distribution to projectors, integration services, and external systems |
| Cache | Redis | Materialized permission cache, session cache, hot projection data |
| File storage | S3-compatible object store (AWS S3, MinIO) | Encrypted document blobs; event store holds metadata references |
| Encryption / KMS | AWS KMS / HashiCorp Vault | Per-tenant encryption key management for events and documents |

## Core Concepts

### Aggregates and Streams

Each aggregate root (DataRoom, Document, QAThread, UserAccess) owns a stream of events. The stream ID encodes the aggregate type and identity:

```
DataRoom-{roomId}
Document-{documentId}
QAThread-{threadId}
UserAccess-{roomId}-{userId}
Permissions-{roomId}
Organization-{orgId}
```

### Command Flow

```
Client --> API Gateway --> Command Handler --> Aggregate Root
                                                  |
                                           Validate invariants
                                                  |
                                           Emit domain events
                                                  |
                                         Append to event store
                                                  |
                                    Publish to message broker
                                                  |
                              +-------------------+-------------------+
                              |                   |                   |
                        Read Model           Analytics           Integration
                        Projector            Projector            Projector
                              |                   |                   |
                        PostgreSQL           ClickHouse          Webhooks/
                        (queries)            (analytics)         Salesforce
```

---

## Event Store Schema

### PostgreSQL-Based Event Store

If using PostgreSQL as the event store (for teams that prefer operational simplicity over a dedicated event store product):

```sql
-- ============================================================
-- EVENT STORE: Core append-only table
-- ============================================================
-- This is the single source of truth for all state in the system.
-- Every state change is recorded here as an immutable event.
-- Partitioned by created_at for retention management.

CREATE TABLE events (
    -- Global sequential position (for ordered replay across all streams)
    global_position     BIGSERIAL NOT NULL,
    -- Stream identity
    stream_name         VARCHAR(500) NOT NULL,
    stream_category     VARCHAR(100) NOT NULL,  -- e.g., 'DataRoom', 'Document', 'QAThread'
    -- Position within this stream (optimistic concurrency control)
    stream_position     BIGINT NOT NULL,
    -- Event payload
    event_type          VARCHAR(200) NOT NULL,   -- e.g., 'DocumentUploaded', 'PermissionGranted'
    event_data          JSONB NOT NULL,           -- The event payload
    event_metadata      JSONB NOT NULL DEFAULT '{}',
    -- Metadata fields extracted for indexing
    organization_id     UUID NOT NULL,
    data_room_id        UUID,                     -- NULL for org-level events
    caused_by_user_id   UUID NOT NULL,
    caused_by_ip        INET,
    correlation_id      UUID NOT NULL,            -- Groups events from one user action
    causation_id        UUID NOT NULL,            -- Links to the command that caused this event
    -- Integrity
    event_id            UUID NOT NULL DEFAULT gen_random_uuid(),
    -- Hash chain for tamper detection (SEC 17a-4 WORM compliance)
    previous_hash       VARCHAR(64),              -- SHA-256 of the previous event in this stream
    event_hash          VARCHAR(64) NOT NULL,      -- SHA-256 of this event (stream_name + position + data + previous_hash)
    -- Timestamp
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Composite primary key for partitioning
    PRIMARY KEY (global_position, created_at),
    -- Unique constraint for optimistic concurrency within a stream
    UNIQUE (stream_name, stream_position)
) PARTITION BY RANGE (created_at);

-- Monthly partitions (automated via pg_partman)
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... automated partition creation via pg_partman

-- Indexes for common access patterns
CREATE INDEX idx_events_stream ON events(stream_name, stream_position);
CREATE INDEX idx_events_category ON events(stream_category, created_at);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_org ON events(organization_id, created_at);
CREATE INDEX idx_events_room ON events(data_room_id, created_at) WHERE data_room_id IS NOT NULL;
CREATE INDEX idx_events_user ON events(caused_by_user_id, created_at);
CREATE INDEX idx_events_correlation ON events(correlation_id);

-- WORM enforcement: prevent updates and deletes
CREATE RULE events_no_update AS ON UPDATE TO events DO INSTEAD NOTHING;
CREATE RULE events_no_delete AS ON DELETE TO events DO INSTEAD NOTHING;

-- ============================================================
-- SNAPSHOTS: Periodic aggregate state captures
-- ============================================================
-- Snapshots accelerate aggregate rehydration by providing a
-- starting point so that only events after the snapshot need
-- to be replayed.

CREATE TABLE snapshots (
    stream_name         VARCHAR(500) NOT NULL,
    stream_position     BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    snapshot_version    INTEGER NOT NULL DEFAULT 1,  -- Schema version of the snapshot format
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_name, stream_position)
);

-- Only keep the most recent snapshot per stream; older ones are pruned
CREATE INDEX idx_snapshots_stream ON snapshots(stream_name, stream_position DESC);

-- ============================================================
-- SUBSCRIPTIONS: Checkpoint tracking for projectors
-- ============================================================
-- Each projector (read model builder) tracks its position in
-- the global event stream so it can resume after restarts.

CREATE TABLE subscription_checkpoints (
    subscription_name   VARCHAR(200) PRIMARY KEY,
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status              VARCHAR(20) NOT NULL DEFAULT 'running'
                        CHECK (status IN ('running', 'paused', 'error', 'rebuilding')),
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- IDEMPOTENCY: Prevent duplicate command processing
-- ============================================================
CREATE TABLE processed_commands (
    command_id          UUID PRIMARY KEY,
    command_type        VARCHAR(200) NOT NULL,
    processed_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    result_event_ids    UUID[] NOT NULL DEFAULT '{}'
);

-- Auto-expire old entries (commands older than 7 days)
CREATE INDEX idx_processed_commands_expiry ON processed_commands(processed_at);
```

---

## Event Type Catalogue

### Organization Events

```typescript
// Stream: Organization-{orgId}
interface OrganizationCreated {
    organizationId: string;
    name: string;
    slug: string;
    planTier: 'free' | 'professional' | 'enterprise';
    billingEmail: string;
    dataResidencyRegion: string;
}

interface OrganizationPlanChanged {
    organizationId: string;
    previousTier: string;
    newTier: string;
    effectiveAt: string;
}

interface OrganizationMemberInvited {
    organizationId: string;
    userId: string;
    email: string;
    orgRole: 'owner' | 'admin' | 'member' | 'billing';
    invitedBy: string;
}

interface OrganizationMemberAccepted {
    organizationId: string;
    userId: string;
    acceptedAt: string;
}

interface OrganizationSuspended {
    organizationId: string;
    reason: string;
    suspendedBy: string;
}
```

### Data Room Lifecycle Events

```typescript
// Stream: DataRoom-{roomId}
interface DataRoomCreated {
    roomId: string;
    organizationId: string;
    name: string;
    description: string;
    dealType: 'mna' | 'fundraising' | 'ipo' | 'regulatory' | 'board_reporting' | 'investor_relations';
    dealValue?: number;
    dealCurrency?: string;
    settings: {
        watermarkEnabled: boolean;
        watermarkTemplate: string;
        invisibleWatermark: boolean;
        downloadEnabled: boolean;
        printEnabled: boolean;
        ndaRequired: boolean;
        aiClassificationEnabled: boolean;
        aiRedactionEnabled: boolean;
        aiReadinessScoring: boolean;
        retentionDays: number;
    };
    createdBy: string;
}

interface DataRoomOpened {
    roomId: string;
    openedAt: string;
    openedBy: string;
}

interface DataRoomClosed {
    roomId: string;
    closedAt: string;
    closedBy: string;
    reason?: string;
}

interface DataRoomArchived {
    roomId: string;
    archivedAt: string;
    archivedBy: string;
    retentionUntil: string;
}

interface DataRoomSettingsUpdated {
    roomId: string;
    changes: Record<string, { previous: unknown; updated: unknown }>;
    updatedBy: string;
}

interface NdaDocumentAttached {
    roomId: string;
    documentId: string;
    attachedBy: string;
}
```

### Folder Events

```typescript
// Stream: DataRoom-{roomId}  (folder events belong to the room aggregate)
interface FolderCreated {
    folderId: string;
    roomId: string;
    parentFolderId?: string;
    name: string;
    indexNumber: string;
    path: string;
    depth: number;
    createdBy: string;
}

interface FolderRenamed {
    folderId: string;
    roomId: string;
    previousName: string;
    newName: string;
    renamedBy: string;
}

interface FolderMoved {
    folderId: string;
    roomId: string;
    previousParentId: string;
    newParentId: string;
    previousPath: string;
    newPath: string;
    movedBy: string;
}

interface FolderHidden {
    folderId: string;
    roomId: string;
    hiddenBy: string;
}

interface FolderDeleted {
    folderId: string;
    roomId: string;
    deletedBy: string;
    cascadedDocumentIds: string[];
}
```

### Document Lifecycle Events

```typescript
// Stream: Document-{documentId}
interface DocumentUploadStarted {
    documentId: string;
    roomId: string;
    organizationId: string;
    folderId: string;
    originalFilename: string;
    displayName: string;
    mimeType: string;
    fileSizeBytes: number;
    storageBucket: string;
    storageKey: string;
    encryptionKeyId: string;
    checksumSha256: string;
    indexNumber: string;
    version: number;
    parentDocumentId?: string;  // for versioned documents
    uploadedBy: string;
}

interface DocumentConversionCompleted {
    documentId: string;
    pdfStorageKey: string;
    pdfPageCount: number;
    conversionDurationMs: number;
}

interface DocumentIndexingCompleted {
    documentId: string;
    contentTextLength: number;
    languageDetected: string;
}

interface DocumentClassified {
    documentId: string;
    classification: string;
    confidence: number;
    tags: string[];
    summary: string;
    classifiedBy: 'ai' | 'manual';
    modelVersion?: string;
}

interface DocumentClassificationCorrected {
    documentId: string;
    previousClassification: string;
    correctedClassification: string;
    correctedTags: string[];
    correctedBy: string;
    // This event feeds back into the AI training pipeline
}

interface DocumentRedactionDetected {
    documentId: string;
    redactions: Array<{
        redactionId: string;
        pageNumber: number;
        xOffset: number;
        yOffset: number;
        width: number;
        height: number;
        piiType: string;
        originalTextEncrypted: string;  // encrypted at application layer
        confidence: number;
        detectionMethod: 'ai_ner' | 'regex';
    }>;
    totalDetected: number;
    modelVersion?: string;
}

interface DocumentRedactionReviewed {
    documentId: string;
    redactionId: string;
    reviewStatus: 'approved' | 'rejected' | 'modified';
    modifiedBounds?: { xOffset: number; yOffset: number; width: number; height: number };
    reviewedBy: string;
}

interface DocumentRedactionApplied {
    documentId: string;
    redactedStorageKey: string;
    totalRedactionsApplied: number;
    appliedBy: string;
}

interface DocumentReady {
    documentId: string;
    processingDurationMs: number;
    pageCount: number;
}

interface DocumentProcessingFailed {
    documentId: string;
    stage: 'uploading' | 'converting' | 'indexing' | 'classifying' | 'redacting';
    errorMessage: string;
    retryable: boolean;
}

interface DocumentSuperseded {
    documentId: string;
    supersededByDocumentId: string;
}

interface DocumentDeleted {
    documentId: string;
    deletedBy: string;
    reason?: string;
}

interface DocumentRemoteWiped {
    documentId: string;
    wipeRequestId: string;
    devicesTargeted: number;
}
```

### Access Control Events

```typescript
// Stream: Permissions-{roomId}
interface RoleCreated {
    roleId: string;
    roomId: string;
    name: string;
    permissions: {
        canView: boolean;
        canDownload: boolean;
        canPrint: boolean;
        canUpload: boolean;
        canDelete: boolean;
        canManageUsers: boolean;
        canManageQa: boolean;
        canViewAnalytics: boolean;
        canManageRedactions: boolean;
    };
    isSystemRole: boolean;
    createdBy: string;
}

interface RolePermissionsUpdated {
    roleId: string;
    roomId: string;
    changes: Record<string, { previous: boolean; updated: boolean }>;
    updatedBy: string;
}

interface UserGroupCreated {
    groupId: string;
    roomId: string;
    name: string;
    roleId: string;
    createdBy: string;
}

interface UserInvitedToRoom {
    roomId: string;
    userId: string;
    email: string;
    groupId?: string;
    roleId: string;
    accessStartsAt?: string;
    accessExpiresAt?: string;
    invitedBy: string;
}

interface UserAcceptedRoomInvitation {
    roomId: string;
    userId: string;
    acceptedAt: string;
}

interface UserNdaAccepted {
    roomId: string;
    userId: string;
    ndaDocumentId: string;
    acceptedAt: string;
    ipAddress: string;
}

interface UserAccessRevoked {
    roomId: string;
    userId: string;
    revokedBy: string;
    reason?: string;
    triggerRemoteWipe: boolean;
}

interface ResourcePermissionGranted {
    roomId: string;
    // Target
    targetType: 'user' | 'group' | 'role';
    targetId: string;
    // Resource
    resourceType: 'folder' | 'document';
    resourceId: string;
    // Permissions
    canView?: boolean;
    canDownload?: boolean;
    canPrint?: boolean;
    canUpload?: boolean;
    inherit: boolean;
    grantedBy: string;
}

interface ResourcePermissionRevoked {
    roomId: string;
    targetType: 'user' | 'group' | 'role';
    targetId: string;
    resourceType: 'folder' | 'document';
    resourceId: string;
    revokedBy: string;
}
```

### Document Engagement Events

```typescript
// Stream: UserAccess-{roomId}-{userId}
interface DocumentViewStarted {
    sessionId: string;
    documentId: string;
    roomId: string;
    userId: string;
    ipAddress: string;
    userAgent: string;
    deviceType: string;
    watermarkId: string;
}

interface DocumentPageViewed {
    sessionId: string;
    documentId: string;
    pageNumber: number;
    viewDurationMs: number;
    scrollDepthPct: number;
}

interface DocumentViewEnded {
    sessionId: string;
    documentId: string;
    totalDurationSeconds: number;
    totalPagesViewed: number;
}

interface DocumentDownloaded {
    documentId: string;
    roomId: string;
    userId: string;
    watermarkId: string;
    ipAddress: string;
    fileFormat: string;  // 'original' | 'pdf' | 'watermarked_pdf'
}

interface DocumentPrinted {
    documentId: string;
    roomId: string;
    userId: string;
    watermarkId: string;
    pageRange: string;  // e.g., "1-5,8,12-15"
}
```

### Q&A Workflow Events

```typescript
// Stream: QAThread-{threadId}
interface QuestionSubmitted {
    questionId: string;
    roomId: string;
    organizationId: string;
    topicId?: string;
    questionNumber: string;
    subject: string;
    body: string;
    relatedFolderId?: string;
    relatedDocumentId?: string;
    priority: 'low' | 'normal' | 'high' | 'urgent';
    visibility: 'all_parties' | 'submitter_and_assignee' | 'sell_side_only' | 'buy_side_only';
    slaDeadline?: string;
    submittedBy: string;
}

interface QuestionAssigned {
    questionId: string;
    assignedTo?: string;
    assignedGroup?: string;
    assignedBy: string;
}

interface QuestionAnswered {
    questionId: string;
    answerId: string;
    body: string;
    attachmentDocumentIds: string[];
    answeredBy: string;
}

interface AnswerApproved {
    questionId: string;
    answerId: string;
    approvedBy: string;
}

interface AnswerRejected {
    questionId: string;
    answerId: string;
    reason: string;
    rejectedBy: string;
}

interface QuestionClosed {
    questionId: string;
    closedBy: string;
    resolution: string;
}

interface QuestionSlaBreached {
    questionId: string;
    slaDeadline: string;
    currentStatus: string;
    hoursOverdue: number;
}
```

### Messaging Events

```typescript
// Stream: DataRoom-{roomId}  (messages are part of the room aggregate)
interface MessageSent {
    messageId: string;
    roomId: string;
    threadId?: string;
    bodyEncrypted: string;  // base64-encoded encrypted content
    bodyNonce: string;
    attachmentDocumentIds: string[];
    visibility: 'all' | 'group' | 'direct';
    targetGroupId?: string;
    targetUserId?: string;
    sentBy: string;
}
```

### Security Events

```typescript
// Stream: UserAccess-{roomId}-{userId}
interface LoginSucceeded {
    userId: string;
    ipAddress: string;
    userAgent: string;
    deviceFingerprint: string;
    authMethod: 'email' | 'saml' | 'oidc' | 'passwordless';
}

interface LoginFailed {
    email: string;
    ipAddress: string;
    reason: 'invalid_password' | 'account_locked' | 'device_not_bound' | 'mfa_failed';
}

interface AnomalousAccessDetected {
    userId: string;
    roomId: string;
    anomalyType: 'unusual_location' | 'bulk_download' | 'off_hours_access' | 'rapid_page_scanning';
    severity: 'low' | 'medium' | 'high' | 'critical';
    details: Record<string, unknown>;
    detectedAt: string;
}

interface RemoteWipeRequested {
    wipeRequestId: string;
    roomId: string;
    targetUserId?: string;
    targetDocumentId?: string;
    devicesTargeted: number;
    requestedBy: string;
}

interface RemoteWipeConfirmed {
    wipeRequestId: string;
    deviceId: string;
    confirmedAt: string;
}
```

---

## Read Model Projections

### Projection 1: Active Rooms (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: data_rooms_view
-- ============================================================
-- Materialized from DataRoomCreated, DataRoomOpened, DataRoomClosed,
-- DataRoomArchived, DataRoomSettingsUpdated events.

CREATE TABLE rm_data_rooms (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    deal_type       VARCHAR(50) NOT NULL,
    deal_value      NUMERIC(18, 2),
    deal_currency   VARCHAR(3),
    status          VARCHAR(20) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Computed aggregates
    document_count  INTEGER NOT NULL DEFAULT 0,
    folder_count    INTEGER NOT NULL DEFAULT 0,
    user_count      INTEGER NOT NULL DEFAULT 0,
    question_count  INTEGER NOT NULL DEFAULT 0,
    -- Lifecycle timestamps
    opened_at       TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    archived_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    created_by      UUID NOT NULL,
    -- Projection metadata
    last_event_position BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_rooms_org ON rm_data_rooms(organization_id);
CREATE INDEX idx_rm_rooms_status ON rm_data_rooms(organization_id, status);
```

### Projection 2: Document Catalogue (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: documents_view
-- ============================================================
-- Materialized from Document* events.

CREATE TABLE rm_documents (
    id              UUID PRIMARY KEY,
    data_room_id    UUID NOT NULL,
    organization_id UUID NOT NULL,
    folder_id       UUID NOT NULL,
    -- File metadata
    original_filename   VARCHAR(500) NOT NULL,
    display_name        VARCHAR(500) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    page_count          INTEGER,
    -- Storage references
    storage_bucket      VARCHAR(255) NOT NULL,
    storage_key         VARCHAR(1000) NOT NULL,
    encryption_key_id   VARCHAR(255) NOT NULL,
    checksum_sha256     VARCHAR(64) NOT NULL,
    -- Versioning
    version             INTEGER NOT NULL DEFAULT 1,
    is_latest           BOOLEAN NOT NULL DEFAULT TRUE,
    parent_document_id  UUID,
    -- Index numbering
    index_number        VARCHAR(50),
    sort_order          INTEGER NOT NULL DEFAULT 0,
    -- Processing
    processing_status   VARCHAR(30) NOT NULL,
    processing_error    TEXT,
    pdf_storage_key     VARCHAR(1000),
    pdf_page_count      INTEGER,
    -- AI classification
    ai_classification   VARCHAR(100),
    ai_classification_confidence NUMERIC(5, 4),
    ai_tags             TEXT[],
    ai_summary          TEXT,
    -- Full-text search
    search_vector       TSVECTOR,
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Timestamps
    uploaded_at         TIMESTAMPTZ NOT NULL,
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL,
    uploaded_by         UUID NOT NULL,
    -- Projection metadata
    last_event_position BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_docs_room ON rm_documents(data_room_id);
CREATE INDEX idx_rm_docs_folder ON rm_documents(folder_id);
CREATE INDEX idx_rm_docs_search ON rm_documents USING gin(search_vector);
CREATE INDEX idx_rm_docs_classification ON rm_documents(ai_classification);
CREATE INDEX idx_rm_docs_status ON rm_documents(data_room_id, status) WHERE status = 'active';
```

### Projection 3: Effective Permissions (Redis + PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: effective_permissions
-- ============================================================
-- Materialized from RoleCreated, RolePermissionsUpdated,
-- UserInvitedToRoom, ResourcePermissionGranted/Revoked events.
-- This projection pre-computes the resolved permissions for
-- every user-resource combination, avoiding expensive
-- recursive walks at query time.

CREATE TABLE rm_effective_permissions (
    data_room_id    UUID NOT NULL,
    user_id         UUID NOT NULL,
    resource_type   VARCHAR(20) NOT NULL,  -- 'folder' or 'document'
    resource_id     UUID NOT NULL,
    -- Resolved permissions
    can_view        BOOLEAN NOT NULL DEFAULT FALSE,
    can_download    BOOLEAN NOT NULL DEFAULT FALSE,
    can_print       BOOLEAN NOT NULL DEFAULT FALSE,
    can_upload      BOOLEAN NOT NULL DEFAULT FALSE,
    -- Resolution metadata
    resolved_via    VARCHAR(20) NOT NULL,  -- 'role', 'group', 'user', 'inherited'
    last_event_position BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (data_room_id, user_id, resource_type, resource_id)
);

CREATE INDEX idx_rm_perms_user ON rm_effective_permissions(user_id, data_room_id);
CREATE INDEX idx_rm_perms_resource ON rm_effective_permissions(resource_type, resource_id);
```

The permission projector listens to all permission-related events and recomputes the effective permissions for affected users by walking the folder hierarchy and applying the override rules (user > group > role; most specific resource wins). This computation happens asynchronously, so there is a brief consistency window after a permission change.

For real-time enforcement, the projector also pushes the computed permissions into Redis:

```
Key: perms:{roomId}:{userId}:{resourceType}:{resourceId}
Value: {"v":true,"d":false,"p":false,"u":false}
TTL: 3600 (re-projected on event or on cache miss)
```

### Projection 4: Engagement Analytics (ClickHouse)

```sql
-- ============================================================
-- READ MODEL: engagement analytics (ClickHouse)
-- ============================================================
-- Projected from DocumentViewStarted, DocumentPageViewed,
-- DocumentViewEnded, DocumentDownloaded, DocumentPrinted events.

CREATE TABLE analytics_page_views (
    event_date      Date,
    event_time      DateTime64(3),
    organization_id UUID,
    data_room_id    UUID,
    document_id     UUID,
    user_id         UUID,
    session_id      UUID,
    page_number     UInt16,
    view_duration_ms UInt32,
    scroll_depth_pct Float32,
    device_type     LowCardinality(String),
    ip_country      LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, data_room_id, document_id, user_id, event_time);

-- Materialized view for per-document engagement aggregates
CREATE MATERIALIZED VIEW mv_document_engagement
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, data_room_id, document_id)
AS SELECT
    toDate(event_time) AS event_date,
    organization_id,
    data_room_id,
    document_id,
    uniqState(user_id) AS unique_viewers,
    uniqState(session_id) AS total_sessions,
    sumState(view_duration_ms) AS total_view_ms,
    maxState(event_time) AS last_viewed_at,
    countState() AS total_page_views
FROM analytics_page_views
GROUP BY event_date, organization_id, data_room_id, document_id;

-- Materialized view for user engagement scoring
CREATE MATERIALIZED VIEW mv_user_engagement
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (organization_id, data_room_id, user_id)
AS SELECT
    toDate(event_time) AS event_date,
    organization_id,
    data_room_id,
    user_id,
    uniqState(document_id) AS documents_viewed,
    sumState(view_duration_ms) AS total_view_ms,
    countState() AS total_page_views,
    maxState(event_time) AS last_activity_at
FROM analytics_page_views
GROUP BY event_date, organization_id, data_room_id, user_id;
```

### Projection 5: Q&A Dashboard (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: Q&A summary
-- ============================================================
CREATE TABLE rm_qa_questions (
    id              UUID PRIMARY KEY,
    data_room_id    UUID NOT NULL,
    organization_id UUID NOT NULL,
    topic_id        UUID,
    topic_name      VARCHAR(255),
    question_number VARCHAR(20) NOT NULL,
    subject         VARCHAR(500) NOT NULL,
    body            TEXT NOT NULL,
    related_folder_id   UUID,
    related_document_id UUID,
    status          VARCHAR(20) NOT NULL,
    priority        VARCHAR(10) NOT NULL,
    visibility      VARCHAR(20) NOT NULL,
    assigned_to     UUID,
    assigned_to_name VARCHAR(255),
    assigned_group  UUID,
    assigned_group_name VARCHAR(100),
    sla_deadline    TIMESTAMPTZ,
    sla_breached    BOOLEAN NOT NULL DEFAULT FALSE,
    answer_count    INTEGER NOT NULL DEFAULT 0,
    latest_answer_at TIMESTAMPTZ,
    submitted_at    TIMESTAMPTZ,
    answered_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    submitted_by    UUID NOT NULL,
    submitted_by_name VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_position BIGINT NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_qa_room ON rm_qa_questions(data_room_id);
CREATE INDEX idx_rm_qa_status ON rm_qa_questions(data_room_id, status);
CREATE INDEX idx_rm_qa_sla ON rm_qa_questions(sla_deadline) WHERE NOT sla_breached;
CREATE INDEX idx_rm_qa_assigned ON rm_qa_questions(assigned_to);
```

### Projection 6: Compliance Audit Export (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: Flattened audit trail for compliance export
-- ============================================================
-- This projection transforms events into a human-readable,
-- regulator-friendly format for SOC 2 / ISO 27001 / SEC 17a-4
-- audit report generation.

CREATE TABLE rm_audit_trail (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    data_room_id    UUID,
    -- Actor
    user_id         UUID NOT NULL,
    user_email      VARCHAR(255) NOT NULL,
    user_display_name VARCHAR(255),
    user_ip         INET,
    user_agent      TEXT,
    -- Action
    action_category VARCHAR(50) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    action_description TEXT NOT NULL,  -- human-readable, e.g., "John Doe uploaded 'Financial Statements Q3.pdf' to folder 'Financials'"
    -- Resource
    resource_type   VARCHAR(50),
    resource_id     UUID,
    resource_name   VARCHAR(500),
    -- Change details
    details         JSONB NOT NULL DEFAULT '{}',
    -- Source event reference
    source_event_id UUID NOT NULL,
    source_stream   VARCHAR(500) NOT NULL,
    source_position BIGINT NOT NULL,
    -- Timestamp
    occurred_at     TIMESTAMPTZ NOT NULL,
    -- Projection metadata
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_audit_org_room ON rm_audit_trail(organization_id, data_room_id, occurred_at);
CREATE INDEX idx_rm_audit_user ON rm_audit_trail(user_id, occurred_at);
CREATE INDEX idx_rm_audit_action ON rm_audit_trail(action_category, occurred_at);
CREATE INDEX idx_rm_audit_resource ON rm_audit_trail(resource_type, resource_id, occurred_at);
```

---

## Deal Readiness Scoring Projection

```sql
-- ============================================================
-- READ MODEL: Deal readiness scoring
-- ============================================================
-- Projected from DocumentClassified, DocumentReady, FolderCreated events.
-- The AI readiness scorer runs as a projector that evaluates
-- document completeness against configurable checklists.

CREATE TABLE rm_deal_readiness (
    data_room_id        UUID NOT NULL,
    organization_id     UUID NOT NULL,
    checklist_id        UUID NOT NULL,
    -- Scores
    overall_score       NUMERIC(5, 2) NOT NULL DEFAULT 0.00,  -- 0-100
    categories_complete INTEGER NOT NULL DEFAULT 0,
    categories_total    INTEGER NOT NULL DEFAULT 0,
    items_complete      INTEGER NOT NULL DEFAULT 0,
    items_total         INTEGER NOT NULL DEFAULT 0,
    -- Per-category breakdown (JSONB for flexibility)
    category_scores     JSONB NOT NULL DEFAULT '[]',
    -- e.g., [{"category": "Financial", "score": 85.0, "items": 12, "matched": 10, "gaps": ["Tax Returns 2024", "Debt Schedule"]}]
    missing_documents   JSONB NOT NULL DEFAULT '[]',
    -- Timestamps
    last_scored_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_event_position BIGINT NOT NULL,
    PRIMARY KEY (data_room_id, checklist_id)
);
```

---

## Saga: Document Processing Pipeline

The document processing pipeline is a long-running process (saga) that coordinates multiple steps. In an event-sourced system, the saga is itself driven by events:

```
DocumentUploadStarted
    --> [PDF Converter Service] --> DocumentConversionCompleted
        --> [Indexer Service] --> DocumentIndexingCompleted
            --> [Classifier Service] --> DocumentClassified
                --> [Redactor Service] --> DocumentRedactionDetected
                    --> [Manual Review or Auto-Approve]
                        --> DocumentRedactionApplied
                            --> DocumentReady
```

```sql
-- ============================================================
-- SAGA STATE: Document processing pipeline
-- ============================================================
CREATE TABLE saga_document_processing (
    document_id     UUID PRIMARY KEY,
    current_stage   VARCHAR(30) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    -- Stage completion tracking
    upload_completed    BOOLEAN NOT NULL DEFAULT FALSE,
    conversion_completed BOOLEAN NOT NULL DEFAULT FALSE,
    indexing_completed  BOOLEAN NOT NULL DEFAULT FALSE,
    classification_completed BOOLEAN NOT NULL DEFAULT FALSE,
    redaction_completed BOOLEAN NOT NULL DEFAULT FALSE,
    -- Error handling
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 3,
    last_error      TEXT,
    -- Timeout monitoring
    stage_timeout_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Anomalous Access Detection Projection

```sql
-- ============================================================
-- READ MODEL: Anomaly detection baselines (ClickHouse)
-- ============================================================
-- Projected from engagement events. The anomaly detector compares
-- real-time behavior against these baselines.

CREATE TABLE analytics_user_baselines (
    organization_id     UUID,
    data_room_id        UUID,
    user_id             UUID,
    -- Behavioral baselines (rolling 30-day averages)
    avg_daily_sessions          Float32,
    avg_session_duration_sec    Float32,
    avg_pages_per_session       Float32,
    avg_documents_per_day       Float32,
    typical_access_hours        Array(UInt8),  -- e.g., [9,10,11,12,13,14,15,16,17]
    typical_ip_country          LowCardinality(String),
    typical_device_types        Array(LowCardinality(String)),
    -- Thresholds (computed as mean + 3*stddev)
    max_pages_per_minute        Float32,
    max_downloads_per_hour      Float32,
    -- Updated
    last_updated_at             DateTime64(3)
) ENGINE = ReplacingMergeTree(last_updated_at)
ORDER BY (organization_id, data_room_id, user_id);
```

---

## Pros and Cons

### Pros

1. **Compliance is built in, not bolted on.** The event store *is* the audit trail. Every state change is immutable, timestamped, and causally linked via correlation IDs. There is no risk of the audit log diverging from the actual state because they are the same thing. Hash chains on events provide cryptographic tamper evidence that satisfies SEC 17a-4 WORM requirements natively.

2. **Complete temporal query capability.** You can reconstruct the exact state of any data room, document, or permission tree at any point in time by replaying events up to that timestamp. This is invaluable for regulatory investigations ("show me exactly who had access to document X on March 15th at 2:00 PM") and dispute resolution.

3. **Optimized read models for each use case.** CQRS lets you build purpose-built projections: a relational projection for room/document browsing, a columnar projection in ClickHouse for analytics, a Redis projection for real-time permission checks, and an Elasticsearch projection for full-text search. Each read model is shaped exactly for its query patterns without compromises.

4. **Natural fit for document processing pipelines.** The saga pattern driven by events elegantly coordinates multi-step processing (upload, convert, index, classify, redact) with built-in retry, timeout, and compensation logic. Each service is decoupled and can scale independently.

5. **AI feedback loop.** When users correct AI classifications, the `DocumentClassificationCorrected` event feeds directly into the training pipeline. The event stream becomes a labeled training dataset that improves over time.

6. **Integration via event subscription.** External systems (Salesforce, DocuSign, webhooks) subscribe to the event stream rather than polling. This produces real-time, reliable integration without custom sync logic.

7. **Debugging and incident response.** When something goes wrong, you can replay the exact sequence of events that led to the current state. Combined with correlation IDs, you can trace a single user action through every system component.

### Cons

1. **Significant architectural complexity.** Event sourcing requires the team to understand aggregates, event streams, projections, eventual consistency, idempotency, saga orchestration, and snapshot management. This is a steep learning curve and a substantial investment in developer training.

2. **Eventual consistency in read models.** After a permission change, there is a window (typically milliseconds to seconds) where the read model has not yet caught up. In a security-critical VDR, this means a user might briefly retain access to a document after their permission is revoked. Mitigation: synchronous permission checks against the event store for security-critical operations, with the cached projection as a fast-path optimization.

3. **Event schema evolution is hard.** Once events are persisted, they are immutable. If the structure of `DocumentUploaded` needs to change, you must handle versioning (upcasting old events to the new schema during replay). Over years of operation, managing dozens of event versions becomes a maintenance burden.

4. **Projection rebuild time.** If a read model projection has a bug and needs to be rebuilt from scratch, replaying millions of events can take hours or days. Snapshots mitigate this, but snapshot management adds its own complexity.

5. **Storage volume.** Every state change generates an event, so the event store grows much faster than a mutable state database. A busy data room with thousands of page-view events per day per user can generate gigabytes of events per month. The storage cost is higher, though the compliance value justifies it.

6. **Testing complexity.** Unit testing aggregate behavior requires setting up event histories. Integration testing requires running projectors. End-to-end tests must account for eventual consistency delays. The testing infrastructure is more complex than traditional CRUD tests.

7. **Operational tooling gap.** Debugging production issues requires tooling to inspect event streams, replay events, and compare projected state against expected state. Off-the-shelf tools for this are less mature than SQL database administration tools.

---

## Migration and Scaling Considerations

### Migrating from a Traditional Relational Model

If starting with the normalized relational model (Suggestion 1) and migrating to event sourcing:

1. **Dual-write transition period.** Run both systems in parallel: the relational database handles writes while an event publisher captures each mutation as an event. Projectors rebuild read models from the event stream. Compare projected state against the relational database to validate correctness.

2. **Event backfill.** For existing data, generate synthetic "initial state" events (e.g., `DataRoomCreated`, `DocumentUploaded`) from the current relational state. Mark these events with a `backfilled: true` metadata flag so they can be distinguished from organically generated events.

3. **Incremental cutover.** Migrate one aggregate at a time (start with the audit-heavy ones like Documents and Permissions), validating each before proceeding.

### Scaling the Event Store

1. **Phase 1 (0-1M events):** Single PostgreSQL instance with partitioned event table. Projectors run in-process.

2. **Phase 2 (1M-100M events):** Kafka as the event bus between the event store and projectors. Multiple projector instances for parallelism. Snapshots every 1,000 events per stream.

3. **Phase 3 (100M-1B events):** Dedicated EventStoreDB cluster for the event store. PostgreSQL only for read models. Kafka for cross-service event distribution. Separate ClickHouse cluster for analytics projections.

4. **Phase 4 (1B+ events):** Shard event streams by organization_id across multiple EventStoreDB nodes. Archive cold event partitions to S3 (Parquet format) for long-term retention. On-demand replay from S3 for historical analysis.

### Snapshot Strategy

```
Events per stream before snapshot: 1,000 (configurable per aggregate type)
Snapshot retention: 3 most recent snapshots per stream
Snapshot format: JSON with schema version tag
Snapshot storage: PostgreSQL snapshots table or S3 for large aggregates
```

### Disaster Recovery

- **Event store replication:** Synchronous replication to a standby for RPO=0. Asynchronous replication to a DR region for geographic redundancy.
- **Read model recovery:** Read models can be completely rebuilt from the event store. This is slower than restoring a backup but guarantees consistency.
- **Point-in-time recovery:** Replay events up to a specific global_position or timestamp to recover to any historical state.

### Performance Optimization

- **Event batching:** Buffer events in memory and flush in batches of 100-500 for write throughput.
- **Projection partitioning:** Each projector can partition work by organization_id for parallel processing.
- **Subscription filtering:** Projectors subscribe only to event types they care about (server-side filtering in EventStoreDB or Kafka topic partitioning).
- **Read model indexing:** Projections are denormalized specifically for their query patterns, so each read model has indexes optimized for its specific access patterns rather than compromising across all use cases.
- **Cache warming:** On projector startup, warm the Redis permission cache from the PostgreSQL permission projection rather than replaying all permission events.
