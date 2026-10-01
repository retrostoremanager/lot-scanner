# Contract: Lot Scan API (`fn-lot-scanner`)

HTTP API exposed by the Azure Function to the React Native client. All endpoints require
the employee's auth token (reuses RSM's existing auth, mechanism TBD at implementation —
not re-decided here).

## `POST /lot-scans`

Create a session and kick off AI identification for one photo.

**Request**: multipart/form-data — `photo` (image).

**Response** `202 Accepted`:
```json
{ "sessionId": "uuid", "status": "in_review" }
```

Identification runs asynchronously (research.md #3 target: ~10s); client polls
`GET /lot-scans/{id}` until items are populated.

## `GET /lot-scans/{id}`

Fetch a session and its items (for the review screen, and to resume after an
app background/close per FR-010).

**Response** `200 OK`:
```json
{
  "sessionId": "uuid",
  "status": "in_review",
  "items": [
    {
      "itemId": "uuid",
      "state": "pending",
      "suggestedTitle": "string",
      "suggestedPlatform": "string",
      "suggestedVariant": "string|null",
      "suggestedPrice": 0.00,
      "confidence": 0.0
    }
  ]
}
```

## `PATCH /lot-scans/{id}/items/{itemId}`

Apply one review action to one item — accept, correct, exclude, manually enter, or
price-override (FR-005, FR-006, FR-008, FR-014). Independent of other items' state
(Principle II — no bulk round-trip required for a single correction).

**Request** (any subset relevant to the action):
```json
{
  "state": "accepted|corrected|excluded|unidentified",
  "finalCatalogGameId": "string|null",
  "finalPrice": 0.00
}
```

**Response** `200 OK`: the updated item (same shape as in `GET`).

## `PATCH /lot-scans/{id}/items:bulk`

Select-all / deselect-all (FR-004) in one round trip rather than N `PATCH` calls.

**Request**:
```json
{ "state": "accepted|pending", "itemIds": ["uuid", "..."] }
```

**Response** `200 OK`: the full updated item list.

## `POST /lot-scans/{id}/confirm`

Finalize the session (FR-009). Fails with `409 Conflict` if any item is still `pending`
or `unidentified` with no final values — nothing is silently included or excluded.

**Response** `200 OK`:
```json
{ "sessionId": "uuid", "status": "confirmed", "confirmedAt": "ISO-8601", "items": [ ] }
```

## Catalog search (proxied — for manual correction, FR-006)

`GET /lot-scans/catalog-search?q={text}` — thin proxy to `api-gamedb`'s existing search
endpoint, so the client only needs to talk to one backend. Response shape matches
`api-gamedb`'s own catalog search contract (not redefined here).
