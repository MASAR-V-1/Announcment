# Module SRS: HR & Volunteer Management

Status: Draft v1.0
Parent doc: `docs/MASAR_SRS_v2.0.md` (Section 5)
Build order: 3 of 6

---

## 1. Purpose

Track staff/volunteer directory, shifts, and hours logged, decoupled from Core so the organization can operate without it if volunteers aren't part of their model.

## 2. In scope

- Volunteer/staff directory (name, contact, role, skills, availability).
- Shift scheduling (create shifts, assign volunteers).
- Hours logging (check-in/check-out or manual entry) per volunteer per shift.
- Basic reporting: total hours by volunteer, by period.

## 3. Out of scope (MVP)

- Payroll/compensation calculation.
- Volunteer self-service mobile check-in (deferred; web only for MVP).
- Automated shift-conflict detection across multiple organizations.

## 4. Data model

```
volunteers: id, tenant_id, full_name, contact_phone, contact_email, skills_json, availability_json, status, created_at
shifts: id, tenant_id, title, location, start_at, end_at, created_by
shift_assignments: id, shift_id, volunteer_id, status (assigned|confirmed|cancelled)
volunteer_logs: id, tenant_id, volunteer_id, shift_id, hours, logged_by, logged_at
```

## 5. API endpoints

```
GET/POST  /api/volunteers
PATCH     /api/volunteers/{id}
GET/POST  /api/shifts
POST      /api/shifts/{id}/assign          (assign volunteer)
POST      /api/volunteer-logs              (log hours)
GET       /api/reports/volunteer-hours?from=&to=
```

## 6. User stories

- **US-1**: As an HR Coordinator, I want to add a volunteer to the directory with their contact info and skills, so I can match them to future shifts.
- **US-2**: As an HR Coordinator, I want to create a shift and assign volunteers to it, so coverage for an activity is planned in advance.
  - AC: Assigned volunteer sees the shift on their schedule; a double-booking on overlapping shifts triggers a warning.
- **US-3**: As a volunteer, I want my hours for a shift to be logged, so my contribution is recorded accurately.
- **US-4**: As an HR Coordinator, I want a report of total hours per volunteer for a date range, so I can recognize contributions or report to donors.
- **US-5**: As a System Admin, I want the Volunteer module to be toggleable independently, so organizations that don't use volunteers aren't forced to have it active.

## 7. Acceptance criteria (module-level)

- The Core system does not directly depend on Volunteer module tables; any cross-module reference (e.g., dashboard KPI) reads via API/event, not a hard foreign key, per the Module Isolation principle in the parent SRS.
- All directory and shift data is tenant-scoped.
- Deactivating this module hides its menu entries and blocks its API routes without affecting other modules.
