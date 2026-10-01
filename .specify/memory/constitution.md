# lot-scanner Constitution

## Core Principles

### I. Honest Automation, Human Confirmation
Every AI-identified item (title, platform, variant, condition, price) MUST pass through
an explicit human review/confirm step before it is treated as accepted inventory or
offered to a customer. The product MUST NOT be marketed or built as fully automated
pricing — the pitch is "AI pre-fills, staff confirms," not "AI decides." No code path
may auto-commit an AI guess without a confirm action tied to it.
Rationale: the scanner is expected to be correct roughly 70-80% of the time, not 100%;
overselling full automation breaks trust the first time a demo misidentifies an item.

### II. Correction Speed Is the Product
The review screen (itemized card list, select/deselect all, per-card confidence, quick
correct/swap) MUST make confirming a correct guess and fixing a wrong one both fast.
Any feature that slows bulk confirmation or makes correcting a miss nearly as slow as
pricing it manually is a regression, not a tradeoff.
Rationale: the entire value proposition versus manual pricing is time saved; if
correction friction approaches manual-pricing time, the pitch collapses even at high
underlying accuracy.

### III. Shared Code First (React Native)
The client is a single React Native codebase targeting iOS and Android. Business logic,
API calls, and the review-screen UI MUST be written once in shared RN code. Native
modules or platform-specific code are permitted only where RN has no viable path (e.g.,
a camera capability RN libraries don't expose) and MUST be justified in the PR/spec that
introduces them.
Rationale: one wedge feature does not justify maintaining two native codebases; shared
code keeps a solo/small team's velocity high.

### IV. Vertical Slices, One at a Time
Work is planned and built as full vertical slices — one frontend item, one function-app
item, and one database item per cycle — tracked as GitHub work items via
`/speckit-taskstoissues`. A new slice MUST NOT start before the current slice's items are
implemented and verified merged (not just marked done).
Rationale: matches how Samuel wants this built given limited time, and avoids the
previously observed failure mode (see RetroStoreManager's `project-strata-done-not-merged`
memory) where "done" drifted from "merged."

### V. Azure-Native, Consistent with RetroStoreManager
The backend is an Azure Function, and any database choice follows the same Azure-native,
low-ops pattern already used by RetroStoreManager (`fn-mystore`, `db-gamedb`). New
infrastructure categories (non-Azure clouds, new hosting paradigms) require an explicit
justification in a spec before adoption.
Rationale: reuses operational knowledge and existing Azure subscription/tooling instead
of introducing a second ops surface for a one-person team to maintain.

## Product Constraints

- Target platforms: iOS and Android via React Native. No web client in scope for this
  repo's specs (RSM's existing web app is a separate product).
- This repo holds specs/plans only. Implementation lives in separate repos (frontend,
  function app, database), created once a slice's plan is ready to implement — mirroring
  RetroStoreManager's `web-mystore` / `fn-mystore` / `db-gamedb` split.
- Database technology is not yet decided; the first slice that needs persistence MUST
  decide and record it in that slice's plan, not here.

## Development Workflow

- Each vertical slice runs the full spec-kit chain: `/speckit-specify` → (optional
  `/speckit-clarify`) → `/speckit-plan` → `/speckit-tasks` → (optional
  `/speckit-analyze` / `/speckit-checklist`) → `/speckit-taskstoissues` →
  `/speckit-implement`.
- A slice's tasks are split so each GitHub work item maps to exactly one of: the RN
  frontend change, the Azure Function change, or the database change — not a mix.
- Before implementation on a slice begins, its spec and plan MUST be reviewed by Samuel.

## Governance

This constitution supersedes ad hoc practice for this repo. Amendments are made via
`/speckit-constitution`, require a stated rationale, and follow semantic versioning:
MAJOR for incompatible principle removals/redefinitions, MINOR for new or materially
expanded principles/sections, PATCH for wording/clarification only. Every spec and plan
produced in this repo should be checked against these principles before moving to tasks.

**Version**: 1.0.0 | **Ratified**: 2026-10-01 | **Last Amended**: 2026-10-01
