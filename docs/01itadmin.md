# Module SRS: IT & Admin

Status: Draft v1.0
Parent doc: `docs/MASAR_SRS_v2.0.md` (Sections 4, 5, 5a)
Build order: 1 of 6 (foundation — everything else depends on this)

---

## 1. Purpose

This module is the platform's kernel-facing admin surface: user accounts, roles/permissions, tenant/module configuration, and the day-to-day IT-department functions an organization expects (helpdesk ticketing, asset tracking, access request lifecycle). It must exist before any other module can be meaningfully used, since it owns authentication, RBAC, and the on/off switch for every other module.

## 2. In scope

- User management: create/edit/deactivate accounts, assign roles.
- RBAC: roles and permissions via Spatie, including `manage-modules` and `manage-access` permissions.
- Org profile: tenant name, contact, timezone, currency.
- Module toggling: tenant admin turns contracted modules on/off.
- Audit logging: every CRUD on users/roles/modules is recorded.
- Helpdesk & ticketing (category, priority, SLA, assignee, comments).
- Asset management (hardware + license inventory, assignment history).
- Access lifecycle (HR-initiated request → IT approval → execution, offboarding checklist).

## 3. Out of scope (MVP)

- Network/device monitoring, remote device management (MDM).
- Automated patch management / software deployment.
- Procurement/purchase-order workflow for new assets (tracked manually, entered after purchase).

## 4. Data model

```
users: id, tenant_id, name, email, password_hash, is_active, roles, profile_json
modules: key, name, type, manifest
tenant_modules: tenant_id, module_key, is_contracted, is_active, config_json, provisioned_at
audit_logs: id, tenant_id, user_id, action, entity, entity_id, before_json, after_json, timestamp

it_tickets: id, tenant_id, requester_id, assignee_id, category, priority, status, sla_due_at, created_at, resolved_at
it_ticket_comments: id, ticket_id, user_id, body, created_at
it_assets: id, tenant_id, asset_tag, type, serial_number, purchase_date, warranty_expiry, status, assigned_to, created_at
it_asset_history: id, asset_id, user_id, action (assigned|unassigned|status_change), note, created_at
it_access_requests: id, tenant_id, employee_id, request_type (new_access|role_change|offboarding), requested_by, approved_by, executed_by, status, created_at, executed_at
```

## 5. API endpoints

```
POST   /api/login | /api/logout | /api/refresh
GET    /api/users            (list, tenant-scoped)
POST   /api/users            (create)          — Admin
PATCH  /api/users/{id}       (edit/deactivate)  — Admin
GET    /tenant/modules
POST   /tenant/modules/{key}/toggle             — manage-modules

GET/POST  /api/it/tickets
PATCH     /api/it/tickets/{id}                  (status/assignee/priority)
POST      /api/it/tickets/{id}/comments

GET/POST  /api/it/assets
PATCH     /api/it/assets/{id}                   (assign/unassign/status)

GET/POST  /api/it/access-requests
PATCH     /api/it/access-requests/{id}/approve
PATCH     /api/it/access-requests/{id}/execute
```

## 6. User stories

### Epic A: User & Role Management
- **US-1**: As a System Admin, I want to create a new staff account and assign a role, so the person can log in with the right access.
  - AC: Account created with a temporary password; role's permissions immediately apply; audit log entry created.
- **US-2**: As a System Admin, I want to deactivate a user's account, so a departed staff member loses access immediately.
  - AC: Deactivated user cannot log in; existing sessions are invalidated within 5 minutes.

### Epic B: Module Control
- **US-3**: As a Tenant Admin, I want to see only the modules my organization has contracted, so I don't get confused by modules we haven't purchased.
- **US-4**: As a Tenant Admin, I want to toggle a contracted module on/off, so I can control what my staff see without contacting support.
  - AC: Toggling off hides the module's menu entry and blocks its API routes for all tenant users within the session.

### Epic C: Helpdesk & Ticketing
- **US-5**: As any Staff member, I want to open an IT support ticket with a category and priority, so my issue gets tracked and resolved.
  - AC: Ticket gets an SLA due date based on priority (Urgent: 4h, High: 1 business day, Medium: 3 business days, Low: 5 business days).
- **US-6**: As IT Staff, I want to see a dashboard of open tickets sorted by SLA due date, so I can prioritize what's overdue.
- **US-7**: As a ticket requester, I want to be notified when my ticket's status changes, so I know when it's resolved.

### Epic D: Asset Management
- **US-8**: As IT Staff, I want to register a new asset (laptop, phone, license) with its serial number and warranty date, so we have a single inventory record.
- **US-9**: As IT Staff, I want to assign an asset to a user and see the assignment history, so I know who has what and when it changed.
- **US-10**: As IT Staff, I want to see licenses/assets nearing expiry, so I can renew or reassign in time.

### Epic E: Access Lifecycle
- **US-11**: As an HR user, I want to submit an access request (new hire, role change, or offboarding) tied to an employee, so IT knows exactly what to provision.
- **US-12**: As IT Staff, I want to approve and then execute an access request, so there's a clear separation between "requested" and "actually done."
  - AC: Audit log distinguishes `requested_by`, `approved_by`, and `executed_by`.
- **US-13**: As IT Staff, I want an offboarding request to auto-flag the employee's assigned assets, so nothing gets lost when someone leaves.

## 7. Acceptance criteria (module-level)

- A user authenticated under Tenant A can never see or modify Tenant B's users, tickets, assets, or requests.
- Every user/role/module-toggle/ticket/asset/access-request change appears in `audit_logs` with actor and timestamp.
- `manage-modules` and `manage-access` permissions are enforced server-side, not just hidden in the UI.
