# Module SRS: M&E & Reporting

Status: Draft v1.0
Parent doc: `docs/MASAR_SRS_v2.0.md` (Sections 5, 7, 9)
Build order: 6 of 6 (highest complexity — depends on AI microservice; build/harden last or defer)

---

## 1. Purpose

Give managers aggregated KPIs and an AI-assisted narrative report generator, reducing manual reporting effort. This module is the most complex in the MVP because it depends on an external/mocked AI microservice rather than pure CRUD.

## Identity model (shared across all modules)

This module does not define its own user identity or login logic. Every user is identified platform-wide by the `employee_code` (canonical routing/identity key) linked 1:1 to their `email` (login credential), as defined in `01-it-admin.md` Section 2a. Any `*_id` field referencing a user in this module (assignee, officer, requester, etc.) resolves back to that same identity.

## 2. In scope

- Dashboard KPI cards (Total Beneficiaries, Active Volunteers, Overdue Tasks, etc.), cached ~5 minutes.
- "Generate Monthly Report" action that calls the AI microservice to draft a narrative summary.
- Persisted report records (content, period, generated_by, status).
- Loading state for the 10–15 second AI generation wait.

## 3. Out of scope (MVP)

- Custom/ad-hoc indicator builder (KPIs are a fixed initial set).
- Real-time (sub-minute) dashboards.
- Multi-language report generation beyond the platform's supported locales.

## 4. Data model

```
reports: id, tenant_id, period_start, period_end, generated_by, content, status, created_at
```
(KPIs are computed on read from existing tenant-scoped tables — beneficiaries, distributions, tasks, volunteer_logs — not stored separately.)

## 5. API endpoints

```
GET   /api/dashboard/kpis
POST  /api/reports/generate                    (tenant scope; calls AI microservice)
GET   /api/reports/{id}
POST  /ai/generate-report                       (internal, requires internal API key)
```

## 6. User stories

- **US-1**: As a System Admin, I want to see aggregated statistics immediately upon login, so I understand the organization's current status at a glance.
  - AC: KPI data is cached for 5 minutes; stale-but-fast is acceptable over real-time-but-slow.
- **US-2**: As a System Admin, I want to click "Generate Monthly Report" and get an AI-drafted narrative summary, so I save hours of manual report writing.
  - AC: A loading indicator is shown during the 10–15 second generation wait; the result is saved as a `reports` record, not just shown transiently.
- **US-3**: As a System Admin, I want to view previously generated reports, so I can reference or re-share them later without regenerating.

## 7. Acceptance criteria (module-level)

- `POST /api/reports/generate` always returns a persisted report record with `generated_by` and timestamp, even if the AI call is mocked in MVP.
- The `/ai/generate-report` endpoint requires an internal API key and is not reachable from the public internet.
- KPI queries respect tenant scoping and do not leak cross-tenant aggregate counts.

## 8. Note on MVP sequencing

Because this module depends on functioning AI integration (even if mocked), consider deferring it to last in the build order or shipping the KPI dashboard first (pure CRUD/aggregation) and the AI report generator as a fast-follow, per the "less complex MVP" guidance discussed for this project.