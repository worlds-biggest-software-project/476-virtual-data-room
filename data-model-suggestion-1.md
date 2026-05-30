# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Virtual Data Room (476) -- Generated: 2026-05-26

## Overview

This model uses a fully normalized relational schema in PostgreSQL, leveraging its mature ACID compliance, row-level security for multi-tenant isolation, and strong indexing capabilities. Every entity is stored in its own table with explicit foreign key relationships. This is the most conventional and well-understood approach, offering predictable performance, straightforward querying, and alignment with compliance frameworks (ISO 27001, SOC 2, SEC 17a-4) that expect structured, auditable data stores.

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary database | PostgreSQL 16+ | ACID compliance, RLS, full-text search, partitioning, JSONB fallback |
| Connection pooling | PgBouncer or Supavisor | Multi-tenant connection efficiency |
| Search index | PostgreSQL tsvector + GIN indexes | Full-text document search without external dependency |
| File storage | S3-compatible object store (AWS S3, MinIO) | Documents stored as encrypted objects; DB holds metadata |
| Encryption at rest | AWS KMS / HashiCorp Vault | Per-tenant or per-document encryption key management |
| Migrations | Flyway or golang-migrate | Versioned, auditable schema migrations |
| Caching | Redis | Session cache, permission cache, analytics aggregation |

## Multi-Tenant Isolation Strategy

The schema uses a shared-database, shared-schema approach with `organization_id` as the tenant discriminator on every table. PostgreSQL Row-Level Security (RLS) policies enforce isolation at the database level, ensuring that even application bugs cannot leak data across tenants.

```sql
-- Example RLS policy applied to every tenant-scoped table
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
    USING (organization_id = current_setting('app.current_org_id')::UUID);

-- Application sets tenant context on each connection
SET app.current_org_id = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';
```

With a composite index on `(organization_id, ...)` as the leading column on all tenant-scoped tables, RLS overhead is typically 2-4% on queries.

---

## Complete Schema Definition

### Organizations and Users

```sql
-- ============================================================
-- ORGANIZATIONS (Tenants)
-- ============================================================
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free'
                    CHECK (plan_tier IN ('free', 'professional', 'enterprise')),
    billing_email   VARCHAR(255),
    logo_url        TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Compliance
    data_residency_region VARCHAR(20) NOT NULL DEFAULT 'us-east-1',
    iso_27001_certified   BOOLEAN NOT NULL DEFAULT FALSE,
    soc2_certified        BOOLEAN NOT NULL DEFAULT FALSE,
    hipaa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    suspended_at    TIMESTAMPTZ
);

CREATE INDEX idx_organizations_slug ON organizations(slug);

-- ============================================================
-- USERS
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),  -- NULL for passwordless/SSO users
    phone           VARCHAR(50),
    avatar_url      TEXT,
    -- Authentication
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'email'
                    CHECK (auth_provider IN ('email', 'saml', 'oidc', 'passwordless')),
    auth_provider_id VARCHAR(255),
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_secret      VARCHAR(255),
    -- Device binding
    device_fingerprints JSONB NOT NULL DEFAULT '[]',
    -- Status
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'deactivated')),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_auth_provider ON users(auth_provider, auth_provider_id);

-- ============================================================
-- ORGANIZATION MEMBERSHIPS
-- ============================================================
CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    org_role        VARCHAR(50) NOT NULL DEFAULT 'member'
                    CHECK (org_role IN ('owner', 'admin', 'member', 'billing')),
    invited_by      UUID REFERENCES users(id),
    invited_at      TIMESTAMPTZ,
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_org_memberships_user ON organization_memberships(user_id);
```

### Data Rooms

```sql
-- ============================================================
-- DATA ROOMS
-- ============================================================
CREATE TABLE data_rooms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- Deal metadata
    deal_type       VARCHAR(50) NOT NULL DEFAULT 'mna'
                    CHECK (deal_type IN ('mna', 'fundraising', 'ipo', 'regulatory',
                                         'board_reporting', 'investor_relations', 'other')),
    deal_value      NUMERIC(18, 2),
    deal_currency   VARCHAR(3) DEFAULT 'USD',
    -- Lifecycle
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'setup', 'active', 'closed', 'archived')),
    opened_at       TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    archived_at     TIMESTAMPTZ,
    -- Settings
    watermark_enabled       BOOLEAN NOT NULL DEFAULT TRUE,
    watermark_template      TEXT DEFAULT '{{user_email}} - {{date}} - Confidential',
    invisible_watermark     BOOLEAN NOT NULL DEFAULT TRUE,
    download_enabled        BOOLEAN NOT NULL DEFAULT FALSE,
    print_enabled           BOOLEAN NOT NULL DEFAULT FALSE,
    nda_required            BOOLEAN NOT NULL DEFAULT FALSE,
    nda_document_id         UUID,  -- FK added after documents table
    -- AI features
    ai_classification_enabled   BOOLEAN NOT NULL DEFAULT TRUE,
    ai_redaction_enabled        BOOLEAN NOT NULL DEFAULT TRUE,
    ai_readiness_scoring        BOOLEAN NOT NULL DEFAULT FALSE,
    -- Compliance
    retention_days          INTEGER NOT NULL DEFAULT 2190,  -- 6 years for SEC 17a-4
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_data_rooms_org ON data_rooms(organization_id);
CREATE INDEX idx_data_rooms_status ON data_rooms(organization_id, status);
CREATE INDEX idx_data_rooms_deal_type ON data_rooms(organization_id, deal_type);

-- ============================================================
-- DEAL READINESS CHECKLISTS
-- ============================================================
CREATE TABLE deal_checklists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    deal_type       VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE deal_checklist_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    checklist_id    UUID NOT NULL REFERENCES deal_checklists(id) ON DELETE CASCADE,
    category        VARCHAR(100) NOT NULL,
    item_name       VARCHAR(255) NOT NULL,
    description     TEXT,
    required        BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    -- AI readiness linkage
    matched_folder_id   UUID,  -- FK to folders
    matched_document_count INTEGER NOT NULL DEFAULT 0,
    completeness_score  NUMERIC(5, 2),  -- 0.00 to 100.00
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_checklist_items_checklist ON deal_checklist_items(checklist_id);
```

### Folders and Documents

```sql
-- ============================================================
-- FOLDERS (Hierarchical via adjacency list + materialized path)
-- ============================================================
CREATE TABLE folders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    parent_id       UUID REFERENCES folders(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    -- Materialized path for efficient hierarchy queries
    -- e.g., '/root-id/parent-id/this-id/'
    path            TEXT NOT NULL,
    depth           INTEGER NOT NULL DEFAULT 0,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    -- Index numbering (e.g., "1.2.3" for VDR standard numbering)
    index_number    VARCHAR(50),
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'hidden', 'deleted')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_folders_room ON folders(data_room_id);
CREATE INDEX idx_folders_parent ON folders(parent_id);
CREATE INDEX idx_folders_path ON folders USING gist (path gist_trgm_ops);
CREATE INDEX idx_folders_org_room ON folders(organization_id, data_room_id);

-- ============================================================
-- DOCUMENTS
-- ============================================================
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    folder_id       UUID NOT NULL REFERENCES folders(id) ON DELETE CASCADE,
    -- File metadata
    original_filename   VARCHAR(500) NOT NULL,
    display_name        VARCHAR(500) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    page_count          INTEGER,
    -- Storage
    storage_bucket      VARCHAR(255) NOT NULL,
    storage_key         VARCHAR(1000) NOT NULL,
    encryption_key_id   VARCHAR(255) NOT NULL,
    checksum_sha256     VARCHAR(64) NOT NULL,
    -- Versioning
    version             INTEGER NOT NULL DEFAULT 1,
    latest              BOOLEAN NOT NULL DEFAULT TRUE,
    parent_document_id  UUID REFERENCES documents(id),
    -- Index numbering
    index_number        VARCHAR(50),
    sort_order          INTEGER NOT NULL DEFAULT 0,
    -- Processing status
    processing_status   VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (processing_status IN (
                            'pending', 'uploading', 'converting',
                            'indexing', 'classifying', 'redacting',
                            'ready', 'failed'
                        )),
    processing_error    TEXT,
    -- Converted PDF (if original was not PDF)
    pdf_storage_key     VARCHAR(1000),
    pdf_page_count      INTEGER,
    -- Full-text search
    content_text        TEXT,
    search_vector       TSVECTOR,
    -- AI classification
    ai_classification   VARCHAR(100),
    ai_classification_confidence NUMERIC(5, 4),  -- 0.0000 to 1.0000
    ai_tags             TEXT[],
    ai_summary          TEXT,
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'superseded', 'deleted', 'wiped')),
    -- Timestamps
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uploaded_by         UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_documents_room ON documents(data_room_id);
CREATE INDEX idx_documents_folder ON documents(folder_id);
CREATE INDEX idx_documents_org_room ON documents(organization_id, data_room_id);
CREATE INDEX idx_documents_search ON documents USING gin(search_vector);
CREATE INDEX idx_documents_classification ON documents(ai_classification);
CREATE INDEX idx_documents_status ON documents(data_room_id, status) WHERE status = 'active';
CREATE INDEX idx_documents_latest ON documents(parent_document_id) WHERE latest = TRUE;

-- Trigger to keep search_vector updated
CREATE FUNCTION documents_search_update() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.display_name, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.ai_summary, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(NEW.content_text, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_documents_search
    BEFORE INSERT OR UPDATE OF display_name, ai_summary, content_text
    ON documents
    FOR EACH ROW
    EXECUTE FUNCTION documents_search_update();
```

### PII Redaction

```sql
-- ============================================================
-- REDACTION RULES (per data room or organization-wide)
-- ============================================================
CREATE TABLE redaction_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    data_room_id    UUID REFERENCES data_rooms(id),  -- NULL = org-wide
    rule_name       VARCHAR(100) NOT NULL,
    pii_type        VARCHAR(50) NOT NULL,  -- e.g., 'ssn', 'credit_card', 'email', 'phone', 'name', 'address'
    pattern         TEXT,  -- regex pattern for rule-based detection
    enabled         BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_redaction_rules_org ON redaction_rules(organization_id);

-- ============================================================
-- REDACTION RESULTS (per document)
-- ============================================================
CREATE TABLE redactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    -- Location in document
    page_number     INTEGER NOT NULL,
    x_offset        NUMERIC(10, 4),
    y_offset        NUMERIC(10, 4),
    width           NUMERIC(10, 4),
    height          NUMERIC(10, 4),
    -- What was detected
    pii_type        VARCHAR(50) NOT NULL,
    original_text   TEXT NOT NULL,  -- stored encrypted
    confidence      NUMERIC(5, 4) NOT NULL,  -- 0.0000 to 1.0000
    detection_method VARCHAR(20) NOT NULL
                    CHECK (detection_method IN ('ai_ner', 'regex', 'manual')),
    -- Review status
    review_status   VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (review_status IN ('pending', 'approved', 'rejected', 'modified')),
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    -- Redacted output
    redacted_storage_key VARCHAR(1000),
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_redactions_document ON redactions(document_id);
CREATE INDEX idx_redactions_status ON redactions(document_id, review_status);
CREATE INDEX idx_redactions_org ON redactions(organization_id);
```

### Access Control (RBAC + Resource-Level Permissions)

```sql
-- ============================================================
-- DATA ROOM ROLES
-- ============================================================
CREATE TABLE data_room_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    -- Permission flags (role-level defaults)
    can_view        BOOLEAN NOT NULL DEFAULT TRUE,
    can_download    BOOLEAN NOT NULL DEFAULT FALSE,
    can_print       BOOLEAN NOT NULL DEFAULT FALSE,
    can_upload      BOOLEAN NOT NULL DEFAULT FALSE,
    can_delete      BOOLEAN NOT NULL DEFAULT FALSE,
    can_manage_users BOOLEAN NOT NULL DEFAULT FALSE,
    can_manage_qa   BOOLEAN NOT NULL DEFAULT FALSE,
    can_view_analytics BOOLEAN NOT NULL DEFAULT FALSE,
    can_manage_redactions BOOLEAN NOT NULL DEFAULT FALSE,
    -- Built-in vs custom
    is_system_role  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_room_roles_room ON data_room_roles(data_room_id);

-- Pre-populated system roles per room
-- 'organizer', 'reviewer', 'bidder', 'legal_counsel', 'viewer'

-- ============================================================
-- USER GROUPS (within a data room)
-- ============================================================
CREATE TABLE user_groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    role_id         UUID NOT NULL REFERENCES data_room_roles(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_groups_room ON user_groups(data_room_id);

-- ============================================================
-- DATA ROOM USER MEMBERSHIPS
-- ============================================================
CREATE TABLE data_room_users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    group_id        UUID REFERENCES user_groups(id),
    role_id         UUID NOT NULL REFERENCES data_room_roles(id),
    -- NDA
    nda_accepted    BOOLEAN NOT NULL DEFAULT FALSE,
    nda_accepted_at TIMESTAMPTZ,
    -- Access window
    access_starts_at TIMESTAMPTZ,
    access_expires_at TIMESTAMPTZ,
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'invited'
                    CHECK (status IN ('invited', 'active', 'suspended', 'revoked')),
    invited_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (data_room_id, user_id)
);

CREATE INDEX idx_room_users_room ON data_room_users(data_room_id);
CREATE INDEX idx_room_users_user ON data_room_users(user_id);
CREATE INDEX idx_room_users_group ON data_room_users(group_id);

-- ============================================================
-- FOLDER / DOCUMENT PERMISSIONS (granular overrides)
-- ============================================================
CREATE TABLE resource_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    -- Target: either a user, group, or role
    user_id         UUID REFERENCES users(id),
    group_id        UUID REFERENCES user_groups(id),
    role_id         UUID REFERENCES data_room_roles(id),
    -- Resource: either a folder or document
    folder_id       UUID REFERENCES folders(id) ON DELETE CASCADE,
    document_id     UUID REFERENCES documents(id) ON DELETE CASCADE,
    -- Permission overrides (NULL = inherit from role/parent)
    can_view        BOOLEAN,
    can_download    BOOLEAN,
    can_print       BOOLEAN,
    can_upload      BOOLEAN,
    -- Inheritance
    inherit         BOOLEAN NOT NULL DEFAULT TRUE,  -- applies to child folders/documents
    -- Constraints
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      UUID NOT NULL REFERENCES users(id),
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

CREATE INDEX idx_resource_perms_folder ON resource_permissions(folder_id);
CREATE INDEX idx_resource_perms_document ON resource_permissions(document_id);
CREATE INDEX idx_resource_perms_user ON resource_permissions(user_id);
CREATE INDEX idx_resource_perms_group ON resource_permissions(group_id);
CREATE INDEX idx_resource_perms_room ON resource_permissions(data_room_id);
```

### Permission Resolution Function

```sql
-- ============================================================
-- PERMISSION RESOLUTION (computed at query time)
-- ============================================================
-- Resolves effective permissions for a user on a document,
-- walking up the folder hierarchy and merging role, group,
-- and user-level overrides.

CREATE OR REPLACE FUNCTION resolve_document_permission(
    p_user_id UUID,
    p_document_id UUID
) RETURNS TABLE (
    can_view BOOLEAN,
    can_download BOOLEAN,
    can_print BOOLEAN,
    can_upload BOOLEAN
) AS $$
DECLARE
    v_room_id UUID;
    v_folder_id UUID;
    v_role_id UUID;
    v_group_id UUID;
    v_result RECORD;
BEGIN
    -- Get document context
    SELECT d.data_room_id, d.folder_id INTO v_room_id, v_folder_id
    FROM documents d WHERE d.id = p_document_id;

    -- Get user's role and group in this room
    SELECT dru.role_id, dru.group_id INTO v_role_id, v_group_id
    FROM data_room_users dru
    WHERE dru.data_room_id = v_room_id AND dru.user_id = p_user_id
      AND dru.status = 'active';

    IF v_role_id IS NULL THEN
        RETURN QUERY SELECT FALSE, FALSE, FALSE, FALSE;
        RETURN;
    END IF;

    -- Start with role defaults
    SELECT drr.can_view, drr.can_download, drr.can_print, drr.can_upload
    INTO v_result
    FROM data_room_roles drr WHERE drr.id = v_role_id;

    -- Override with folder hierarchy permissions (walk up)
    -- Check from document's folder up to root, applying
    -- overrides at each level (most specific wins)
    WITH RECURSIVE folder_chain AS (
        SELECT f.id, f.parent_id, 0 AS depth
        FROM folders f WHERE f.id = v_folder_id
        UNION ALL
        SELECT f.id, f.parent_id, fc.depth + 1
        FROM folders f
        JOIN folder_chain fc ON f.id = fc.parent_id
    )
    SELECT
        COALESCE(
            (SELECT rp.can_view FROM resource_permissions rp
             WHERE rp.folder_id = fc.id
               AND (rp.user_id = p_user_id OR rp.group_id = v_group_id OR rp.role_id = v_role_id)
             ORDER BY
                 CASE WHEN rp.user_id IS NOT NULL THEN 0
                      WHEN rp.group_id IS NOT NULL THEN 1
                      ELSE 2 END,
                 fc.depth
             LIMIT 1),
            v_result.can_view
        ) INTO v_result.can_view
    FROM folder_chain fc
    ORDER BY fc.depth
    LIMIT 1;

    -- Direct document-level override (highest priority)
    SELECT
        COALESCE(rp.can_view, v_result.can_view),
        COALESCE(rp.can_download, v_result.can_download),
        COALESCE(rp.can_print, v_result.can_print),
        COALESCE(rp.can_upload, v_result.can_upload)
    INTO v_result
    FROM resource_permissions rp
    WHERE rp.document_id = p_document_id
      AND (rp.user_id = p_user_id OR rp.group_id = v_group_id OR rp.role_id = v_role_id)
    ORDER BY
        CASE WHEN rp.user_id IS NOT NULL THEN 0
             WHEN rp.group_id IS NOT NULL THEN 1
             ELSE 2 END
    LIMIT 1;

    RETURN QUERY SELECT v_result.can_view, v_result.can_download,
                        v_result.can_print, v_result.can_upload;
END;
$$ LANGUAGE plpgsql STABLE;
```

### Q&A Workflow

```sql
-- ============================================================
-- Q&A TOPICS (grouping for questions)
-- ============================================================
CREATE TABLE qa_topics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- Q&A QUESTIONS
-- ============================================================
CREATE TABLE qa_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    topic_id        UUID REFERENCES qa_topics(id),
    -- Question details
    question_number VARCHAR(20) NOT NULL,  -- e.g., "Q-001"
    subject         VARCHAR(500) NOT NULL,
    body            TEXT NOT NULL,
    -- Related documents
    related_folder_id   UUID REFERENCES folders(id),
    related_document_id UUID REFERENCES documents(id),
    -- Workflow
    status          VARCHAR(20) NOT NULL DEFAULT 'submitted'
                    CHECK (status IN ('draft', 'submitted', 'assigned', 'answered',
                                      'follow_up', 'closed', 'rejected')),
    priority        VARCHAR(10) NOT NULL DEFAULT 'normal'
                    CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    -- Routing
    assigned_to     UUID REFERENCES users(id),
    assigned_group  UUID REFERENCES user_groups(id),
    -- SLA tracking
    sla_deadline    TIMESTAMPTZ,
    sla_breached    BOOLEAN NOT NULL DEFAULT FALSE,
    -- Visibility
    visibility      VARCHAR(20) NOT NULL DEFAULT 'submitter_and_assignee'
                    CHECK (visibility IN ('all_parties', 'submitter_and_assignee',
                                          'sell_side_only', 'buy_side_only')),
    -- Timestamps
    submitted_at    TIMESTAMPTZ,
    answered_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    submitted_by    UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_qa_questions_room ON qa_questions(data_room_id);
CREATE INDEX idx_qa_questions_status ON qa_questions(data_room_id, status);
CREATE INDEX idx_qa_questions_assigned ON qa_questions(assigned_to);
CREATE INDEX idx_qa_questions_sla ON qa_questions(sla_deadline) WHERE NOT sla_breached;

-- ============================================================
-- Q&A ANSWERS
-- ============================================================
CREATE TABLE qa_answers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id     UUID NOT NULL REFERENCES qa_questions(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    body            TEXT NOT NULL,
    -- Attachments
    attachment_document_ids UUID[],
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'submitted', 'approved', 'rejected')),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    answered_by     UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_qa_answers_question ON qa_answers(question_id);
```

### Audit Trail

```sql
-- ============================================================
-- AUDIT LOG (Append-only, immutable)
-- ============================================================
-- Partitioned by month for retention management and query performance.
-- This table is the compliance backbone for SEC 17a-4, ISO 27001,
-- and SOC 2 Type II audit requirements.

CREATE TABLE audit_log (
    id              BIGSERIAL,
    organization_id UUID NOT NULL,
    data_room_id    UUID,
    -- Actor
    user_id         UUID NOT NULL,
    user_email      VARCHAR(255) NOT NULL,
    user_ip         INET,
    user_agent      TEXT,
    -- Action
    action          VARCHAR(50) NOT NULL,
    -- e.g., 'document.viewed', 'document.downloaded', 'document.printed',
    -- 'document.uploaded', 'permission.changed', 'user.invited',
    -- 'qa.submitted', 'qa.answered', 'redaction.approved',
    -- 'room.opened', 'room.closed', 'login.success', 'login.failed'
    -- Resource
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    resource_name   VARCHAR(500),
    -- Details
    details         JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"page_numbers": [1,2,3], "view_duration_seconds": 45,
    --        "previous_value": "...", "new_value": "..."}
    -- Timestamp
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create monthly partitions (automated via pg_partman or cron)
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... etc. automated partition creation

CREATE INDEX idx_audit_org_room ON audit_log(organization_id, data_room_id, created_at);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_action ON audit_log(action, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id, created_at);

-- Prevent updates and deletes (WORM compliance)
CREATE RULE audit_log_no_update AS ON UPDATE TO audit_log DO INSTEAD NOTHING;
CREATE RULE audit_log_no_delete AS ON DELETE TO audit_log DO INSTEAD NOTHING;

-- For SEC 17a-4 WORM compliance, additionally use:
-- 1. A non-superuser application role that owns the insert-only policy
-- 2. pg_audit extension for PostgreSQL-level audit of the audit table itself
-- 3. Periodic hash-chain verification (each row includes hash of previous row)
```

### Analytics and Engagement Tracking

```sql
-- ============================================================
-- DOCUMENT VIEW SESSIONS (page-level engagement tracking)
-- ============================================================
CREATE TABLE document_view_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(id),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    -- Session data
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at        TIMESTAMPTZ,
    total_duration_seconds INTEGER,
    -- Device info
    ip_address      INET,
    user_agent      TEXT,
    device_type     VARCHAR(20),
    -- Watermark ID for this session
    watermark_id    VARCHAR(100) NOT NULL
);

CREATE INDEX idx_view_sessions_doc ON document_view_sessions(document_id);
CREATE INDEX idx_view_sessions_user ON document_view_sessions(user_id);
CREATE INDEX idx_view_sessions_room ON document_view_sessions(data_room_id, started_at);

-- ============================================================
-- PAGE-LEVEL HEATMAP DATA
-- ============================================================
CREATE TABLE page_view_events (
    id              BIGSERIAL PRIMARY KEY,
    session_id      UUID NOT NULL REFERENCES document_view_sessions(id),
    document_id     UUID NOT NULL,
    organization_id UUID NOT NULL,
    page_number     INTEGER NOT NULL,
    view_duration_ms INTEGER NOT NULL,
    scroll_depth_pct NUMERIC(5, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_page_views_session ON page_view_events(session_id);
CREATE INDEX idx_page_views_document ON page_view_events(document_id, page_number);

-- ============================================================
-- USER ENGAGEMENT SCORES (materialized, refreshed periodically)
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
    COUNT(DISTINCT qq.id) AS questions_asked,
    -- Engagement score (weighted composite)
    (
        COALESCE(COUNT(DISTINCT dvs.document_id), 0) * 2 +
        COALESCE(SUM(dvs.total_duration_seconds)::NUMERIC / 3600, 0) * 10 +
        COALESCE(COUNT(DISTINCT qq.id), 0) * 5
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

### Watermarking and Remote Wipe

```sql
-- ============================================================
-- WATERMARK CONFIGURATIONS
-- ============================================================
CREATE TABLE watermark_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    -- Visible watermark
    visible_enabled     BOOLEAN NOT NULL DEFAULT TRUE,
    visible_template    TEXT NOT NULL DEFAULT '{{user_email}} | {{datetime}} | CONFIDENTIAL',
    visible_opacity     NUMERIC(3, 2) NOT NULL DEFAULT 0.15,
    visible_position    VARCHAR(20) NOT NULL DEFAULT 'diagonal',
    visible_font_size   INTEGER NOT NULL DEFAULT 14,
    -- Invisible watermark (steganographic)
    invisible_enabled   BOOLEAN NOT NULL DEFAULT TRUE,
    invisible_payload_fields TEXT[] NOT NULL DEFAULT ARRAY['user_id', 'session_id', 'timestamp'],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- REMOTE WIPE RECORDS
-- ============================================================
CREATE TABLE remote_wipe_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    -- Scope
    target_user_id  UUID REFERENCES users(id),       -- NULL = all users
    target_document_id UUID REFERENCES documents(id), -- NULL = all documents
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'in_progress', 'completed', 'failed')),
    devices_targeted INTEGER NOT NULL DEFAULT 0,
    devices_confirmed INTEGER NOT NULL DEFAULT 0,
    -- Timestamps
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    requested_by    UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_wipe_requests_room ON remote_wipe_requests(data_room_id);
```

### Messaging and Notifications

```sql
-- ============================================================
-- ENCRYPTED MESSAGES (in-room communication)
-- ============================================================
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id    UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    -- Thread support
    thread_id       UUID REFERENCES messages(id),
    -- Content (encrypted at rest via application-layer encryption)
    body_encrypted  BYTEA NOT NULL,
    body_nonce      BYTEA NOT NULL,
    -- Attachments
    attachment_document_ids UUID[],
    -- Visibility
    visibility      VARCHAR(20) NOT NULL DEFAULT 'all'
                    CHECK (visibility IN ('all', 'group', 'direct')),
    target_group_id UUID REFERENCES user_groups(id),
    target_user_id  UUID REFERENCES users(id),
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_by         UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_messages_room ON messages(data_room_id, created_at);
CREATE INDEX idx_messages_thread ON messages(thread_id);

-- ============================================================
-- NOTIFICATIONS
-- ============================================================
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    -- Content
    notification_type VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    -- Link target
    data_room_id    UUID REFERENCES data_rooms(id),
    resource_type   VARCHAR(50),
    resource_id     UUID,
    -- Status
    read            BOOLEAN NOT NULL DEFAULT FALSE,
    read_at         TIMESTAMPTZ,
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, read, created_at);
```

### Integrations

```sql
-- ============================================================
-- EXTERNAL INTEGRATIONS
-- ============================================================
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    integration_type VARCHAR(50) NOT NULL
                    CHECK (integration_type IN ('salesforce', 'docusign', 'microsoft365',
                                                 'slack', 'webhook', 'saml_sso', 'custom_api')),
    name            VARCHAR(255) NOT NULL,
    -- Credentials (encrypted)
    config_encrypted BYTEA NOT NULL,
    config_nonce    BYTEA NOT NULL,
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'paused', 'error', 'revoked')),
    last_sync_at    TIMESTAMPTZ,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_integrations_org ON integrations(organization_id);

-- ============================================================
-- WEBHOOK SUBSCRIPTIONS
-- ============================================================
CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    data_room_id    UUID REFERENCES data_rooms(id),  -- NULL = org-wide
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,  -- HMAC-SHA256 signing secret
    events          TEXT[] NOT NULL,  -- e.g., ARRAY['document.uploaded', 'room.opened']
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_delivery_at TIMESTAMPTZ,
    failure_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhooks_org ON webhook_subscriptions(organization_id);
```

---

## Pros and Cons

### Pros

1. **Compliance alignment.** Normalized relational schemas map directly to the record structures expected by ISO 27001, SOC 2 Type II, SEC 17a-4, and HIPAA auditors. Every entity has a clear table, every relationship has a foreign key, and the append-only audit log with partition-based retention meets WORM requirements.

2. **Mature ecosystem.** PostgreSQL has decades of production hardening, extensive tooling (pg_dump, pg_restore, pg_partman, pgAudit), and broad cloud support (AWS RDS, Google Cloud SQL, Supabase, Neon). Hiring developers with PostgreSQL experience is straightforward.

3. **Row-Level Security for multi-tenancy.** RLS policies enforce tenant isolation at the database layer, adding a defense-in-depth layer beyond application code. If an application bug bypasses tenant filtering, the database still blocks cross-tenant access.

4. **Built-in full-text search.** PostgreSQL's `tsvector` and GIN indexes eliminate the need for a separate search engine (Elasticsearch) for basic document search, reducing infrastructure complexity.

5. **ACID transactions.** Permission changes, document uploads, and Q&A workflows all benefit from transactional consistency. An upload that fails mid-processing never leaves orphaned metadata.

6. **Straightforward querying.** Standard SQL with JOINs makes complex queries (e.g., "which documents has user X viewed in room Y for more than 30 seconds?") easy to write and optimize.

### Cons

1. **Permission resolution is expensive.** Walking the folder hierarchy with recursive CTEs to resolve inherited permissions on every document access adds latency. For rooms with deep folder trees (10+ levels) and hundreds of permission overrides, this can become a bottleneck. Mitigation: materialize effective permissions in a cache table and invalidate on permission changes.

2. **Schema rigidity.** Adding new document metadata fields (e.g., new AI classification dimensions, industry-specific attributes) requires ALTER TABLE migrations. In a SaaS environment with continuous deployment, this can cause brief lock contention on large tables.

3. **Analytics at scale.** Page-level heatmap data grows rapidly (thousands of events per document per user). The `page_view_events` table can reach billions of rows in active deployments. Mitigation: partition by date and archive to columnar storage (e.g., TimescaleDB, ClickHouse) for analytics queries.

4. **No native graph traversal.** Permission inheritance through folder hierarchies is a graph problem shoehorned into a relational model. Recursive CTEs work but are less elegant and slower than purpose-built graph databases for deep, complex permission trees.

5. **Document content outside the database.** Files live in S3 while metadata lives in PostgreSQL, creating a dual-system consistency challenge. A failed S3 upload with committed metadata (or vice versa) requires compensating transactions or a saga pattern.

6. **Audit log volume.** Append-only audit logs with WORM constraints cannot be pruned within retention windows. For high-volume rooms (thousands of users, millions of page views), the audit_log table grows to terabytes. Partitioning and archival to cold storage (S3 + Parquet) are essential.

---

## Migration and Scaling Considerations

### Horizontal Scaling Path

1. **Phase 1 (0-100 data rooms):** Single PostgreSQL instance with connection pooling. RLS handles multi-tenancy. All tables on one server.

2. **Phase 2 (100-1,000 data rooms):** Read replicas for analytics queries. Materialized views refreshed asynchronously. Audit log partitioned monthly.

3. **Phase 3 (1,000-10,000 data rooms):** Citus or native PostgreSQL 16 declarative partitioning to shard tenant-heavy tables by `organization_id`. Analytics moved to a dedicated ClickHouse or TimescaleDB instance.

4. **Phase 4 (10,000+ data rooms):** Consider per-organization schema isolation (schema-per-tenant) for the largest enterprise customers. Separate hot (active rooms) and cold (archived rooms) storage tiers.

### Migration Strategy

- Use Flyway or golang-migrate with numbered, version-controlled migration files.
- All migrations are forward-only (no rollback scripts in production -- instead, write a new forward migration to undo changes).
- Blue-green deployment: apply migrations to the green database, verify, then switch traffic.
- For zero-downtime schema changes on large tables, use `pg_repack` or `CREATE INDEX CONCURRENTLY`.

### Backup and Disaster Recovery

- Continuous WAL archiving to S3 with Point-in-Time Recovery (PITR).
- Cross-region replication for data residency compliance (EU customers get EU replicas).
- Daily logical backups (`pg_dump`) retained for 30 days.
- Audit log partitions archived to immutable S3 buckets with Object Lock (WORM) for SEC 17a-4 compliance.

### Performance Optimization

- Permission cache: materialize effective permissions per user per room in Redis; invalidate on permission changes via LISTEN/NOTIFY.
- Connection pooling: PgBouncer in transaction mode with 100-200 connections per pool.
- Prepared statements for hot paths (document access checks, audit log inserts).
- Partial indexes: `WHERE status = 'active'` on documents and rooms to skip archived/deleted rows.
- Covering indexes for the most common access patterns (e.g., listing documents in a folder for a specific user).
