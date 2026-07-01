# Module SRS: Tasks & Workflow

Status: Draft v1.0
Parent doc: `docs/MASAR_SRS_v2.0.md` (Section 5)
Build order: 2 of 6 (simplest domain module, no AI dependency)

---

## 1. Purpose

A lightweight task/Kanban system so managers can assign work and track progress across any department, independent of which other domain modules (Field Ops, HR, Logistics) are active.

## Identity model (shared across all modules)

This module does not define its own user identity or login logic. Every user is identified platform-wide by the `employee_code` (canonical routing/identity key) linked 1:1 to their `email` (login credential), as defined in `01-it-admin.md` Section 2a. Any `*_id` field referencing a user in this module (assignee, officer, requester, etc.) resolves back to that same identity.

## 2. In scope

- Create tasks with title, description, assignee, due date, priority.
- Kanban board with statuses: To Do, In Progress, Review, Done.
- Status history per task (who moved it, when).
- Task comments.
- Filter/search by assignee, status, due date.

## 3. Out of scope (MVP)

- Recurring/repeating tasks.
- Gantt charts / dependency chains between tasks.
- Time tracking / timesheets.

## 4. Data model

```
tasks: id, tenant_id, title, description, assignee_id, created_by, priority, status, due_date, created_at
task_status_history: id, task_id, from_status, to_status, changed_by, changed_at
task_comments: id, task_id, user_id, body, created_at
```

## 5. API endpoints

```
GET/POST  /api/tasks
PATCH     /api/tasks/{id}                (update fields)
PATCH     /api/tasks/{id}/status         (move on Kanban board)
POST      /api/tasks/{id}/comments
GET       /api/tasks?assignee=&status=&due_before=
```

## 6. User stories

- **US-1**: As a Manager, I want to create a task with a deadline and assign it to a staff member, so operational work is tracked.
  - AC: Task appears on the assignee's "My Tasks" view immediately; assignee gets a notification.
- **US-2**: As a Staff member, I want to drag a task across the Kanban board to update its status, so my manager sees progress without asking me.
  - AC: Status change is recorded in `task_status_history` with actor and timestamp.
- **US-3**: As a Manager, I want to filter tasks by assignee and due date, so I can see who's overloaded or who has overdue work.
- **US-4**: As a Staff member, I want to comment on a task, so I can ask questions or note blockers without a separate channel.
- **US-5**: As a Manager, I want to see a count of overdue tasks on the dashboard, so I know where to follow up.

## 7. Acceptance criteria (module-level)

- Only the assignee, the creator, or a user with a `task.manage` permission can edit a task.
- All task data is tenant-scoped; no cross-tenant visibility.
- Kanban board loads and reflects status changes within the same session without a full page reload.