# Open Questions & Missing Requirements

**Project:** AI-Powered Developer Portfolio
**Document status:** Draft v0.1 — Phase 1
**Last updated:** 2026-11-11

These are requirements that are either unspecified, ambiguous, or would materially change the
architecture. **Phase 1 approval is blocked on the "Must answer" (B) items below.** Each item
lists options, a recommendation, and the impact if left unresolved.

Legend: **B = blocking**, **S = should resolve before the relevant phase**, **D = can defer**.

---

## A. Product scope

### Q1 (B) — What exactly is public vs. private?
The SRS assumes: public portfolio + anonymous assistant; everything else admin-only.
- **Confirm** there are no public user accounts and no public sign-up.
- **Confirm** whether the assistant transcript is visible to the visitor only (session token) or
  to no one after the session ends.

**Impact:** auth model, data retention, privacy notice.

### Q2 (B) — Is the admin dashboard part of the same React SPA or a separate app?
- Option A: One SPA, admin routes code-split behind auth. (Recommended for MVP.)
- Option B: Separate admin build/subdomain.

**Impact:** repository layout, routing, CSP, deployment, auth cookie scope.

### Q3 (S) — Which portfolio sections are in MVP?
Currently assumed: profile, skills, projects, experience, education, social links, contact,
resume download.
- Confirm education and certifications are required (certifications are **not** in the current
  ERD).
- Is a blog/articles section required? (Currently out of scope.)

**Impact:** data model, admin CRUD scope, milestones.

### Q4 (D) — Multi-language support?
Assumed English-only for MVP. Confirm if i18n is required at launch.

**Impact:** schema (translated fields), routing, content workflow.

---

## B. AI / RAG

### Q5 (B) — Which LLM and embedding providers?
- Chat provider + embedding provider (may differ).
- Hosted API (e.g., OpenAI/Anthropic/etc.) vs self-hosted (e.g., local model).
- **Embedding model determines vector dimensions** (default assumed 1536). Changing later forces
  a full re-index.

**Impact:** env/config, `knowledge_chunks.embedding` dimension, cost, latency, security review.

### Q6 (B) — Grounding/refusal policy
Confirm the behavior when no relevant knowledge is found:
- Option A: Strict grounded refusal only ("I don't have that information"). (Recommended.)
- Option B: Allow limited general small-talk with strict guardrails.

**Impact:** prompt design, evals, UX.

### Q7 (S) — Chat transcript persistence & retention
- Option A: Persist for quality/analytics, default 90-day retention. (Recommended.)
- Option B: Do not persist assistant messages beyond the live session.
- Confirm whether visitors must be told and whether they can request deletion.

**Impact:** data model usage, privacy notice, purge jobs, storage.

### Q8 (S) — Answer feedback
Should visitors be able to rate answers (thumbs up/down)? Not in current ERD/API.

**Impact:** new table/endpoints, analytics.

### Q9 (S) — Knowledge ingestion formats
Supported now: manual text/markdown + file upload (`.md`, `.txt`, assumed `.pdf`).
- Confirm PDF is required for MVP (adds parsing + OCR questions).
- Are web URLs to be scraped? (Assumed no; admin pastes text.)

**Impact:** worker complexity, dependencies, security (untrusted files).

### Q10 (S) — Assistant access model
Assumed anonymous, rate-limited by IP/session. Confirm no login is required.

**Impact:** abuse controls, cost exposure.

---

## C. Authentication & security

### Q11 (B) — Is MFA required for admin accounts at launch?
- Option A: Not in MVP; roadmap for Phase 2. (Recommended given single-owner use.)
- Option B: TOTP required before production launch.

**Impact:** auth scope, UI, recovery flow, timeline.

### Q12 (B) — SPA/API hosting topology (same site or cross-site)?
Affects refresh cookie `SameSite` and CSRF strategy:
- Same site/subdomain (e.g., `app.example.com` + `api.example.com`) → `SameSite=Lax`/`Strict`
  feasible.
- Fully cross-site → `SameSite=None; Secure` + CSRF token required.

**Impact:** cookie, CORS, CSRF design.

### Q13 (S) — Password reset flow
- Option A: Admin-assisted reset only, no public reset endpoint. (Recommended for MVP.)
- Option B: Email-based self-service reset (requires email infra + token flow).

**Impact:** endpoints, email provider, security surface.

### Q14 (S) — How many admin users / roles are needed?
Confirm the four roles (`super_admin`, `admin`, `editor`, `viewer`) are appropriate, or collapse
to `super_admin` + `editor` for MVP.

**Impact:** seed data, UI complexity, testing matrix.

### Q15 (D) — Social/OAuth login for admins?
Currently out of scope. Confirm it can defer to Phase 2.

---

## D. Infrastructure & operations

### Q16 (B) — Where will this be hosted?
Candidates affect `infrastructure/` and deployment pipeline:
- Managed PaaS (e.g., Render/Fly/Railway/AWS ECS) vs single VPS + Docker Compose.
- Managed PostgreSQL with pgvector support vs self-hosted Postgres container.
- Object storage: S3-compatible vs local disk (affects media handling).

**Impact:** Docker/proxy config, CI/CD, backups, TLS, cost.

### Q17 (S) — Background job execution
- Option A: In-process FastAPI background tasks for MVP. (Recommended to start.)
- Option B: Redis-backed queue (RQ/Celery/Arq) from day one.

**Impact:** dependencies, reliability of indexing, scaling.

### Q18 (S) — Email provider for contact notifications and (if chosen) password reset.
Options: transactional provider (e.g., Resend/SES/Postmark) vs SMTP.

**Impact:** env config, deliverability, security of credentials.

### Q19 (S) — Media storage
- Object storage with signed URLs (recommended) vs local filesystem volume.
- Confirm max upload size and allowed types (assumed 10 MB; images + PDF resume).

**Impact:** `media_assets`, deployment, backups.

### Q20 (S) — Backups / RPO / RTO
SRS assumes RPO ≤ 24h, RTO ≤ 4h. Confirm acceptable for a personal portfolio.

**Impact:** DB plan, backup automation, restore drills.

---

## E. SEO, performance & UX

### Q21 (B) — SEO strategy for a Vite SPA
Portfolio discoverability matters. Options:
- Option A: Prerender public routes at build time (e.g., SSG/prerender plugin) and serve static
  HTML. (Recommended.)
- Option B: Adopt a meta-framework for SSR/SSG (changes the stated stack).
- Option C: SPA-only with dynamic meta tags (weakest SEO).

**Impact:** architecture, build pipeline, hosting, milestone scope.

### Q22 (S) — Analytics
Any analytics required (privacy-preserving vs Google Analytics)? Affects consent notice.

**Impact:** frontend dependencies, privacy policy.

### Q23 (D) — Branding/domain
Domain name, display name, logo, color system, and dark-mode default.

**Impact:** UI design phase, SEO metadata.

---

## F. Content & data

### Q24 (S) — Contact form abuse protection
- Option A: Honeypot + rate limit. (Recommended for MVP.)
- Option B: Add CAPTCHA/Turnstile (third-party dependency, privacy implications).

### Q25 (S) — Rate limits / LLM budget
Confirm target caps:
- Assistant: ~20 messages/min per session, ~60/hour per IP.
- LLM monthly budget ceiling and alert threshold.

**Impact:** middleware config, cost controls.

### Q26 (D) — Public resume file
Will the resume be a static file or generated from structured data? Assumed uploaded PDF.

---

## G. Decisions log (to be filled on approval)

| # | Question | Decision | Date | Owner |
|---|----------|----------|------|-------|
| Q1 | Public vs private surfaces | | | |
| Q2 | Admin app topology | | | |
| Q5 | LLM + embedding providers | | | |
| Q6 | Refusal policy | | | |
| Q11 | MFA in MVP | | | |
| Q12 | Hosting topology (cookie/CSRF) | | | |
| Q16 | Hosting platform | | | |
| Q21 | SEO strategy | | | |

---

## H. Recommended baseline (if you want to approve quickly)

If you accept the following defaults, Phase 1 can be approved immediately:

1. Single SPA with code-split admin routes (Q2-A).
2. Anonymous, strictly grounded assistant; no sign-up (Q1, Q6-A, Q10).
3. Persist chat transcripts with 90-day retention and a purge job (Q7-A).
4. No MFA at launch; roadmap for Phase 2 (Q11-A).
5. Same-site SPA/API with `SameSite=Lax` refresh cookie (Q12).
6. In-process background indexing for MVP; queue-ready interface (Q17-A).
7. Prerender public routes for SEO (Q21-A).
8. Hosting: managed PostgreSQL (pgvector) + containerized backend/frontend; object storage for
   media (Q16).
9. English-only, no blog, education included, certifications deferred (Q3, Q4).
10. Honeypot + rate limit for contact; no CAPTCHA initially (Q24-A).

**Still required before Phase 5:** the concrete LLM + embedding provider and model (Q5),
because vector dimensions and cost depend on it.
