# F5 — Projects, questions, runs and project roles

**Milestone:** M1 Deployable shell · **Status:** ⬜ Not started · **Depends on:** [F1](f01-cargo-workspace.md) · **Unblocks:** [F6](f06-job-queue-launcher.md), [F8](../m02-local-knowledge/f08-model-routing.md), [H3](../mh-pipeline-core/h03-turn-store-tracer.md)

[← Back to plan](../PLAN.md)

## Goal

Create the core data model and the first real pages: projects that hold questions, which hold runs. Projects are private until shared, and each member gets a project role. Every action records who did it. This is the spine that research, the ledger and chat all hang from.

## Scope

**In scope**

- SQLite (WAL mode) via `sqlx`, with migrations that run at hub startup.
- Tables for projects, questions, runs, project memberships and an activity log.
- Project roles (owner, editor, viewer) and their permission checks.
- Sharing: the owner adds members by user.
- Attribution: every write records the acting user in the activity log.
- htmx pages for the Projects list, Project page, and a basic New question form. Phone-first.

**Out of scope**

- Running anything: job queue → [F6](f06-job-queue-launcher.md); real research → [F12](../m02-local-knowledge/f12-quick-depth-e2e.md), [F24](../m06-async-research/f24-planner-checkpoint.md).
- Depth, profile and budget settings in the New question form. Fields can exist, but behavior comes later: [F24](../m06-async-research/f24-planner-checkpoint.md), [F25](../m06-async-research/f25-fan-out-budget.md), [F41](../m10-profiles-media/f41-profile-system.md).
- The claim ledger, sources and notes on the Project page → [F13](../m03-ledger-verification/f13-claim-ledger.md) and later.
- An activity feed UI and comments with mentions. These are gap items in PLAN.md.
- Watches → [F27](../m06-async-research/f27-watches.md).

## Design notes

**Organization** (Design: Knowledge store › Organization)

- Project → questions → runs. A question can be rerun or put on a watch.

**Project roles** (Design: Teams and permissions › Roles)

| Role | Can |
| --- | --- |
| Owner | Share the project, set its default profile and budget, delete it |
| Editor | Start runs and reviews, chat, edit the claim ledger, save outputs |
| Viewer | Read reports, sources, outputs and chat threads; comment |

Instance admins and members come from [F4](f04-accounts-auth.md). Members can create projects.

**Sharing and privacy** (Design: Teams and permissions › Sharing and privacy)

- Projects are private to their creator until shared.
- Personal notes and datasets are private by default. Adding one to a project is an explicit share. This is relevant from F11 and F26 on; the schema should leave room for it.

**Attribution** (Design: Teams and permissions › Attribution and coordination)

- Every run, review, ledger edit and accepted verdict records who did it, shown in a project activity feed.

**Async** (Design: Web interface › Async behavior): all state lives in SQLite on the host, so closing the tab changes nothing.

**Screens** (Design: Web interface › Screens): Projects ("list and create projects; each shows its open questions and last activity"), Project, and New question ("type or dictate a question, pick a depth… set a budget").

**Run states** (Design: Research pipeline): runs pause in `awaiting_approval` after planning. Other states are implied: queued, running, finished, failed.

**Operations** (Design: Deployment › Upgrades and operations): migrations run at hub startup.

## Implementation sketch

Suggestions only:

- **Migrations**: `sqlx::migrate!` embedded in the binary. Set `PRAGMA journal_mode=WAL`, `foreign_keys=ON` and `busy_timeout` on every connection.
- **Tables**:
  - `projects(id, name, description, owner_id, default_profile, default_budget, created_at, updated_at, deleted_at)`
  - `project_members(project_id, user_id, role, added_by, added_at)` (the owner is also a member row)
  - `questions(id, project_id, text, depth, profile, budget, created_by, created_at)`
  - `runs(id, question_id, status, depth, profile, started_by, created_at, started_at, finished_at, error)`
    - `status` ∈ `queued | running | awaiting_approval | finished | failed | cancelled`
  - `activity(id, project_id, actor_id, verb, object_type, object_id, at, details_json)`
- **Authorization**: one helper, `require_project_role(user, project, min_role)`, used by every handler. Admins don't automatically see private projects. The design says projects are private to their creator.
- **Writes and activity**: wrap each write and its activity row in one transaction.
- **Pages**: askama templates with htmx forms. One column and large tap targets. Make "New question" a prominent button on the Project page.
- **IDs**: ULIDs or UUIDv7 give sortable, unguessable IDs in URLs.

## Acceptance criteria

- [ ] Create a project from a phone; a second user sees it only after being added
- [ ] Migrations run automatically on hub startup against an empty and an existing database.
- [ ] A viewer can't create questions or runs (403). An editor can. Only the owner can share or delete.
- [ ] Every create, share and delete writes an activity row with the acting user.
- [ ] Creating a question with a placeholder run records `started_by` and shows on the Project page.
- [ ] Pages render usably at 380 px wide.

## Open questions

- Activity feed UI: F5 records the data, but no feature builds the feed screen. See the PLAN.md gap item.
- Should instance admins be able to see or delete any project for moderation? The design is silent. The default here is no.
- Does deleting a project soft-delete it? Sources are shared across projects, so hard deletes need care later.
