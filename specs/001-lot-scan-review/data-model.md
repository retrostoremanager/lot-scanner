# Phase 1 Data Model: Lot Scan & Review

## LotScanSession

One photographing event for one customer's lot.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `employee_id` | text | Who ran the scan (FR-011); populated from the authenticated request identity (plan.md Technical Context) |
| `status` | enum: `processing` \| `in_review` \| `confirmed` \| `discarded` \| `failed` | |
| `photo_url` | text | Blob storage reference to the captured photo |
| `error_message` | text, nullable | Set only when `status = failed` |
| `created_at` | timestamptz | |
| `confirmed_at` | timestamptz, nullable | Set only on confirm |

**State transitions**: `processing` (AI call in flight) → `in_review` (items landed) or
`failed` (AI call errored/timed out, contracts/lot-scan-api.md `GET` response). From
`in_review`: → `confirmed` (via confirm action, FR-009) or → `discarded` (employee
abandons the session, FR-015, `POST /lot-scans/{id}/discard`). No transition out of
`confirmed`/`discarded`/`failed` (immutable once finalized, supporting Principle I —
honest, one-way confirmation); a `failed` scan is retried via a new session, not by
reviving this one.

## ScannedItem

One physical game detected within a session.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `session_id` | uuid (FK → LotScanSession) | |
| `suggested_catalog_game_id` | text, nullable | `api-gamedb` catalog ID the AI proposed; null if unidentified |
| `suggested_title` / `suggested_platform` / `suggested_variant` | text, nullable | AI's raw guess, kept even after correction for later accuracy analysis (SC-002) |
| `suggested_price` | numeric, nullable | Catalog reference price for the suggested match |
| `confidence` | numeric (0-1), nullable | Primarily the catalog fuzzy-match score, with the vision model's self-reported certainty as a secondary signal where available — see research.md #3/#6 on why this isn't a single calibrated number |
| `final_catalog_game_id` | text, nullable | Set once accepted or corrected |
| `final_title` / `final_platform` / `final_variant` | text, nullable | |
| `final_price` | numeric, nullable | May be manually overridden per FR-014 |
| `state` | enum: `pending` \| `accepted` \| `corrected` \| `excluded` \| `unidentified` | |
| `manually_entered` | boolean | True if the employee typed details with no catalog match (FR-008) |

**Validation rules**:
- A `ScannedItem` can only move into the session's final accepted list (contributing to
  the confirmed quote) if `state` is `accepted` or `corrected` — `excluded` and
  unresolved `unidentified`/`pending` items are left out (FR-009).
- `final_price` MUST be set for any item in `accepted`/`corrected` state before the
  parent session can transition to `confirmed`.

**State transitions**: `pending` (AI result landed) → one of `accepted` (confirmed as-is,
including any manual price edit via FR-014), `corrected` (catalog match swapped via
FR-006), `excluded` (left out), or — if the AI had no match at all — starts directly at
`unidentified` → either `corrected`/`accepted` (after manual entry) or `excluded`.

## CatalogGame (external reference, not owned by this database)

Represents an existing `api-gamedb` catalog entry. Not stored in `lotscanner`'s
database — looked up live via `api-gamedb`'s HTTP API for both AI-match resolution and
the employee's manual-correction search (research.md #4).

| Field | Notes |
|---|---|
| `id` | Catalog's own identifier, stored on `ScannedItem` as a foreign reference only |
| `title` / `platform` / `variant` / `reference_price` | Read-only from this product's perspective |
