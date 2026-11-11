# Software Requirements Specification (SRS)

**Project:** AI-Powered Developer Portfolio
**Working name:** rajan-portfolio
**Document status:** Draft v0.1 — Phase 1 (architecture & planning only)
**Owner:** Project Owner (Rajan)
**Last updated:** 2026-11-11

---

## 1. Purpose

This document defines the functional and non-functional requirements for a production-oriented,
AI-powered developer portfolio. It is the contract between product expectations and the
architecture/implementation that follows in later phases.

This SRS is intentionally implementation-agnostic where possible. Application code is **out of
scope for Phase 1**; only documentation and architectural design are produced.

---

## 2. Scope

The system is a single web product composed of four capability areas:

| # | Capability | Audience | Description |
|---|------------|----------|-------------|
| C1 | Public developer portfolio | Everyone | Showcase profile, skills, projects, experience, and contact channel. |
| C2 | Floating AI portfolio assistant | Everyone | A chat assistant grounded in the portfolio knowledge base (RAG). |
| C3 | Private admin dashboard | Authenticated staff | Manage all portfolio content, users, and assistant configuration. |
| C4 | AI knowledge base management | Admin/editor | Ingest, version, index, and re-index content used by the assistant. |

### 2.1 In scope (MVP, Phase 1 documentation baseline)

- Public, SEO-aware portfolio site.
- Floating AI assistant with streaming answers and source citations.
- Admin authentication + RBAC.
- Dynamic CRUD for skills, projects, profile, experience, education, social links.
- Knowledge base ingestion (manual text + file upload) and vector search (pgvector).
- Audit logging of privileged actions.
- Docker-based local development and deployable images.

### 2.2 Out of scope (for MVP)

- Multi-tenant portfolio hosting (one portfolio owner per deployment).
- Native mobile apps.
- Real-time collaboration / simultaneous editing.
- Payments, billing, or subscriptions.
- Public user self-registration and public user accounts.
- Third-party OAuth/Social login (revisit in Phase 2).
- Fine-tuning or training custom LLMs (inference only).
- Automated content generation for the portfolio itself.

---

## 3. Definitions

| Term | Meaning |
|------|---------|
| Portfolio Owner | Person whose portfolio is published; default super-admin. |
| Visitor | Anonymous user of the public site and assistant. |
| Admin/Editor | Authenticated staff who edit content. |
| RAG | Retrieval-Augmented Generation: retrieve relevant knowledge chunks, then prompt the LLM. |
| Knowledge Document | A source of truth (markdown, PDF, URL text) stored for retrieval. |
| Knowledge Chunk | A smaller segment of a document with an embedding vector. |
| Embedding | Numeric vector representation of text used for semantic search. |
| RBAC | Role-Based Access Control. |
| Access Token | Short-lived JWT authorizing API calls. |
| Refresh Token | Long-lived rotating token stored in an HttpOnly cookie. |
| LLM Provider | External or local model API used to generate answers/embeddings. |

---

## 4. Stakeholders and personas

| Stakeholder | Goals | Pain points |
|-------------|-------|-------------|
| Portfolio Owner / Super Admin | Publish and update portfolio effortlessly; control the assistant's knowledge. | Manual HTML edits, stale content, opaque AI answers. |
| Recruiter / Visitor | Quickly understand skills and projects; ask questions naturally. | Long resumes, hard-to-search project details. |
| Editor (optional) | Maintain content without infrastructure access. | Heavy CMS, unclear permissions. |
| Developer/Maintainer | Operate, secure, and evolve the system. | Hidden coupling, poor observability, unsafe defaults. |

---

## 5. Functional requirements

Requirements use **MoSCoW** priority and stable IDs (`FR-<area>-<n>`). Acceptance criteria are
summarized and detailed further in the API and test plans in later phases.

### 5.1 Public portfolio (FR-PUB)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-PUB-01 | Display a profile section (name, headline, bio, avatar, location, availability). | Must |
| FR-PUB-02 | Display a filterable/searchable list of projects with detail pages. | Must |
| FR-PUB-03 | Display skills grouped by category with proficiency indicators. | Must |
| FR-PUB-04 | Display work experience and education timelines. | Must |
| FR-PUB-05 | Display social/professional links (GitHub, LinkedIn, email, etc.). | Must |
| FR-PUB-06 | Provide a contact form that stores messages and notifies the owner. | Must |
| FR-PUB-07 | Serve responsive layouts for mobile, tablet, and desktop. | Must |
| FR-PUB-08 | Provide SEO metadata, Open Graph tags, sitemap, and robots configuration. | Should |
| FR-PUB-09 | Support light/dark theme with persisted preference. | Should |
| FR-PUB-10 | Show a resume/CV download link (hosted asset). | Should |
| FR-PUB-11 | Support project filtering by tag/technology and sorting. | Should |
| FR-PUB-12 | Gracefully degrade when the AI assistant or backend is unavailable. | Must |

### 5.2 AI portfolio assistant (FR-AI)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-AI-01 | Provide a floating chat launcher available on all public pages. | Must |
| FR-AI-02 | Accept natural-language questions about the owner's skills, projects, experience, and background. | Must |
| FR-AI-03 | Answer using only retrieved portfolio knowledge (RAG); do not fabricate. | Must |
| FR-AI-04 | Stream responses token-by-token to the UI. | Must |
| FR-AI-05 | Cite the knowledge sources/documents used in an answer. | Should |
| FR-AI-06 | Maintain short-term conversational context within a session. | Must |
| FR-AI-07 | Refuse or redirect on out-of-scope, unsafe, or abusive requests. | Must |
| FR-AI-08 | Persist chat transcripts for moderation and analytics (configurable retention). | Should |
| FR-AI-09 | Rate-limit and throttle abusive usage per IP/session. | Must |
| FR-AI-10 | Allow the admin to configure system prompt, model, temperature, and max tokens. | Should |
| FR-AI-11 | Handle LLM provider failure/timeout with a user-safe fallback message. | Must |
| FR-AI-12 | Support semantic search over the knowledge base independent of chat. | Could |

### 5.3 Admin dashboard (FR-ADM)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-ADM-01 | Authenticate admins with email + password. | Must |
| FR-ADM-02 | Enforce role-based access on every admin route and API endpoint. | Must |
| FR-ADM-03 | Manage profile, skills, projects, experience, education, and links (CRUD). | Must |
| FR-ADM-04 | Support draft/published states and ordered display. | Must |
| FR-ADM-05 | Upload and manage media assets (images, resume). | Must |
| FR-ADM-06 | Manage knowledge documents and trigger re-indexing. | Must |
| FR-ADM-07 | Preview content before publishing. | Should |
| FR-ADM-08 | Manage users and roles (super-admin only). | Must |
| FR-ADM-09 | View contact messages with read/unread status. | Must |
| FR-ADM-10 | Display audit log of privileged actions. | Should |
| FR-ADM-11 | Show assistant usage analytics (questions, latency, token usage) without exposing secrets. | Could |
| FR-ADM-12 | Prevent locking out the last super-admin. | Must |

### 5.4 Knowledge base & RAG (FR-KB)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-KB-01 | Create knowledge documents from pasted text/markdown. | Must |
| FR-KB-02 | Upload supported files (e.g., `.md`, `.txt`, `.pdf`) for ingestion. | Must |
| FR-KB-03 | Split documents into configurable chunks with overlap. | Must |
| FR-KB-04 | Generate and store embeddings per chunk in pgvector. | Must |
| FR-KB-05 | Re-index a single document or the whole corpus on demand. | Must |
| FR-KB-06 | Soft-delete/unpublish documents so they stop influencing answers. | Must |
| FR-KB-07 | Show indexing status, chunk counts, and errors per document. | Should |
| FR-KB-08 | Use cosine similarity and a configurable top-K / similarity threshold. | Must |
| FR-KB-09 | Support metadata filtering (e.g., tag = "projects"). | Should |
| FR-KB-10 | Record embedding model/version per chunk for re-index compatibility. | Should |

### 5.5 Authentication, users, and audit (FR-AUTH)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-AUTH-01 | Issue short-lived JWT access tokens and rotating refresh tokens. | Must |
| FR-AUTH-02 | Store refresh tokens in `HttpOnly`, `Secure`, `SameSite` cookies. | Must |
| FR-AUTH-03 | Support logout that revokes the active refresh token. | Must |
| FR-AUTH-04 | Support password change and forced re-authentication. | Must |
| FR-AUTH-05 | Lock accounts after repeated failed logins (configurable). | Should |
| FR-AUTH-06 | Provide a bootstrap/super-admin creation mechanism (no public registration). | Must |
| FR-AUTH-07 | Log authentication events and privileged CRUD to an audit log. | Must |
| FR-AUTH-08 | Support token revocation (per-session and global). | Should |

---

## 6. Non-functional requirements

### 6.1 Performance

| ID | Requirement |
|----|-------------|
| NFR-PERF-01 | Public pages: LCP < 2.5s on a mid-tier mobile connection; TTFB < 800ms for cached content. |
| NFR-PERF-02 | Read APIs: p95 < 300ms (excluding LLM/embedding calls). |
| NFR-PERF-03 | Assistant first token: p95 < 3s under normal provider conditions. |
| NFR-PERF-04 | Semantic search: p95 < 250ms for corpora up to 100k chunks. |
| NFR-PERF-05 | Frontend bundle follows route-level code splitting; initial JS budget < 200KB gzipped (target). |

### 6.2 Availability & reliability

| ID | Requirement |
|----|-------------|
| NFR-AVL-01 | Target 99.5% monthly availability for the public site. |
| NFR-AVL-02 | LLM/embedding provider failures must not take down the portfolio. |
| NFR-AVL-03 | All external calls have timeouts, bounded retries, and circuit-breaking. |
| NFR-AVL-04 | Health and readiness endpoints exposed for orchestration. |
| NFR-AVL-05 | Database backups with defined RPO ≤ 24h and RTO ≤ 4h for MVP. |

### 6.3 Security

| ID | Requirement |
|----|-------------|
| NFR-SEC-01 | All traffic over HTTPS; HSTS enabled in production. |
| NFR-SEC-02 | Passwords hashed with Argon2id (fallback bcrypt cost ≥ 12). |
| NFR-SEC-03 | Strict CORS allowlist; no wildcard in production. |
| NFR-SEC-04 | Input validation on every boundary; output encoding to prevent XSS. |
| NFR-SEC-05 | Secrets only via environment/secret manager; never committed. |
| NFR-SEC-06 | Least-privilege RBAC enforced server-side. |
| NFR-SEC-07 | Structured audit logging for auth and privileged mutations. |
| NFR-SEC-08 | Dependency and container vulnerability scanning in CI. |
| NFR-SEC-09 | Prompt-injection and data-exfiltration mitigations for the assistant. |
| NFR-SEC-10 | Rate limiting on auth, contact, and AI endpoints. |

### 6.4 Maintainability & quality

| ID | Requirement |
|----|-------------|
| NFR-MNT-01 | Type checking (TypeScript strict, Python mypy/pyright) passes in CI. |
| NFR-MNT-02 | Lint/format (ESLint/Prettier, Ruff/Black) enforced in CI. |
| NFR-MNT-03 | Backend unit test coverage ≥ 80% for domain/critical paths. |
| NFR-MNT-04 | All DB changes via Alembic migrations; no manual schema drift. |
| NFR-MNT-05 | API documented via OpenAPI with request/response examples. |
| NFR-MNT-06 | Architecture Decision Records (ADRs) for significant choices. |

### 6.5 Observability

| ID | Requirement |
|----|-------------|
| NFR-OBS-01 | Structured JSON logs with correlation/request IDs. |
| NFR-OBS-02 | Health, readiness, and metrics endpoints. |
| NFR-OBS-03 | Error tracking integration (e.g., Sentry) for frontend and backend. |
| NFR-OBS-04 | LLM cost/token/latency metrics tracked per request (no PII in logs). |

### 6.6 Usability & accessibility

| ID | Requirement |
|----|-------------|
| NFR-UX-01 | WCAG 2.1 AA for public site and assistant (keyboard, focus, contrast, ARIA). |
| NFR-UX-02 | Assistant usable with keyboard and screen readers; streaming announced politely. |
| NFR-UX-03 | Responsive from 320px to 4K. |
| NFR-UX-04 | Clear loading, empty, and error states everywhere. |

### 6.7 Compatibility

| ID | Requirement |
|----|-------------|
| NFR-COMP-01 | Latest 2 versions of Chrome, Firefox, Safari, Edge. |
| NFR-COMP-02 | Backend/DB run via Docker on Linux containers; dev supported on Windows/macOS. |
| NFR-COMP-03 | PostgreSQL ≥ 15 with pgvector ≥ 0.7. |

---

## 7. Assumptions

- Single portfolio owner per deployment; content is curated by a small trusted admin group.
- An external LLM API (or self-hosted inference) will be selected before Phase 3.
- The frontend is a SPA; SEO-critical content is either prerendered or served with metadata at the edge.
- Admin dashboard is served from the same SPA behind an authenticated route (or a separate build later).
- English-only UI for MVP.

---

## 8. Constraints

- Stack is fixed: React + Vite + TypeScript; Python + FastAPI; PostgreSQL + pgvector; SQLAlchemy + Alembic; Docker.
- No application implementation code during Phase 1.
- AI answers must be grounded; hallucination is treated as a correctness defect.
- Budget for LLM usage must remain controllable via limits and caching.

---

## 9. Acceptance criteria (Phase 1)

Phase 1 is complete when, and only when:

1. SRS, architecture, ER diagram, API spec, auth/RBAC design, repository structure, security
   requirements, and milestones are documented under `docs/`.
2. Mermaid diagrams render without syntax errors.
3. All open questions are listed and answered or explicitly deferred by the owner.
4. Owner approves the documentation baseline before implementation begins.

---

## 10. Open questions

See [`open-questions.md`](./open-questions.md). Approval of this SRS is blocked on the
must-answer items listed there.

---

## 11. Traceability

| Capability | Requirements | Architecture | API | Data |
|------------|--------------|--------------|-----|------|
| Public portfolio | FR-PUB | `architecture.md` §4 | `api.md` §6 | `database.md` §5 |
| AI assistant | FR-AI | `architecture.md` §5 | `api.md` §9 | `database.md` §7 |
| Admin dashboard | FR-ADM | `architecture.md` §6 | `api.md` §7 | `database.md` §4–8 |
| Knowledge base | FR-KB | `architecture.md` §5 | `api.md` §8 | `database.md` §7 |
| Auth & RBAC | FR-AUTH | `authentication-rbac.md` | `api.md` §5 | `database.md` §4 |
