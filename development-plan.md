# Virtual Data Room — Phased Development Plan

> Project: 476-virtual-data-room · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (the normalized PostgreSQL model, chosen as the MVP data layer). It targets the mid-market VDR opportunity: transparent pricing, modern UX, and AI-native document processing accessible at every tier.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The product is AI-heavy (PII NER redaction, classification, deal-readiness scoring, summarization). Python has the strongest ecosystem for document processing (PyMuPDF, Presidio, Tesseract) and LLM SDKs. |
| API framework | **FastAPI** | Async-first (critical for streaming LLM calls and large file I/O), native Pydantic validation, and auto-generates the OpenAPI 3.1 document mandated by `standards.md`. |
| Primary database | **PostgreSQL 16** | Chosen per `data-model-suggestion-1.md`: ACID, Row-Level Security for multi-tenant isolation, `tsvector`/GIN full-text search (no separate Elasticsearch), JSONB fallback, declarative partitioning for the WORM audit log (SEC 17a-4). |
| ORM / DB access | **SQLAlchemy 2.0 (async) + Alembic** | Mature async ORM; Alembic provides versioned, forward-only, auditable migrations as required for compliance change-control. |
| Connection pooling | **PgBouncer** (transaction mode) | Multi-tenant connection efficiency at scale. |
| Object storage | **S3-compatible (MinIO dev, AWS S3 prod)** | Documents stored as encrypted objects; DB holds only metadata. MinIO gives a zero-cost local dev parity with prod. Object Lock for WORM audit archival. |
| Task queue | **Celery + Redis** | Document ingestion (PDF conversion, OCR, indexing, classification, redaction) is long-running and must not block the API. Celery gives retries, chains, and a saga-style pipeline. |
| Cache / pub-sub | **Redis 7** | Session cache, materialized permission cache, Celery broker/result backend, WebSocket fan-out for analytics. |
| LLM provider | **Pluggable provider via `litellm`** (OpenAI / Anthropic / Azure OpenAI) | Avoids vendor lock-in; lets enterprise/regulated tenants route to a FIPS-validated or EU-resident endpoint. Default model: a mid-tier reasoning model for classification, a stronger model for summarization. |
| PII detection | **Microsoft Presidio + spaCy** | Open-source, extensible NER for 100+ entity types, supports custom recognizers and multilingual models — matches the "100+ data types" and multilingual goals. LLM used as a second-pass validator. |
| PDF / document processing | **PyMuPDF (fitz)** + **LibreOffice headless** + **Tesseract OCR** | PyMuPDF for rendering, watermarking, page extraction, and applying redaction boxes; LibreOffice for Office→PDF conversion; Tesseract for scanned-document OCR. |
| Vector search (v1.1) | **pgvector extension** | Keeps embeddings in Postgres for the AI Q&A / MCP layer — avoids adding a separate vector DB during MVP. |
| Frontend | **Next.js 15 (App Router) + TypeScript + Tailwind + shadcn/ui** | Server components for fast first paint; the document viewer is a client component using PDF.js. WCAG 2.2 AA is a hard requirement (`standards.md`) — shadcn/Radix primitives are accessible by default. |
| Document viewer | **PDF.js (canvas render)** with view-only mode | Server renders watermarked page images / streams; download/print disabled at the viewer level and enforced server-side. CSP Level 3 locks the viewer origin. |
| Auth | **OAuth 2.0 + OIDC (Authlib) + JWT (RFC 9068)** | `standards.md` mandates OAuth 2.0 Authorization Code + Client Credentials, JWT access tokens, and JWT client auth (RFC 7523) for service accounts. Passwordless (WebAuthn/passkeys) on the roadmap. |
| Encryption | **AES-256-GCM at rest, TLS 1.3 in transit; per-document keys via envelope encryption (KMS/Vault)** | FIPS 140-3 expectations and NIST SP 800-57 key lifecycle. Vault (dev) / AWS KMS (prod) as the KEK store. |
| Realtime | **WebSockets (FastAPI) / Server-Sent Events** | Live page-view heatmap ingest, Q&A notifications, processing status. |
| Containerisation | **Docker + docker-compose (dev), Helm chart (prod)** | Reproducible multi-service local stack; cloud-portable. |
| Testing | **pytest + pytest-asyncio + testcontainers + Playwright** | Unit + integration (real Postgres/Redis/MinIO via testcontainers) + browser E2E. |
| Code quality | **ruff (lint+format), mypy (strict), bandit (security)** | Bandit aligns with OWASP ASVS L2 expectations. |
| Package manager | **uv** (backend), **pnpm** (frontend) | Fast, reproducible lockfiles. |
| CI/CD | **GitHub Actions** | Lint → typecheck → unit → integration → build → deploy; SARIF upload for security scans. |

### Project Structure

```
virtual-data-room/
├── docker-compose.yml              # postgres, redis, minio, mailhog, api, worker, web
├── Makefile                        # make dev / test / migrate / seed
├── README.md
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── Dockerfile
│   ├── alembic/
│   │   └── versions/               # numbered, forward-only migrations
│   ├── app/
│   │   ├── main.py                 # FastAPI app factory, middleware, OpenAPI config
│   │   ├── config.py               # Pydantic Settings (env-driven)
│   │   ├── db/
│   │   │   ├── session.py          # async engine, session, RLS tenant context
│   │   │   ├── base.py             # declarative base, mixins (TenantMixin, TimestampMixin)
│   │   │   └── models/             # SQLAlchemy models, one module per domain
│   │   ├── schemas/                # Pydantic request/response models
│   │   ├── api/
│   │   │   ├── deps.py             # auth, tenant, permission dependencies
│   │   │   └── v1/                 # routers: auth, rooms, folders, documents,
│   │   │                           #   permissions, qa, analytics, audit, webhooks, ai
│   │   ├── services/               # business logic (permission_resolver, audit, etc.)
│   │   ├── security/               # jwt, oauth, crypto (envelope encryption), watermark
│   │   ├── ai/                     # presidio_redactor, classifier, summarizer, readiness
│   │   ├── storage/                # s3 client, encrypted object I/O
│   │   ├── workers/                # celery app + tasks (ingest pipeline)
│   │   └── integrations/           # salesforce, docusign, m365, webhook dispatcher
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/               # sample PDFs, DOCX, redaction goldens
├── frontend/
│   ├── package.json
│   ├── app/                        # Next.js App Router
│   │   ├── (auth)/                 # login, sso callback
│   │   ├── (org)/rooms/            # room list, room detail, folder tree
│   │   ├── viewer/[documentId]/    # PDF.js view-only viewer
│   │   ├── qa/                     # Q&A board
│   │   └── analytics/              # engagement dashboard
│   ├── components/                 # shadcn/ui-based components
│   ├── lib/                        # api client (generated from OpenAPI), auth
│   └── tests/                      # Playwright e2e
└── deploy/
    ├── helm/
    └── github/workflows/
```

---

## Phase 1: Foundation — Project Skeleton, Config, Multi-Tenant DB Core

### Purpose
Establish the runnable application skeleton, the multi-tenant database foundation with Row-Level Security, and the core organization/user/membership entities. After this phase, the stack boots via `docker-compose up`, migrations apply, and the API serves a health check and authenticated `/me` endpoint scoped to a tenant. Everything else builds on this isolation guarantee.

### Tasks

#### 1.1 — Application skeleton, config, and Docker stack

**What**: A FastAPI app factory, environment-driven settings, and a docker-compose stack (api, worker, postgres, redis, minio, mailhog).

**Design**:
- `app/config.py` using `pydantic-settings`:
```python
class Settings(BaseSettings):
    environment: Literal["dev", "staging", "prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str
    s3_access_key: str
    s3_secret_key: SecretStr
    s3_bucket_documents: str = "vdr-documents"
    jwt_private_key_path: str
    jwt_public_key_path: str
    jwt_issuer: str = "https://vdr.local"
    access_token_ttl_seconds: int = 900
    kms_provider: Literal["vault", "aws", "local"] = "local"
    llm_provider: str = "openai"
    data_residency_region: str = "us-east-1"
    model_config = SettingsConfigDict(env_file=".env", env_prefix="VDR_")
```
- `app/main.py`: `create_app()` registers routers, exception handlers, CORS, security middleware (CSP Level 3, HSTS, X-Frame-Options DENY), and configures OpenAPI metadata (`openapi_version="3.1.0"`, title, servers).
- Health endpoints: `GET /healthz` (liveness), `GET /readyz` (checks DB + Redis + S3 connectivity).

**Testing**:
- `Unit: Settings loads from env with prefix VDR_ → correct typed fields`
- `Unit: missing VDR_DATABASE_URL → ValidationError naming the field`
- `Integration: GET /healthz → 200 {"status":"ok"}`
- `Integration (testcontainers): GET /readyz with all deps up → 200; with Redis down → 503 listing the failed dependency`

#### 1.2 — Database base, RLS tenant context, migration tooling

**What**: Async SQLAlchemy engine/session, a tenant-context mechanism that sets `app.current_org_id` per connection, and Alembic configured for forward-only migrations.

**Design**:
- `db/session.py`: async engine; a `get_session()` dependency that, after acquiring a connection, executes `SET app.current_org_id = :org_id` from the request's authenticated tenant. A `SystemSession` context manager bypasses RLS for migrations/admin using a privileged role.
- `db/base.py` mixins:
```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    organization_id: Mapped[UUID] = mapped_column(ForeignKey("organizations.id"), index=True)
```
- RLS policy applied in migration to every tenant-scoped table:
```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON <t>
  USING (organization_id = current_setting('app.current_org_id')::uuid);
```
- Migration `0001_init` enables extensions: `pgcrypto`, `pg_trgm`, `btree_gin`.

**Testing**:
- `Integration (real PG): connection with org A context → SELECT on documents returns only org A rows`
- `Integration: query without app.current_org_id set → RLS blocks all rows (0 returned)`
- `Integration: SystemSession bypasses RLS → returns rows across orgs`
- `Unit: alembic upgrade head then downgrade base on a scratch DB → no errors (sanity only; prod is forward-only)`

#### 1.3 — Organizations, users, memberships

**What**: The `organizations`, `users`, and `organization_memberships` tables and CRUD for org/user provisioning.

**Design**: Implement the DDL from `data-model-suggestion-1.md` (Organizations and Users section) as SQLAlchemy models. Pydantic schemas:
```python
class OrganizationCreate(BaseModel):
    name: str; slug: constr(pattern=r"^[a-z0-9-]{3,100}$")
    plan_tier: Literal["free","professional","enterprise"] = "free"
    data_residency_region: str = "us-east-1"

class UserPublic(BaseModel):
    id: UUID; email: EmailStr; display_name: str; status: str
```
Endpoints (admin-scoped): `POST /v1/organizations`, `POST /v1/organizations/{id}/members`, `GET /v1/me`.

**Testing**:
- `Unit: OrganizationCreate with invalid slug "A B" → ValidationError`
- `Integration: POST /v1/organizations → 201, row created, owner membership auto-created`
- `Integration: duplicate slug → 409`
- `Integration: GET /v1/me without token → 401`

### Definition of Done
Stack boots; migrations apply cleanly; RLS isolation proven by tests; org/user/me endpoints work; ruff+mypy+bandit pass; OpenAPI doc served at `/openapi.json`.

---

## Phase 2: Authentication, Authorization & Audit Backbone

### Purpose
Implement the security spine: OAuth 2.0 / OIDC login, JWT issuance (RFC 9068), the data-room RBAC model, the granular permission-resolution engine, and the immutable, hash-chained audit log. Every later feature logs through this and checks permissions through it. Building it now prevents retrofitting security.

### Tasks

#### 2.1 — OAuth 2.0 / OIDC auth and JWT access tokens

**What**: Email/password and OIDC login flows issuing RFC 9068 JWT access tokens, plus Client Credentials (RFC 7523) for service accounts.

**Design**:
- `security/jwt.py`: RS256 signing. Access token claims: `iss, sub (user_id), aud, exp, iat, jti, org_id, scope`. Public JWKS at `GET /.well-known/jwks.json`.
- Flows: Authorization Code + PKCE for the web app; Client Credentials for integrations. Use Authlib.
- `POST /v1/auth/login` (password), `POST /v1/auth/token` (OAuth token endpoint), `POST /v1/auth/refresh`.
- Password hashing: argon2id. MFA (TOTP) optional per user.

**Testing**:
- `Unit: token signed then verified → claims match; tampered token → InvalidSignature`
- `Unit: expired token → 401 token_expired`
- `Integration: login with valid creds → access+refresh tokens; bad password → 401, audit login.failed written`
- `Integration: JWKS endpoint returns the public key matching the signer`

#### 2.2 — Data-room RBAC: roles, groups, memberships

**What**: `data_room_roles`, `user_groups`, `data_room_users` tables with seeded system roles (organizer, reviewer, bidder, legal_counsel, viewer).

**Design**: Models from the Access Control DDL. On room creation a migration/service seeds the five system roles with the flag defaults from the schema (e.g., organizer: all flags true; viewer: only `can_view`). Endpoints: `POST /v1/rooms/{room}/users` (invite), `PATCH .../users/{id}` (change role/group/access window), `DELETE` (revoke).

**Testing**:
- `Integration: invite user to room → data_room_users row status 'invited', audit user.invited written`
- `Integration: assign 'viewer' role → resolved flags allow view only`
- `Unit: access_expires_at in the past → user treated as no-access`

#### 2.3 — Granular permission resolution engine

**What**: A service that computes effective `(can_view, can_download, can_print, can_upload)` for a user on a folder/document, merging role → group → user overrides up the folder hierarchy, with a Redis cache.

**Design**:
- Port `resolve_document_permission` (the recursive-CTE SQL function from the data model) into a migration, AND mirror the logic in `services/permission_resolver.py` for cache warming. Specificity order: document-level user > document-level group > document-level role > folder-level (deepest first, same specificity order) > room role default.
- Cache key `perm:{room_id}:{user_id}:{document_id}` → flags, TTL 300s, invalidated on any `resource_permissions`/`data_room_users` change via Redis pub-sub (`perm.invalidate` channel).
- `api/deps.py`: `require_permission("view"|"download"|...)` dependency raising 403 on deny.

**Testing**:
- `Unit: role allows view, folder override denies view → resolved deny`
- `Unit: folder denies, document-level user override allows → allow (most specific wins)`
- `Unit: deny propagates to child folders when inherit=True`
- `Integration: permission change publishes invalidation → next resolve recomputes`
- `Integration: user with no membership → all flags false`

#### 2.4 — Immutable, hash-chained audit log

**What**: The partitioned `audit_log` table with WORM rules and a per-row hash chain, plus an `AuditService` used by all write paths.

**Design**:
- DDL from the Audit Trail section, partitioned monthly; add `prev_hash` and `row_hash` columns. `row_hash = sha256(prev_hash || canonical_json(row_without_hash))`. A nightly Celery task creates next month's partition.
- `AuditService.record(action, resource_type, resource_id, details, request)` captures actor, IP, user-agent. Standard action vocabulary enumerated as a `StrEnum` (`document.viewed`, `document.downloaded`, `permission.changed`, `room.opened`, `login.failed`, …).
- WORM: `audit_log_no_update`/`audit_log_no_delete` rules; app uses an insert-only DB role. Export endpoint `GET /v1/rooms/{room}/audit?format=csv|jsonl` (organizer-only) for regulatory submission.
- Chain-verification endpoint/CLI recomputes hashes and reports the first break.

**Testing**:
- `Unit: row_hash deterministic for same input; differs if any field changes`
- `Integration: UPDATE on audit_log is silently ignored (row unchanged); DELETE ignored`
- `Integration: insert 3 events → chain verifies; manually corrupt middle event → verification reports break at that index`
- `Integration: audit export CSV has header + one row per event, RLS-scoped to the room`

### Definition of Done
Login issues verifiable JWTs; RBAC seeded; permission engine passes the override matrix; audit log is append-only and chain-verifiable; all auth/authz endpoints in OpenAPI; ASVS L2 auth checks (rate limiting on login, no user enumeration) covered by tests.

---

## Phase 3: Data Rooms, Folders & Encrypted Document Storage

### Purpose
Deliver the structural heart of a VDR: data rooms with lifecycle, a hierarchical folder tree with VDR-standard index numbering, and encrypted document upload/storage. After this phase an organizer can create a room, build a folder structure, and upload a file that lands encrypted in object storage with metadata in Postgres — guarded by the permission engine and recorded in the audit log.

### Tasks

#### 3.1 — Data rooms with lifecycle and settings

**What**: `data_rooms` and `deal_checklists`/`deal_checklist_items` tables, plus room lifecycle transitions.

**Design**: Models from the Data Rooms DDL. Lifecycle state machine: `draft → setup → active → closed → archived` (forward-only except admin reopen `closed→active`). `POST /v1/rooms`, `PATCH /v1/rooms/{id}` (settings: watermark, download/print/NDA flags, AI flags, retention_days), `POST /v1/rooms/{id}/open` (sets status active + opened_at, emits `room.opened` audit + webhook), `POST /v1/rooms/{id}/close`.

**Testing**:
- `Unit: transition active→draft → ValueError invalid_transition`
- `Integration: open room → status active, opened_at set, audit room.opened recorded`
- `Integration: PATCH retention_days < 1 → 422`

#### 3.2 — Hierarchical folders with materialized path & index numbering

**What**: `folders` table using adjacency list + materialized path; auto-generated VDR index numbers ("1", "1.2", "1.2.3").

**Design**: Model from the Folders DDL. On create, compute `path = parent.path || new_id || '/'`, `depth = parent.depth + 1`, and `index_number` from sibling `sort_order`. Move operation rewrites `path`/`depth` for the subtree in one transaction. Endpoints: `POST /v1/rooms/{room}/folders`, `PATCH .../folders/{id}` (rename/move/reorder), `GET .../folders/tree` (returns nested tree honoring per-user view permissions).

**Testing**:
- `Unit: child of root depth 0 → depth 1, path contains both ids`
- `Unit: index numbering for 3 siblings → "1","2","3"; nested child of #2 → "2.1"`
- `Integration: move subtree → all descendant paths/depths updated`
- `Integration: GET tree as viewer with folder denied → that folder omitted`

#### 3.3 — Encrypted object storage layer (envelope encryption)

**What**: `storage/` module that writes/reads objects to S3/MinIO with per-document AES-256-GCM envelope encryption.

**Design**:
- `EncryptionService.envelope_encrypt(plaintext) -> (ciphertext, encrypted_dek, nonce)`: generate a random 256-bit DEK, encrypt content with AES-256-GCM, wrap the DEK with the tenant KEK from the KMS provider (`local` = libsodium sealed key for dev; `aws` = KMS; `vault` = Transit). Store `encryption_key_id` referencing the wrapped DEK.
- `StorageService.put_document(room_id, document_id, stream) -> StorageRef` writes to key `org/{org}/room/{room}/doc/{doc}/v{n}`. Streaming, never buffering full file in memory.
- NIST SP 800-57 key lifecycle: keys are per-document; rotation re-wraps DEKs without re-encrypting content.

**Testing**:
- `Unit: encrypt then decrypt → original bytes; wrong DEK → AEAD auth failure`
- `Integration (MinIO testcontainer): put then get → byte-identical content; object at rest is ciphertext (not plaintext-searchable)`
- `Unit: KMS provider 'local' wraps/unwraps DEK round-trips`

#### 3.4 — Document upload & metadata (pre-processing)

**What**: `documents` table and a chunked/multipart upload endpoint that validates files (OWASP File Upload), stores them encrypted, and records metadata in `processing_status='pending'`.

**Design**: Model from the Documents DDL. `POST /v1/rooms/{room}/folders/{folder}/documents` accepts multipart upload. Validation per OWASP File Upload Cheat Sheet: extension+MIME allow-list (PDF, DOCX, XLSX, PPTX, images, TXT), magic-byte sniffing (`python-magic`), max size, filename sanitization, metadata stripping deferred to processing. Compute `checksum_sha256` while streaming. On success: enqueue the Celery ingest chain (Phase 4), emit `document.uploaded` audit + webhook. `require_permission("upload")` enforced.

**Testing**:
- `Unit: filename "../../etc/passwd" → sanitized to safe basename`
- `Integration: upload PDF → 201, document row processing_status 'pending', object in MinIO, ingest task enqueued (mock broker)`
- `Integration: upload .exe → 415 unsupported_media_type, no row, no object`
- `Integration: MIME/extension mismatch (renamed .exe→.pdf) → 415 via magic-byte check`
- `Integration: upload without upload permission → 403`

### Definition of Done
Rooms, folders (with index numbering and move), and encrypted uploads all work and are permission-gated and audited; objects verified encrypted at rest; OWASP upload controls tested; OpenAPI updated.

---

## Phase 4: Document Processing Pipeline — Conversion, OCR, Indexing, Search

### Purpose
Turn raw uploads into viewable, searchable documents. This Celery pipeline converts Office files to PDF, OCRs scanned pages, extracts text for full-text search, and counts pages — the non-AI processing that every VDR needs. After this phase, uploaded documents reach `processing_status='ready'` and are discoverable via full-text search.

### Tasks

#### 4.1 — Celery pipeline orchestration with saga-style status

**What**: A Celery chain `convert → ocr → extract_text → index → finalize`, driving the `processing_status` state machine with failure handling.

**Design**:
- `workers/tasks.py`: each step updates `documents.processing_status` (`converting → indexing → ... → ready`) within a transaction. Any step failure sets `processing_status='failed'` + `processing_error`, emits `document.processing_failed` webhook, and does NOT poison later steps. Idempotent steps keyed on `document_id` + step so retries are safe.
- Real-time status pushed over WebSocket `ws/rooms/{room}/documents` and reflected in `GET /v1/documents/{id}` (returns status + progress).
- Configuration: `MAX_RETRIES=3`, exponential backoff, dead-letter handling logged to audit.

**Testing**:
- `Integration (real Celery eager mode): upload PDF → chain runs → status 'ready'`
- `Integration: convert step raises → status 'failed', error captured, later steps not run`
- `Unit: re-running 'index' for already-indexed doc → no duplicate, idempotent`

#### 4.2 — Office→PDF conversion and PDF normalization

**What**: Convert DOCX/XLSX/PPTX to PDF via headless LibreOffice; normalize/validate existing PDFs; record `pdf_storage_key`, `pdf_page_count`.

**Design**: `ai/.../converter.py` shells out to `libreoffice --headless --convert-to pdf` in an isolated temp dir (sandboxed, no network). Output re-encrypted and stored at `pdf_storage_key`. PDFs validated with PyMuPDF; corrupt files → failed status with a clear error.

**Testing**:
- `Fixture: sample.docx → produces a PDF with expected page_count`
- `Fixture: corrupt.pdf → status failed, error "invalid_pdf"`
- `Integration: converted PDF stored encrypted, retrievable, page_count populated`

#### 4.3 — OCR for scanned documents

**What**: Detect image-only pages and run Tesseract OCR, merging recognized text into the extractable layer.

**Design**: For each page, if PyMuPDF extracts < N chars but the page has a large image, render at 300 DPI and OCR with Tesseract (`pytesseract`), language auto-detected (default `eng`, configurable list for multilingual). OCR text appended to `content_text`.

**Testing**:
- `Fixture: scanned-invoice.pdf (image-only) → content_text contains known words`
- `Fixture: native-text.pdf → OCR skipped (fast path), text extracted directly`

#### 4.4 — Full-text indexing & search

**What**: Populate `content_text`/`search_vector` and expose ranked full-text search scoped to permissions.

**Design**: The `documents_search_update` trigger (from the data model) maintains `search_vector` (display_name weight A, ai_summary B, content_text C). `GET /v1/rooms/{room}/search?q=...` runs `websearch_to_tsquery` with `ts_rank` ordering, then filters results through the permission engine (a user must have `can_view` on each hit). Returns snippets via `ts_headline`.

**Testing**:
- `Integration: index 3 docs, search a term in doc 2 only → returns doc 2 ranked first with highlighted snippet`
- `Integration: search term in a doc the user cannot view → excluded from results`
- `Unit: malicious tsquery input → safely parsed (no error), no injection`

### Definition of Done
Upload→ready pipeline runs end-to-end with status visible in real time; Office conversion, OCR, and search all tested with fixtures; failures are isolated and audited.

---

## Phase 5: Secure Viewer, Watermarking & View-Only Enforcement

### Purpose
Deliver the controlled-viewing experience that distinguishes a VDR from file sharing: a browser viewer that renders watermarked pages with download/print disabled, enforced both client- and server-side. This is where access control becomes tangible to end users.

### Tasks

#### 5.1 — Dynamic watermarking (visible + invisible)

**What**: `watermark_configs` table and a service that stamps per-session visible watermarks and embeds invisible (steganographic) payloads.

**Design**: Model from the Watermarking DDL. `WatermarkService.render_page(doc, page, session)`:
- Visible: PyMuPDF overlays the templated text (`{{user_email}} | {{datetime}} | CONFIDENTIAL`) diagonally at configured opacity/font size.
- Invisible: embed a payload (`user_id`, `session_id`, `timestamp`) via least-significant-bit perturbation of rendered page image / metadata, recoverable later for leak attribution. A `watermark_id` (random, stored on the view session) ties a leaked page back to a user.

**Testing**:
- `Unit: rendered page contains the user's email text (extractable via OCR of the output)`
- `Unit: embedded payload recoverable from output image → matches session watermark_id`
- `Unit: template with unknown variable → rendered literally, no crash`

#### 5.2 — View-only streaming endpoint

**What**: `GET /v1/documents/{id}/pages/{n}` that returns a watermarked, rasterized page image (not the source PDF) for view-only users, creating/continuing a `document_view_sessions` record.

**Design**: Permission check: `can_view` required. If `can_download` is false, the source file is never sent — only per-page PNG/WebP renders with watermark applied server-side. Response carries `Cache-Control: no-store` and CSP headers. Opens/continues a view session, logs `document.viewed` (page number, watermark_id) to audit. Download endpoint `GET /v1/documents/{id}/download` exists separately and requires `can_download` (returns decrypted PDF, watermarked); print is a client capability gated by `can_print`.

**Testing**:
- `Integration: view-only user GETs page 1 → 200 image with watermark, view session created, audit document.viewed`
- `Integration: view-only user hits /download → 403`
- `Integration: download-enabled user GETs /download → 200 watermarked PDF, audit document.downloaded`
- `Integration: page request beyond page_count → 404`

#### 5.3 — Frontend viewer (Next.js + PDF.js) with engagement capture

**What**: An accessible (WCAG 2.2 AA) viewer component that displays streamed pages, disables right-click/save/print when not permitted, and reports page-level engagement.

**Design**: Client component renders page images sequentially with virtualized scrolling. Captures per-page dwell time and scroll depth, batching `page_view_events` to `POST /v1/documents/{id}/events` (feeds heatmaps in Phase 7). Print/download UI hidden and the corresponding shortcuts intercepted when permissions deny (defense-in-depth; server is the real gate). CSP Level 3 restricts script/style/img sources; keyboard navigation and focus order meet AA.

**Testing**:
- `E2E (Playwright): view-only user opens viewer → pages render, no download button, ctrl+S blocked`
- `E2E: dwell on page 2 for 3s → batched event POSTed with ~3000ms duration`
- `E2E: axe-core scan of viewer → no WCAG AA violations`

### Definition of Done
Viewer renders watermarked pages; view-only is enforced server-side (source never leaves server); engagement events captured; download/print gated by permissions; viewer passes axe-core AA scan.

---

## Phase 6: AI-Native Document Processing — Redaction, Classification, Summarization

### Purpose
Deliver the AI-native advantage central to the product thesis: automatic PII redaction with confidence scoring, document classification/auto-tagging that learns from organizer corrections, and generative summarization — all available at every tier, not just enterprise. This is the primary differentiator versus incumbents.

### Tasks

#### 6.1 — PII detection & redaction with confidence scoring

**What**: `redaction_rules` and `redactions` tables; a pipeline step that detects PII (Presidio + spaCy NER, LLM second-pass) and produces redacted page outputs with a human review queue.

**Design**: Models from the PII Redaction DDL. `ai/presidio_redactor.py` runs Presidio analyzers across `content_text` mapped to page coordinates (via PyMuPDF text spans), producing `redactions` rows with `pii_type`, bounding box, `confidence`, `detection_method` ('ai_ner'|'regex'|'manual'), `review_status='pending'`. An optional LLM pass validates low-confidence hits. `original_text` stored encrypted. Organizer review endpoints: `GET /v1/documents/{id}/redactions`, `PATCH .../redactions/{id}` (approve/reject/modify). Approved redactions are burned into a redacted PDF (PyMuPDF black-box + remove underlying text) stored at `redacted_storage_key`; counterparties only ever receive the redacted render.

**Testing**:
- `Fixture: PDF containing an SSN → redaction row pii_type 'ssn', confidence > 0.8, correct page`
- `Unit: regex rule for custom keyword → matches expected spans`
- `Integration: approve redaction → redacted PDF has the region blacked out AND underlying text removed (not recoverable via copy)`
- `Integration: reject redaction → original text remains, audit redaction.rejected`
- `Integration: counterparty fetch of a doc with approved redactions → served the redacted render`

#### 6.2 — AI document classification & auto-tagging with learning loop

**What**: Pipeline step that classifies each document (e.g., "Financial Statement", "Contract", "Cap Table") and suggests tags/folder placement, recording confidence and learning from organizer corrections.

**Design**: `ai/classifier.py` sends extracted text (truncated/sampled) to the LLM with a structured-output prompt returning `{classification, confidence, tags[], suggested_folder}`. Stored in `documents.ai_classification`, `ai_classification_confidence`, `ai_tags`. Organizer corrections are written to a `classification_feedback` table (new, additive) and used as few-shot exemplars (retrieved per tenant) on subsequent classifications — the learning loop. EU AI Act Art. 50: classification results carry an `ai_generated=true` disclosure flag in the API.

Prompt template (structure):
```
System: You are a document classifier for M&A due diligence. Classify into one of {taxonomy}. Return JSON: {classification, confidence (0-1), tags, suggested_folder, rationale}.
User: Filename: {name}\nExcerpt:\n{first_2000_chars}\nKnown tenant examples:\n{few_shot}
```

**Testing**:
- `Fixture (mocked LLM): financial-statement.pdf → classification "Financial Statement", confidence captured, ai_generated flag true`
- `Integration: organizer corrects classification → feedback row written; next classification of similar doc includes it as few-shot (assert prompt contains the example)`
- `Unit: LLM returns malformed JSON → step retried then marked failed gracefully, doc still 'ready' without classification`

#### 6.3 — Generative document summarization & explanation

**What**: On-demand and pipeline summarization producing `ai_summary`, plus a Q&A-style "explain this document" endpoint.

**Design**: `ai/summarizer.py` chunks long documents, map-reduces summaries via the LLM, stores `documents.ai_summary` (also fed into `search_vector` weight B). `POST /v1/documents/{id}/explain {question}` answers grounded only in that document's text, returning citations (page numbers). All AI output carries the `ai_generated` disclosure.

**Testing**:
- `Fixture (mocked LLM): 50-page doc → summary stored, non-empty, length-bounded`
- `Integration: /explain with a question answerable from page 3 → answer cites page 3`
- `Integration: /explain without can_view on the doc → 403`

### Definition of Done
Redaction produces reviewable, irreversibly-burned redactions; classification populates metadata and improves from corrections; summaries generated and searchable; all AI outputs carry the AI Act disclosure flag; LLM calls mocked in CI with at least one optional real-provider smoke test.

---

## Phase 7: Q&A Workflow, Analytics & Engagement Intelligence

### Purpose
Add the collaboration and intelligence layers buyers evaluate VDRs on: a structured, SLA-tracked Q&A workflow with role-based visibility, and engagement analytics (per-user view time, page-level heatmaps, engagement scoring). After this phase organizers can run due-diligence Q&A and read buyer interest from the data.

### Tasks

#### 7.1 — Structured Q&A workflow with SLA and visibility

**What**: `qa_topics`, `qa_questions`, `qa_answers` tables with the question lifecycle, routing, SLA deadlines, and visibility scoping.

**Design**: Models from the Q&A DDL. Lifecycle `draft → submitted → assigned → answered → (follow_up|closed|rejected)`. On submit, assign `question_number` (`Q-001`…) and compute `sla_deadline` from room SLA config; a periodic Celery task flags `sla_breached` and notifies. Visibility (`all_parties|submitter_and_assignee|sell_side_only|buy_side_only`) filters list/read results per requesting user. Endpoints: `POST .../questions`, `POST .../questions/{id}/assign`, `POST .../questions/{id}/answers`, `POST .../answers/{id}/approve`. Each transition audited.

**Testing**:
- `Integration: submit question → number assigned, sla_deadline set, audit qa.submitted`
- `Integration: bidder lists questions → sees own + all_parties, not sell_side_only ones`
- `Integration: SLA task past deadline → sla_breached true, assignee notified`
- `Unit: answer approval by non-organizer → 403`

#### 7.2 — Engagement tracking & page-level heatmaps

**What**: Ingest `document_view_sessions` and `page_view_events` (from Phase 5 viewer) and expose heatmap + session analytics.

**Design**: Models from the Analytics DDL. `POST /v1/documents/{id}/events` batches page events (page, duration_ms, scroll_depth). `GET /v1/documents/{id}/heatmap` aggregates total dwell per page; `GET /v1/rooms/{room}/users/{id}/activity` returns session timeline. `page_view_events` partitioned by month. Analytics endpoints require `can_view_analytics`.

**Testing**:
- `Integration: post events for pages 1-3 → heatmap shows per-page totals`
- `Integration: analytics endpoint without can_view_analytics → 403`
- `Unit: batch with negative duration → rejected/clamped`

#### 7.3 — Engagement scoring & buyer-readiness signals

**What**: The `mv_user_engagement` materialized view plus a buyer-readiness heuristic, exposed on a dashboard endpoint.

**Design**: Materialized view from the Analytics DDL (composite score over documents viewed, dwell hours, questions asked), refreshed on a schedule (`REFRESH MATERIALIZED VIEW CONCURRENTLY`). `GET /v1/rooms/{room}/engagement` returns ranked users with scores; a buyer-readiness band (cold/warm/hot) derived from score percentiles + recency + Q&A activity.

**Testing**:
- `Integration: user with many views + questions → higher engagement_score than a passive user`
- `Integration: refresh after new sessions → scores update`
- `Unit: readiness band thresholds map scores to cold/warm/hot correctly`

#### 7.4 — Deal-readiness AI scoring

**What**: Score a room's document completeness against its `deal_checklists`, surfacing gaps before the room opens.

**Design**: `ai/readiness.py` matches uploaded documents (by classification + folder + name) to `deal_checklist_items`, computing per-item `completeness_score` and `matched_document_count`; an LLM judges quality/sufficiency of matched docs for ambiguous items. `GET /v1/rooms/{room}/readiness` returns overall % and the list of missing/weak items. Seeded default checklists per `deal_type`.

**Testing**:
- `Fixture: room with 8/10 required items matched → overall 80%, two gaps listed`
- `Integration: upload the missing doc + reclassify → readiness recomputes upward`
- `Unit: checklist item with no matches → completeness_score 0, flagged required gap`

### Definition of Done
Q&A workflow runs with SLA + visibility enforcement; heatmaps and engagement scores populate from real viewer events; deal-readiness reports gaps; all analytics permission-gated and audited.

---

## Phase 8: Integrations, Webhooks, MCP & Remote Wipe

### Purpose
Connect the VDR to the deal-workflow ecosystem and complete the security feature set: outbound webhooks, Salesforce/DocuSign/M365 connectors, an MCP server exposing permissioned room contents to AI agents, and remote-wipe of distributed documents.

### Tasks

#### 8.1 — Outbound webhooks (HMAC-signed, retrying)

**What**: `webhook_subscriptions` table and a dispatcher emitting HMAC-SHA256-signed events with retries.

**Design**: Model from the Integrations DDL. Events: `document.uploaded`, `document.processing_failed`, `room.opened/closed`, `qa.submitted/answered`, `user.invited`. Dispatcher (Celery) POSTs JSON with header `X-VDR-Signature: sha256=<hmac(secret, body)>` and `X-VDR-Event`. Exponential-backoff retries up to N; `failure_count` increments, auto-pauses after threshold. Secret rotation supported.

**Testing**:
- `Integration: subscribed event fires → POST sent with valid HMAC signature`
- `Unit: signature computed matches independent HMAC verification; altered body → mismatch`
- `Integration: endpoint returns 500 thrice → retried, failure_count incremented, then paused`

#### 8.2 — DocuSign & Salesforce connectors

**What**: `integrations` table (encrypted config) and OAuth-based connectors to send documents for signature (DocuSign) and sync deal/room metadata (Salesforce).

**Design**: Model from the Integrations DDL; `config_encrypted` holds OAuth tokens (envelope-encrypted). DocuSign: `POST /v1/documents/{id}/send-for-signature` creates an envelope via DocuSign eSign API (OAuth Authorization Code / JWT Grant), webhook back updates status. Salesforce: push room/deal records via Client Credentials. Token refresh handled centrally; `last_error`/`status` tracked.

**Testing**:
- `Integration (mocked DocuSign API): send-for-signature → envelope created, status tracked`
- `Integration: expired token → auto-refresh attempted; refresh fail → integration status 'error', last_error set`
- `Unit: config encrypted at rest (stored bytes ≠ plaintext token)`

#### 8.3 — MCP server for AI-agent access

**What**: An MCP server exposing room documents, folder structure, and metadata as resources, plus tools for Q&A submission and metadata retrieval — all permission- and audit-scoped.

**Design**: Implement `resources/list` (folders+documents the agent's bound user may view), `resources/read` (returns text/summary, never bypassing redaction or `can_view`), and tools `submit_question`, `get_document_metadata`. Every access flows through the permission engine and is audited as `mcp.resource.read`. Agent identity bound via Client Credentials JWT (RFC 7523) mapped to a data-room user.

**Testing**:
- `Integration: resources/list for a viewer → only viewable docs listed`
- `Integration: resources/read on a denied doc → MCP error, audit shows denied attempt`
- `Integration: submit_question tool → qa_questions row created, audited`

#### 8.4 — Remote wipe

**What**: `remote_wipe_requests` table and a mechanism to revoke and destroy cached document material on viewer devices after access is revoked.

**Design**: Model from the Watermarking/Remote Wipe DDL. Because the viewer is view-only (server-streamed, `no-store`), "wipe" = immediate key revocation + viewer session invalidation: revoking access publishes a `session.revoked` signal over the document WebSocket; the client clears rendered buffers and the server refuses further page requests. For download-enabled docs, wipe re-wraps/rotates the DEK and marks the document `status='wiped'` so the prior decrypted copies' re-fetch is impossible. Tracks `devices_targeted`/`devices_confirmed`.

**Testing**:
- `Integration: revoke user access → open viewer session receives session.revoked, subsequent page request → 403`
- `Integration: wipe a document → status 'wiped', further view/download → 410 gone, audit document.wiped`
- `Integration: device confirms wipe → devices_confirmed incremented`

### Definition of Done
Webhooks deliver signed, retried events; DocuSign/Salesforce connectors work against mocked APIs with token refresh; MCP server enforces permissions and audits access; remote wipe revokes live sessions and invalidates keys.

---

## Phase 9: Hardening, Compliance & Production Readiness

### Purpose
Bring the platform to production and certification-readiness: encrypted messaging, compliance exports, OWASP ASVS L2 hardening, accessibility conformance, observability, and the deployment artifacts. This phase makes the product sellable to the regulated mid-market.

### Tasks

#### 9.1 — Encrypted in-room messaging & notifications

**What**: `messages` (application-layer encrypted) and `notifications` tables with in-room threads and a notification feed.

**Design**: Models from the Messaging DDL. Message bodies AES-256-GCM encrypted (`body_encrypted`/`body_nonce`) with the room key; visibility `all|group|direct`. WebSocket delivery; `notifications` populated on Q&A events, mentions, SLA breaches, processing completion.

**Testing**:
- `Integration: send group message → only group members receive/list it; stored bytes are ciphertext`
- `Integration: Q&A answered → submitter gets a notification row`

#### 9.2 — Compliance: retention, WORM export, GDPR/CCPA tooling

**What**: Retention enforcement, regulatory audit export (SEC 17a-4 WORM), and data-subject request (DSAR) / erasure tooling.

**Design**: Scheduled task archives audit partitions older than `retention_days` to S3 with Object Lock (WORM). DSAR endpoint exports all PII for a subject across rooms; erasure honors GDPR right-to-erasure where not under legal hold (legal-hold flag blocks erasure, returns the conflicting retention basis). CCPA/CPRA: privacy-risk-assessment metadata recorded for AI features (redaction/classification/scoring) per the 2026 amendments.

**Testing**:
- `Integration: audit older than retention → archived to WORM bucket, original partition detached`
- `Integration: DSAR export → returns subject's PII across rooms`
- `Integration: erasure under legal hold → 409 with retention basis; without hold → data removed, tombstone audited`

#### 9.3 — Security hardening to OWASP ASVS L2

**What**: Rate limiting, brute-force protection, CSP/security headers, dependency and SAST scanning with SARIF output.

**Design**: Per-IP and per-account rate limits (Redis token bucket) on auth and upload; account lockout with backoff; security headers middleware (CSP3, HSTS, X-Content-Type-Options, Referrer-Policy). CI runs `bandit`, `pip-audit`, and a container scan, all emitting SARIF uploaded to GitHub code scanning. Map controls to ASVS L2 chapters in a checklist doc.

**Testing**:
- `Integration: 10 failed logins in window → account temporarily locked, audit login.locked`
- `Integration: responses carry CSP, HSTS, X-Frame-Options DENY`
- `CI: bandit/pip-audit run and produce SARIF; build fails on high-severity findings`

#### 9.4 — Observability & deployment

**What**: Structured logging, metrics, tracing, and production deployment artifacts (Helm chart, GitHub Actions CD).

**Design**: JSON structured logs with request/trace IDs; OpenTelemetry traces across API→worker→DB; Prometheus metrics (request latency, queue depth, processing duration, redaction throughput). Helm chart for api/worker/web/redis with PgBouncer; HPA on the worker pool. GitHub Actions: lint→typecheck→unit→integration→build→push→deploy (blue-green). DB migrations gated and run pre-cutover.

**Testing**:
- `Integration: a request emits a trace spanning API and a worker task with a shared trace ID`
- `Integration: /metrics exposes the documented counters`
- `CI: helm lint + helm template render with no errors; smoke deploy to ephemeral env passes /readyz`

### Definition of Done
Messaging encrypted; retention/WORM/DSAR/erasure work with legal-hold respected; ASVS L2 controls in place with SARIF in CI; observability live; Helm + CD pipeline deploys a green environment that passes readiness checks. Platform is certification-ready (ISO 27001 / SOC 2 Type II evidence collectable from the audit log and controls).

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (DB, RLS, org/user)          ─── required by everything
    │
Phase 2: Auth, RBAC, Permission Engine, Audit    ─── requires 1
    │
Phase 3: Rooms, Folders, Encrypted Storage       ─── requires 2
    │
Phase 4: Processing Pipeline (convert/OCR/index) ─── requires 3
    │
    ├── Phase 5: Viewer & Watermarking           ─── requires 4 (can parallel with 6)
    └── Phase 6: AI (redaction/classify/summary) ─── requires 4 (can parallel with 5)
         │
Phase 7: Q&A & Analytics & Readiness             ─── requires 5 (events) + 6 (readiness)
    │
Phase 8: Integrations, Webhooks, MCP, Wipe       ─── requires 3 (webhooks) ; MCP requires 6 ; wipe requires 5
    │
Phase 9: Hardening, Compliance, Production        ─── requires all
```

**Parallelism opportunities:**
- **Phases 5 and 6** can be developed concurrently once Phase 4 is complete (viewer team vs. AI team).
- Within Phase 8, **8.1 (webhooks)** and **8.2 (connectors)** can start as soon as Phase 3 events exist; **8.3 (MCP)** waits on Phase 6; **8.4 (remote wipe)** waits on Phase 5.
- Within Phase 9, **9.3 (security)** and **9.4 (observability)** can proceed in parallel with **9.1/9.2**.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks implemented as specified.
2. All unit and integration tests pass (integration tests use testcontainers for real Postgres/Redis/MinIO).
3. `ruff` lint + format clean; `mypy --strict` passes; `bandit` reports no high-severity findings.
4. New/changed endpoints appear in the auto-generated OpenAPI 3.1 document and validate against JSON Schema 2020-12.
5. Alembic migration(s) created, forward-only, and apply cleanly on a fresh database.
6. RLS policy present on every new tenant-scoped table; tenant-isolation test added.
7. Every state-changing operation writes an audit-log entry with the correct action vocabulary.
8. New config options documented (env vars with `VDR_` prefix and defaults).
9. Docker build succeeds; `docker-compose up` boots the affected services; `/readyz` green.
10. Frontend changes pass an axe-core WCAG 2.2 AA scan (where UI is involved).
11. Any AI-generated output exposed to users carries the `ai_generated` disclosure flag (EU AI Act Art. 50).
```
