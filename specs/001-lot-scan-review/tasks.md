# Tasks: Lot Scan & Review

**Input**: Design documents from `/specs/001-lot-scan-review/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/lot-scan-api.md, quickstart.md (all present)

**Organizing principle (deviation from spec-kit's default)**: The constitution's
Principle IV ("Vertical Slices, One at a Time") and Development Workflow section
require this slice's tasks to be delivered as exactly three GitHub work items — one
database item, one function-app item, one frontend item — not one item per user story.
Phases below are therefore organized **by component**, not by user story. Every task
still carries a `[US#]` label for traceability back to spec.md's user stories, and each
component phase is internally ordered P1 → P2 → P3 so an MVP can still be cut early
within a component if needed.

**Tests**: Included. `plan.md`'s Technical Context already commits to a testing stack
(xUnit/Moq/FluentAssertions; Jest/RNTL) matching RSM convention, so tests are treated as
in-scope deliverables, not optional extras.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Maps to spec.md's user stories (US1/US2/US3)

## Path Conventions (per plan.md Project Structure)

- `db-lot-scanner/` — THE database work item
- `fn-lot-scanner/` — THE function-app work item
- `app-lot-scanner/` — THE frontend work item

None of these three repos exist yet; T001-T003 create them.

---

## Phase 1: Setup

- [ ] T001 [P] Create `db-lot-scanner` repo with `migrations/` and `schema/` directories (plan.md Project Structure)
- [ ] T002 [P] Create `fn-lot-scanner` repo: .NET 10 isolated-worker Azure Functions project with `Functions/`, `Services/`, `Models/`, `Repos/` folders (plan.md Project Structure)
- [ ] T003 [P] Create `app-lot-scanner` repo: Expo (managed) TypeScript project with `app/`, `src/components/`, `src/api/`, `src/state/` folders (plan.md Project Structure)

**Checkpoint**: All three repos exist with empty scaffolding.

---

## Phase 2: Foundational (Blocking Prerequisites)

**⚠️ CRITICAL**: No component-phase work below can begin until this phase is complete.

- [ ] T004 Provision the `lotscanner` database on `db-gamedb`'s existing Azure Postgres Flexible Server (research.md #5 — no new server)
- [ ] T005 [P] Configure `fn-lot-scanner`'s HTTP client for `api-gamedb`'s catalog search/lookup API (research.md #4 — proxy, not direct DB access)
- [ ] T006 [P] Configure `fn-lot-scanner`'s Anthropic Claude API client for vision calls (research.md #3)
- [ ] T007 Resolve how `fn-lot-scanner` obtains employee identity from the authenticated request (reuse RSM's existing auth scheme; plan.md Technical Context note on FR-011) — needed before any function sets `employee_id`
- [ ] T008 [P] Set up xUnit + Moq + FluentAssertions test project for `fn-lot-scanner` (plan.md Testing)
- [ ] T009 [P] Set up Jest + React Native Testing Library for `app-lot-scanner` (plan.md Testing)

**Checkpoint**: Database reachable, catalog/AI clients configured, auth resolved, test harnesses in place.

---

## Phase 3: Database (`db-lot-scanner`) — THE database work item

**Goal**: Persist `LotScanSession` and `ScannedItem` exactly as specified in data-model.md, with the validation rule enforced at the schema level.

**Independent Test**: Apply migrations to a fresh `lotscanner` database; insert a session + items by hand; confirm the final-price CHECK constraint rejects an `accepted` item with a null `final_price`.

- [ ] T010 [P] [US1] Migration for `lot_scan_sessions`: `id` uuid PK, `employee_id` text not null, `status` enum (`processing`\|`in_review`\|`confirmed`\|`discarded`\|`failed`), `photo_url` text, `error_message` text nullable, `created_at` timestamptz not null default now(), `confirmed_at` timestamptz nullable — `db-lot-scanner/migrations/001_lot_scan_sessions.sql` (data-model.md LotScanSession)
- [ ] T011 [P] [US1] Migration for `scanned_items`: `id` uuid PK, `session_id` uuid FK → `lot_scan_sessions`, `suggested_catalog_game_id` text nullable, `suggested_title`/`suggested_platform`/`suggested_variant` text nullable, `suggested_price` numeric nullable, `confidence` numeric(0-1) nullable, `final_catalog_game_id` text nullable, `final_title`/`final_platform`/`final_variant` text nullable, `final_price` numeric nullable, `state` enum (`pending`\|`accepted`\|`corrected`\|`excluded`\|`unidentified`), `manually_entered` boolean not null default false — `db-lot-scanner/migrations/002_scanned_items.sql` (data-model.md ScannedItem)
- [ ] T012 [US2] CHECK constraint: `final_price IS NOT NULL` whenever `state IN ('accepted','corrected')` — "`final_price` MUST be set for any item in `accepted`/`corrected` state before the parent session can transition to `confirmed`" (data-model.md validation rule) — `db-lot-scanner/migrations/003_scanned_items_final_price_check.sql`
- [ ] T013 [P] [US2] Indexes on `scanned_items(session_id)` and `scanned_items(final_catalog_game_id)` to support correction-search lookups — `db-lot-scanner/migrations/004_indexes.sql`
- [ ] T014 [US1] Verify all migrations apply cleanly to a fresh `lotscanner` database on `db-gamedb`'s server; document the connection string pattern — `db-lot-scanner/README.md`

**Checkpoint**: Schema exists, matches data-model.md field-for-field, validation rule enforced by the database itself.

---

## Phase 4: Function App (`fn-lot-scanner`) — THE function-app work item

**Goal**: Implement every endpoint in `contracts/lot-scan-api.md` against the schema from Phase 3.

**Independent Test**: Run `quickstart.md` scenarios 1-7 against this function app with a mocked or seeded `lotscanner` database and a fixture `api-gamedb` instance — all seven should pass without the frontend existing yet (use curl/Postman).

- [ ] T015 [P] [US1] EF Core models for `LotScanSession` and `ScannedItem` matching the migrations exactly (field names, nullability, enums) — `fn-lot-scanner/src/Models/LotScanSession.cs`, `fn-lot-scanner/src/Models/ScannedItem.cs`
- [ ] T016 [US1] Postgres repository (EF Core `DbContext` + CRUD) — `fn-lot-scanner/src/Repos/LotScanRepository.cs` (depends on T015)
- [ ] T017 [US1] AI identification service: send photo to Claude vision, resolve candidates against `api-gamedb` via fuzzy match, derive `confidence` as primarily the fuzzy-match score with vision self-report as a secondary signal (research.md #3/#6) — `fn-lot-scanner/src/Services/IdentificationService.cs`
- [ ] T018 [US1] `POST /lot-scans`: accept photo upload, create session (`status = processing`), store photo in blob storage, resolve `employee_id` (T007), kick off async identification — `fn-lot-scanner/src/Functions/CreateScan.cs` (contracts/lot-scan-api.md)
- [ ] T019 [US1] `GET /lot-scans/{id}`: return session + items in the documented item shape, including `final*` fields and `manuallyEntered` — `fn-lot-scanner/src/Functions/GetScan.cs`
- [ ] T020 [US1] AI-call failure handling: on identification error/timeout, set `status = failed` + `error_message` instead of leaving the session stuck in `processing` (spec.md Edge Case, contracts/lot-scan-api.md `failed` status) — `fn-lot-scanner/src/Services/IdentificationService.cs`
- [ ] T021 [US1] `PATCH /lot-scans/{id}/items:bulk`: apply select-all/deselect-all, scoped to only items currently `pending`/`accepted` — any `corrected`/`excluded`/`unidentified` item in the request is left untouched (contracts/lot-scan-api.md scoping rule, spec Acceptance Scenario 2.2) — `fn-lot-scanner/src/Functions/BulkUpdateItems.cs`
- [ ] T022 [US1] `POST /lot-scans/{id}/confirm`: `409 Conflict` if any item is still `pending` or `unidentified` with no final values (FR-009); otherwise set `status = confirmed` + `confirmed_at` — `fn-lot-scanner/src/Functions/ConfirmScan.cs`
- [ ] T023 [P] [US1] `POST /lot-scans/{id}/discard`: idempotent no-op if already `confirmed`/`discarded`, else set `status = discarded` (FR-015) — `fn-lot-scanner/src/Functions/DiscardScan.cs`
- [ ] T024 [US2] `PATCH /lot-scans/{id}/items/{itemId}` — catalog-correction path: given `finalCatalogGameId`, server fills `finalTitle`/`finalPlatform`/`finalVariant`/`finalPrice` from `api-gamedb`, sets `state = corrected` (FR-006) — `fn-lot-scanner/src/Functions/UpdateItem.cs`
- [ ] T025 [US2] `GET /lot-scans/catalog-search`: thin proxy to `api-gamedb`'s existing search endpoint (FR-006) — `fn-lot-scanner/src/Functions/CatalogSearch.cs`
- [ ] T026 [US2] Manual price override on `PATCH /lot-scans/{id}/items/{itemId}`: accept `finalPrice` with `finalCatalogGameId` unchanged, on any state (FR-014) — `fn-lot-scanner/src/Functions/UpdateItem.cs` (extends T024)
- [ ] T027 [US3] Manual-entry path on `PATCH /lot-scans/{id}/items/{itemId}`: `finalTitle`/`finalPlatform`/`finalVariant` set directly with `finalCatalogGameId` left null → `manuallyEntered = true` (FR-008) — `fn-lot-scanner/src/Functions/UpdateItem.cs` (extends T024)
- [ ] T028 [US3] Exclude path on `PATCH /lot-scans/{id}/items/{itemId}`: `state = excluded` — `fn-lot-scanner/src/Functions/UpdateItem.cs` (extends T024)
- [ ] T029 [P] xUnit tests for `IdentificationService` (confidence derivation, fuzzy-match resolution) and for the confirm/bulk-update scoping rules — `fn-lot-scanner/tests/`

**Checkpoint**: Every contract endpoint works end-to-end against a real `lotscanner` database; `quickstart.md` scenarios 1-7 pass without a frontend.

---

## Phase 5: Frontend (`app-lot-scanner`) — THE frontend work item

**Goal**: Implement the capture → review → confirm flow against `fn-lot-scanner`'s API.

**Independent Test**: With `fn-lot-scanner` running (Phase 4 complete), walk through `quickstart.md` scenarios 1-6 using the actual app UI instead of curl.

- [ ] T030 [P] [US1] Typed API client for every endpoint in `contracts/lot-scan-api.md` — `app-lot-scanner/src/api/lotScanClient.ts`
- [ ] T031 [P] [US1] Capture screen: camera permission + `expo-camera` capture + upload via `POST /lot-scans` — `app-lot-scanner/app/capture.tsx`
- [ ] T032 [US1] Polling hook for `GET /lot-scans/{id}` while `status = processing`, surfacing `status = failed` + `errorMessage` as a retry prompt instead of polling forever (spec.md Edge Case) — `app-lot-scanner/src/state/useScanSession.ts` (depends on T030)
- [ ] T033 [US1] Review screen: itemized card list rendering suggested vs. final fields; confidence badge visually distinguishing anything below the 0.7 threshold (FR-007, research.md #6) — `app-lot-scanner/app/review.tsx`, `app-lot-scanner/src/components/ItemCard.tsx`
- [ ] T034 [US1] Select-all/deselect-all control calling `PATCH /lot-scans/{id}/items:bulk` — `app-lot-scanner/src/components/SelectAllBar.tsx`
- [ ] T035 [US1] Confirm action calling `POST /lot-scans/{id}/confirm`; surface a `409` as "finish reviewing every item first," not a generic error — `app-lot-scanner/app/review.tsx`
- [ ] T036 [US1] Persist the active session id locally so backgrounding/closing the app before confirming doesn't lose the in-progress review (FR-010) — `app-lot-scanner/src/state/useScanSession.ts`
- [ ] T037 [P] [US1] Discard action (confirmation prompt → `POST /lot-scans/{id}/discard`) (FR-015) — `app-lot-scanner/app/review.tsx`
- [ ] T038 [US2] Per-card correction flow: tap card → catalog search (`GET /lot-scans/catalog-search`) → select → `PATCH` with `finalCatalogGameId` (FR-006) — `app-lot-scanner/app/item-correct.tsx`
- [ ] T039 [US2] Manual price-override control on any card, correctly identified or not (FR-014) — `app-lot-scanner/src/components/ItemCard.tsx`
- [ ] T040 [US3] Unidentified-item UI: manual-entry form (title/platform/variant) or exclude action, `PATCH`ing accordingly (FR-008) — `app-lot-scanner/src/components/UnidentifiedItemCard.tsx`
- [ ] T041 [P] Jest + RNTL tests for `ItemCard`, `SelectAllBar`, and `useScanSession` — `app-lot-scanner/__tests__/`

**Checkpoint**: All three user stories are usable end-to-end through the actual app.

---

## Phase 6: Polish & Cross-Cutting

- [ ] T042 [P] Run `quickstart.md` scenarios 1-7 end-to-end against all three deployed dev components together
- [ ] T043 [P] Structured logging across `fn-lot-scanner` functions for scan/update/confirm/discard actions (supports FR-011's audit requirement) — `fn-lot-scanner/src/Functions/`
- [ ] T044 Timed trial with a real 10-20 item lot to check SC-001 (<5 min), SC-002 (≥70% correct), SC-003 (<15s per correction); record results in `specs/001-lot-scan-review/` as a follow-up note

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies.
- **Foundational (Phase 2)**: Depends on Setup — blocks all three component phases.
- **Database (Phase 3)**: Depends on Foundational. Blocks Function App (schema must exist first).
- **Function App (Phase 4)**: Depends on Database (Phase 3) + Foundational's catalog/AI client config (T005, T006) and auth resolution (T007). Blocks Frontend for any task that calls a real endpoint.
- **Frontend (Phase 5)**: Depends on Function App (Phase 4) for real integration; T030 (typed client) and T031 (capture screen UI only) can start against the *contract* in parallel with late Function App work, but T032-T040 need working endpoints to test against.
- **Polish (Phase 6)**: Depends on all three component phases.

This is a deliberately **serial, component-ordered** chain (DB → Function App → Frontend), not the parallel-by-user-story model spec-kit defaults to — see the Organizing principle note at the top.

### Within Each Component Phase

- Tasks are still ordered P1 → P2 → P3 by `[Story]` label, so a component can be cut short at a story boundary if needed (e.g., ship Function App with only US1 done, revisit US2/US3 later) — matching the constitution's "one work item at a time" cadence without forcing all three stories to land simultaneously within a component.

### Parallel Opportunities

- T001-T003 (repo creation) — fully parallel.
- T005, T006, T008, T009 — parallel within Foundational.
- T010, T011 (the two base migrations) — parallel; T012-T013 depend on both.
- T015 — parallel for the two models; most Function App tasks after that are sequential (same files/dependencies).
- T030, T031 — parallel; most Frontend tasks after that are sequential.

---

## Parallel Example: Database Phase

```bash
Task: "Migration for lot_scan_sessions in db-lot-scanner/migrations/001_lot_scan_sessions.sql"
Task: "Migration for scanned_items in db-lot-scanner/migrations/002_scanned_items.sql"
```

---

## Implementation Strategy

### One component at a time (per constitution Principle IV)

1. Phase 1 (Setup) → Phase 2 (Foundational).
2. Phase 3 (Database) — ship as one GitHub work item, fully done (all three stories' schema needs, since the schema cost of including all 5 status values / 5 item states up front is trivial compared to migrating later).
3. Phase 4 (Function App) — ship as one GitHub work item. Within it, US1's tasks (T015-T023) are the MVP; US2 (T024-T026) and US3 (T027-T028) can follow in the same work item or a quick follow-up, per Samuel's call at `/speckit-taskstoissues` time.
4. Phase 5 (Frontend) — ship as one GitHub work item, same US1-first internal ordering.
5. Phase 6 (Polish) once all three land.

### MVP slice within this structure

Phase 1 + 2 + Phase 3 (full) + Phase 4's US1 tasks only + Phase 5's US1 tasks only =
a working bulk-confirm flow (User Story 1) with no correction/unidentified handling yet
— deployable and demoable before US2/US3 are built.

---

## Notes

- Every `[P]` pair touches different files with no unmet dependency.
- `[Story]` labels are for traceability only in this task list — delivery grouping for
  GitHub work items is by *component* (Phase 3/4/5), per the constitution.
- Commit after each task or logical group; stop at any Checkpoint to validate independently.
