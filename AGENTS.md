# AGENTS.md

**University of Sri Jayewardenepura — Incident Reporting & Resolution Management System**

Read this before modifying the project. For visual/UI work, also read `DESIGN.md`.

---

## Mission

Institutional incident management (not a public complaint form): students report campus issues; staff verify, assign, resolve, and close through a controlled workflow.

```text
SUBMITTED → UNDER_REVIEW → VERIFIED | REJECTED
VERIFIED → FORWARDED_TO_DEAN → ASSIGNED → IN_PROGRESS → RESOLVED → CLOSED
```

Officials mark **RESOLVED**; only the **Dean** closes incidents. Backend must validate allowed status transitions (no arbitrary jumps, e.g. SUBMITTED → CLOSED).

---

## Principles

- **Institutional & professional** — clarity, trust, predictable workflows; not generic SaaS / AI-slop UI.
- **Simple** — smallest architecture that meets requirements.
- **Secure** — authorization on Django; never trust frontend-only or client-supplied roles.
- **Maintainable** — clear names, explicit business logic, reviewed serializers (avoid `fields = "__all__"` on sensitive models).

---

## Stack & deployment

| Layer | Choice |
|--------|--------|
| Frontend | Next.js, React, TypeScript, Tailwind (optional: shadcn/ui, Lucide) → **Vercel** |
| Backend | Django, DRF → **Railway** (same region as DB) |
| Database | PostgreSQL → **Railway** |
| Media | **Cloudinary** (store references in DB, not binary blobs) |

```text
Next.js (Vercel) --HTTPS--> Django REST (Railway) --> PostgreSQL (Railway)
                                    └── Cloudinary (images)
```

Frontend talks to Django via REST only (no DB credentials in Next.js). Production: HTTPS, explicit CORS for the Vercel origin, secrets in env vars (provide `.env.example` placeholders only).

**Do not add without a real requirement:** Redis, Celery, WebSockets, microservices, dedicated audit-log subsystems.

---

## Authentication & accounts

- JWT (or one consistent SPA-friendly scheme): login → access + refresh tokens.
- **Students:** roster-provisioned; MC = username, CPM = initial password; may change password after first login; **no self-registration**.
- **Officials:** created by Dean → invitation → activate → set password (no passwords in email/URLs).
- **Dean:** initial account via controlled provisioning only; never public `role=DEAN|ADMIN|OFFICIAL` from clients.
- Account statuses: `INVITED`, `ACTIVE`, `INACTIVE`, `SUSPENDED` — prefer deactivation over delete when history exists.

**Roles:** `STUDENT`, `ADMIN`, `DEAN`, `OFFICIAL`. **Official positions** (separate from role): `VICE_CHANCELLOR`, `HOD`, `MAINTENANCE_OFFICER`, `SECURITY_OFFICER`, `OTHER`. Dean has highest application authority.

### Permission summary

| Role | Core capabilities |
|------|-------------------|
| Student | Own incidents, public incidents, permitted comms; cannot verify/assign/close/manage users |
| Admin | Review, verify, reject, request info; cannot create Dean or override Dean |
| Dean | Assign/reassign, priority, close, manage officials/parties/categories/locations/settings |
| Official | Assigned incidents only — progress, evidence, comms, mark resolved; cannot close or manage users |

Every privileged and object-level action: authenticate user, role, and **authorization for that specific incident** (ID tampering must not grant access).

---

## Incident domain

- **ID:** server-generated (e.g. `INC-2026-00142`).
- **Fields (minimum):** number, title, description, category, location, visibility, reporter, status, timestamps; optional priority, image, verified/resolved/closed times.
- **Visibility:** `PUBLIC`, `PRIVATE`, `RESTRICTED` — public serializers must not expose reporter phone/email/MC, private comms, or internal notes.
- **Categories (controlled):** e.g. INFRASTRUCTURE, MAINTENANCE, WATER_PLUMBING, ELECTRICAL, CLEANLINESS, SECURITY, STUDENT_CONFLICT, ACADEMIC, IT_TECHNOLOGY, SAFETY, OTHER.
- **Priority (official):** LOW, MEDIUM, HIGH, CRITICAL.
- **Responsible parties:** entities (divisions/offices), not hard-coded user IDs; assignments record incident, official, assigner, time, comment.
- **Communication:** incident-scoped messages only (not a general chat product).
- **Lists:** paginated; optimize queries (`select_related` / `prefetch_related`); index common filter fields.

Custom Django user model from the start (e.g. name, email, phone, mc_number, role, position, status, department, year).

---

## API & security checklist

- RESTful, purpose-specific endpoints (verify, reject, assign, resolve, close, messages, etc.).
- Validate on frontend and backend; rate-limit login, registration, password reset, public APIs, incident create, comms.
- No plain-text passwords, exposed hashes, or hard-coded secrets; no hidden admin/Dean backdoors.
- Production: `DEBUG=False`, migrations applied, permission tests for the matrix in project tests (student/admin/official/dean boundaries).

---

## Frontend

- Structure: `components/`, `services/`, `hooks/`, `lib/`, `types/`; explicit TypeScript for enums like `IncidentStatus`.
- UI follows **`DESIGN.md`**; match institutional SJP visual language (see footer below).
- Forms: labels, validation, error handling; mobile-friendly reporting; skeleton/loading and duplicate-submit prevention on critical actions.

---

## Agent workflow

Before changes: read this file, `DESIGN.md` for UI, and existing models/serializers/views/permissions/tests for the area you touch.

**Source of truth:** explicit requirements → AGENTS.md → DESIGN.md → existing code/APIs → framework defaults.

**Definition of done:** backend + auth + validation + DB behavior + frontend + responsive UX + safe errors + relevant tests; sensitive data protected.

---

## Non-negotiable

1. No student/official self-registration; Dean manages officials.
2. Backend authorization mandatory; object-level incident access enforced.
3. Public incidents never expose reporter PII; private/restricted protected at API.
4. Officials resolve; Dean closes; no audit-log feature unless requirements change.
5. Vercel (frontend), Railway (Django + Postgres, co-located), Cloudinary (images).
6. Keep architecture simple; optional infra only when justified.

---

## Visual reference

Redesign aligns with https://www.sjp.ac.lk/ — use **`DESIGN.md`** for colors, typography, masthead, layouts, footer. Visual reference only; do not copy university services, links, or promotional content into this app.
