# Data Model Suggestion 3: Hybrid Relational + Document (JSONB) Model

> Project: Virtual Data Room (476) -- Generated: 2026-05-26

## Overview

This model takes a pragmatic middle path: stable, well-understood entities (organizations, users, data rooms, folders) live in normalized relational tables with foreign keys and constraints, while inherently flexible or rapidly evolving data (document metadata, AI classification results, redaction coordinates, deal-specific custom fields, integration configurations, notification payloads) lives in JSONB columns within those same tables or in lightweight JSONB-centric tables.

The insight driving this design is that a Virtual Data Room has two distinct categories of data. The first is structural data -- the hierarchy of rooms, folders, users, roles, and permissions -- which is stable, query-intensive, and benefits from relational integrity. The second is contextual data -- AI-generated classifications that evolve as models improve, redaction bounding boxes with variable schemas per document type, custom metadata fields that differ per deal type (M&A vs. fundraising vs. regulatory), and integration payloads from external systems with their own evolving schemas. Forcing the second category into rigid relational tables leads to constant schema migrations, sparse columns, and EAV anti-patterns. Storing it in JSONB preserves flexibility while keeping it co-located with the structural data in a single PostgreSQL instance.

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary database | PostgreSQL 16+ | JSONB with GIN indexes, partial indexes, generated columns, RLS |
| Connection pooling | PgBouncer | Transaction-mode pooling for multi-tenant workloads |
| Search | PostgreSQL tsvector + optional OpenSearch | Built-in FTS for basic search; OpenSearch for advanced faceted search at scale |
| File storage | S3-compatible (AWS S3, MinIO, R2) | Encrypted document blobs with server-side encryption |
| Cache | Redis 7+ | Permission cache, session store, JSONB query result cache |
| Encryption / KMS | AWS KMS / HashiCorp Vault | Per-tenant key management |
| Migrations | Flyway or Sqitch | Schema migrations for relational parts; JSONB schema validation at application layer |
| Analytics | PostgreSQL + optional ClickHouse | Start with PostgreSQL materialized views; graduate analytics to ClickHouse if volume demands it |
| JSONB validation | Application-layer (Zod/JSON Schema) | Validate JSONB payloads before write; database stores validated data |

## Design Philosophy: When to Use Columns vs. JSONB

| Use relational columns when... | Use JSONB when... |
|-------------------------------|-------------------|
| The field is used in WHERE clauses frequently | The schema varies by deal type, document type, or integration |
| The field participates in JOIN conditions | The field is write-once, read-occasionally (AI results, processing metadata) |
| The field has referential integrity requirements | The structure is nested or hierarchical (redaction coordinates, checklist trees) |
| The field is part of a unique or foreign key constraint | The schema evolves rapidly (new AI model outputs, new PII types) |
| Compliance requires explicit auditability of the field | The data is consumed as a blob by the frontend (settings, templates, layouts) |
| The field is indexed for range queries (dates, numbers) | Custom fields defined by the tenant at runtime |

---

## Complete Schema Definition

### Organizations and Users (Fully Relational)

```sql
-- ============================================================
-- ORGANIZATIONS
-- ============================================================
-- Core identity and billing are relational. Per-org settings
-- and feature flags use JSONB because they evolve with the product.

CREATE TABLE organizations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    plan_tier           VARCHAR(50) NOT NULL DEFAULT 'free'
                        CHECK (plan_tier IN ('free', 'professional', 'enterprise')),
    billing_email       VARCHAR(255),
    logo_url            TEXT,
    data_residency_region VARCHAR(20) NOT NULL DEFAULT 'us-east-1',
    -- Compliance flags (relational: frequently filtered)
    iso_27001_certified BOOLEAN NOT NULL DEFAULT FALSE,
    soc2_certified      BOOLEAN NOT NULL DEFAULT FALSE,
    hipaa_enabled       BOOLEAN NOT NULL DEFAULT FALSE,
    -- Flexible settings (JSONB: evolves with product features)
    settings            JSONB NOT NULL DEFAULT '{
        "branding": {"primaryColor": "#1a73e8", "customDomain": null},
        "security": {"sessionTimeoutMinutes": 480, "ipWhitelist": [], "enforceDeviceBinding": false},
        "notifications": {"emailDigestFrequency": "daily", "slackWebhook": null},
        "features": {"aiClassification": true, "aiRedaction": true, "dealReadiness": false}
    }'::JSONB,
    -- Timestamps
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    suspended_at        TIMESTAMPTZ
);

-- GIN index on settings for containment queries
CREATE INDEX idx_org_settings ON organizations USING gin(settings jsonb_path_ops);

-- ============================================================
-- USERS
-- ============================================================
-- Core identity is relational. Device fingerprints and auth
-- metadata use JSONB because they vary by auth provider.

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(255) NOT NULL UNIQUE,
    display_name        VARCHAR(255) NOT NULL,
    password_hash       VARCHAR(255),
    phone               VARCHAR(50),
    avatar_url          TEXT,
    -- Auth core (relational: indexed, filtered)
    auth_provider       VARCHAR(50) NOT NULL DEFAULT 'email'
                        CHECK (auth_provider IN ('email', 'saml', 'oidc', 'passwordless')),
    auth_provider_id    VARCHAR(255),
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    email_verified      BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'suspended', 'deactivated')),
    last_login_at       TIMESTAMPTZ,
    -- Auth details (JSONB: varies by provider, evolves with security features)
    auth_metadata       JSONB NOT NULL DEFAULT '{
        "mfaSecret": null,
        "deviceFingerprints": [],
        "samlAttributes": null,
        "oidcClaims": null,
        "passwordlessTokens": [],
        "loginHistory": []
    }'::JSONB,
    -- Timestamps
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_auth ON users(auth_provider, auth_provider_id);

-- ============================================================
-- ORGANIZATION MEMBERSHIPS (Fully relational)
-- ============================================================
CREATE TABLE organization_memberships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_role            VARCHAR(50) NOT NULL DEFAULT 'member'
                        CHECK (org_role IN ('owner', 'admin', 'member', 'billing')),
    invited_by          UUID REFERENCES users(id),
    invited_at          TIMESTAMPTZ,
    accepted_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_memberships(organization_id);
CREATE INDEX idx_org_members_user ON organization_memberships(user_id);
```

### Data Rooms (Relational Core + JSONB Settings)

```sql
-- ============================================================
-- DATA ROOMS
-- ============================================================
-- Core deal metadata is relational (filtered, aggregated, reported).
-- Room-level settings use JSONB because they are consumed as a
-- configuration blob and new settings are added frequently.

CREATE TABLE data_rooms (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    -- Deal metadata (relational: filtered, sorted, aggregated)
    deal_type           VARCHAR(50) NOT NULL DEFAULT 'mna'
                        CHECK (deal_type IN ('mna', 'fundraising', 'ipo', 'regulatory',
                                             'board_reporting', 'investor_relations', 'other')),
    deal_value          NUMERIC(18, 2),
    deal_currency       VARCHAR(3) DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'setup', 'active', 'closed', 'archived')),
    -- Lifecycle timestamps (relational: range queries)
    opened_at           TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    archived_at         TIMESTAMPTZ,
    -- Room settings (JSONB: complex nested config, evolves with features)
    room_settings       JSONB NOT NULL DEFAULT '{
        "watermark": {
            "enabled": true,
            "template": "{{user_email}} - {{date}} - Confidential",
            "opacity": 0.15,
            "position": "diagonal",
            "fontSize": 14,
            "invisibleEnabled": true,
            "invisiblePayloadFields": ["user_id", "session_id", "timestamp"]
        },
        "access": {
            "downloadEnabled": false,
            "printEnabled": false,
            "ndaRequired": false,
            "ndaDocumentId": null,
            "sessionTimeoutMinutes": 480,
            "ipRestrictions": []
        },
        "ai": {
            "classificationEnabled": true,
            "redactionEnabled": true,
            "readinessScoring": false,
            "summarizationEnabled": true,
            "classificationModelVersion": "v3.2"
        },
        "compliance": {
            "retentionDays": 2190,
            "wormEnabled": false,
            "dataResidencyOverride": null
        },
        "ui": {
            "customBranding": null,
            "defaultSortOrder": "index_number",
            "showPageThumbnails": true
        }
    }'::JSONB,
    -- Deal-type-specific custom fields (JSONB: schema varies by deal_type)
    deal_metadata       JSONB NOT NULL DEFAULT '{}',
    -- Examples by deal type:
    -- M&A: {"targetCompany": "...", "acquirer": "...", "dealStage": "LOI", "exclusivityDeadline": "2026-06-01"}
    -- Fundraising: {"fundName": "...", "targetRaise": 50000000, "roundType": "Series B", "leadInvestor": "..."}
    -- IPO: {"exchangeTarget": "NYSE", "filingType": "S-1", "quietPeriodStart": "2026-07-01"}
    -- Regulatory: {"regulatoryBody": "SEC", "filingDeadline": "2026-09-15", "filingType": "10-K"}
    -- Timestamps
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_rooms_org ON data_rooms(organization_id);
CREATE INDEX idx_rooms_status ON data_rooms(organization_id, status);
CREATE INDEX idx_rooms_deal_type ON data_rooms(organization_id, deal_type);
-- Expression index on a commonly queried JSONB field
CREATE INDEX idx_rooms_nda_required ON data_rooms((room_settings->'access'->>'ndaRequired'));
-- GIN index for arbitrary JSONB queries on deal_metadata
CREATE INDEX idx_rooms_deal_metadata ON data_rooms USING gin(deal_metadata jsonb_path_ops);

-- ============================================================
-- DEAL READINESS CHECKLISTS
-- ============================================================
-- The checklist structure itself is JSONB because checklists
-- vary significantly by deal type and are consumed as trees.

CREATE TABLE deal_checklists (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    deal_type           VARCHAR(50) NOT NULL,
    overall_score       NUMERIC(5, 2) NOT NULL DEFAULT 0.00,
    -- Checklist items as a JSONB tree (variable depth, variable fields per deal type)
    items               JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "id": "uuid",
    --     "category": "Financial",
    --     "items": [
    --       {
    --         "id": "uuid",
    --         "name": "Audited Financial Statements (3 years)",
    --         "description": "...",
    --         "required": true,
    --         "matchedFolderId": "uuid",
    --         "matchedDocumentCount": 3,
    --         "completenessScore": 100.0,
    --         "subitems": [
    --           {"id": "uuid", "name": "FY 2023", "matched": true, "documentId": "uuid"},
    --           {"id": "uuid", "name": "FY 2024", "matched": true, "documentId": "uuid"},
    --           {"id": "uuid", "name": "FY 2025", "matched": true, "documentId": "uuid"}
    --         ]
    --       },
    --       {
    --         "id": "uuid",
    --         "name": "Tax Returns (3 years)",
    --         "required": true,
    --         "matchedFolderId": null,
    --         "matchedDocumentCount": 0,
    --         "completenessScore": 0.0,
    --         "subitems": []
    --       }
    --     ],
    --     "categoryScore": 75.0
    --   }
    -- ]
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_checklists_room ON deal_checklists(data_room_id);
CREATE INDEX idx_checklists_items ON deal_checklists USING gin(items jsonb_path_ops);
```

### Folders and Documents

```sql
-- ============================================================
-- FOLDERS (Relational hierarchy + JSONB metadata)
-- ============================================================
-- The folder tree structure is relational (adjacency list + materialized
-- path) because it is traversed in permission resolution and folder
-- listing queries. Custom folder metadata is JSONB.

CREATE TABLE folders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    parent_id           UUID REFERENCES folders(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    path                TEXT NOT NULL,
    depth               INTEGER NOT NULL DEFAULT 0,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    index_number        VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'hidden', 'deleted')),
    -- Custom metadata per folder (JSONB: user-defined labels, notes, deal-specific tags)
    metadata            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"customLabel": "Phase 1 Materials", "color": "#ff5722", "notes": "Buyer eyes only"}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_folders_room ON folders(data_room_id);
CREATE INDEX idx_folders_parent ON folders(parent_id);
CREATE INDEX idx_folders_org_room ON folders(organization_id, data_room_id);
CREATE INDEX idx_folders_path ON folders(path text_pattern_ops);

-- ============================================================
-- DOCUMENTS
-- ============================================================
-- Core file metadata (name, size, status, storage refs) is relational.
-- AI processing results and variable metadata are JSONB because:
-- 1. AI output schemas change with each model version
-- 2. Different document types produce different classification structures
-- 3. Redaction coordinates are nested arrays of variable length
-- 4. Custom metadata fields are tenant-defined

CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    folder_id           UUID NOT NULL REFERENCES folders(id) ON DELETE CASCADE,
    -- File identity (relational: indexed, filtered, sorted)
    original_filename   VARCHAR(500) NOT NULL,
    display_name        VARCHAR(500) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    page_count          INTEGER,
    index_number        VARCHAR(50),
    sort_order          INTEGER NOT NULL DEFAULT 0,
    -- Storage (relational: referenced by processing services)
    storage_bucket      VARCHAR(255) NOT NULL,
    storage_key         VARCHAR(1000) NOT NULL,
    encryption_key_id   VARCHAR(255) NOT NULL,
    checksum_sha256     VARCHAR(64) NOT NULL,
    -- Versioning (relational: tree structure needs FK)
    version             INTEGER NOT NULL DEFAULT 1,
    is_latest           BOOLEAN NOT NULL DEFAULT TRUE,
    parent_document_id  UUID REFERENCES documents(id),
    -- Processing status (relational: filtered, drives workflow UI)
    processing_status   VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (processing_status IN (
                            'pending', 'uploading', 'converting',
                            'indexing', 'classifying', 'redacting',
                            'ready', 'failed'
                        )),
    processing_error    TEXT,
    -- Converted PDF refs (relational: referenced by viewer)
    pdf_storage_key     VARCHAR(1000),
    pdf_page_count      INTEGER,
    -- Full-text search (relational: used by search queries)
    content_text        TEXT,
    search_vector       TSVECTOR,
    -- Status (relational: filtered everywhere)
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'superseded', 'deleted', 'wiped')),
    -- AI processing results (JSONB: schema varies by model version and document type)
    ai_results          JSONB NOT NULL DEFAULT '{
        "classification": null,
        "classificationConfidence": null,
        "tags": [],
        "summary": null,
        "language": null,
        "modelVersion": null,
        "classifiedAt": null,
        "entities": [],
        "keyPhrases": [],
        "sentiment": null,
        "documentType": null
    }'::JSONB,
    -- Example populated ai_results:
    -- {
    --   "classification": "financial_statement",
    --   "classificationConfidence": 0.9542,
    --   "tags": ["audited", "annual", "consolidated"],
    --   "summary": "Consolidated financial statements for FY2025...",
    --   "language": "en",
    --   "modelVersion": "classify-v3.2",
    --   "classifiedAt": "2026-05-26T10:30:00Z",
    --   "entities": [
    --     {"type": "company", "value": "Acme Corp", "count": 47},
    --     {"type": "currency", "value": "USD", "count": 128},
    --     {"type": "date_range", "value": "Jan 2025 - Dec 2025", "count": 3}
    --   ],
    --   "keyPhrases": ["revenue recognition", "goodwill impairment", "lease obligations"],
    --   "sentiment": "neutral",
    --   "documentType": "pdf_text"
    -- }

    -- Redaction results (JSONB: nested arrays of variable-length coordinate data)
    redaction_results   JSONB NOT NULL DEFAULT '{
        "status": "pending",
        "totalDetected": 0,
        "totalApproved": 0,
        "totalRejected": 0,
        "redactedStorageKey": null,
        "modelVersion": null,
        "detections": []
    }'::JSONB,
    -- Example populated redaction_results:
    -- {
    --   "status": "reviewed",
    --   "totalDetected": 12,
    --   "totalApproved": 10,
    --   "totalRejected": 2,
    --   "redactedStorageKey": "docs/redacted/abc123.pdf",
    --   "modelVersion": "redact-v2.1",
    --   "detections": [
    --     {
    --       "id": "uuid",
    --       "pageNumber": 3,
    --       "bounds": {"x": 120.5, "y": 340.2, "w": 200.0, "h": 18.0},
    --       "piiType": "ssn",
    --       "confidence": 0.9876,
    --       "detectionMethod": "ai_ner",
    --       "reviewStatus": "approved",
    --       "reviewedBy": "uuid",
    --       "reviewedAt": "2026-05-26T11:00:00Z"
    --     }
    --   ]
    -- }

    -- Processing pipeline metadata (JSONB: variable per processing stage)
    processing_metadata JSONB NOT NULL DEFAULT '{
        "pipeline": [],
        "totalDurationMs": null,
        "retryCount": 0
    }'::JSONB,
    -- Example:
    -- {
    --   "pipeline": [
    --     {"stage": "upload", "startedAt": "...", "completedAt": "...", "durationMs": 1200},
    --     {"stage": "conversion", "startedAt": "...", "completedAt": "...", "durationMs": 4500, "converter": "libreoffice"},
    --     {"stage": "indexing", "startedAt": "...", "completedAt": "...", "durationMs": 800, "wordsIndexed": 12450},
    --     {"stage": "classification", "startedAt": "...", "completedAt": "...", "durationMs": 2100, "modelVersion": "v3.2"},
    --     {"stage": "redaction", "startedAt": "...", "completedAt": "...", "durationMs": 3200, "modelVersion": "v2.1"}
    --   ],
    --   "totalDurationMs": 11800,
    --   "retryCount": 0
    -- }

    -- Custom metadata (JSONB: tenant-defined fields that vary per organization)
    custom_metadata     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"department": "Legal", "confidentialityLevel": "highly_confidential", "reviewDeadline": "2026-06-15"}

    -- Timestamps
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uploaded_by         UUID NOT NULL REFERENCES users(id)
);

-- Relational column indexes
CREATE INDEX idx_docs_room ON documents(data_room_id);
CREATE INDEX idx_docs_folder ON documents(folder_id);
CREATE INDEX idx_docs_org_room ON documents(organization_id, data_room_id);
CREATE INDEX idx_docs_search ON documents USING gin(search_vector);
CREATE INDEX idx_docs_status ON documents(data_room_id, status) WHERE status = 'active';
CREATE INDEX idx_docs_latest ON documents(parent_document_id) WHERE is_latest = TRUE;
CREATE INDEX idx_docs_processing ON documents(processing_status) WHERE processing_status != 'ready';

-- JSONB expression indexes for commonly queried AI fields
CREATE INDEX idx_docs_ai_classification ON documents((ai_results->>'classification'))
    WHERE ai_results->>'classification' IS NOT NULL;
CREATE INDEX idx_docs_ai_confidence ON documents(((ai_results->>'classificationConfidence')::NUMERIC))
    WHERE ai_results->>'classificationConfidence' IS NOT NULL;
-- GIN index for tag containment queries (e.g., "find all documents tagged 'audited'")
CREATE INDEX idx_docs_ai_tags ON documents USING gin((ai_results->'tags') jsonb_path_ops);
-- GIN index for full JSONB containment on custom metadata
CREATE INDEX idx_docs_custom_meta ON documents USING gin(custom_metadata jsonb_path_ops);
-- GIN index for redaction queries
CREATE INDEX idx_docs_redaction_status ON documents((redaction_results->>'status'))
    WHERE redaction_results->>'status' IS NOT NULL;

-- Full-text search trigger (same as Suggestion 1)
CREATE FUNCTION documents_search_update() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.display_name, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.ai_results->>'summary', '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(NEW.content_text, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_documents_search
    BEFORE INSERT OR UPDATE OF display_name, content_text, ai_results
    ON documents
    FOR EACH ROW
    EXECUTE FUNCTION documents_search_update();
```

### Access Control (Fully Relational)

```sql
-- ============================================================
-- DATA ROOM ROLES
-- ============================================================
-- Permissions are relational because they participate in security-
-- critical queries and need explicit boolean column semantics.

CREATE TABLE data_room_roles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(100) NOT NULL,
    description         TEXT,
    can_view            BOOLEAN NOT NULL DEFAULT TRUE,
    can_download        BOOLEAN NOT NULL DEFAULT FALSE,
    can_print           BOOLEAN NOT NULL DEFAULT FALSE,
    can_upload          BOOLEAN NOT NULL DEFAULT FALSE,
    can_delete          BOOLEAN NOT NULL DEFAULT FALSE,
    can_manage_users    BOOLEAN NOT NULL DEFAULT FALSE,
    can_manage_qa       BOOLEAN NOT NULL DEFAULT FALSE,
    can_view_analytics  BOOLEAN NOT NULL DEFAULT FALSE,
    can_manage_redactions BOOLEAN NOT NULL DEFAULT FALSE,
    is_system_role      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_roles_room ON data_room_roles(data_room_id);

-- ============================================================
-- USER GROUPS
-- ============================================================
CREATE TABLE user_groups (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(100) NOT NULL,
    description         TEXT,
    role_id             UUID NOT NULL REFERENCES data_room_roles(id),
    -- Group-level metadata (JSONB: custom labels, display preferences)
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_groups_room ON user_groups(data_room_id);

-- ============================================================
-- DATA ROOM USERS
-- ============================================================
CREATE TABLE data_room_users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    group_id            UUID REFERENCES user_groups(id),
    role_id             UUID NOT NULL REFERENCES data_room_roles(id),
    nda_accepted        BOOLEAN NOT NULL DEFAULT FALSE,
    nda_accepted_at     TIMESTAMPTZ,
    access_starts_at    TIMESTAMPTZ,
    access_expires_at   TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'invited'
                        CHECK (status IN ('invited', 'active', 'suspended', 'revoked')),
    invited_by          UUID REFERENCES users(id),
    -- User-specific access metadata (JSONB: tracks NDA details, access restrictions)
    access_metadata     JSONB NOT NULL DEFAULT '{
        "ndaVersion": null,
        "ndaIpAddress": null,
        "customRestrictions": [],
        "lastActiveAt": null,
        "totalSessions": 0
    }'::JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (data_room_id, user_id)
);

CREATE INDEX idx_room_users_room ON data_room_users(data_room_id);
CREATE INDEX idx_room_users_user ON data_room_users(user_id);
CREATE INDEX idx_room_users_group ON data_room_users(group_id);

-- ============================================================
-- RESOURCE PERMISSIONS (granular overrides - fully relational)
-- ============================================================
CREATE TABLE resource_permissions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    user_id             UUID REFERENCES users(id),
    group_id            UUID REFERENCES user_groups(id),
    role_id             UUID REFERENCES data_room_roles(id),
    folder_id           UUID REFERENCES folders(id) ON DELETE CASCADE,
    document_id         UUID REFERENCES documents(id) ON DELETE CASCADE,
    can_view            BOOLEAN,
    can_download        BOOLEAN,
    can_print           BOOLEAN,
    can_upload          BOOLEAN,
    inherit             BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID NOT NULL REFERENCES users(id),
    CHECK (
        (user_id IS NOT NULL AND group_id IS NULL AND role_id IS NULL) OR
        (user_id IS NULL AND group_id IS NOT NULL AND role_id IS NULL) OR
        (user_id IS NULL AND group_id IS NULL AND role_id IS NOT NULL)
    ),
    CHECK (
        (folder_id IS NOT NULL AND document_id IS NULL) OR
        (folder_id IS NULL AND document_id IS NOT NULL)
    )
);

CREATE INDEX idx_perms_folder ON resource_permissions(folder_id);
CREATE INDEX idx_perms_document ON resource_permissions(document_id);
CREATE INDEX idx_perms_user ON resource_permissions(user_id);
CREATE INDEX idx_perms_group ON resource_permissions(group_id);
CREATE INDEX idx_perms_room ON resource_permissions(data_room_id);
```

### Q&A Workflow (Relational Core + JSONB Details)

```sql
-- ============================================================
-- Q&A TOPICS (Relational)
-- ============================================================
CREATE TABLE qa_topics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- Q&A QUESTIONS
-- ============================================================
-- Core workflow fields are relational (status, priority, SLA tracking).
-- Structured content and thread history use JSONB.

CREATE TABLE qa_questions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    topic_id            UUID REFERENCES qa_topics(id),
    question_number     VARCHAR(20) NOT NULL,
    subject             VARCHAR(500) NOT NULL,
    body                TEXT NOT NULL,
    related_folder_id   UUID REFERENCES folders(id),
    related_document_id UUID REFERENCES documents(id),
    -- Workflow (relational: filtered, sorted, SLA-tracked)
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted'
                        CHECK (status IN ('draft', 'submitted', 'assigned', 'answered',
                                          'follow_up', 'closed', 'rejected')),
    priority            VARCHAR(10) NOT NULL DEFAULT 'normal'
                        CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    assigned_to         UUID REFERENCES users(id),
    assigned_group      UUID REFERENCES user_groups(id),
    sla_deadline        TIMESTAMPTZ,
    sla_breached        BOOLEAN NOT NULL DEFAULT FALSE,
    visibility          VARCHAR(20) NOT NULL DEFAULT 'submitter_and_assignee'
                        CHECK (visibility IN ('all_parties', 'submitter_and_assignee',
                                              'sell_side_only', 'buy_side_only')),
    -- Answer thread (JSONB: variable-length list of answers with review history)
    answers             JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "id": "uuid",
    --     "body": "Please refer to document 3.2.1...",
    --     "attachmentDocumentIds": ["uuid1", "uuid2"],
    --     "status": "approved",
    --     "answeredBy": "uuid",
    --     "answeredAt": "2026-05-26T14:00:00Z",
    --     "approvedBy": "uuid",
    --     "approvedAt": "2026-05-26T15:00:00Z",
    --     "revisions": [
    --       {"body": "Original answer text...", "revisedAt": "2026-05-26T13:00:00Z"}
    --     ]
    --   }
    -- ]
    -- Timestamps
    submitted_at        TIMESTAMPTZ,
    answered_at         TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    submitted_by        UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_qa_room ON qa_questions(data_room_id);
CREATE INDEX idx_qa_status ON qa_questions(data_room_id, status);
CREATE INDEX idx_qa_assigned ON qa_questions(assigned_to);
CREATE INDEX idx_qa_sla ON qa_questions(sla_deadline) WHERE NOT sla_breached;
CREATE INDEX idx_qa_answers ON qa_questions USING gin(answers jsonb_path_ops);
```

### Audit Trail (Relational Envelope + JSONB Details)

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================
-- The audit log uses a relational envelope (who, what, when, where)
-- for indexed querying and compliance exports, with JSONB for the
-- variable detail payload. This avoids creating separate audit
-- detail tables for each action type.

CREATE TABLE audit_log (
    id                  BIGSERIAL,
    organization_id     UUID NOT NULL,
    data_room_id        UUID,
    -- Actor (relational: always present, always queried)
    user_id             UUID NOT NULL,
    user_email          VARCHAR(255) NOT NULL,
    user_ip             INET,
    user_agent          TEXT,
    -- Action (relational: indexed for filtering and compliance reports)
    action_category     VARCHAR(50) NOT NULL,
    -- 'document', 'permission', 'user', 'qa', 'room', 'auth', 'security'
    action              VARCHAR(50) NOT NULL,
    -- 'document.viewed', 'document.downloaded', 'permission.granted', etc.
    -- Resource (relational: indexed for per-resource audit trails)
    resource_type       VARCHAR(50) NOT NULL,
    resource_id         UUID,
    resource_name       VARCHAR(500),
    -- Action-specific details (JSONB: varies per action type)
    details             JSONB NOT NULL DEFAULT '{}',
    -- Examples by action type:
    -- document.viewed: {"pageNumbers": [1,2,3], "viewDurationSeconds": 45, "deviceType": "desktop"}
    -- document.downloaded: {"format": "pdf", "watermarked": true, "watermarkId": "abc123"}
    -- permission.granted: {"targetUserId": "...", "resourceType": "folder", "permissions": {"canView": true, "canDownload": false}}
    -- redaction.approved: {"redactionId": "...", "piiType": "ssn", "pageNumber": 3}
    -- login.failed: {"reason": "invalid_password", "attemptCount": 3}
    -- anomaly.detected: {"anomalyType": "bulk_download", "severity": "high", "documentsAccessed": 47, "timeWindowMinutes": 5}
    -- Timestamp
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... automated via pg_partman

CREATE INDEX idx_audit_org_room ON audit_log(organization_id, data_room_id, created_at);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_action ON audit_log(action_category, action, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id, created_at);
-- GIN index on details for ad-hoc compliance queries
CREATE INDEX idx_audit_details ON audit_log USING gin(details jsonb_path_ops);

-- WORM compliance
CREATE RULE audit_log_no_update AS ON UPDATE TO audit_log DO INSTEAD NOTHING;
CREATE RULE audit_log_no_delete AS ON DELETE TO audit_log DO INSTEAD NOTHING;
```

### Analytics and Engagement Tracking

```sql
-- ============================================================
-- DOCUMENT VIEW SESSIONS
-- ============================================================
-- Session envelope is relational. Per-page engagement data
-- is stored as a JSONB array within the session, avoiding a
-- separate high-volume table for page-level events.

CREATE TABLE document_view_sessions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID NOT NULL REFERENCES documents(id),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at            TIMESTAMPTZ,
    total_duration_seconds INTEGER,
    ip_address          INET,
    user_agent          TEXT,
    device_type         VARCHAR(20),
    watermark_id        VARCHAR(100) NOT NULL,
    -- Page-level engagement data (JSONB: variable-length per session)
    page_events         JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"page": 1, "viewMs": 12000, "scrollDepth": 100.0, "viewedAt": "2026-05-26T10:00:01Z"},
    --   {"page": 2, "viewMs": 45000, "scrollDepth": 85.5, "viewedAt": "2026-05-26T10:00:13Z"},
    --   {"page": 3, "viewMs": 3000, "scrollDepth": 30.0, "viewedAt": "2026-05-26T10:00:58Z"}
    -- ]
    -- Session-level analytics (JSONB: computed on session end)
    session_analytics   JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "totalPagesViewed": 15,
    --   "uniquePagesViewed": 12,
    --   "avgPageViewMs": 8500,
    --   "maxPageViewMs": 45000,
    --   "maxPageViewPage": 7,
    --   "scrollCompletionPct": 72.5,
    --   "readingPattern": "sequential"  -- or "random", "focused"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_doc ON document_view_sessions(document_id);
CREATE INDEX idx_sessions_user ON document_view_sessions(user_id);
CREATE INDEX idx_sessions_room ON document_view_sessions(data_room_id, started_at);
-- Expression index for finding sessions with specific reading patterns
CREATE INDEX idx_sessions_pattern ON document_view_sessions((session_analytics->>'readingPattern'))
    WHERE session_analytics->>'readingPattern' IS NOT NULL;

-- ============================================================
-- USER ENGAGEMENT SCORES (Materialized view)
-- ============================================================
CREATE MATERIALIZED VIEW mv_user_engagement AS
SELECT
    dru.data_room_id,
    dru.organization_id,
    dru.user_id,
    u.display_name,
    u.email,
    COUNT(DISTINCT dvs.id) AS total_sessions,
    COUNT(DISTINCT dvs.document_id) AS documents_viewed,
    SUM(dvs.total_duration_seconds) AS total_view_seconds,
    MAX(dvs.started_at) AS last_activity_at,
    COUNT(DISTINCT qq.id) FILTER (WHERE qq.id IS NOT NULL) AS questions_asked,
    (
        COALESCE(COUNT(DISTINCT dvs.document_id), 0) * 2 +
        COALESCE(SUM(dvs.total_duration_seconds)::NUMERIC / 3600, 0) * 10 +
        COALESCE(COUNT(DISTINCT qq.id) FILTER (WHERE qq.id IS NOT NULL), 0) * 5
    ) AS engagement_score
FROM data_room_users dru
JOIN users u ON u.id = dru.user_id
LEFT JOIN document_view_sessions dvs ON dvs.user_id = dru.user_id
    AND dvs.data_room_id = dru.data_room_id
LEFT JOIN qa_questions qq ON qq.submitted_by = dru.user_id
    AND qq.data_room_id = dru.data_room_id
GROUP BY dru.data_room_id, dru.organization_id, dru.user_id,
         u.display_name, u.email;

CREATE UNIQUE INDEX idx_mv_engagement ON mv_user_engagement(data_room_id, user_id);
```

### Messaging, Notifications, and Integrations

```sql
-- ============================================================
-- ENCRYPTED MESSAGES
-- ============================================================
CREATE TABLE messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    thread_id           UUID REFERENCES messages(id),
    body_encrypted      BYTEA NOT NULL,
    body_nonce          BYTEA NOT NULL,
    attachment_document_ids UUID[],
    visibility          VARCHAR(20) NOT NULL DEFAULT 'all'
                        CHECK (visibility IN ('all', 'group', 'direct')),
    target_group_id     UUID REFERENCES user_groups(id),
    target_user_id      UUID REFERENCES users(id),
    -- Message metadata (JSONB: reactions, read receipts, delivery status)
    message_metadata    JSONB NOT NULL DEFAULT '{
        "readBy": [],
        "reactions": [],
        "deliveryStatus": "sent"
    }'::JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_by             UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_messages_room ON messages(data_room_id, created_at);
CREATE INDEX idx_messages_thread ON messages(thread_id);

-- ============================================================
-- NOTIFICATIONS (JSONB-heavy: payload varies by notification type)
-- ============================================================
CREATE TABLE notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    notification_type   VARCHAR(50) NOT NULL,
    -- Notification content (JSONB: varies entirely by type)
    content             JSONB NOT NULL,
    -- Examples:
    -- type=document.uploaded: {"title": "New document in Financials", "documentName": "Q3 Report.pdf", "uploaderName": "John Doe", "roomId": "...", "folderId": "..."}
    -- type=qa.assigned: {"title": "New question assigned", "questionNumber": "Q-042", "subject": "Revenue recognition policy", "roomId": "...", "questionId": "..."}
    -- type=sla.breach: {"title": "SLA breached on Q-042", "questionNumber": "Q-042", "hoursOverdue": 4.5, "roomId": "...", "questionId": "..."}
    -- type=anomaly.detected: {"title": "Suspicious access detected", "userName": "Jane Smith", "anomalyType": "bulk_download", "severity": "high", "roomId": "..."}
    -- Status
    read                BOOLEAN NOT NULL DEFAULT FALSE,
    read_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, read, created_at);
CREATE INDEX idx_notifications_type ON notifications(notification_type, created_at);

-- ============================================================
-- INTEGRATIONS (relational identity + JSONB config)
-- ============================================================
CREATE TABLE integrations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    integration_type    VARCHAR(50) NOT NULL
                        CHECK (integration_type IN ('salesforce', 'docusign', 'microsoft365',
                                                     'slack', 'webhook', 'saml_sso', 'custom_api')),
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'paused', 'error', 'revoked')),
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    -- Integration-specific configuration (JSONB: varies completely by integration type)
    config              JSONB NOT NULL DEFAULT '{}',
    -- Encrypted credentials stored separately for security
    credentials_encrypted BYTEA NOT NULL,
    credentials_nonce   BYTEA NOT NULL,
    -- Sync state (JSONB: tracks integration-specific cursors and checkpoints)
    sync_state          JSONB NOT NULL DEFAULT '{}',
    -- Example for Salesforce:
    -- {"lastSyncCursor": "2026-05-26T10:00:00Z", "syncedObjects": {"Opportunity": 45, "Contact": 120}, "fieldMappings": {"dealName": "Opportunity.Name"}}
    -- Example for DocuSign:
    -- {"envelopeTemplateId": "...", "signerRoleMappings": {"seller_counsel": "role_id_1"}, "completedEnvelopes": 12}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_integrations_org ON integrations(organization_id);
CREATE INDEX idx_integrations_type ON integrations(integration_type);

-- ============================================================
-- WEBHOOK SUBSCRIPTIONS
-- ============================================================
CREATE TABLE webhook_subscriptions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    data_room_id        UUID REFERENCES data_rooms(id),
    url                 TEXT NOT NULL,
    secret_hash         VARCHAR(255) NOT NULL,
    events              TEXT[] NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Delivery tracking (JSONB: recent delivery history for debugging)
    delivery_log        JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"eventType": "document.uploaded", "deliveredAt": "2026-05-26T10:01:00Z", "statusCode": 200, "durationMs": 120},
    --   {"eventType": "room.opened", "deliveredAt": "2026-05-26T09:00:00Z", "statusCode": 200, "durationMs": 95}
    -- ]
    -- (keep last 100 deliveries; older entries pruned by application)
    failure_count       INTEGER NOT NULL DEFAULT 0,
    last_delivery_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhooks_org ON webhook_subscriptions(organization_id);
```

### Watermarking and Remote Wipe

```sql
-- ============================================================
-- WATERMARK CONFIGURATIONS (JSONB: complex visual styling config)
-- ============================================================
CREATE TABLE watermark_configs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    -- Config stored as JSONB because watermark styling is a complex
    -- nested structure that evolves as new watermark features ship
    config              JSONB NOT NULL DEFAULT '{
        "visible": {
            "enabled": true,
            "template": "{{user_email}} | {{datetime}} | CONFIDENTIAL",
            "opacity": 0.15,
            "position": "diagonal",
            "fontSize": 14,
            "color": "#000000",
            "rotation": -45
        },
        "invisible": {
            "enabled": true,
            "payloadFields": ["user_id", "session_id", "timestamp"],
            "algorithm": "dct_steganography",
            "strength": "medium"
        }
    }'::JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- REMOTE WIPE REQUESTS
-- ============================================================
CREATE TABLE remote_wipe_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    target_user_id      UUID REFERENCES users(id),
    target_document_id  UUID REFERENCES documents(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'in_progress', 'completed', 'failed')),
    devices_targeted    INTEGER NOT NULL DEFAULT 0,
    devices_confirmed   INTEGER NOT NULL DEFAULT 0,
    -- Device-level wipe tracking (JSONB: variable per device type)
    device_results      JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"deviceId": "...", "deviceType": "desktop", "os": "macOS", "confirmedAt": "2026-05-26T11:00:00Z", "method": "cache_clear"},
    --   {"deviceId": "...", "deviceType": "mobile", "os": "iOS", "confirmedAt": null, "status": "pending", "lastAttempt": "2026-05-26T11:05:00Z"}
    -- ]
    requested_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ,
    requested_by        UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_wipe_room ON remote_wipe_requests(data_room_id);
```

---

## Querying Patterns: Leveraging Both Worlds

### Finding documents by AI classification (JSONB expression index)

```sql
-- Fast: uses the expression index on ai_results->>'classification'
SELECT id, display_name, ai_results->>'classification' AS classification,
       (ai_results->>'classificationConfidence')::NUMERIC AS confidence
FROM documents
WHERE data_room_id = $1
  AND status = 'active'
  AND ai_results->>'classification' = 'financial_statement'
ORDER BY (ai_results->>'classificationConfidence')::NUMERIC DESC;
```

### Finding documents with specific tags (GIN containment)

```sql
-- Uses GIN index on ai_results->'tags'
SELECT id, display_name, ai_results->'tags' AS tags
FROM documents
WHERE data_room_id = $1
  AND status = 'active'
  AND ai_results->'tags' @> '["audited", "annual"]'::JSONB;
```

### Filtering by custom metadata (GIN containment)

```sql
-- Tenant-defined custom fields, zero schema migration required
SELECT id, display_name, custom_metadata
FROM documents
WHERE data_room_id = $1
  AND status = 'active'
  AND custom_metadata @> '{"confidentialityLevel": "highly_confidential"}'::JSONB;
```

### Page-level heatmap from session JSONB

```sql
-- Aggregate page-level data from JSONB arrays across sessions
SELECT
    page_data->>'page' AS page_number,
    COUNT(*) AS total_views,
    AVG((page_data->>'viewMs')::INTEGER) AS avg_view_ms,
    SUM((page_data->>'viewMs')::INTEGER) AS total_view_ms
FROM document_view_sessions,
     jsonb_array_elements(page_events) AS page_data
WHERE document_id = $1
  AND data_room_id = $2
GROUP BY page_data->>'page'
ORDER BY (page_data->>'page')::INTEGER;
```

### Deal-type-specific queries

```sql
-- Find all M&A rooms past their exclusivity deadline
SELECT id, name, deal_metadata->>'targetCompany' AS target,
       deal_metadata->>'exclusivityDeadline' AS deadline
FROM data_rooms
WHERE organization_id = $1
  AND deal_type = 'mna'
  AND status = 'active'
  AND (deal_metadata->>'exclusivityDeadline')::DATE < CURRENT_DATE;

-- Find all fundraising rooms by round type
SELECT id, name, deal_metadata->>'roundType' AS round,
       (deal_metadata->>'targetRaise')::NUMERIC AS target_raise
FROM data_rooms
WHERE organization_id = $1
  AND deal_type = 'fundraising'
  AND deal_metadata @> '{"roundType": "Series B"}'::JSONB;
```

---

## JSONB Schema Validation Strategy

Since JSONB columns have no database-level schema enforcement, validation happens at the application layer:

```typescript
// Example: Zod schema for document AI results
const aiResultsSchema = z.object({
    classification: z.string().nullable(),
    classificationConfidence: z.number().min(0).max(1).nullable(),
    tags: z.array(z.string()),
    summary: z.string().nullable(),
    language: z.string().nullable(),
    modelVersion: z.string().nullable(),
    classifiedAt: z.string().datetime().nullable(),
    entities: z.array(z.object({
        type: z.string(),
        value: z.string(),
        count: z.number().int().positive()
    })),
    keyPhrases: z.array(z.string()),
    sentiment: z.enum(['positive', 'negative', 'neutral', 'mixed']).nullable(),
    documentType: z.string().nullable()
});

// Validate before every write
function updateAiResults(documentId: string, results: unknown) {
    const validated = aiResultsSchema.parse(results);
    return db.query(
        `UPDATE documents SET ai_results = $1, updated_at = NOW() WHERE id = $2`,
        [JSON.stringify(validated), documentId]
    );
}
```

For schema evolution, version the JSONB structure:

```typescript
// Migration function applied on read when schema version is outdated
function migrateAiResults(raw: unknown): AiResults {
    const data = raw as Record<string, unknown>;
    // v1 -> v2: added 'entities' and 'keyPhrases'
    if (!data.entities) data.entities = [];
    if (!data.keyPhrases) data.keyPhrases = [];
    // v2 -> v3: added 'sentiment' and 'documentType'
    if (!('sentiment' in data)) data.sentiment = null;
    if (!('documentType' in data)) data.documentType = null;
    return aiResultsSchema.parse(data);
}
```

---

## Pros and Cons

### Pros

1. **Single database, no operational complexity.** The entire data model lives in PostgreSQL. There is no separate document database to maintain, no synchronization to manage, and no additional failure modes. Operational teams manage one database engine with familiar tools (pg_dump, pg_restore, pgAudit).

2. **Zero-migration flexibility for AI and custom fields.** When the AI classification model changes its output format, or when a new PII type is added, or when a tenant adds custom metadata fields, no ALTER TABLE is needed. The JSONB column absorbs the change immediately. This is crucial for a product iterating rapidly on AI capabilities.

3. **Relational integrity where it matters.** Foreign keys enforce that every document belongs to a real folder, every permission references a real user and role, and every Q&A question lives in a real data room. The structural integrity of the room hierarchy is never compromised by JSONB flexibility.

4. **Excellent query performance on both data types.** Relational columns use B-tree indexes for equality and range queries. JSONB columns use GIN indexes for containment queries and expression indexes for specific key lookups. PostgreSQL's query planner efficiently combines both in a single query.

5. **Natural fit for deal-type polymorphism.** M&A rooms, fundraising rooms, IPO rooms, and regulatory rooms share the same `data_rooms` table structure but have completely different custom fields in `deal_metadata`. This avoids either a rigid union of all possible columns or an entity-attribute-value anti-pattern.

6. **Embedded page-level analytics.** Storing page view events as a JSONB array within the session record avoids a separate high-volume table with billions of rows. For rooms with moderate traffic, this is sufficient. For high-volume analytics, the JSONB data can be extracted and loaded into ClickHouse without changing the core schema.

7. **Compliance-friendly audit trail.** The audit log uses relational columns for the indexed envelope (who, what, when) and JSONB for variable details. Compliance queries ("show all document downloads by user X in room Y") run on relational indexes. Forensic deep-dives into specific events read the JSONB details.

8. **Gradual migration path.** If a JSONB field stabilizes and becomes critical for filtering (e.g., `ai_results->>'classification'` is queried constantly), it can be promoted to a generated column or extracted to its own relational column with zero application changes.

### Cons

1. **JSONB schema drift risk.** Without database-level enforcement, different application versions or services might write inconsistent JSONB structures. A bug in the classification service could write `"classificaton"` (typo) without any database error. Mitigation: strict application-layer validation (Zod/JSON Schema) and CI tests that verify JSONB schemas.

2. **JSONB indexing limitations.** GIN indexes support containment (`@>`) and existence (`?`) operators but not range queries on nested numeric values. The expression index `((ai_results->>'classificationConfidence')::NUMERIC)` works for a specific key but does not generalize to arbitrary nested paths. Complex JSONB queries can fall back to sequential scans.

3. **No referential integrity within JSONB.** The `answers` array in `qa_questions` references user IDs and document IDs as strings, but PostgreSQL cannot enforce that those IDs actually exist. If a referenced document is deleted, the JSONB retains a dangling reference. Mitigation: application-layer integrity checks and cascade handlers.

4. **JSONB storage overhead.** JSONB stores field names with every row (unlike relational columns where the field name lives only in the catalog). For documents with large `redaction_results` (dozens of detections), the repeated key names ("pageNumber", "bounds", "piiType") add 20-40% storage overhead compared to a normalized approach.

5. **Limited JSONB update granularity.** Updating a single field within a JSONB column requires reading the full JSONB value, modifying it, and writing it back. PostgreSQL's `jsonb_set()` function helps but still rewrites the entire JSONB value on disk. For the `page_events` array in `document_view_sessions`, appending events during a long session means repeatedly rewriting a growing JSON array. Mitigation: buffer page events in Redis and flush to the JSONB column on session end.

6. **Testing and debugging JSONB is harder.** SQL queries with JSONB operators (`->`, `->>`, `@>`, `#>`) are less intuitive than simple column references. Developers need to understand JSONB path expressions, containment semantics, and GIN index behavior. Error messages for malformed JSONB queries are less helpful than standard SQL errors.

7. **Reporting and BI tool compatibility.** Many BI tools (Metabase, Looker, Tableau) handle relational columns well but struggle with JSONB. Building reports on data inside JSONB columns often requires creating views or materialized views that extract the JSONB fields into flat relational columns, adding maintenance overhead.

---

## Migration and Scaling Considerations

### From Fully Normalized (Suggestion 1) to Hybrid

1. **Identify migration candidates.** Review tables with frequent ALTER TABLE history (columns added every sprint), sparse columns (many NULLs), or EAV patterns. These are prime candidates for JSONB consolidation.

2. **Add JSONB columns alongside existing relational columns.** During a transition period, write to both the relational column and the JSONB field. Validate that reads from JSONB match the relational column.

3. **Migrate reads to JSONB.** Update application queries to read from JSONB. Remove the relational column in a subsequent migration once all reads are migrated.

4. **Create expression indexes.** For any JSONB field that is filtered frequently, create an expression index to maintain query performance parity with the former relational column.

### Scaling Strategy

1. **Phase 1 (0-100 data rooms):** Single PostgreSQL 16 instance. JSONB queries are fast with GIN indexes. Page events stored inline in session JSONB.

2. **Phase 2 (100-1,000 data rooms):** Read replicas for analytics. Materialized views for engagement scores. Audit log partitioned monthly. Consider extracting page-level event data from session JSONB into a dedicated table if query patterns demand per-page aggregation.

3. **Phase 3 (1,000-10,000 data rooms):** Shard by `organization_id` using Citus. Export analytics data to ClickHouse for heavy aggregation. Promote heavily-queried JSONB fields to generated columns.

4. **Phase 4 (10,000+ data rooms):** Per-tenant schema isolation for enterprise customers. Archive closed room data to cold storage. JSONB compression (PostgreSQL TOAST handles this automatically for large values, but explicit pglz or lz4 tuning may help).

### JSONB Column Maintenance

```sql
-- Promote a frequently-queried JSONB field to a generated column
ALTER TABLE documents ADD COLUMN ai_classification VARCHAR(100)
    GENERATED ALWAYS AS (ai_results->>'classification') STORED;

-- This creates a regular relational column that is automatically
-- maintained from the JSONB column, giving you B-tree index capability
-- without changing any write logic.

CREATE INDEX idx_docs_ai_class_gen ON documents(ai_classification);
```

### Backup and Disaster Recovery

- Same PostgreSQL backup strategy as the fully relational model: continuous WAL archiving, PITR, cross-region replication.
- JSONB data is included in standard pg_dump output and WAL shipping -- no special handling required.
- For compliance exports, create views that flatten JSONB into relational columns for auditor-friendly CSV/PDF generation.

### Performance Optimization

- **JSONB TOAST compression:** Large JSONB values (redaction results with many detections) are automatically compressed and stored out-of-line via TOAST. Monitor `pg_stat_user_tables.n_tup_hot_upd` to ensure JSONB updates use HOT updates when possible.
- **Partial indexes on JSONB expressions:** Only index the JSONB paths you actually query. Avoid broad GIN indexes on large JSONB columns unless containment queries are common.
- **Connection pooling:** Same PgBouncer setup as the relational model.
- **Redis caching:** Cache frequently-accessed JSONB values (room settings, watermark configs) in Redis to avoid repeated JSONB parsing on the PostgreSQL side.
- **Batch JSONB writes:** For page-level events, buffer in Redis and write the complete JSONB array once per session rather than updating the JSONB column on every page view.
