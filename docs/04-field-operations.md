# Module SRS: Field Operations

Status: Draft v1.0
Parent doc: `docs/MASAR_SRS_v2.0.md` (Sections 5, 7, 9)
Build order: 4 of 6 (or swap with HR & Volunteers depending on target customer)

---

## 1. Purpose

Register and track beneficiaries/individuals receiving aid, and record distributions to them. This is the primary "field" module for NGO-style organizations.

## Identity model (shared across all modules)

This module does not define its own user identity or login logic. Every user is identified platform-wide by the `employee_code` (canonical routing/identity key) linked 1:1 to their `email` (login credential), as defined in `01-it-admin.md` Section 2a. Any `*_id` field referencing a user in this module (assignee, officer, requester, etc.) resolves back to that same identity.

## 2. In scope

- Beneficiary registration (name, national ID, phone, family size, address).
- Duplicate check before creating a new beneficiary (candidate match list; merge or proceed).
- Distribution recording (item, quantity, date, officer).
- Search beneficiaries by name (fuzzy) or national ID (exact, via blind index).

## 3. Out of scope (MVP)

- Offline-first mobile data collection (deferred to v2, per parent SRS Section 2).
- Automated eligibility scoring / needs assessment logic.
- GPS/geofencing validation of distribution locations.

## 4. Data model

```
beneficiaries: id, tenant_id, full_name, normalized_name, national_id_blind_index, national_id_encrypted, phone, family_size, address_json, created_by, created_at
distributions: id, tenant_id, beneficiary_id, item, quantity, distributed_at, officer_id
```

## 5. API endpoints

```
GET    /api/beneficiaries?query=              (paginated, tenant-scoped)
POST   /api/beneficiaries/check-duplicate     (pre-save candidate match; calls AI microservice)
POST   /api/beneficiaries                     (create)
GET/POST /api/distributions
```

## 6. User stories

- **US-1**: As a Field Officer, I want to search for a beneficiary by name or ID before registering a new one, so I don't create duplicates.
  - AC: `check-duplicate` returns a ranked list of likely matches; officer can choose "Merge" (link to existing) or "Proceed" (create new).
- **US-2**: As a Field Officer, I want to register a new beneficiary with their basic details, so we have a record for future distributions.
  - AC: An audit log entry is created on every insertion; national ID is encrypted at rest with a blind index for exact search.
- **US-3**: As a Field Officer, I want to record a distribution (item, quantity, date) against a beneficiary, so we have proof of aid delivered.
- **US-4**: As a Manager, I want to see distribution totals per beneficiary/item, so I can report to donors.

## 7. Acceptance criteria (module-level)

- National ID is never stored or returned in plaintext over the API except to roles with explicit `beneficiary.view-pii` permission.
- Duplicate-check must respond before the beneficiary is saved (blocking pre-save call), not after.
- All beneficiary/distribution data is tenant-scoped; cross-tenant queries are impossible even with a crafted request.