# Software Requirements Specification (SRS)

**Project Name:** MASAR (مَسَار)
**Methodology:** Agile / Scrum
**Version:** 1.4 Final
**Date:** June 6, 2026
**Document Owner:** Product Owner / Team Lead
**Status:** Approved for MVP Development

---

## Table of Contents

1. Introduction & Product Vision
2. Scrum Roles & Responsibilities
3. User Personas & Roles
4. Functional Requirements (Epics & User Stories)
5. Non-Functional Requirements (NFRs)
6. Release Plan (June – 4 Sprints)
7. System Architecture Overview
8. Data Model (High-Level)
9. API Contract (High-Level)
10. Risks & Mitigations
11. Glossary
12. Document Approval
13. Definition of Done (DoD) & Error Handling Guidelines

---

## 1. Introduction & Product Vision

### 1.1 Purpose
The purpose of this document is to define the functional and non-functional requirements for the MASAR platform, specifically for its Minimum Viable Product (MVP) release. This document serves as the primary Product Backlog and a guide for the development team during the 8-week delivery cycle.

### 1.2 Product Vision
MASAR is an intelligent, centralized management platform designed for humanitarian and community organizations. It aims to eliminate scattered data (e.g., decentralized Excel sheets) and streamline administrative and field operations. By integrating Artificial Intelligence (AI), MASAR significantly reduces administrative burden, detects beneficiary duplication, and automates reporting.

### 1.3 MVP Scope (8-Week Timeline)
The MVP will focus on a single-tenant architecture (one organization per deployment) focusing on the core operations:

- User & Role Management
- Beneficiary Management (with AI-driven Duplicate Detection and Blind Indexing for searchability)
- Volunteer Management (decoupled into an independent, toggleable module using a Feature Flag)
- Task & Workflow Management
- Smart Dashboard & AI Report Generation

### 1.4 Out of Scope for MVP
- Multi-tenant / multi-organization architecture
- Donor / fundraising management
- Inventory & warehouse management
- Financial accounting modules
- Offline mobile mode (planned for v2)
- SMS / WhatsApp integration

### 1.5 Document Conventions
- **SHALL / MUST:** Mandatory requirement.
- **SHOULD:** Recommended requirement.
- **MAY:** Optional requirement.
- **US-X.Y:** User Story identifier (Epic.Story).

---

## 2. Scrum Roles & Responsibilities

| Role | Responsibilities |
|---|---|
| Product Owner / Team Lead | Owns the Product Backlog, defines priorities, ensures the product meets business needs, and oversees the AI integration strategy. |
| Scrum Master | Facilitates daily standups, sprint planning, sprint reviews, retrospectives, and removes blockers. |
| Laravel Developer | Implements the Core REST API, authentication, business logic, and database layer. |
| Python Developer | Builds the FastAPI AI microservice (fuzzy matching + Claude API integration). |
| Frontend Developer | Implements the web UI (RTL Arabic-first). |
| Flutter Developer | Implements the mobile UI for field officers. |
| API Integration Specialist | Ensures stable contracts between Laravel ↔ Python ↔ Frontend/Mobile clients. |

---

## 3. User Personas & Roles

**👤 General Manager / System Admin (Level 0)**
Has full access to the organization's data via Dynamic Role-Based Access Control (RBAC) implemented via the Spatie package. Focuses on high-level KPIs, aggregated data, and AI-generated reports to make strategic decisions.

**👤 Staff / Data Entry / Field Officer (Level 1)**
Interacts with the system daily to register beneficiaries, update case statuses, and manage assigned tasks. Prioritizes speed, ease of use, and mobile responsiveness.

**👤 Volunteer Coordinator (Level 1)**
Manages the volunteer database, assigns volunteers to specific events or tasks, and tracks their contributed hours.

### 3.1 Role Permissions Matrix

| Capability | Admin | Staff | Coordinator |
|---|:---:|:---:|:---:|
| Manage users & roles | ✅ | ❌ | ❌ |
| View KPI dashboard | ✅ | Partial | Partial |
| Register / edit beneficiaries | ✅ | ✅ | ❌ |
| Trigger AI duplicate check | ✅ | ✅ | ❌ |
| Manage volunteers | ✅ | ❌ | ✅ |
| Log volunteer hours | ✅ | ❌ | ✅ |
| Create / assign tasks | ✅ | ❌ | ✅ |
| Update own task status | ✅ | ✅ | ✅ |
| Generate AI reports | ✅ | ❌ | ❌ |

---

## 4. Functional Requirements

### Epic 1: Authentication & Authorization
**Description:** Establishing the core security layer using the Spatie package for dynamic Role-Based Access Control (RBAC).

#### US-1.1: Secure Login
As a system user, I want to log in using my email and password, so that I can access the platform securely.

**Acceptance Criteria:**
- Frontend communicates with `POST /api/login` (Laravel).
- System returns a JWT Token upon successful validation.
- Clear Arabic error messages are displayed for invalid credentials.
- Account is temporarily locked after 5 consecutive failed attempts.

#### US-1.2: Role Management
As a System Admin, I want to create new staff accounts and assign specific permissions through Dynamic RBAC using the Spatie package, so that I can restrict access to sensitive data.

**Acceptance Criteria:**
- Admin can access a user management dashboard.
- Standard "Staff" cannot access the user management routes (HTTP 403).
- Role changes take effect on the user's next request.

### Epic 2: Beneficiary Management (Core Module)
**Description:** The primary module for registering and tracking individuals or families receiving aid.

#### US-2.1: Register Beneficiary
As a Staff member, I want to input beneficiary details (Name, ID, Phone, Address, Category), so that their data is centralized.

**Acceptance Criteria:**
- Strict validation on phone numbers and national IDs.
- Data is saved via `POST /api/beneficiaries`.
- An audit log entry is created on every insertion.

#### US-2.2: Beneficiary Directory & Search
As a User, I want to view a paginated list of beneficiaries and search by name or ID, so that I can quickly locate their files.

**Acceptance Criteria:**
- Server-side pagination (default 25 per page).
- A fast search bar connected to a search API (debounced 300ms).
- Search supports Arabic and Latin scripts.

#### US-2.3: AI-Driven Duplicate Detection (Fuzzy Matching)
As a Staff member, I want the system to alert me if I am registering a beneficiary that already exists (even with slight spelling variations), so that we prevent duplicate aid distribution.

**Acceptance Criteria:**
- Laravel sends the input data to the Python Microservice (`POST /ai/duplicate-check`).
- Python service uses Fuzzy Matching (e.g., RapidFuzz / Levenshtein) against existing records.
- If a match > 85% is found, a warning prompt appears: "يوجد مستفيد مشابه. هل تريد المتابعة أم الدمج؟" — "A similar beneficiary exists. Proceed or merge?"
- The user can choose: Proceed, Merge, or Cancel.

### Epic 3: Volunteer Management
**Description:** Independent module managed via Feature Flags for flexible deployments, used for tracking volunteer resources and their engagement.

#### US-3.1: Volunteer Registry
As a Volunteer Coordinator, I want to add a new volunteer with their skills and availability, so that I have a ready-to-deploy database.

**Acceptance Criteria:**
- Form includes fields for Name, Phone, Skills (multi-select), and Available Hours.
- Volunteers list supports filtering by skill.

#### US-3.2: Hours Tracking
As a Volunteer Coordinator, I want to log the hours a volunteer has worked after an activity, so that their total contribution is updated accurately.

**Acceptance Criteria:**
- Logging form: volunteer, activity, date, hours.
- Volunteer profile shows running total of hours.

### Epic 4: Tasks & Workflow
**Description:** Internal coordination and task delegation.

#### US-4.1: Task Creation & Assignment
As a Manager, I want to create a task, set a deadline, and assign it to a specific staff member, so that operational work is tracked.

**Acceptance Criteria:**
- Task fields: Title, Description, Priority (Low/Medium/High), Deadline, Assignee.
- Assignee receives an in-app notification.

#### US-4.2: Kanban Board View
As a Staff member, I want to view my assigned tasks in a Kanban board (Todo, In Progress, Done), so that I can easily update task statuses via drag-and-drop or simple clicks.

**Acceptance Criteria:**
- Three columns: Todo, In Progress, Done.
- Status updates persist via `PATCH /api/tasks/{id}`.
- Overdue tasks are visually highlighted (red border).

### Epic 5: Dashboard & AI Insights
**Description:** High-level overview and automated reporting for decision-makers.

#### US-5.1: KPI Dashboard
As a System Admin, I want to see aggregated statistics immediately upon login (Total Beneficiaries, Active Volunteers, Overdue Tasks), so that I understand the organization's current status.

**Acceptance Criteria:**
- Laravel aggregates data efficiently (cached for 5 minutes) and serves it to frontend KPI cards.
- Dashboard renders in under 2 seconds.

#### US-5.2: AI Report Generator
As a System Admin, I want to click a "Generate Monthly Report" button to have AI draft a narrative summary of our activities, so that I save hours of manual report writing.

**Acceptance Criteria:**
- Laravel compiles monthly statistics (JSON) and sends them to the Python service (`POST /ai/generate-report`).
- Python service constructs a prompt and interfaces with the Claude API.
- The system returns a drafted, editable narrative report in Arabic, exportable as a PDF.
- Loading spinner shown during the 10–15 second wait.

---

## 5. Non-Functional Requirements (NFRs)

### 5.1 Performance & Responsiveness
- Web (Frontend) and Mobile (Flutter) interfaces MUST render and respond to basic CRUD operations in under 2 seconds.
- AI operations (duplicate detection, report generation) are permitted a latency of up to 10–15 seconds, provided a clear loading indicator (spinner) is displayed to the user.
- Backend API endpoints SHOULD respond in under 500 ms at the 95th percentile (excluding AI calls).

### 5.2 Usability & Localization
- **Arabic-First:** The platform MUST natively support Right-to-Left (RTL) alignment and Arabic typography. English will be supported as a secondary language.
- The UI MUST be intuitive for non-technical NGO field workers, utilizing clear iconography and logical workflows.
- All date/time values SHALL display in the user's local timezone, defaulting to Arabia Standard Time.

### 5.3 Architecture & Integration
- **Core API:** Laravel will act as the central repository and main REST API.
- **AI Microservice:** Python (FastAPI) will operate as an internal microservice, communicating exclusively with the Laravel backend — never directly with the frontend.
- **Security:** JWT-based authentication. The Python service MUST require an internal API key to accept requests from Laravel.
- All inter-service traffic SHALL occur over HTTPS or within a private VPC.

### 5.4 Security
- Passwords hashed using bcrypt (cost ≥ 12).
- JWT access tokens expire in 60 minutes; refresh tokens in 7 days.
- All API endpoints MUST enforce role-based authorization (RBAC).
- Sensitive PII (national ID, phone) SHALL be encrypted at rest, utilizing Blind Indexing on National ID for secure searchability.
- Full audit log of CRUD actions on beneficiaries and users.

### 5.5 Hosting & Availability
- The platform will be designed for cloud deployment (e.g., Vercel for Frontend, DigitalOcean / AWS for Backend services).
- Target availability is 99% during standard operational hours.
- Daily automated database backups, retained for 30 days.

### 5.6 Maintainability
- Code SHALL follow PSR-12 (PHP) and PEP-8 (Python).
- Unit test coverage target: ≥ 70% for backend services.
- CI/CD pipeline using GitHub Actions for automated build, test, and deploy.

---

## 6. Release Plan (June – 4 Sprints)

| Sprint | Dates | Focus | Deliverable |
|---|---|---|---|
| Sprint 0 & 1 | June 1 – June 14 | Foundation — Foundational infrastructure and Mock APIs for the AI service. | A functional core where users can log in and manage beneficiaries safely. |
| Sprint 2 | June 15 – June 21 | Expansion — Epic 3 (Volunteers) + AI algorithm development for duplicate detection. | Volunteer module live; Task API endpoints ready. |
| Sprint 3 | June 22 – June 28 | Workflow & AI — Epic 4 (Tasks Kanban) + Epic 5 (Dashboard & AI Report Generation) | Full MVP feature set complete. |
| End of June | — | MVP Demo | First full MVP Demo presentation to the training organization. |

---

## 7. System Architecture Overview

```
┌──────────────┐     ┌──────────────┐
│  Web (RTL)   │     │  Flutter App │
│   Frontend   │     │    Mobile    │
└──────┬───────┘     └──────┬───────┘
       │     HTTPS / JWT    │
       └─────────┬──────────┘
                 ▼
       ┌──────────────────┐
       │  Laravel Core API│
       │  (REST + Auth)   │
       └─────────┬────────┘
                 │  Internal API Key (HTTPS)
                 ▼
       ┌──────────────────┐         ┌────────────┐
       │  Python FastAPI  │ ──────▶ │ Claude API │
       │  AI Microservice │         └────────────┘
       └─────────┬────────┘
                 │
                 ▼
          ┌────────────┐
          │ PostgreSQL │
          └────────────┘
```

### CRITICAL ARCHITECTURE RULES (Added post-review)
1. **Search Constraints:** Use `pg_trgm` (Fuzzy Search) for Names ONLY. National IDs must use Blind Indexing for Exact Match ONLY. Do not attempt to fuzzy-match encrypted hashes.
2. **Module Isolation (Volunteer Module):** The Core system must NOT depend on the Volunteer Module. Communication should be strictly via Read-Only Foreign Keys or Laravel Events. Developers must draft an ADR (Architecture Decision Record) for this before coding.
3. **Mobile Offline-First:** The Flutter app MUST use a local database (e.g., Isar). Beneficiary registration must work offline and sync when online (flagged as "Pending AI Check" until synced).

> **Note:** PostgreSQL will utilize the `pg_trgm` extension for high-performance fuzzy matching while minimizing RAM usage during search.

---

## 8. Data Model (High-Level)

| Entity | Key Fields |
|---|---|
| User | id, name, email, password_hash, role, created_at |
| Beneficiary | id, full_name, national_id (Blind Indexed), phone, address, category, created_by, created_at |
| Volunteer | id, full_name, phone, skills[], available_hours, total_hours |
| VolunteerLog | id, volunteer_id, activity, date, hours |
| Task | id, title, description, priority, deadline, assignee_id, status, created_by |
| AuditLog | id, user_id, action, entity, entity_id, timestamp |

---

## 9. API Contract (High-Level)

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| POST | `/api/login` | User login, returns JWT | Public |
| POST | `/api/users` | Create user | Admin |
| GET | `/api/beneficiaries` | List (paginated, searchable) | Staff+ |
| POST | `/api/beneficiaries` | Register beneficiary (triggers AI check) | Staff+ |
| GET | `/api/volunteers` | List volunteers | Coordinator+ |
| POST | `/api/volunteers` | Create volunteer | Coordinator+ |
| POST | `/api/volunteers/{id}/log-hours` | Log volunteer hours | Coordinator+ |
| GET | `/api/tasks` | List tasks (Kanban) | Staff+ |
| POST | `/api/tasks` | Create task | Coordinator/Admin |
| PATCH | `/api/tasks/{id}` | Update task status | Assignee/Admin |
| GET | `/api/dashboard/kpis` | Aggregated KPIs | Admin |
| POST | `/api/reports/generate` | Generate AI monthly report | Admin |
| POST | `/ai/duplicate-check` (internal) | Fuzzy match beneficiary | Internal Key |
| POST | `/ai/generate-report` (internal) | Claude-based report draft | Internal Key |

---

## 10. Risks & Mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Claude API downtime / rate limits | Reports cannot be generated | Add retries + fallback "manual draft" template |
| R2 | False positives in duplicate detection | Staff frustration | Tunable similarity threshold (default 85%) + always allow override |
| R3 | RTL/Arabic rendering bugs | Poor UX for primary audience | Dedicated RTL QA pass each sprint |
| R4 | 8-week timeline pressure | Scope slip | Strict MVP scope; defer non-essential items to v2 |
| R5 | Data privacy (PII) | Compliance/security risk | Encryption at rest, audit logging, role-based access |

---

## 11. Glossary

| Term | Definition |
|---|---|
| MVP | Minimum Viable Product |
| RBAC | Role-Based Access Control |
| JWT | JSON Web Token |
| NGO | Non-Governmental Organization |
| RTL | Right-to-Left (text direction) |
| KPI | Key Performance Indicator |
| PII | Personally Identifiable Information |
| Fuzzy Matching | String similarity algorithm tolerating typos/variants |

---

## 12. Document Approval

| Role | Name | Signature | Date |
|---|---|---|---|
| Product Owner | | | |
| Scrum Master | | | |
| Tech Lead (Laravel) | | | |
| Tech Lead (Python AI) | | | |

---

## 13. Definition of Done (DoD) & Error Handling Guidelines

- **Definition of Done (DoD) for Sprints:** A user story is only considered 'Done' if it has: (1) Peer code review completed, (2) No hardcoded credentials/API keys in the repository, (3) Basic unit testing passed, and (4) Proper Arabic RTL alignment verified on standard screen sizes.
- **Global Error Handling:** All APIs must return standardized JSON error payloads. For instance, if the Python AI microservice is unreachable, Laravel must log a critical error and return a graceful custom message to the client (e.g., 'Verification service temporarily unavailable') instead of crashing or returning raw system errors.
- **Mock API Standards for Sprint 1:** Mock endpoints deployed in Sprint 1 must simulate realistic network latency (e.g., 500ms delay) and include test cases for both successful operations and failure states (e.g., simulating invalid National ID format errors).

---

*End of Document — MASAR SRS v1.4 Final*
