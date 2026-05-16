# Portal Architecture

## 1) System Overview

This document defines the target architecture for the portal platform so implementation across frontend, backend, and infrastructure stays consistent.

### Core layers

1. **Client-facing Pages frontend (Cloudflare Pages)**
   - Delivers the web UI for public and authenticated users.
   - Owns route rendering, client-side data fetching, and session-aware UX state.
   - Must not directly access privileged data stores (D1/KV/R2) without going through Worker APIs.

2. **Worker API layer (Cloudflare Workers)**
   - Single trusted backend boundary for business logic, validation, authorization, and data access.
   - Exposes versioned JSON APIs under `/api/v1/*`.
   - Mediates access to D1, KV, and R2 bindings.

3. **Auth0 authentication and authorization**
   - Auth0 is the identity provider (IdP).
   - Pages frontend initiates login and receives Auth0 session/tokens.
   - Worker APIs validate Auth0 JWT access tokens and enforce role/scope checks.

4. **Data and storage layer**
   - **D1** for relational, queryable business records.
   - **KV** for lightweight key-value caching, idempotency artifacts, and short-lived app state.
   - **R2** for binary/document storage (uploads, attachments, exports).

---

## 2) Frontend Architecture (Client-facing Pages)

### Frontend responsibilities

- Route rendering and layout composition.
- Auth-aware route guards and navigation behavior.
- API orchestration to Worker endpoints.
- Display-only transformations and user input collection.

### Frontend boundaries

- No direct SQL access.
- No direct write access to KV or R2 from browser clients.
- No embedding of privileged secrets in client bundles.
- All privileged operations must be proxied through Worker APIs.

---

## 3) Worker API Layer Boundaries

### Worker responsibilities

- Validate and parse inbound JSON.
- Enforce authentication and role-based authorization.
- Execute business logic and workflow orchestration.
- Read/write D1 records.
- Manage KV cache/session/idempotency records.
- Issue R2 pre-signed URLs or perform brokered upload/download operations.
- Emit structured audit logs and request correlation IDs.

### Worker boundaries

- No UI-specific concerns (presentation formatting stays in frontend).
- No unauthenticated access to privileged endpoints.
- No cross-tenant data access without explicit tenant checks.

---

## 4) Auth0 Authentication Flow

1. User visits frontend route on Cloudflare Pages.
2. If route is protected, frontend redirects to Auth0 Universal Login.
3. Auth0 authenticates user and returns to callback route.
4. Frontend obtains/refreshes access token for API audience.
5. Frontend sends `Authorization: Bearer <token>` to Worker API.
6. Worker validates token signature/issuer/audience/expiry.
7. Worker maps claims to application roles (client, partner, dev/admin) and enforces route-level scopes.
8. Worker returns JSON response; frontend renders view.

### Role model (minimum)

- `public`: no login required.
- `client`: authenticated end users creating and tracking requests.
- `partner`: authenticated partner users for intranet operations.
- `developer` or `admin`: elevated access to internal tooling and operational endpoints.

---

## 5) D1 Schema Boundaries

D1 should be separated by functional domains to avoid accidental coupling and simplify permission checks.

### Suggested schema domains

- **identity domain**
  - `users`, `roles`, `user_roles`, optional `partner_orgs`.
- **request domain**
  - `requests`, `request_events`, `request_status_history`, `request_messages`.
- **chat domain**
  - `chat_sessions`, `chat_messages`, `chat_participants`.
- **partner operations domain**
  - `partner_upload_jobs`, `partner_upload_items`, `partner_job_errors`.
- **audit domain**
  - `audit_log`, `api_access_log`.

### D1 boundary rules

- Worker layer is the only writer to D1.
- Cross-domain joins should be minimized and reviewed.
- Migrations are append-only and versioned.
- Sensitive columns should be minimized and protected via tokenized references where possible.

---

## 6) KV and R2 Usage Boundaries

### KV namespaces (recommended)

- `PORTAL_CACHE_KV`
  - Response/cache fragments and computed read models.
  - Strict TTL defaults; never system-of-record.
- `PORTAL_SESSION_KV`
  - Ephemeral session adjunct state, CSRF nonces, one-time tokens.
  - Short TTL and key prefixing by environment + tenant.
- `PORTAL_IDEMPOTENCY_KV`
  - Idempotency keys for create/update endpoints.
  - TTL aligned with retry windows.

### R2 buckets (recommended)

- `portal-client-assets-r2`
  - Client-provided files tied to requests.
- `portal-partner-ingest-r2`
  - Partner bulk upload files and staged ingestion artifacts.
- `portal-export-r2`
  - Generated reports/exports with lifecycle expiration rules.

### KV/R2 boundary rules

- KV is not authoritative persistence for business records.
- R2 objects are referenced by D1 metadata (owner, checksum, size, classification).
- Use signed URL workflows with short expiration and content-type constraints.
- Enforce per-role read/write policies through Worker endpoints.

---

## 7) Route Groups and Ownership

## Public routes (no auth required)

- **Marketing**
  - Examples: `/`, `/about`, `/pricing`, `/contact`.
- **Blog list/detail**
  - Examples: `/blog`, `/blog/:slug`.

**Owner:** Growth/Marketing + Frontend team.

## Authenticated client routes

- **Request form**
  - Example: `/app/request/new`.
- **Request history**
  - Example: `/app/requests` and `/app/requests/:id`.
- **Chat**
  - Example: `/app/chat` and `/app/chat/:sessionId`.

**Owner:** Client Experience team + API team.

## Partner / developer routes

- **Intranet dashboard**
  - Example: `/intranet`.
- **Data-loading tools**
  - Example: `/intranet/uploads`, `/intranet/tools/data-load`.

**Owner:** Partner Operations + Platform/API team.

Route access must be enforced both in frontend route guards and Worker API authorization checks.

---

## 8) Deployment and Binding Map

See `docs/worker-api-contracts.md` for endpoint contracts. This section standardizes infrastructure mapping.

### Cloudflare resources

- **Pages project:** `portal-pages`
  - Hosts the frontend app.
- **Worker service:** `portal-api-worker`
  - Handles all `/api/v1/*` requests.
- **D1 database binding:** `DB`
  - Primary relational datastore.
- **KV bindings:**
  - `PORTAL_CACHE_KV`
  - `PORTAL_SESSION_KV`
  - `PORTAL_IDEMPOTENCY_KV`
- **R2 bindings:**
  - `CLIENT_ASSETS_R2`
  - `PARTNER_INGEST_R2`
  - `EXPORT_R2`

### Environment segmentation

Use isolated resources per environment:

- `dev`
- `staging`
- `prod`

Do not share D1/KV/R2 resources across environments. Use environment-prefixed naming and binding separation.

### Security controls

- Principle of least privilege for service tokens and API access.
- Strict CORS allowlist for frontend origins.
- JWT validation with issuer and audience pinning.
- Rate limiting per route group and role.
- Audit logging for partner/dev operations.
- Data retention policies for R2 and log stores.

