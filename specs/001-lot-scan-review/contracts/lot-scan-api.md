# Contract: Lot Scan API (`fn-lot-scanner`)

HTTP API exposed by the Azure Function to the React Native client. All endpoints require
the employee's auth token (reuses RSM's existing auth, mechanism TBD at implementation —
not re-decided here; the resulting identity is what populates `employeeId`, FR-011).

Every item in every response below includes both the AI's original suggestion and the
current final values (null until accepted/corrected), so the client can always render
"what the AI said" vs. "what's actually going to be quoted":

```json
{
  "itemId": "uuid",
  "state": "pending|accepted|corrected|excluded|unidentified",
  "suggestedTitle": "string|null",
  "suggestedPlatform": "string|null",
  "suggestedVariant": "string|null",
  "suggestedPrice": 0.00,
  "confidence": 0.0,
  "finalCatalogGameId": "string|null",
  "finalTitle": "string|null",
  "finalPlatform": "string|null",
  "finalVariant": "string|null",
  "finalPrice": 0.00,
  "manuallyEntered": false
}
```
This is the "item shape" referenced by every endpoint below.

## `POST /lot-scans`

Create a session and kick off AI identification for one photo.

**Request**: multipart/form-data — `photo` (image).

**Response** `202 Accepted`:
```json
{ "sessionId": "uuid", "status": "in_review" }
```

Identification runs asynchronously (research.md #3 target: ~10s); client polls
`GET /lot-scans/{id}` until `status` leaves `processing` (see status values below).

## `GET /lot-scans/{id}`

Fetch a session and its items (for the review screen, and to resume after an
app background/close per FR-010).

**Response** `200 OK`:
```json
{
  "sessionId": "uuid",
  "status": "processing|in_review|confirmed|discarded|failed",
  "errorMessage": "string|null",
  "items": [ /* item shape, see above — empty while status is "processing" */ ]
}
```

`status: "failed"` (with `errorMessage` set) is returned if the AI identification call
errors out or times out, so the client can distinguish "still working" from "broken"
instead of polling forever. A failed session can be retried by issuing a new
`POST /lot-scans` with the same photo; this endpoint does not auto-retry.

## `PATCH /lot-scans/{id}/items/{itemId}`

Apply one review action to one item — accept, correct, exclude, or manually enter
(FR-005, FR-006, FR-008), and/or override the price (FR-014). Independent of other
items' state (Principle II — no bulk round-trip required for a single correction).

**Request** (any subset relevant to the action):
```json
{
  "state": "accepted|corrected|excluded|unidentified",
  "finalCatalogGameId": "string|null",
  "finalTitle": "string|null",
  "finalPlatform": "string|null",
  "finalVariant": "string|null",
  "finalPrice": 0.00
}
```
`finalCatalogGameId` is set when the employee picks an existing catalog entry (search
result or AI suggestion) — in that case `finalTitle`/`finalPlatform`/`finalVariant` are
filled in by the server from the catalog, not the client. `finalTitle`/`finalPlatform`/
`finalVariant` are set directly by the client (with `finalCatalogGameId` left null) only
for the no-catalog-match, manual-entry path (FR-008); the server sets
`manuallyEntered: true` whenever a `PATCH` sets final fields without a
`finalCatalogGameId`.

**Response** `200 OK`: the updated item (item shape, see above).

## `PATCH /lot-scans/{id}/items:bulk`

Select-all / deselect-all (FR-004) in one round trip rather than N `PATCH` calls.

**Request**:
```json
{ "state": "accepted|pending", "itemIds": ["uuid", "..."] }
```

**Scoping rule**: this endpoint only ever affects items whose *current* state is
`pending` or `accepted` — any `corrected`, `excluded`, or `unidentified` item passed in
`itemIds` is left untouched and returned as-is. This guarantees select-all/deselect-all
can never silently revert a correction or re-include an excluded item (spec Acceptance
Scenario 2.2).

**Response** `200 OK`: the full updated item list (item shape, see above).

## `POST /lot-scans/{id}/confirm`

Finalize the session (FR-009). Fails with `409 Conflict` if any item is still `pending`
or `unidentified` with no final values — nothing is silently included or excluded.

**Response** `200 OK`:
```json
{
  "sessionId": "uuid",
  "status": "confirmed",
  "confirmedAt": "ISO-8601",
  "items": [ /* item shape, see above — final accepted/corrected items only */ ]
}
```

## `POST /lot-scans/{id}/discard`

Abandon an in-review session without confirming it (FR-015). No-op if the session is
already `confirmed` or `discarded` (idempotent).

**Response** `200 OK`: `{ "sessionId": "uuid", "status": "discarded" }`

## Catalog search (proxied — for manual correction, FR-006)

`GET /lot-scans/catalog-search?q={text}` — thin proxy to `api-gamedb`'s existing search
endpoint, so the client only needs to talk to one backend. Response shape matches
`api-gamedb`'s own catalog search contract (not redefined here).
