# Module SRS Set

Per-module requirements docs, derived from `docs/MASAR_SRS_v2.0.md`, written so each module's team can start building independently. Suggested build order (simplest/foundation first):

1. [IT & Admin](01-it-admin.md) — foundation: auth, RBAC, module toggling, helpdesk, assets, access lifecycle
2. [Tasks & Workflow](02-tasks-workflow.md) — simplest domain module, no external dependencies
3. [HR & Volunteer Management](03-hr-volunteers.md)
4. [Field Operations](04-field-operations.md) — pick 3 or 4 first depending on target customer
5. [Logistics & Supply Chain](05-logistics-supply-chain.md)
6. [M&E & Reporting](06-me-reporting.md) — highest complexity (AI dependency); build/harden last

Each doc includes: purpose, in/out of scope, data model, API endpoints, user stories with acceptance criteria, and module-level acceptance criteria.
