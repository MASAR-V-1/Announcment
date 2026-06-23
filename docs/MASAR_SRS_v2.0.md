# MASAR (مَسَار) — Software Requirements Specification v2.0

Status: Draft v2.0
Date: 2026-06-08
Owner: Product Owner / Team Lead

This document replaces and extends SRS v1.4. It re-scopes MASAR from a single-organization app into a configurable B2B platform with a Module Catalog and tenant provisioning. It documents tenancy model, Module Engine architecture, catalog of modules, core data model, API contract, provisioning & demo flows, security constraints, and acceptance criteria for the MVP.

---

1. Executive summary

MASAR v2.0 is a multi-tenant (logical) B2B platform designed to be provisioned per-customer (tenant). The platform provides a core kernel and a Module Engine that manages a catalog of administrative modules (approx. 30). Each customer purchases a subset of modules at contract time (provisioning). After provisioning, the customer's tenant admin activates/deactivates (On/Off) modules at runtime. For the MVP we will implement a shared-database, tenant-scoped architecture (tenant_id), and deliver 6 fully-implemented modules (backend + frontend). The remaining modules will be delivered as frontend preview widgets in the MVP and can be upgraded to full later.

Decisions confirmed by product owner:
- Tenancy model: Shared DB + tenant_id (logical multi-tenancy). Upgrade path to DB-per-tenant supported in architecture.
- MVP full modules: IT & Admin, Field Operations, HR & Volunteers, M&E & Reporting, Tasks & Workflow, Logistics & Supply Chain.
- Remaining modules: preview-only widgets in MVP.

---

2. Goals & constraints

Goals:
- Deliver a configurable platform that can be sold to NGOs and configured per contract.
- Provide a Module Catalog for provisioning and a tenant On/Off slicer for runtime activation.
- Ensure tenant isolation (logical) and protect PII with blind indexing and encryption.

Constraints:
- MVP timeline and scope: implement Module Engine, 6 full modules + previews, AI microservice mocks.
- No dynamic uploadable plugins in MVP: modules are part of the codebase and enabled by flags.
- Offline-first mobile deferred to v2.

---

3. Tenancy model

MVP choice: Shared database with tenant_id column on all tenant-scoped tables. This provides a fast path to a multi-tenant B2B product while keeping per-tenant isolation via global query scopes and middleware.

Upgrade path: The system shall be designed to allow per-tenant migration to a DB-per-tenant model later. Code must be modular to support migration tooling.

Tenant lifecycle:
- Tenant record created (tenant_id, contract metadata, admin contact).
- Provisioning: provider marks modules as contracted for tenant (tenant_modules.is_contracted=true). Provisioning triggers module provisioning tasks (migrations/seeds/config).
- Runtime activation: tenant admin toggles tenant_modules.is_active to enable/disable modules at runtime.

Data isolation rules:
- All tenant-scoped models must include tenant_id and apply a global scope for tenant filtering.
- All API endpoints must verify the authenticated user's tenant_id matches resources being accessed.

---

4. Module Engine (Module Kernel)

Overview:
- Core provides user management, authentication, RBAC (Spatie), module registry, ModuleManager, tenant provisioning APIs, audit logging, and shared infrastructure (DB, storage, secrets).
- Modules are self-contained units inside the codebase (not dynamically uploaded in MVP). Each module includes a manifest describing routes, migrations, required permissions, and menu entries.

Module lifecycle:
- register: module declares metadata and permissions to the Module Registry.
- provision (contract): provider-side API sets tenant_modules.is_contracted and queues a provisioning job.
- provision job: runs module migrations/seeds for tenant if needed, creates tenant-specific config entries.
- activate/deactivate: tenant admin toggles is_active. ModuleManager loads only modules where is_contracted && is_active for the tenant.

Data model for modules:
- modules (module_key, name, type [full|preview], manifest_json)
- tenant_modules (tenant_id, module_key, is_contracted, is_active, config JSON, provisioned_at)

Security considerations:
- Restrict provisioning APIs to provider super_admin role.
- Restrict tenant module toggles to tenant_admin with permission manage-modules.

Implementation notes:
- Recommend using a modular structure such as nwidart/laravel-modules (or similar) to organize module code, routes, controllers and migrations.
- Provisioning should be asynchronous (queued job) and idempotent.

---

5. Module catalog and MVP module list

Catalog size: ~30 administrative modules modeled on common NGO/org departments.

MVP full modules (backend + frontend):
1. IT & Admin (system administration, users, roles, org profile)
2. Field Operations (beneficiary registry, distributions, dedup)
3. HR & Volunteer Management (volunteer directory, shifts, hours)
4. M&E & Reporting (KPIs, indicators, AI report generation)
5. Tasks & Workflow (Kanban board, assignments)
6. Logistics & Supply Chain (basic inventory and distributions)

Preview-only modules (frontend widgets for demo/marketing):
- Finance & Procurement, Donor & Grants, Complaints & Feedback (CFM), Protection, Training & Capacity, Knowledge Management, Coordination (Cluster), Health, WASH, Education, Food Security & Livelihoods, Emergency Response, Legal, Communications, Asset Management, Security/HSS, etc.

Each catalog entry includes: key, display name, short description, type (full|preview), default permissions list, icon, and suggested menu path.

---

6. Core data model (high-level)

Tenant & user:
- tenants: id, name, contact, timezone, currency, contract_metadata
- users: id, tenant_id, name, email, password_hash, is_active, roles (Spatie), profile_json

Modules & provisioning:
- modules: key, name, type, manifest
- tenant_modules: tenant_id, module_key, is_contracted, is_active, config_json, provisioned_at

Domain models (tenant-scoped, examples):
- beneficiaries: id, tenant_id, full_name, normalized_name, national_id_blind_index, national_id_encrypted, phone, family_size, address_json, created_by, created_at
- distributions: id, tenant_id, beneficiary_id, item, quantity, distributed_at, officer_id
- volunteers, volunteer_logs, shifts
- tasks, task_status_history
- reports: id, tenant_id, period_start, period_end, generated_by, content, status, created_at
- audit_logs: id, tenant_id, user_id, action, entity, entity_id, before_json, after_json, timestamp

Indexing & search:
- PostgreSQL with pg_trgm for name similarity searches (normalized_name).
- Blind indexing strategy for national_id exact-match search.

---

7. API Contract (critical endpoints)

Auth & tenancy:
- POST /api/login — returns JWT, tenant_id included in session
- POST /api/logout
- POST /api/refresh

Provider / provisioning (super_admin only):
- GET /admin/modules/catalog — full catalog
- POST /admin/tenants/{tenant_id}/contract — contract a module(s) for tenant (body: module_keys[])
- POST /internal/modules/{module_key}/provision?tenant={id} — internal trigger (queue) for provisioning

Tenant admin (tenant-scoped):
- GET /tenant/modules — list contracted modules for tenant with is_active flag
- POST /tenant/modules/{module_key}/toggle — toggle is_active (requires manage-modules permission)

Module-specific endpoints (examples):
- GET /api/beneficiaries?query= — paginated, tenant-scoped
- POST /api/beneficiaries/check-duplicate — pre-save duplicate check (calls AI microservice)
- POST /api/beneficiaries — create beneficiary (tenant scope)
- POST /api/reports/generate — request AI report (tenant scope)

AI microservice (internal):
- POST /ai/duplicate-check (internal key required)
- POST /ai/generate-report (internal key required)

Notes:
- All tenant-scoped endpoints must validate req.user.tenant_id and apply global tenant scope.
- All AI endpoints require internal API key and network restrictions.

---

8. Provisioning & deployment flow

Provisioning (provider perspective):
1. Create tenant and initial tenant_admin account.
2. Provider selects modules for the tenant in the catalog (is_contracted = true). This calls provisioning API.
3. System enqueues module provisioning jobs for each contracted full module. Jobs run migrations/seeds and create tenant-specific config.
4. Provider verifies provisioning success and notifies tenant.

Tenant activation (tenant_admin):
1. Tenant_admin logs in to tenant console.
2. Tenant_admin sees only contracted modules. They toggle is_active for the modules they want to enable.
3. Once activated, the ModuleManager loads modules for users of the tenant; front-end menus reflect active modules.

Edge cases & rollback:
- Provisioning jobs must be idempotent and transactional where possible. If a provisioning job fails, the system rolls back the tenant_modules entry and records error in audit logs.

---

9. Demo (presentation) script

Two-perspective demo from single running instance:

1) Provider view (super_admin):
- Login as super_admin.
- Open Modules Catalog (all ~30 modules visible).
- Select 6 modules for Tenant X and click Contract.
- Demonstrate provisioning job queued and completed (show logs).

2) Tenant view (tenant_admin X):
- Login as tenant_admin for Tenant X.
- Show tenant dashboard with only the 6 contracted modules visible.
- Use On/Off slicer to activate/deactivate modules and show that menus and APIs change accordingly.

3) User view (field officer):
- Login as field officer.
- Show that only active modules for the tenant and the user's assigned department are visible.
- Create a beneficiary and run pre-save duplicate-check (calls AI microservice mock), demonstrate Merge/Proceed flow.

---

10. Security & privacy

- Tenant isolation via tenant_id global scopes and middleware checks.
- Early design: treat PII as sensitive: national_id encrypted at rest, blind-index used for exact search, limited access and logging.
- Roles and permissions via Spatie package. A permission `manage-modules` is reserved for provider super_admin and tenant_admin roles.
- AI microservice requires internal API key; calls are restricted to backend API only.
- Transport: TLS for all traffic. Secrets in env or secret manager; no secrets in repo.

---

11. Acceptance criteria (select examples)

Provisioning:
- When provider contracts a full module for tenant, tenant_modules.is_contracted becomes true and the provisioning job completes successfully, creating tenant-level database artifacts.

Tenant activation:
- Tenant_admin can only see modules where is_contracted=true. Toggling is_active updates UI and module availability immediately for tenant users.

Isolation:
- A user authenticated under tenant A can never read resources (beneficiaries, volunteers, tasks) belonging to tenant B; automated tests validate this.

Module loading:
- ModuleManager only registers routes/controllers/menus for modules active for the tenant.

Duplicate-check:
- POST /api/beneficiaries/check-duplicate returns a ranked list of candidate matches before create.

Report generation:
- POST /api/reports/generate returns a persisted report record and stores generated content in reports table with generated_by and timestamp.

---

12. Implementation roadmap (high-level tasks for initial sprints)

Sprint 0 & 1 (Foundation):
- Scaffold repo skeletons, tenant model, user auth, Spatie RBAC, modules table and tenant_modules table, ModuleManager skeleton, provisioning API (mock job), AI microservice mock endpoints.
- Implement IT & Admin module minimal features (user management, roles) as full module.

Sprint 2:
- Implement Field Operations (beneficiaries, distributions), AI duplicate-check integration with mock service, migrations for beneficiary model.
- Implement HR & Volunteers module basics.

Sprint 3:
- Implement M&E & Reporting, Tasks & Workflow, Logistics basics, dashboard KPIs, AI report pipeline (using mocked AI or Gemini in production later).

---

13. Module manifest example (manifest JSON)

```json
{
  "key": "field_operations",
  "name": "Field Operations",
  "version": "1.0.0",
  "type": "full",
  "permissions": ["beneficiary.view","beneficiary.create","distribution.create"],
  "routes": "Modules/FieldOperations/routes/api.php",
  "migrations": "Modules/FieldOperations/Database/Migrations",
  "menu": { "label": "Beneficiaries", "path": "/beneficiaries", "icon": "users" }
}
```

---

14. Open decisions & questions

- Confirm preview vs full lists for any modules you want changed.
- Confirm whether you want provider-side UI to run provisioning live during demo or simulate provisioning success (recommended: run on queue but keep a small dataset to avoid long DB tasks during demo).

---

15. Next steps

If you confirm this SRS v2.0 outline, I will produce the final Markdown file and commit it to `MASAR-V-1/Announcment/docs/MASAR_SRS_v2.0.md`. I will also create a short checklist of immediate implementation tasks (issues) for Sprint 0 & 1.

Please reply "Start" to create the final SRS v2.0 in the repo, or provide edits if you want changes before committing.
