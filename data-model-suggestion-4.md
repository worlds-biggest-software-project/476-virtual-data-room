# Data Model Suggestion 4: Polyglot Persistence with Graph-Centric Access Control

> Project: Virtual Data Room (476) -- Generated: 2026-05-26

## Overview

This model uses a polyglot persistence strategy, assigning each data category to the storage engine that best fits its access patterns, rather than forcing all data through a single database. The centerpiece is a **graph database (Neo4j)** for the access control and permission system -- the most complex and performance-critical subsystem in a Virtual Data Room. Surrounding the graph are specialized stores for each other domain: a relational database for transactional entities, a time-series database for engagement analytics, a vector database for AI-powered document similarity and search, and an append-only log store for the compliance audit trail.

The key insight is that a VDR's permission model is inherently a graph problem. Users belong to groups. Groups have roles. Roles grant permissions. Permissions are overridden at the folder level. Folders inherit from parent folders. Documents inherit from their folder. Exceptions exist for specific users on specific documents. Resolving "can user X view document Y?" requires traversing this multi-layered graph, and a graph database executes this traversal in constant time relative to the total dataset size, whereas a relational database requires recursive CTEs that scale with hierarchy depth and permission override count.

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Access control & relationships | Neo4j 5.x (Aura or self-hosted) | Native graph traversal for permission resolution, relationship-first data model |
| Transactional entities | PostgreSQL 16+ | ACID transactions for rooms, documents, users, Q&A workflow |
| Engagement analytics | TimescaleDB (on PostgreSQL) or ClickHouse | Time-series optimized storage for page views, session data, heatmaps |
| Document search & similarity | Weaviate or Pgvector (on PostgreSQL) | Vector embeddings for semantic search, document similarity, AI classification |
| Audit trail | Apache Kafka + S3 (Parquet/Iceberg) | Append-only event stream with immutable archival to object storage |
| Full-text search | OpenSearch / Elasticsearch | Faceted search across document content, metadata, Q&A threads |
| File storage | S3-compatible (AWS S3, MinIO) | Encrypted document blobs |
| Cache | Redis 7+ | Permission resolution cache, session store, real-time notifications |
| Encryption / KMS | AWS KMS / HashiCorp Vault | Per-tenant encryption key management |
| Message broker | Apache Kafka | Cross-service event distribution, audit log ingestion |
| Orchestration | Temporal or Apache Airflow | Document processing pipeline orchestration |

## Architecture Overview

```
                              +------------------+
                              |   API Gateway    |
                              +--------+---------+
                                       |
              +------------------------+------------------------+
              |                        |                        |
    +---------v---------+   +----------v----------+   +---------v---------+
    |  Permission       |   |  Core Business      |   |  Analytics        |
    |  Service          |   |  Service             |   |  Service          |
    +---------+---------+   +----------+----------+   +---------+---------+
              |                        |                        |
    +---------v---------+   +----------v----------+   +---------v---------+
    |  Neo4j            |   |  PostgreSQL         |   |  TimescaleDB /    |
    |  (Graph)          |   |  (Relational)       |   |  ClickHouse       |
    +-------------------+   +---------------------+   +-------------------+
                                       |
              +------------------------+------------------------+
              |                        |                        |
    +---------v---------+   +----------v----------+   +---------v---------+
    |  Weaviate /       |   |  OpenSearch          |   |  Kafka --> S3     |
    |  Pgvector         |   |  (Full-text search)  |   |  (Audit trail)   |
    |  (Vector search)  |   +---------------------+   +-------------------+
    +-------------------+
```

---

## Neo4j Graph Schema: Access Control and Relationships

### Node Types

```cypher
// ============================================================
// NODE DEFINITIONS
// ============================================================

// Organizations (tenants)
CREATE CONSTRAINT org_id IF NOT EXISTS FOR (o:Organization) REQUIRE o.id IS UNIQUE;

// Users
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT user_email IF NOT EXISTS FOR (u:User) REQUIRE u.email IS UNIQUE;

// Data Rooms
CREATE CONSTRAINT room_id IF NOT EXISTS FOR (r:DataRoom) REQUIRE r.id IS UNIQUE;

// User Groups (within a data room)
CREATE CONSTRAINT group_id IF NOT EXISTS FOR (g:UserGroup) REQUIRE g.id IS UNIQUE;

// Roles (permission templates)
CREATE CONSTRAINT role_id IF NOT EXISTS FOR (rl:Role) REQUIRE rl.id IS UNIQUE;

// Folders (hierarchical)
CREATE CONSTRAINT folder_id IF NOT EXISTS FOR (f:Folder) REQUIRE f.id IS UNIQUE;

// Documents
CREATE CONSTRAINT doc_id IF NOT EXISTS FOR (d:Document) REQUIRE d.id IS UNIQUE;

// Permission nodes (explicit overrides)
CREATE CONSTRAINT perm_id IF NOT EXISTS FOR (p:Permission) REQUIRE p.id IS UNIQUE;
```

### Relationship Types

```cypher
// ============================================================
// RELATIONSHIP DEFINITIONS
// ============================================================

// Organization structure
// (User)-[:MEMBER_OF {role: 'admin', since: datetime()}]->(Organization)
// (DataRoom)-[:BELONGS_TO]->(Organization)

// Data room membership
// (User)-[:PARTICIPATES_IN {
//     roleId: 'uuid',
//     groupId: 'uuid',
//     status: 'active',
//     ndaAccepted: true,
//     ndaAcceptedAt: datetime(),
//     accessStartsAt: datetime(),
//     accessExpiresAt: datetime(),
//     invitedBy: 'uuid'
// }]->(DataRoom)

// Group and role assignments
// (User)-[:IN_GROUP]->(UserGroup)
// (UserGroup)-[:IN_ROOM]->(DataRoom)
// (UserGroup)-[:HAS_ROLE]->(Role)
// (Role)-[:DEFINED_IN]->(DataRoom)

// Folder hierarchy
// (Folder)-[:CHILD_OF]->(Folder)
// (Folder)-[:IN_ROOM]->(DataRoom)
// (Document)-[:IN_FOLDER]->(Folder)
// (Document)-[:IN_ROOM]->(DataRoom)
// (Document)-[:VERSION_OF]->(Document)   // document versioning chain

// Permission grants (the core of the graph model)
// (User)-[:GRANTED {canView: true, canDownload: false, canPrint: false, canUpload: false, inherit: true}]->(Folder)
// (User)-[:GRANTED {canView: true, canDownload: true}]->(Document)
// (UserGroup)-[:GRANTED {canView: true, canDownload: false, inherit: true}]->(Folder)
// (Role)-[:GRANTS {canView: true, canDownload: false, canPrint: false, canUpload: false}]->(DataRoom)

// Permission denials (explicit deny overrides any grant)
// (User)-[:DENIED {canView: true, canDownload: true}]->(Folder)
// (User)-[:DENIED {canView: true}]->(Document)

// Document relationships (AI-discovered or manual)
// (Document)-[:RELATED_TO {type: 'references', confidence: 0.85}]->(Document)
// (Document)-[:SUPERSEDES]->(Document)
// (Document)-[:ATTACHMENT_OF]->(QAQuestion)  // stored as node in graph for traversal
```

### Creating the Graph

```cypher
// ============================================================
// EXAMPLE: Setting up a data room with users, groups, and permissions
// ============================================================

// Create organization and room
CREATE (org:Organization {id: $orgId, name: 'Acme Advisors', slug: 'acme-advisors', planTier: 'enterprise'})
CREATE (room:DataRoom {id: $roomId, name: 'Project Falcon M&A', dealType: 'mna', status: 'active'})
CREATE (room)-[:BELONGS_TO]->(org)

// Create roles
CREATE (organizer:Role {id: $organizerRoleId, name: 'Organizer',
    canView: true, canDownload: true, canPrint: true, canUpload: true,
    canDelete: true, canManageUsers: true, canManageQa: true,
    canViewAnalytics: true, canManageRedactions: true, isSystem: true})
CREATE (organizer)-[:DEFINED_IN]->(room)

CREATE (bidder:Role {id: $bidderRoleId, name: 'Bidder',
    canView: true, canDownload: false, canPrint: false, canUpload: false,
    canDelete: false, canManageUsers: false, canManageQa: false,
    canViewAnalytics: false, canManageRedactions: false, isSystem: true})
CREATE (bidder)-[:DEFINED_IN]->(room)

CREATE (counsel:Role {id: $counselRoleId, name: 'Legal Counsel',
    canView: true, canDownload: true, canPrint: true, canUpload: false,
    canDelete: false, canManageUsers: false, canManageQa: true,
    canViewAnalytics: false, canManageRedactions: true, isSystem: true})
CREATE (counsel)-[:DEFINED_IN]->(room)

// Create groups
CREATE (sellSide:UserGroup {id: $sellSideGroupId, name: 'Sell-Side Team'})
CREATE (sellSide)-[:IN_ROOM]->(room)
CREATE (sellSide)-[:HAS_ROLE]->(organizer)

CREATE (buyerA:UserGroup {id: $buyerAGroupId, name: 'Buyer A - Falcon Capital'})
CREATE (buyerA)-[:IN_ROOM]->(room)
CREATE (buyerA)-[:HAS_ROLE]->(bidder)

// Create folder hierarchy
CREATE (root:Folder {id: $rootId, name: 'Root', indexNumber: '0', depth: 0})
CREATE (root)-[:IN_ROOM]->(room)

CREATE (financial:Folder {id: $finId, name: 'Financial', indexNumber: '1', depth: 1})
CREATE (financial)-[:CHILD_OF]->(root)
CREATE (financial)-[:IN_ROOM]->(room)

CREATE (legal:Folder {id: $legalId, name: 'Legal', indexNumber: '2', depth: 1})
CREATE (legal)-[:CHILD_OF]->(root)
CREATE (legal)-[:IN_ROOM]->(room)

CREATE (tax:Folder {id: $taxId, name: 'Tax', indexNumber: '1.1', depth: 2})
CREATE (tax)-[:CHILD_OF]->(financial)
CREATE (tax)-[:IN_ROOM]->(room)

CREATE (contracts:Folder {id: $contractsId, name: 'Material Contracts', indexNumber: '2.1', depth: 2})
CREATE (contracts)-[:CHILD_OF]->(legal)
CREATE (contracts)-[:IN_ROOM]->(room)

// Add users to groups and room
CREATE (alice:User {id: $aliceId, email: 'alice@acme.com', displayName: 'Alice Chen'})
CREATE (alice)-[:MEMBER_OF {role: 'admin'}]->(org)
CREATE (alice)-[:IN_GROUP]->(sellSide)
CREATE (alice)-[:PARTICIPATES_IN {roleId: $organizerRoleId, groupId: $sellSideGroupId, status: 'active'}]->(room)

CREATE (bob:User {id: $bobId, email: 'bob@falcon.com', displayName: 'Bob Falcon'})
CREATE (bob)-[:IN_GROUP]->(buyerA)
CREATE (bob)-[:PARTICIPATES_IN {roleId: $bidderRoleId, groupId: $buyerAGroupId, status: 'active', ndaAccepted: true}]->(room)

// Grant group-level folder permissions
CREATE (buyerA)-[:GRANTED {canView: true, canDownload: false, canPrint: false, inherit: true}]->(financial)
CREATE (buyerA)-[:GRANTED {canView: true, canDownload: false, canPrint: false, inherit: true}]->(legal)

// Deny access to sensitive subfolder for Buyer A
CREATE (buyerA)-[:DENIED {canView: true}]->(tax)

// Grant specific user an exception: Bob can download contracts
CREATE (bob)-[:GRANTED {canView: true, canDownload: true, canPrint: false, inherit: false}]->(contracts)
```

### Permission Resolution Queries

```cypher
// ============================================================
// QUERY: Can user X view document Y?
// ============================================================
// This is the critical hot-path query. The graph traversal resolves
// permissions in O(path_length) time, independent of total dataset size.

// Step 1: Find the user's role and group in the document's room
MATCH (user:User {id: $userId})-[participation:PARTICIPATES_IN]->(room:DataRoom)
MATCH (doc:Document {id: $documentId})-[:IN_ROOM]->(room)
WHERE participation.status = 'active'

// Step 2: Check for direct document-level DENY (highest priority)
OPTIONAL MATCH (user)-[deny:DENIED]->(doc)

// Step 3: Check for direct document-level GRANT
OPTIONAL MATCH (user)-[directGrant:GRANTED]->(doc)

// Step 4: Walk up the folder hierarchy checking for user-level grants/denies
MATCH (doc)-[:IN_FOLDER]->(folder:Folder)
OPTIONAL MATCH path = (folder)-[:CHILD_OF*0..20]->(ancestor:Folder)
WITH user, doc, room, participation, deny, directGrant, folder,
     collect(ancestor) AS ancestors

// Step 5: Check group-level permissions on folder hierarchy
OPTIONAL MATCH (user)-[:IN_GROUP]->(group:UserGroup)-[:IN_ROOM]->(room)
OPTIONAL MATCH (group)-[groupGrant:GRANTED]->(grantedFolder:Folder)
WHERE grantedFolder IN ancestors OR grantedFolder = folder

// Step 6: Get role defaults
OPTIONAL MATCH (role:Role {id: participation.roleId})-[:DEFINED_IN]->(room)

// Step 7: Resolve (deny > direct user > group > role)
RETURN
    CASE
        WHEN deny IS NOT NULL AND deny.canView = true THEN false
        WHEN directGrant IS NOT NULL THEN directGrant.canView
        WHEN groupGrant IS NOT NULL THEN groupGrant.canView
        ELSE role.canView
    END AS canView,
    CASE
        WHEN deny IS NOT NULL AND deny.canDownload = true THEN false
        WHEN directGrant IS NOT NULL THEN COALESCE(directGrant.canDownload, false)
        WHEN groupGrant IS NOT NULL THEN COALESCE(groupGrant.canDownload, false)
        ELSE role.canDownload
    END AS canDownload,
    CASE
        WHEN deny IS NOT NULL AND deny.canPrint = true THEN false
        WHEN directGrant IS NOT NULL THEN COALESCE(directGrant.canPrint, false)
        WHEN groupGrant IS NOT NULL THEN COALESCE(groupGrant.canPrint, false)
        ELSE role.canPrint
    END AS canPrint
```

```cypher
// ============================================================
// QUERY: List all documents user X can view in room Y
// ============================================================
// This is the document listing query, run when a user opens a room.

MATCH (user:User {id: $userId})-[p:PARTICIPATES_IN]->(room:DataRoom {id: $roomId})
WHERE p.status = 'active'

// Get user's role and group
MATCH (role:Role {id: p.roleId})-[:DEFINED_IN]->(room)
OPTIONAL MATCH (user)-[:IN_GROUP]->(group:UserGroup)-[:IN_ROOM]->(room)

// Find all documents in the room
MATCH (doc:Document)-[:IN_ROOM]->(room)
MATCH (doc)-[:IN_FOLDER]->(folder:Folder)

// Check for DENY on document or any ancestor folder
OPTIONAL MATCH (user)-[denyDoc:DENIED]->(doc)
OPTIONAL MATCH deniedPath = (folder)-[:CHILD_OF*0..20]->(deniedAncestor:Folder)
WHERE EXISTS { (user)-[:DENIED]->(deniedAncestor) }
   OR EXISTS { (group)-[:DENIED]->(deniedAncestor) }

// Check for GRANT on folder hierarchy
OPTIONAL MATCH grantPath = (folder)-[:CHILD_OF*0..20]->(grantedAncestor:Folder)
WHERE EXISTS { (user)-[:GRANTED]->(grantedAncestor) }
   OR EXISTS { (group)-[:GRANTED]->(grantedAncestor) }

// Resolve visibility
WITH doc, folder, role, denyDoc, deniedAncestor, grantedAncestor
WHERE denyDoc IS NULL AND deniedAncestor IS NULL  // no deny found
  AND (grantedAncestor IS NOT NULL OR role.canView = true)  // grant found or role allows

RETURN doc.id, doc.displayName, folder.id AS folderId, folder.name AS folderName
ORDER BY folder.indexNumber, doc.sortOrder
```

```cypher
// ============================================================
// QUERY: Who has access to a specific document? (Compliance query)
// ============================================================
MATCH (doc:Document {id: $documentId})-[:IN_ROOM]->(room:DataRoom)
MATCH (user:User)-[p:PARTICIPATES_IN]->(room)
WHERE p.status = 'active'

// Get each user's effective permission
MATCH (role:Role {id: p.roleId})
OPTIONAL MATCH (user)-[:IN_GROUP]->(group:UserGroup)-[:IN_ROOM]->(room)
OPTIONAL MATCH (user)-[deny:DENIED]->(doc)

MATCH (doc)-[:IN_FOLDER]->(folder:Folder)
OPTIONAL MATCH (folder)-[:CHILD_OF*0..20]->(ancestor:Folder)
OPTIONAL MATCH (user)-[userGrant:GRANTED]->(grantedFolder:Folder)
WHERE grantedFolder = folder OR grantedFolder = ancestor
OPTIONAL MATCH (group)-[groupGrant:GRANTED]->(groupGrantedFolder:Folder)
WHERE groupGrantedFolder = folder OR groupGrantedFolder = ancestor

WITH user, role, deny, userGrant, groupGrant,
     CASE
         WHEN deny IS NOT NULL THEN false
         WHEN userGrant IS NOT NULL THEN userGrant.canView
         WHEN groupGrant IS NOT NULL THEN groupGrant.canView
         ELSE role.canView
     END AS effectiveCanView

WHERE effectiveCanView = true

RETURN user.id, user.email, user.displayName, effectiveCanView
ORDER BY user.displayName
```

```cypher
// ============================================================
// QUERY: Document relationship graph (AI-discovered connections)
// ============================================================
// Find documents related to a given document within 2 hops

MATCH (source:Document {id: $documentId})
MATCH path = (source)-[:RELATED_TO*1..2]-(related:Document)
WHERE related.id <> source.id
RETURN DISTINCT related.id, related.displayName,
       [r IN relationships(path) | r.type] AS relationshipTypes,
       [r IN relationships(path) | r.confidence] AS confidences,
       length(path) AS distance
ORDER BY distance, related.displayName
```

---

## PostgreSQL Schema: Transactional Entities

The relational database stores the mutable state of all entities that require ACID transactions. The graph database is the authority for permissions and relationships; PostgreSQL is the authority for entity data.

```sql
-- ============================================================
-- ORGANIZATIONS
-- ============================================================
CREATE TABLE organizations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    plan_tier           VARCHAR(50) NOT NULL DEFAULT 'free',
    billing_email       VARCHAR(255),
    logo_url            TEXT,
    settings            JSONB NOT NULL DEFAULT '{}',
    data_residency_region VARCHAR(20) NOT NULL DEFAULT 'us-east-1',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- USERS
-- ============================================================
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(255) NOT NULL UNIQUE,
    display_name        VARCHAR(255) NOT NULL,
    password_hash       VARCHAR(255),
    auth_provider       VARCHAR(50) NOT NULL DEFAULT 'email',
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    last_login_at       TIMESTAMPTZ,
    auth_metadata       JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- DATA ROOMS
-- ============================================================
CREATE TABLE data_rooms (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    deal_type           VARCHAR(50) NOT NULL DEFAULT 'mna',
    deal_value          NUMERIC(18, 2),
    deal_currency       VARCHAR(3) DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    room_settings       JSONB NOT NULL DEFAULT '{}',
    deal_metadata       JSONB NOT NULL DEFAULT '{}',
    opened_at           TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    archived_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_rooms_org ON data_rooms(organization_id);
CREATE INDEX idx_rooms_status ON data_rooms(organization_id, status);

-- ============================================================
-- FOLDERS
-- ============================================================
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
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_folders_room ON folders(data_room_id);
CREATE INDEX idx_folders_parent ON folders(parent_id);

-- ============================================================
-- DOCUMENTS
-- ============================================================
CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    folder_id           UUID NOT NULL REFERENCES folders(id) ON DELETE CASCADE,
    original_filename   VARCHAR(500) NOT NULL,
    display_name        VARCHAR(500) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    page_count          INTEGER,
    storage_bucket      VARCHAR(255) NOT NULL,
    storage_key         VARCHAR(1000) NOT NULL,
    encryption_key_id   VARCHAR(255) NOT NULL,
    checksum_sha256     VARCHAR(64) NOT NULL,
    version             INTEGER NOT NULL DEFAULT 1,
    is_latest           BOOLEAN NOT NULL DEFAULT TRUE,
    parent_document_id  UUID REFERENCES documents(id),
    index_number        VARCHAR(50),
    sort_order          INTEGER NOT NULL DEFAULT 0,
    processing_status   VARCHAR(30) NOT NULL DEFAULT 'pending',
    processing_error    TEXT,
    pdf_storage_key     VARCHAR(1000),
    pdf_page_count      INTEGER,
    content_text        TEXT,
    search_vector       TSVECTOR,
    -- AI results stored as JSONB (see Suggestion 3 rationale)
    ai_results          JSONB NOT NULL DEFAULT '{}',
    redaction_results   JSONB NOT NULL DEFAULT '{}',
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uploaded_by         UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_docs_room ON documents(data_room_id);
CREATE INDEX idx_docs_folder ON documents(folder_id);
CREATE INDEX idx_docs_search ON documents USING gin(search_vector);
CREATE INDEX idx_docs_status ON documents(data_room_id, status) WHERE status = 'active';

-- ============================================================
-- Q&A WORKFLOW
-- ============================================================
CREATE TABLE qa_topics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    name                VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

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
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted',
    priority            VARCHAR(10) NOT NULL DEFAULT 'normal',
    assigned_to         UUID REFERENCES users(id),
    assigned_group_id   UUID,  -- references Neo4j group
    sla_deadline        TIMESTAMPTZ,
    sla_breached        BOOLEAN NOT NULL DEFAULT FALSE,
    visibility          VARCHAR(20) NOT NULL DEFAULT 'submitter_and_assignee',
    submitted_at        TIMESTAMPTZ,
    answered_at         TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    submitted_by        UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_qa_room ON qa_questions(data_room_id);
CREATE INDEX idx_qa_status ON qa_questions(data_room_id, status);
CREATE INDEX idx_qa_sla ON qa_questions(sla_deadline) WHERE NOT sla_breached;

CREATE TABLE qa_answers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id         UUID NOT NULL REFERENCES qa_questions(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    body                TEXT NOT NULL,
    attachment_document_ids UUID[],
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    answered_by         UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_qa_answers_question ON qa_answers(question_id);

-- ============================================================
-- MESSAGES (encrypted in-room communication)
-- ============================================================
CREATE TABLE messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_room_id        UUID NOT NULL REFERENCES data_rooms(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    thread_id           UUID REFERENCES messages(id),
    body_encrypted      BYTEA NOT NULL,
    body_nonce          BYTEA NOT NULL,
    attachment_document_ids UUID[],
    visibility          VARCHAR(20) NOT NULL DEFAULT 'all',
    target_group_id     UUID,  -- references Neo4j group
    target_user_id      UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_by             UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_messages_room ON messages(data_room_id, created_at);
CREATE INDEX idx_messages_thread ON messages(thread_id);

-- ============================================================
-- INTEGRATIONS
-- ============================================================
CREATE TABLE integrations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    integration_type    VARCHAR(50) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    config              JSONB NOT NULL DEFAULT '{}',
    credentials_encrypted BYTEA NOT NULL,
    credentials_nonce   BYTEA NOT NULL,
    sync_state          JSONB NOT NULL DEFAULT '{}',
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_integrations_org ON integrations(organization_id);
```

---

## TimescaleDB Schema: Engagement Analytics

```sql
-- ============================================================
-- TIMESCALEDB: Page-level engagement tracking
-- ============================================================
-- TimescaleDB hypertables automatically partition by time and
-- provide efficient time-range queries, continuous aggregates,
-- and data retention policies.

CREATE TABLE page_view_events (
    time                TIMESTAMPTZ NOT NULL,
    organization_id     UUID NOT NULL,
    data_room_id        UUID NOT NULL,
    document_id         UUID NOT NULL,
    user_id             UUID NOT NULL,
    session_id          UUID NOT NULL,
    page_number         SMALLINT NOT NULL,
    view_duration_ms    INTEGER NOT NULL,
    scroll_depth_pct    NUMERIC(5, 2),
    device_type         VARCHAR(20),
    ip_country          VARCHAR(2),
    watermark_id        VARCHAR(100) NOT NULL
);

-- Convert to hypertable (partitioned by time, 1 day chunks)
SELECT create_hypertable('page_view_events', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Add space partitioning by organization for multi-tenant performance
SELECT add_dimension('page_view_events', 'organization_id', 4);

-- Indexes
CREATE INDEX idx_pve_room_doc ON page_view_events(data_room_id, document_id, time DESC);
CREATE INDEX idx_pve_user ON page_view_events(user_id, time DESC);
CREATE INDEX idx_pve_session ON page_view_events(session_id, time DESC);

-- ============================================================
-- CONTINUOUS AGGREGATE: Document engagement summary
-- ============================================================
CREATE MATERIALIZED VIEW cagg_document_engagement
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    organization_id,
    data_room_id,
    document_id,
    COUNT(DISTINCT user_id) AS unique_viewers,
    COUNT(DISTINCT session_id) AS total_sessions,
    COUNT(*) AS total_page_views,
    SUM(view_duration_ms) AS total_view_ms,
    AVG(view_duration_ms) AS avg_view_ms
FROM page_view_events
GROUP BY day, organization_id, data_room_id, document_id
WITH NO DATA;

-- Refresh policy: aggregate data older than 1 hour
SELECT add_continuous_aggregate_policy('cagg_document_engagement',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- CONTINUOUS AGGREGATE: User engagement scoring
-- ============================================================
CREATE MATERIALIZED VIEW cagg_user_engagement
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    organization_id,
    data_room_id,
    user_id,
    COUNT(DISTINCT document_id) AS documents_viewed,
    COUNT(DISTINCT session_id) AS sessions,
    SUM(view_duration_ms) AS total_view_ms,
    COUNT(*) AS total_page_views,
    MAX(time) AS last_activity_at
FROM page_view_events
GROUP BY day, organization_id, data_room_id, user_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_user_engagement',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- CONTINUOUS AGGREGATE: Page-level heatmap
-- ============================================================
CREATE MATERIALIZED VIEW cagg_page_heatmap
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    organization_id,
    data_room_id,
    document_id,
    page_number,
    COUNT(DISTINCT user_id) AS unique_viewers,
    SUM(view_duration_ms) AS total_view_ms,
    AVG(view_duration_ms) AS avg_view_ms,
    AVG(scroll_depth_pct) AS avg_scroll_depth
FROM page_view_events
GROUP BY day, organization_id, data_room_id, document_id, page_number
WITH NO DATA;

SELECT add_continuous_aggregate_policy('cagg_page_heatmap',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- ============================================================
-- DATA RETENTION: Automatically drop raw data older than 1 year
-- ============================================================
SELECT add_retention_policy('page_view_events', INTERVAL '1 year');
-- Continuous aggregates retain summarized data indefinitely

-- ============================================================
-- DOCUMENT VIEW SESSIONS (session-level rollup in PostgreSQL)
-- ============================================================
-- Session start/end events stored in PostgreSQL for relational queries
CREATE TABLE document_view_sessions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID NOT NULL,
    data_room_id        UUID NOT NULL,
    organization_id     UUID NOT NULL,
    user_id             UUID NOT NULL,
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at            TIMESTAMPTZ,
    total_duration_seconds INTEGER,
    total_pages_viewed  INTEGER,
    ip_address          INET,
    user_agent          TEXT,
    device_type         VARCHAR(20),
    watermark_id        VARCHAR(100) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_doc ON document_view_sessions(document_id);
CREATE INDEX idx_sessions_user ON document_view_sessions(user_id);
CREATE INDEX idx_sessions_room ON document_view_sessions(data_room_id, started_at);
```

---

## Vector Database Schema: AI-Powered Document Search

```sql
-- ============================================================
-- VECTOR EMBEDDINGS (using Pgvector extension on PostgreSQL)
-- ============================================================
-- Alternative: Weaviate as a dedicated vector database for larger scale.
-- Pgvector is chosen here for operational simplicity (same PostgreSQL instance).

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_embeddings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    organization_id     UUID NOT NULL,
    data_room_id        UUID NOT NULL,
    -- Chunk-level embeddings (documents are split into chunks for better retrieval)
    chunk_index         INTEGER NOT NULL,
    chunk_text          TEXT NOT NULL,
    chunk_start_page    INTEGER,
    chunk_end_page      INTEGER,
    -- Vector embedding (1536 dimensions for OpenAI ada-002, 1024 for Cohere, etc.)
    embedding           vector(1536) NOT NULL,
    -- Metadata
    model_version       VARCHAR(50) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (document_id, chunk_index)
);

-- HNSW index for approximate nearest neighbor search
CREATE INDEX idx_embeddings_hnsw ON document_embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 200);

-- Filter index for scoping searches to a specific room
CREATE INDEX idx_embeddings_room ON document_embeddings(data_room_id);
CREATE INDEX idx_embeddings_org ON document_embeddings(organization_id);

-- ============================================================
-- SEMANTIC SEARCH QUERY
-- ============================================================
-- Find documents semantically similar to a query
-- (query embedding computed at application layer)

-- Search within a specific data room
SELECT
    de.document_id,
    d.display_name,
    de.chunk_text,
    de.chunk_start_page,
    de.chunk_end_page,
    1 - (de.embedding <=> $queryEmbedding::vector) AS similarity
FROM document_embeddings de
JOIN documents d ON d.id = de.document_id
WHERE de.data_room_id = $roomId
  AND d.status = 'active'
ORDER BY de.embedding <=> $queryEmbedding::vector
LIMIT 20;

-- ============================================================
-- DOCUMENT SIMILARITY (for "related documents" feature)
-- ============================================================
-- Find documents similar to a specific document
SELECT
    target_de.document_id,
    d.display_name,
    AVG(1 - (source_de.embedding <=> target_de.embedding)) AS avg_similarity
FROM document_embeddings source_de
JOIN document_embeddings target_de ON target_de.data_room_id = source_de.data_room_id
    AND target_de.document_id != source_de.document_id
JOIN documents d ON d.id = target_de.document_id AND d.status = 'active'
WHERE source_de.document_id = $documentId
GROUP BY target_de.document_id, d.display_name
HAVING AVG(1 - (source_de.embedding <=> target_de.embedding)) > 0.75
ORDER BY avg_similarity DESC
LIMIT 10;

-- ============================================================
-- AI CLASSIFICATION SUPPORT
-- ============================================================
-- Store classification reference embeddings for zero-shot or
-- few-shot classification of new documents

CREATE TABLE classification_reference_embeddings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    classification      VARCHAR(100) NOT NULL,
    label_description   TEXT NOT NULL,
    embedding           vector(1536) NOT NULL,
    example_document_ids UUID[] NOT NULL DEFAULT '{}',
    model_version       VARCHAR(50) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_class_ref_org ON classification_reference_embeddings(organization_id);
CREATE INDEX idx_class_ref_hnsw ON classification_reference_embeddings
    USING hnsw (embedding vector_cosine_ops);
```

---

## Audit Trail: Kafka + S3 Immutable Archive

```
                                    +------------------+
                                    |  Application     |
                                    |  Services        |
                                    +--------+---------+
                                             |
                                     Produce audit events
                                             |
                                    +--------v---------+
                                    |  Kafka Topic:    |
                                    |  audit.events    |
                                    |  (partitioned    |
                                    |   by org_id)     |
                                    +----+--------+----+
                                         |        |
                            +------------+        +-------------+
                            |                                   |
                   +--------v---------+               +---------v--------+
                   |  Kafka Connect   |               |  Audit Query     |
                   |  S3 Sink         |               |  Service         |
                   +--------+---------+               +---------+--------+
                            |                                   |
                   +--------v---------+               +---------v--------+
                   |  S3 Bucket       |               |  PostgreSQL      |
                   |  (Parquet files, |               |  (Recent audit   |
                   |   Object Lock,   |               |   events, 90     |
                   |   immutable)     |               |   day window)    |
                   +------------------+               +------------------+
```

```json
// Kafka audit event schema (Avro/JSON)
{
    "eventId": "uuid",
    "timestamp": "2026-05-26T10:30:00.000Z",
    "organizationId": "uuid",
    "dataRoomId": "uuid",
    "actor": {
        "userId": "uuid",
        "email": "alice@acme.com",
        "displayName": "Alice Chen",
        "ipAddress": "203.0.113.42",
        "userAgent": "Mozilla/5.0...",
        "deviceFingerprint": "abc123"
    },
    "action": {
        "category": "document",
        "type": "document.viewed",
        "description": "Alice Chen viewed 'Q3 Financial Statements.pdf' (pages 1-15)"
    },
    "resource": {
        "type": "document",
        "id": "uuid",
        "name": "Q3 Financial Statements.pdf",
        "parentFolderId": "uuid",
        "parentFolderName": "Financials"
    },
    "details": {
        "pagesViewed": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15],
        "viewDurationSeconds": 342,
        "watermarkId": "wm-session-abc",
        "deviceType": "desktop"
    },
    "correlationId": "uuid",
    "schemaVersion": 2
}
```

```sql
-- PostgreSQL table for recent audit events (queryable, 90-day window)
CREATE TABLE audit_log_recent (
    id                  BIGSERIAL,
    organization_id     UUID NOT NULL,
    data_room_id        UUID,
    user_id             UUID NOT NULL,
    user_email          VARCHAR(255) NOT NULL,
    user_ip             INET,
    action_category     VARCHAR(50) NOT NULL,
    action              VARCHAR(50) NOT NULL,
    resource_type       VARCHAR(50) NOT NULL,
    resource_id         UUID,
    resource_name       VARCHAR(500),
    details             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create 3 monthly partitions (rolling window)
-- Older partitions are dropped automatically; data persists in S3

CREATE INDEX idx_audit_org_room ON audit_log_recent(organization_id, data_room_id, created_at);
CREATE INDEX idx_audit_user ON audit_log_recent(user_id, created_at);
CREATE INDEX idx_audit_action ON audit_log_recent(action, created_at);

-- WORM compliance
CREATE RULE audit_no_update AS ON UPDATE TO audit_log_recent DO INSTEAD NOTHING;
CREATE RULE audit_no_delete AS ON DELETE TO audit_log_recent DO INSTEAD NOTHING;
```

For compliance queries spanning more than 90 days, the audit query service reads directly from S3 Parquet files using Athena (AWS) or DuckDB, providing SQL query capability over immutable archived data.

---

## Cross-Service Synchronization

### Neo4j <-> PostgreSQL Consistency

When a new document is created in PostgreSQL, a corresponding `Document` node and `IN_FOLDER` / `IN_ROOM` relationships must be created in Neo4j. This is handled via a synchronization service:

```
PostgreSQL LISTEN/NOTIFY  --->  Sync Service  --->  Neo4j
        (or Kafka)                                 (Create nodes/edges)
```

The sync service is idempotent: if it crashes and restarts, it re-reads the WAL or Kafka offset and replays missed operations. The PostgreSQL record is the source of truth for entity data; the Neo4j graph is the source of truth for relationships and permissions.

### Conflict Resolution

- **Permission changes:** Applied to Neo4j first (source of truth), then cached in Redis. PostgreSQL is not involved in permission storage.
- **Document CRUD:** Applied to PostgreSQL first (source of truth), then synced to Neo4j for relationship management and to OpenSearch for full-text indexing.
- **Analytics events:** Written to TimescaleDB and Kafka simultaneously. Kafka feeds the S3 audit archive.

---

## Pros and Cons

### Pros

1. **Sub-millisecond permission resolution.** Graph traversal for "can user X access document Y?" executes in constant time relative to the total number of documents and users. Where a relational recursive CTE might take 10-50ms for a deep folder hierarchy with many overrides, Neo4j traverses the same path in under 1ms. For a VDR where every document access requires a permission check, this is a significant performance advantage.

2. **Natural permission visualization.** The graph structure directly maps to how permissions are conceptualized: users belong to groups, groups have roles, roles grant access, folders inherit from parents, overrides exist at specific nodes. This makes the permission model easier to reason about, debug, and explain to auditors. Permission queries in Cypher are more readable than recursive SQL CTEs.

3. **AI-powered document discovery via vector search.** The vector embedding store enables semantic search ("find documents about revenue recognition policies") and document similarity ("show me documents related to this tax filing"). This goes far beyond keyword-based full-text search and enables the AI-native features described in the project README.

4. **Purpose-built analytics storage.** TimescaleDB's hypertables with continuous aggregates handle the page-view analytics workload efficiently. Automatic data retention drops raw events after one year while preserving aggregated summaries indefinitely. This keeps analytics queries fast without manual partition management.

5. **Compliance through immutable archival.** The Kafka-to-S3 audit pipeline creates an immutable, cryptographically verifiable archive of all events. S3 Object Lock provides WORM compliance. Querying historical data via Athena/DuckDB gives SQL access without any database to manage.

6. **Rich relationship queries.** Graph queries like "show all documents user X can access that are related to document Y" or "find all users who have transitively received access through group changes in the last 30 days" are natural in Neo4j but extremely complex in SQL.

7. **Independent scaling per workload.** The permission service (Neo4j) scales independently of the document service (PostgreSQL), which scales independently of the analytics service (TimescaleDB). Each can be sized and optimized for its specific access patterns without compromise.

### Cons

1. **Operational complexity is the highest of all suggestions.** Running Neo4j, PostgreSQL, TimescaleDB (or ClickHouse), Kafka, Redis, and S3 requires a mature DevOps team. Each system has its own backup strategy, monitoring, upgrade path, and failure modes. For an early-stage startup, this is a significant burden.

2. **Cross-database consistency challenges.** When a document is created, data must be written to PostgreSQL (entity data), Neo4j (permission graph node), and OpenSearch (search index). If any write fails, the systems are inconsistent. The sync service adds complexity and introduces eventual consistency windows. A user might see a document in the folder listing (PostgreSQL) but be unable to access it because the Neo4j permission node has not yet been created.

3. **Neo4j licensing costs.** Neo4j Enterprise Edition (required for clustering, role-based access control, and performance features) is not open source. Aura (cloud) pricing scales with database size. For a startup targeting transparent, affordable pricing, this is a meaningful cost component that must be passed to customers or absorbed.

4. **Limited Neo4j transaction capabilities.** Neo4j supports ACID transactions within a single database, but cross-database transactions (PostgreSQL + Neo4j) require a saga pattern or two-phase commit, adding complexity to operations that need to atomically update both systems (e.g., deleting a user should remove their Neo4j nodes and PostgreSQL records simultaneously).

5. **Team skill requirements.** Developers must be proficient in SQL, Cypher (Neo4j's query language), TimescaleDB-specific features (hypertables, continuous aggregates), Kafka operations, and vector database concepts. This is a significantly broader skill set than a single-database approach.

6. **Testing infrastructure overhead.** Integration tests require running Neo4j, PostgreSQL, TimescaleDB, and potentially Kafka via Docker Compose or similar. Test setup is slower and more complex. CI pipelines take longer.

7. **Vendor lock-in risk.** Neo4j's Cypher query language and graph model are not directly portable to other graph databases (though GQL standardization is progressing). If Neo4j pricing becomes prohibitive, migrating the permission model to a different system is a substantial effort.

8. **Data duplication.** User, folder, and document data exists in both PostgreSQL (full entity data) and Neo4j (identity + relationship data). Changes must be synchronized. The total storage footprint is higher than a single-database approach, and stale data in the graph can cause permission errors.

---

## Migration and Scaling Considerations

### Migrating from a Single-Database Model

1. **Start with the permission subsystem.** Export the `resource_permissions`, `data_room_roles`, `user_groups`, `data_room_users`, and `folders` tables from PostgreSQL into Neo4j as nodes and relationships. Keep the PostgreSQL tables for a dual-read validation period.

2. **Build the sync service.** Implement a service that listens to PostgreSQL change data capture (Debezium or LISTEN/NOTIFY) and creates/updates/deletes corresponding Neo4j nodes and relationships in near-real-time.

3. **Validate permission resolution.** For every permission check, query both the old PostgreSQL recursive CTE and the new Neo4j traversal. Compare results. Log discrepancies. Fix sync issues.

4. **Cut over reads.** Once the error rate is zero for a sustained period, switch the permission service to read exclusively from Neo4j. Remove the PostgreSQL permission tables.

5. **Add vector search and analytics progressively.** These can be added as independent services without changing the core data model. Document embeddings are generated asynchronously from the document processing pipeline. TimescaleDB analytics start receiving events once the page-view tracking service is updated.

### Scaling Each Component

**Neo4j:**
- Phase 1: Single instance (handles millions of nodes and relationships).
- Phase 2: Neo4j read replicas for permission resolution during peak load.
- Phase 3: Neo4j Fabric for sharding across multiple databases (one per large enterprise tenant).

**PostgreSQL:**
- Same scaling path as Suggestions 1 and 3: single instance, read replicas, Citus sharding.

**TimescaleDB:**
- Phase 1: Single TimescaleDB instance (built on PostgreSQL, so same operational model).
- Phase 2: TimescaleDB multi-node for distributing hypertable chunks across workers.
- Phase 3: Archive raw data to S3/Parquet; continuous aggregates serve historical queries.

**Vector Search (Pgvector):**
- Phase 1: Pgvector on the existing PostgreSQL instance.
- Phase 2: Dedicated PostgreSQL instance for vector search with more memory for HNSW index.
- Phase 3: Migrate to Weaviate for dedicated vector database with built-in multi-tenancy and horizontal scaling.

**Kafka / Audit:**
- Phase 1: Managed Kafka (AWS MSK, Confluent Cloud) with 3 partitions for the audit topic.
- Phase 2: Increase partitions by organization volume. Add Kafka Connect S3 Sink for automated archival.
- Phase 3: Tiered storage (Kafka's built-in tiered storage or manual S3 archival).

### Disaster Recovery

- **Neo4j:** Online backups with point-in-time recovery. Causal cluster provides automatic failover.
- **PostgreSQL:** Standard WAL archiving and PITR.
- **TimescaleDB:** Same as PostgreSQL (built on PostgreSQL).
- **Kafka:** Topic replication factor 3 across availability zones. Consumer offsets survive broker failures.
- **S3:** Cross-region replication for the audit archive. Object Lock prevents deletion.
- **Full rebuild capability:** In the worst case, PostgreSQL is the source of truth for all entity data. Neo4j can be rebuilt from PostgreSQL export. TimescaleDB can be rebuilt from Kafka replay. Vector embeddings can be regenerated from document content. The system is resilient to the loss of any single specialized store.

### When to Choose This Architecture

This model is the right choice when:
- The VDR serves large enterprise deals with thousands of users, deep folder hierarchies (15+ levels), and complex permission override patterns.
- Sub-millisecond permission resolution is a hard requirement (e.g., real-time document rendering where every page turn requires a permission check).
- AI-powered document discovery (semantic search, similarity, classification) is a core differentiator, not a nice-to-have.
- The team has the operational maturity to manage a polyglot persistence stack (or is using managed cloud services for each component).
- The business model supports the infrastructure cost of multiple specialized databases.

This model is NOT the right choice when:
- The team is small (fewer than 5 engineers) and needs to move fast with minimal infrastructure overhead.
- The typical data room has fewer than 1,000 documents and fewer than 50 users (the relational model handles this fine).
- Budget constraints preclude Neo4j Enterprise licensing and managed Kafka.
- The initial product does not need AI-powered search or advanced analytics.
