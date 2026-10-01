# lot-scanner

Specs and planning for **AI game lot scanning** — a standalone-marketable feature for
[RetroStoreManager](https://github.com/retrostoremanager): a customer drops off a lot of
games, staff photographs it, AI identifies and prices each item, staff confirms/corrects
in a review screen instead of pricing everything by hand.

## Product shape

- **Client**: React Native app (iOS + Android, shared codebase) — camera capture →
  AI-assisted item identification → itemized card review screen (select/deselect all,
  per-card confidence, quick correct) → confirm.
- **Backend**: Azure Function (AI scan/identify + pricing lookup).
- **Data**: a database for scan sessions / identified items / pricing, shape TBD during
  planning.

This repo holds the spec-kit specs and plans. Implementation will live in separate repos
(one for the RN frontend, one for the function app, one for the database), following the
same multi-repo pattern as RetroStoreManager's `web-mystore` / `fn-mystore` / `db-gamedb`.

## Workflow

This repo is set up with [GitHub spec-kit](https://github.com/github/spec-kit). Work is
planned one vertical slice at a time (one frontend item, one function app item, one
database item), using:

- `/speckit-constitution` — project principles
- `/speckit-specify` — baseline spec for a feature/slice
- `/speckit-plan` — implementation plan
- `/speckit-tasks` — actionable tasks
- `/speckit-taskstoissues` — push tasks out as GitHub work items
- `/speckit-implement` — execute

Run these from a Claude Code session rooted in this repo directory.
