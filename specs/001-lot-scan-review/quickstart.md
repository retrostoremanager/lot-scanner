# Quickstart: Validating Lot Scan & Review

Prerequisites: `fn-lot-scanner` running locally (Azurite + local settings pointing at a
test `lotscanner` Postgres DB and a test `api-gamedb` instance/catalog fixture),
`app-lot-scanner` running in Expo Go or a simulator pointed at the local function app.

## 1. Happy path — bulk confirm (User Story 1)

1. Photograph (or load a fixture image of) a lot with 5-10 known games.
2. `POST /lot-scans` with the photo → expect `202` with a `sessionId`.
3. Poll `GET /lot-scans/{id}` until `items` is populated (expect < ~10s per
   research.md #3) → verify one item per physical game, each with
   title/platform/price/confidence (contract: `lot-scan-api.md`).
4. In the app, tap "select all", then confirm → `POST /lot-scans/{id}/confirm` → expect
   `200` with `status: "confirmed"`.
5. Verify in the `lotscanner` DB that the `LotScanSession` row is `confirmed` and every
   `ScannedItem` has `state IN (accepted, corrected)` with a non-null `final_price`
   (data-model.md validation rule) — proves SC-004 (nothing silently included).

## 2. Correction path (User Story 2)

1. Using a fixture with one deliberately mislabeled item, repeat steps 1-3 above.
2. `PATCH /lot-scans/{id}/items/{itemId}` with a corrected `finalCatalogGameId` →
   expect `200` and `state: "corrected"`.
3. Confirm the session → verify only the corrected item's final fields differ from its
   suggested fields; all other items are untouched (spec Acceptance Scenario 2.2).
4. Time step 2 end-to-end → should be under 15s (SC-003).

## 3. Unidentified item path (User Story 3)

1. Using a fixture with one item the catalog can't match, repeat steps 1-3 above and
   verify that item's `state` is `unidentified`.
2. `PATCH /lot-scans/{id}/items/{itemId}` with `state: "excluded"`.
3. Confirm the session → verify the final item list excludes it and the rest confirm
   normally (spec Acceptance Scenario 3.2).

## 4. Manual price override (FR-014)

1. On any `accepted` item (correctly identified), `PATCH` a `finalPrice` different from
   `suggestedPrice` without changing `finalCatalogGameId`.
2. Confirm → verify the persisted `final_price` reflects the override, not the AI
   suggestion.

## 5. Resume after interruption (FR-010, Edge Case)

1. Start a scan, let items populate, then stop the client (simulating app close) before
   confirming.
2. Re-open and `GET /lot-scans/{id}` → verify the same in-review items are still there,
   none regenerated or lost.
