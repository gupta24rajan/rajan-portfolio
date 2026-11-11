# API Specification

**Project:** AI-Powered Developer Portfolio
**Base URL (dev):** `http://localhost:8000`
**Base path:** `/api/v1`
**Document status:** Draft v0.1 — Phase 1
**Last updated:** 2026-11-11

> The backend is OpenAPI-first (FastAPI auto-generates `/openapi.json` and `/docs`). This
> document is the design contract; generated schemas are authoritative once implemented.

---

## 1. Conventions

### 1.1 Protocol & format

- JSON (`application/json; charset=utf-8`) unless noted; chat streaming uses SSE.
- UTF-8; timestamps ISO 8601 UTC (`2026-11-11T12:00:00Z`).
- IDs are UUID strings.
- Field names `snake_case` on the wire (consistent backend contract).

### 1.2 Versioning

- URI versioning: `/api/v1`.
- Breaking changes require `/api/v2`; additive changes (new optional fields) are non-breaking.

### 1.3 Authentication

- Protected endpoints require `Authorization: Bearer <access_token>`.
- Access token: JWT, short-lived (default 15 min).
- Refresh: `POST /api/v1/auth/refresh` using the `HttpOnly` refresh cookie.
- Public endpoints are explicitly marked **Public** below.

### 1.4 Error envelope

All non-2xx responses return:

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed.",
    "details": [{ "field": "email", "issue": "invalid format" }],
    "request_id": "01J9Z8YQ2K3M4N5P6Q7R8S9T0V"
  }
}
```

| HTTP | `code` (examples) | Meaning |
|------|-------------------|---------|
| 400 | `bad_request` | Malformed request. |
| 401 | `unauthenticated`, `token_expired` | Missing/invalid credentials. |
| 403 | `forbidden`, `insufficient_permissions` | Authenticated but not allowed. |
| 404 | `not_found` | Resource missing. |
| 409 | `conflict`, `slug_exists` | State conflict. |
| 422 | `validation_error` | Schema/semantic validation failed. |
| 429 | `rate_limited` | Quota exceeded; includes `Retry-After`. |
| 502/503 | `upstream_error`, `assistant_unavailable` | External provider/availability failure. |
| 500 | `internal_error` | Unexpected failure (no internals leaked). |

### 1.5 Pagination, filtering, sorting

List endpoints accept:

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `page` | int ≥ 1 | 1 | Page number. |
| `page_size` | int 1–100 | 20 | Items per page. |
| `sort` | string | endpoint default | e.g., `-created_at`, `sort_order`. |
| `q` | string | — | Optional text search. |

Response envelope:

```json
{
  "items": [],
  "page": 1,
  "page_size": 20,
  "total": 0,
  "total_pages": 0
}
```

### 1.6 Idempotency & concurrency

- `POST` create endpoints may honor `Idempotency-Key` where duplicates are harmful (contact,
  chat session creation).
- Content update endpoints accept optional `If-Match`/`version` for optimistic concurrency;
  conflicts return `409 conflict`.

### 1.7 Rate limits (baseline)

| Scope | Limit |
|-------|-------|
| `POST /auth/login` | 5/min per IP + per account lockout policy |
| `POST /contact` | 3/hour per IP |
| `POST /chat/*/messages` | 20/min per session, 60/hour per IP |
| General public reads | 120/min per IP |
| Authenticated admin | 300/min per user |

Limits return `429` with `Retry-After`.

---

## 2. Endpoint index

| Area | Endpoints |
|------|-----------|
| Health | `GET /health`, `GET /health/ready` |
| Auth | `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`, `POST /auth/change-password` |
| Users | `GET/POST /users`, `GET/PATCH/DELETE /users/{id}` |
| Roles | `GET /roles`, `GET /permissions`, `PUT /users/{id}/roles` |
| Profile | `GET/PUT /profile`, `GET/POST /profile/links`, `PATCH/DELETE /profile/links/{id}` |
| Skills | `GET/POST /skill-categories`, `PATCH/DELETE /skill-categories/{id}`, `GET/POST /skills`, `PATCH/DELETE /skills/{id}` |
| Projects | `GET/POST /projects`, `GET/PATCH/DELETE /projects/{id}`, `GET /projects/slug/{slug}`, tags |
| Experience | `GET/POST /experiences`, `PATCH/DELETE /experiences/{id}` |
| Education | `GET/POST /education`, `PATCH/DELETE /education/{id}` |
| Media | `POST /media`, `GET /media`, `DELETE /media/{id}` |
| Knowledge | `GET/POST /knowledge/documents`, `GET/PATCH/DELETE /knowledge/documents/{id}`, `POST /knowledge/documents/{id}/reindex`, `POST /knowledge/reindex`, `GET /knowledge/search` |
| Chat | `POST /chat/sessions`, `GET /chat/sessions/{id}/messages`, `POST /chat/sessions/{id}/messages`, `DELETE /chat/sessions/{id}` |
| Contact | `POST /contact`, `GET /contact`, `PATCH /contact/{id}` |
| Settings | `GET/PUT /settings` |
| Audit | `GET /audit-logs` |

Legend: **Public** = no auth; **Auth** = any admin role; **Perm** = specific permission.

---

## 3. Health

### `GET /health` — Public

Liveness. Returns `200` if the process is running.

```json
{ "status": "ok", "version": "0.1.0", "time": "2026-11-11T12:00:00Z" }
```

### `GET /health/ready` — Public (internal use)

Readiness. Checks DB connectivity (and optionally Redis/LLM config). Returns `503` when not
ready.

```json
{ "status": "ready", "checks": { "database": "ok", "redis": "ok" } }
```

---

## 4. Authentication

### `POST /auth/login` — Public

```json
{ "email": "admin@example.com", "password": "••••••••" }
```

Response `200` sets refresh cookie and returns access token:

```json
{
  "access_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 900,
  "user": { "id": "…", "email": "admin@example.com", "full_name": "Rajan", "roles": ["admin"] }
}
```

Errors: `401 unauthenticated`, `403 forbidden` (inactive), `429 rate_limited` (lockout).

### `POST /auth/refresh` — Public (cookie)

Uses `refresh_token` HttpOnly cookie. Rotates the token (old one revoked) and returns a new
access token. Detects reuse of a revoked token and revokes the whole chain.

Response `200` same shape as login (without password). Errors: `401 token_expired`,
`401 invalid_refresh`, `401 token_reuse_detected`.

### `POST /auth/logout` — Auth

Revokes the current refresh token and clears the cookie. `204 No Content`.

### `GET /auth/me` — Auth

Returns the current principal:

```json
{
  "id": "…",
  "email": "admin@example.com",
  "full_name": "Rajan",
  "is_active": true,
  "roles": ["admin"],
  "permissions": ["project:read", "project:write", "knowledge:write"]
}
```

### `POST /auth/change-password` — Auth

```json
{ "current_password": "…", "new_password": "…" }
```

Revokes all other refresh tokens. `204 No Content`. Errors: `422 weak_password`,
`401 invalid_credentials`.

---

## 5. Users, roles, permissions

> All endpoints in this section require `user:read` / `user:write` permissions. Only
> `super_admin` may grant/revoke `super_admin`.

### `GET /users` — Perm `user:read`

Query: `q`, `is_active`, `role`, pagination.

### `POST /users` — Perm `user:write`

```json
{ "email": "editor@example.com", "full_name": "Editor", "password": "…", "roles": ["editor"] }
```

`201 Created`. Errors: `409 email_exists`, `422 weak_password`.

### `GET /users/{id}` — Perm `user:read`

Returns user with roles and effective permissions.

### `PATCH /users/{id}` — Perm `user:write`

Editable: `full_name`, `is_active`, `roles`. Cannot deactivate/demote the last active
`super_admin` (`409 last_super_admin`).

### `DELETE /users/{id}` — Perm `user:write`

Soft-deactivates or hard-deletes (configurable). Cannot delete self or last super-admin.

### `GET /roles` · `GET /permissions` — Perm `role:read`

Lists seeded roles/permissions.

### `PUT /users/{id}/roles` — Perm `role:assign`

```json
{ "roles": ["editor", "viewer"] }
```

Replaces role set. `204 No Content`.

---

## 6. Public portfolio content

### `GET /profile` — Public

Returns published profile:

```json
{
  "display_name": "Rajan",
  "headline": "Full-Stack Developer",
  "bio": "…",
  "location": "…",
  "availability": "open_to_work",
  "avatar_url": "https://…",
  "resume_url": "https://…",
  "social_links": [{ "id": "…", "label": "GitHub", "url": "https://github.com/…", "icon": "github" }]
}
```

### `PUT /profile` — Perm `profile:write`

Updates the singleton profile. `meta` JSONB accepts known keys only.

### `GET /profile/links` — Public · `POST /profile/links` — Perm `profile:write`

### `PATCH /profile/links/{id}` · `DELETE /profile/links/{id}` — Perm `profile:write`

### `GET /skill-categories` — Public · `GET /skills` — Public

Public list returns only `is_published` items ordered by `sort_order`.

```json
{
  "items": [
    { "id": "…", "name": "Languages", "skills": [{ "id": "…", "name": "Python", "proficiency": 90 }] }
  ]
}
```

Admin (`skill:read`) may include unpublished items via `?include_unpublished=true`.

### `POST /skill-categories` · `PATCH/DELETE /skill-categories/{id}` — Perm `skill:write`

### `POST /skills` · `PATCH/DELETE /skills/{id}` — Perm `skill:write`

`DELETE` is soft-deactivate when referenced (`409 in_use`) unless `?force=true` (super-admin).

### `GET /projects` — Public

Query: `q`, `tag`, `skill`, `featured`, `status`, `page`, `page_size`, `sort`.

Public response contains only published, non-deleted projects and summary fields.

### `GET /projects/slug/{slug}` — Public

Full project detail, 404 if unpublished/deleted (unless authenticated with `project:read`).

### `POST /projects` — Perm `project:write`

```json
{
  "title": "AI Portfolio Assistant",
  "slug": "ai-portfolio-assistant",
  "summary": "RAG-grounded assistant…",
  "description": "## Overview\n…",
  "repo_url": "https://github.com/…",
  "live_url": "https://…",
  "tags": ["ai", "rag"],
  "skill_ids": ["…"],
  "is_featured": true,
  "is_published": false
}
```

`201 Created`. Errors: `409 slug_exists`; invalid URLs → `422`.

### `PATCH /projects/{id}` · `DELETE /projects/{id}` — Perm `project:write`

`DELETE` soft-deletes by default; `?hard=true` (super-admin) hard-deletes.

### Tags

`GET /tags` — Public · `POST/PATCH/DELETE /tags/{id}` — Perm `project:write`

### `GET /experiences` — Public · CRUD — Perm `content:write`

### `GET /education` — Public · CRUD — Perm `content:write`

---

## 7. Media

### `POST /media` — Perm `media:write`

`multipart/form-data` with `file`. Allowed types: `image/png`, `image/jpeg`, `image/webp`,
`image/svg+xml`, `application/pdf` (resume). Max size 10 MB (configurable). Returns asset
metadata + signed/derived URLs.

```json
{
  "id": "…", "storage_key": "media/2026/11/uuid.webp",
  "mime_type": "image/webp", "size_bytes": 83421,
  "url": "https://cdn.example.com/media/…"
}
```

### `GET /media` — Perm `media:read` · `DELETE /media/{id}` — Perm `media:write`

Prevent deleting assets referenced by content (`409 in_use`) unless `?force=true`.

---

## 8. Knowledge base

### `GET /knowledge/documents` — Perm `knowledge:read`

Query: `q`, `status`, `tag`, pagination. Excludes soft-deleted by default (`?include_deleted`).

### `POST /knowledge/documents` — Perm `knowledge:write`

Two creation modes:

**a) Manual text**

```json
{
  "title": "About my AI work",
  "source_type": "manual",
  "content": "…# markdown …",
  "tags": ["projects", "ai"]
}
```

**b) File upload** (`multipart/form-data`): `file` + `title` + `tags`. Extracted text becomes
`content`.

Both return `202 Accepted` with `status: "queued"` and a document id. Indexing is asynchronous.

### `GET /knowledge/documents/{id}` — Perm `knowledge:read`

Returns document metadata + indexing status/chunk count/last error (redacted).

### `PATCH /knowledge/documents/{id}` — Perm `knowledge:write`

Update title/content/tags. Content change marks document `stale`; call reindex to refresh.

### `DELETE /knowledge/documents/{id}` — Perm `knowledge:write`

Soft-deletes and removes from retrieval. Chunks are purged by the worker (or immediately via
`?purge=true`).

### `POST /knowledge/documents/{id}/reindex` — Perm `knowledge:write`

Enqueues single-document re-index. `202 Accepted`.

### `POST /knowledge/reindex` — Perm `knowledge:write`

Enqueues full-corpus re-index (e.g., after embedding model change). `202 Accepted` with job id.

### `GET /knowledge/search` — Perm `knowledge:read`

Debug/verification endpoint:

```
GET /knowledge/search?q=python+experience&top_k=5&tag=projects
```

```json
{
  "query": "python experience",
  "results": [
    {
      "document_id": "…", "chunk_id": "…",
      "title": "About my AI work", "score": 0.83,
      "content": "…", "metadata": { "heading": "Experience" }
    }
  ]
}
```

---

## 9. AI assistant / chat

### `POST /chat/sessions` — Public (rate-limited)

Creates an anonymous chat session. Honors `Idempotency-Key`.

```json
{ "context": "portfolio" }
```

```json
{ "session_id": "…", "session_token": "opaque-token", "expires_at": "2026-11-11T13:00:00Z" }
```

The client stores the opaque token (memory/sessionStorage) and sends it with each message.
Never trust a client-provided session id without its token.

### `POST /chat/sessions/{session_id}/messages` — Public (rate-limited)

Send a user question and receive a streamed answer.

Request:

```json
{ "message": "What backend technologies does Rajan use?" }
```

Response: `200` with `Content-Type: text/event-stream`.

```
event: token
data: {"delta":"Rajan"}

event: token
data: {"delta":" primarily"}

event: sources
data: {"sources":[{"document_id":"…","title":"Skills","chunk_id":"…","score":0.81}]}

event: done
data: {"message_id":"…","usage":{"prompt_tokens":812,"completion_tokens":143},"latency_ms":2140}
```

Error events (mid-stream) use `event: error` with the standard error shape, then close.

Behavior:

- Short-term context = last N messages within token budget.
- If no relevant context: grounded refusal (`I don't have that information…`) rather than
  fabrication; still returns `done` with `refusal_reason`.
- Provider timeout → `event: error` with `assistant_unavailable` and a safe message.
- Respect `429` on quota; include `Retry-After`.

### `GET /chat/sessions/{session_id}/messages` — Public (token)

Returns transcript for the session (paginated). 404 if token invalid/expired.

### `DELETE /chat/sessions/{session_id}` — Public (token)

Closes the session and (optionally) schedules transcript purge. `204 No Content`.

---

## 10. Contact

### `POST /contact` — Public (rate-limited)

```json
{
  "name": "Jane Recruiter",
  "email": "jane@example.com",
  "subject": "Opportunity",
  "message": "…",
  "website": ""
}
```

`website` is a honeypot field (must be empty). Response `202 Accepted` (do not reveal spam
detection details). Notifies owner via email (best-effort).

Errors: `422 validation_error`, `429 rate_limited`.

### `GET /contact` — Perm `contact:read`

Query: `status`, pagination. Never public.

### `PATCH /contact/{id}` — Perm `contact:read`

Update `status` (`read`, `archived`, `spam`). `204 No Content`.

---

## 11. Settings & audit

### `GET /settings` — Public (safe subset) / Perm `settings:read` (full)

Public subset: SEO defaults, assistant enabled flag, assistant display name. Full config
(model, temperature, top_k, system prompt) is admin-only and must never include provider keys.

### `PUT /settings` — Perm `settings:write`

```json
{
  "assistant": {
    "enabled": true,
    "display_name": "Portfolio Assistant",
    "model": "…",
    "temperature": 0.2,
    "max_output_tokens": 512,
    "top_k": 6,
    "similarity_threshold": 0.72,
    "system_prompt": "You are a helpful assistant…"
  }
}
```

Validation enforces ranges and length limits.

### `GET /audit-logs` — Perm `audit:read`

Query: `actor_id`, `action`, `entity_type`, `from`, `to`, pagination. Newest first.

---

## 12. OpenAPI & docs

- `/openapi.json` — machine-readable spec.
- `/docs` — Swagger UI (disabled or auth-gated in production).
- `/redoc` — ReDoc (optional).

Security schemes: `bearerAuth` (JWT) and `refreshCookie` (`apiKey` in cookie).

---

## 13. Frontend contract notes

- Typed client generated from OpenAPI to keep FE/BE in sync.
- Access token held in memory (not localStorage); refresh via cookie on `401` once, then
  redirect to login.
- SSE consumed via `fetch` + `ReadableStream` (not `EventSource`) to send POST bodies and
  auth headers.
- All mutating admin calls surface the standard error envelope in a toast/inline form.

---

## 14. Open API questions

See [`open-questions.md`](./open-questions.md). Key items: public vs. authenticated chat,
transcript persistence/retention, whether visitors can submit feedback on answers, and the
chosen LLM streaming compatibility.
