# Documentation Index

**Project:** AI-Powered Developer Portfolio
**Baseline:** Phase 1 — architecture & planning (no application code)

This directory is the single source of truth for product and technical design. Start here.

---

## Phase 1 documents

| Document | Purpose |
|----------|---------|
| [SRS.md](./SRS.md) | Scope, personas, functional and non-functional requirements, acceptance criteria. |
| [architecture.md](./architecture.md) | System context, components, RAG pipeline, deployment topology, technology decisions, risks. |
| [database.md](./database.md) | Data model, Mermaid ER diagram, table specs, indexes, retention, migrations. |
| [api.md](./api.md) | REST API contract, error envelope, endpoints, SSE streaming, rate limits. |
| [authentication-rbac.md](./authentication-rbac.md) | Auth flows, token rotation, RBAC model, permission matrix, audit events. |
| [repository-structure.md](./repository-structure.md) | Target monorepo layout and conventions for frontend, backend, infra, docs, CI. |
| [security.md](./security.md) | Security requirements, OWASP/LLM mappings, secret management, testing gates. |
| [milestones.md](./milestones.md) | Phased plan with deliverables, exit criteria, dependencies, and risks. |
| [open-questions.md](./open-questions.md) | **Blocking questions and missing requirements — approval required.** |

---

## Reading order

1. **SRS** — what we are building and why.
2. **architecture + database + api + auth** — how it is built.
3. **security** — the non-negotiable constraints.
4. **repository-structure** — where code will live.
5. **milestones** — how and when it ships.
6. **open-questions** — what still needs your decision.

---

## Status

| Area | Status |
|------|--------|
| Requirements | Draft — awaiting approval |
| Architecture | Draft — awaiting approval |
| Data model | Draft — embedding dimensions pending (Q5) |
| API | Draft — chat/transcript decisions pending |
| Auth/RBAC | Draft — MFA and hosting topology pending |
| Security | Draft |
| Milestones | Draft |

**Next step:** resolve the blocking items in [open-questions.md](./open-questions.md), then
approve the baseline to begin Phase 2 (scaffolding). No implementation code will be written
before approval.

---

## Conventions

- All diagrams are authored in **Mermaid** and must render on GitHub.
- Significant decisions become **ADRs** under `docs/adr/`.
- Documents are versioned with the codebase; update them when requirements change.
