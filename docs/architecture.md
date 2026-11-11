# System Architecture

**Project:** AI-Powered Developer Portfolio
**Document status:** Draft v0.1 — Phase 1
**Last updated:** 2026-11-11

---

## 1. Architectural goals

1. **Separation of concerns** — a static-first public UI, an API layer, an async AI/RAG
   pipeline, and a relational/vector data layer.
2. **Graceful degradation** — the portfolio must render and be usable even if the LLM provider
   or vector search is down.
3. **Security by default** — closed admin surface, least-privilege RBAC, strict input and
   secret handling.
4. **Operability** — one-command local environment, containerized deployables, observable
   services.
5. **Evolvability** — provider-agnostic LLM/embeddings, versioned migrations, ADR-driven
   decisions.

---

## 2. Architecture style

- **Modular monolith** backend (FastAPI) organized by domain modules, with clearly separated
  layers: API → Services → Repositories → Models.
- **SPA frontend** (React + Vite + TypeScript) with two routing surfaces: public and admin.
- **Async post-processing** for embedding/indexing via background tasks (in-process worker for
  MVP; queue-ready interface for later scaling).
- **PostgreSQL as the single source of truth**, including vector storage via `pgvector`.
- **Stateless API instances** (JWT access tokens) so the backend can scale horizontally.

Why a modular monolith over microservices for MVP: a single portfolio has low, bursty traffic;
a monolith minimizes operational cost and cross-service failure modes while remaining internally
modular enough to split the AI pipeline out later if needed. See ADR-001 (planned).

---

## 3. System context

```mermaid
flowchart LR
    Visitor([Visitor])
    Admin([Admin / Editor])
    Owner([Portfolio Owner])

    subgraph System["AI-Powered Portfolio Platform"]
        FE["Frontend SPA<br/>React + Vite + TS"]
        API["Backend API<br/>FastAPI"]
        DB[("PostgreSQL<br/>+ pgvector")]
        Worker["Indexing Worker<br/>(background tasks)"]
    end

    LLM["LLM Provider<br/>(chat + embeddings)"]
    Mail["Email Provider<br/>(contact notifications)"]
    Storage["Object Storage<br/>(media, resumes, uploads)"]

    Visitor -->|HTTPS| FE
    Admin -->|HTTPS| FE
    Owner -->|HTTPS| FE
    FE -->|REST / JSON + SSE| API
    API -->|SQL / vector queries| DB
    API -->|enqueue indexing| Worker
    Worker -->|embed + write chunks| DB
    Worker -->|embeddings| LLM
    API -->|chat completion| LLM
    API -->|send notification| Mail
    API -->|signed URLs / read-write| Storage
```

---

## 4. Container / component diagram

```mermaid
flowchart TB
    subgraph Client["Browser"]
        PublicUI["Public Portfolio<br/>(routes, sections, SEO)"]
        Assistant["Floating AI Assistant<br/>(chat widget, streaming)"]
        AdminUI["Admin Dashboard<br/>(content, KB, users)"]
        AuthClient["Auth Client<br/>(token refresh, guards)"]
        PublicUI --- Assistant
        AdminUI --- AuthClient
    end

    subgraph Edge["Edge / Reverse Proxy"]
        Nginx["Nginx / Reverse Proxy<br/>TLS, static assets, gzip/brotli"]
    end

    subgraph Backend["Backend (FastAPI)"]
        direction TB
        MW["Middleware<br/>CORS · request-id · rate limit · error handler"]
        subgraph Modules["Domain modules"]
            AuthMod["auth"]
            UsersMod["users & RBAC"]
            ContentMod["content (profile, skills, projects, exp)"]
            KBMod["knowledge base"]
            ChatMod["assistant / chat"]
            ContactMod["contact"]
            MediaMod["media"]
            AuditMod["audit"]
        end
        AISvc["AI service<br/>prompting · retrieval · streaming"]
        EmbSvc["Embedding service<br/>chunking · vectors"]
        Repo["Repositories<br/>SQLAlchemy async"]
    end

    subgraph Data["Data & Infra"]
        PG[("PostgreSQL + pgvector")]
        Redis[("Redis (optional)<br/>rate limit · cache · queue")]
        Obj[("Object Storage")]
    end

    subgraph External["External Providers"]
        LLM["LLM / Embeddings API"]
        Mail["Email"]
        Obs["Error tracking / metrics"]
    end

    PublicUI --> Nginx
    Assistant --> Nginx
    AdminUI --> Nginx
    Nginx --> MW
    MW --> Modules
    ChatMod --> AISvc
    KBMod --> EmbSvc
    AISvc --> Repo
    EmbSvc --> Repo
    Modules --> Repo
    Repo --> PG
    MW --> Redis
    AISvc --> LLM
    EmbSvc --> LLM
    ContentMod --> Obj
    MediaMod --> Obj
    ContactMod --> Mail
    Backend --> Obs
```

---

## 5. AI / RAG pipeline

### 5.1 Ingestion (write path)

```mermaid
sequenceDiagram
    autonumber
    participant Admin
    participant API as Backend API
    participant Store as PostgreSQL
    participant Worker as Indexing Worker
    participant LLM as Embedding Provider

    Admin->>API: Create/upload knowledge document
    API->>Store: Persist document (status=queued)
    API-->>Admin: 202 Accepted (document id)
    API->>Worker: Enqueue index job
    Worker->>Store: Load document
    Worker->>Worker: Normalize + chunk (size, overlap)
    Worker->>LLM: Generate embeddings (batch)
    LLM-->>Worker: Vectors
    Worker->>Store: Replace chunks + embeddings (status=indexed)
    Worker-->>API: Update job status/errors
```

### 5.2 Retrieval + answer (read path)

```mermaid
sequenceDiagram
    autonumber
    participant V as Visitor
    participant FE as Chat Widget
    participant API as Backend API
    participant Store as PostgreSQL + pgvector
    participant LLM as LLM Provider

    V->>FE: Ask question
    FE->>API: POST /chat/sessions/{id}/messages (SSE)
    API->>API: Validate, rate-limit, moderation checks
    API->>LLM: Embed query
    LLM-->>API: Query vector
    API->>Store: Cosine top-K search (threshold, filters)
    Store-->>API: Relevant chunks + metadata
    API->>API: Build grounded prompt (context + guardrails)
    API->>LLM: Chat completion (stream)
    LLM-->>API: Token stream
    API-->>FE: SSE tokens + citations
    API->>Store: Persist message, sources, usage metrics
    FE-->>V: Rendered answer with sources
```

**Grounding rules**

- If retrieval yields no chunk above the similarity threshold, return a safe "I don't have that
  information" response; do not call the LLM with an empty context (configurable exception for
  small-talk with a strict system prompt).
- Every answer must include at least one source when factual claims are made.
- Retrieved content is treated as **data, not instructions** (prompt-injection defense).
- Context window is bounded; top-K and token budget are configurable.

---

## 6. Request lifecycle (admin write example)

```mermaid
sequenceDiagram
    participant A as Admin UI
    participant API as Backend API
    participant Auth as Auth/RBAC
    participant Svc as Content Service
    participant DB as PostgreSQL
    participant Aud as Audit Log

    A->>API: PUT /api/v1/projects/{id} (Bearer access token)
    API->>Auth: Verify JWT + required permission
    Auth-->>API: Principal + roles/permissions
    API->>Svc: Validate payload (Pydantic)
    Svc->>DB: Transactional update
    DB-->>Svc: Updated row
    Svc->>Aud: Record actor, action, before/after (redacted)
    Svc-->>API: DTO
    API-->>A: 200 OK
```

---

## 7. Layer responsibilities

| Layer | Responsibility | Must not |
|-------|----------------|----------|
| Routers (API) | HTTP concerns, dependency injection, status codes, OpenAPI metadata | Contain business logic or direct SQL |
| Services | Business rules, authorization checks, transactions, orchestration | Know about HTTP details |
| Repositories | Data access, query composition, persistence | Enforce business rules |
| Models | SQLAlchemy ORM definitions + DB constraints | Contain transport logic |
| Schemas | Pydantic request/response contracts, validation | Contain persistence logic |
| Workers | Async ingestion/embedding, retries, idempotency | Mutate unrelated domains |
| AI service | Retrieval, prompt construction, provider abstraction | Access DB outside repositories |

---

## 8. Deployment topology

### 8.1 Local / development (Docker Compose)

```mermaid
flowchart LR
    Dev([Developer])
    Dev -->|http://localhost:5173| FE["frontend<br/>Vite dev server"]
    Dev -->|http://localhost:8000/docs| API["backend<br/>uvicorn --reload"]
    FE -->|/api proxy| API
    API --> PG[("postgres:pgvector")]
    API --> Redis[("redis")]
    Adminer["adminer / pgadmin (optional)"] --> PG
```

### 8.2 Production (baseline)

```mermaid
flowchart TB
    Internet([Internet]) -->|HTTPS 443| Proxy["Reverse proxy / load balancer<br/>TLS termination"]
    Proxy -->|/| Static["Static frontend build<br/>(CDN or nginx)"]
    Proxy -->|/api,/chat stream| API1["backend replica 1"]
    Proxy -->|/api,/chat stream| API2["backend replica 2"]
    API1 --> PG[("Managed PostgreSQL<br/>+ pgvector, backups")]
    API2 --> PG
    API1 --> Redis[("Managed Redis")]
    API2 --> Redis
    API1 --> Obj[("Object storage / CDN")]
    API2 --> Obj
    API1 --> Obs["Logs / metrics / traces"]
    API2 --> Obs
```

**Notes**

- SSE requires proxy buffering disabled for `/api/v1/chat/*` and appropriate timeouts.
- Embedding/indexing either runs in a dedicated worker process/container or as a separate
  service; the interface is queue-based to allow extraction later.
- The static frontend is immutable-build + long cache; API is versioned under `/api/v1`.

---

## 9. Technology decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Frontend | React + Vite + TypeScript | Fast DX, typed contracts, simple SPA hosting. |
| Backend | Python + FastAPI | Async I/O, OpenAPI-first, strong typing with Pydantic. |
| ORM | SQLAlchemy 2.x (async) + Alembic | Mature, migration-friendly, async support. |
| Database | PostgreSQL 15+ | Reliable relational model + pgvector in one store. |
| Vectors | pgvector | Avoids a second datastore at MVP scale. |
| Auth | JWT access + rotating refresh cookie | Stateless API, revocable sessions. |
| Hashing | Argon2id | Memory-hard, modern default. |
| Streaming | Server-Sent Events | Simple over HTTP, proxy-friendly, good browser support. |
| Containers | Docker + Compose | Reproducible dev and deployable images. |
| Config | Environment variables + `.env` (never committed) | Twelve-factor friendly. |

---

## 10. Cross-cutting concerns

- **Configuration**: single typed settings object (`pydantic-settings`), validated at startup,
  fail-fast on missing required secrets.
- **Errors**: consistent error envelope (`code`, `message`, `details`, `request_id`); no stack
  traces or secrets leaked to clients.
- **IDs**: UUIDv7 (time-ordered) for primary keys to aid indexing.
- **Concurrency**: optimistic concurrency via `updated_at`/version where editing collisions
  matter (admin content).
- **Time**: store UTC (`timestamptz`); render in client timezone.
- **Caching**: HTTP cache headers for public read endpoints; optional Redis caching for
  expensive aggregate queries.
- **Rate limiting**: per-IP for public endpoints, per-session for AI, per-account for auth.
- **Observability**: request IDs propagated FE→BE→logs; metrics for latency, error rate, LLM
  tokens/cost; health/readiness separate from liveness.

---

## 11. Risks and mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| LLM hallucination | Reputation/incorrect info | Strict grounding, citations, refusal path, eval set. |
| Prompt injection via KB/docs | Data leak or misuse | Treat retrieved text as data, sanitize, allowlist tools, no secrets in context. |
| Provider outage / rate limits | Assistant unavailable | Timeouts, retries, fallback message, provider abstraction. |
| Cost overrun | Budget | Token budgets, per-session limits, caching, usage metrics, budget alerts. |
| Vector table growth | Slow queries | Index strategy (HNSW/IVFFlat), chunk pruning, re-index lifecycle. |
| Admin credential compromise | Full content control | Argon2, lockout, rotation, audit, MFA (Phase 2 candidate). |
| SEO with SPA | Poor indexing | Metadata/prerender strategy, sitemap, semantic HTML. |
| Contact form abuse | Spam | Rate limit, honeypot/CAPTCHA option, server-side validation. |

---

## 12. Future evolution (post-MVP)

- Extract AI/RAG into a dedicated service if load or model diversity grows.
- Add MFA (TOTP/WebAuthn) for admins.
- Queue-backed worker (Celery/RQ/Arq) for robust indexing and retries.
- Content versioning/history and scheduled publishing.
- Multi-language portfolio and assistant.
- Analytics dashboard for assistant quality (feedback thumbs, evaluation scores).
- CDN/edge rendering for SEO-critical routes.
