# Feature Specification: Lot Scan & Review

**Feature Branch**: `001-lot-scan-review`

**Created**: 2026-10-01

**Status**: Draft

**Input**: User description: "A store employee photographs a lot of physical games a
customer brings in. The app uses AI to identify each game (title, platform, variant) and
suggest a price. Because the AI will not be perfect (~70-80% expected accuracy), the
employee reviews an itemized card list of all detected items, with select/deselect-all
and per-card correction, before the lot is accepted."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Scan a lot and bulk-confirm AI suggestions (Priority: P1)

A store employee has a customer waiting with a lot of games. The employee opens the scan
screen, photographs the lot, and gets back a card for each detected game with a
suggested title, platform, variant, price, and confidence level. For the items the AI
got right, the employee uses select-all and confirms in bulk instead of pricing each one
by hand.

**Why this priority**: This is the core value proposition — replacing the fully manual
pricing loop. Without this, there is no product.

**Independent Test**: Photograph a known lot of games, verify the review screen lists one
card per physical item with a suggestion and confidence, select all, confirm, and verify
the resulting accepted list matches the lot.

**Acceptance Scenarios**:

1. **Given** a customer's lot of physical games, **When** the employee photographs it,
   **Then** the app returns one card per detected item, each with a suggested title,
   platform, variant (if applicable), price, and confidence level.
2. **Given** the review screen is showing identified items, **When** the employee taps
   "select all" and confirms, **Then** all shown items are accepted as-is into the final
   lot.
3. **Given** the review screen is showing identified items, **When** the employee taps
   "deselect all", **Then** no items are marked accepted and none are included if they
   confirm at that point.

---

### User Story 2 - Correct a misidentified item (Priority: P2)

One or more cards show the wrong game, platform, or variant. The employee corrects just
those cards — searching the catalog and swapping in the right game — without disturbing
the rest of the lot's accepted state.

**Why this priority**: Required to handle the ~20-30% of items the AI gets wrong without
forcing a full manual re-price of the whole lot.

**Independent Test**: Given a lot with one deliberately-misidentified item, correct that
one card via search and confirm the final list shows the corrected item while all other
items are unaffected.

**Acceptance Scenarios**:

1. **Given** a card shows a wrong title/platform/variant, **When** the employee taps the
   card and searches the catalog, **Then** they can select the correct game and the
   card's title, platform, variant, and price update accordingly.
2. **Given** one card has been corrected, **When** the employee confirms the lot,
   **Then** the corrected card's final values are used and no other card's accepted
   state or values change.
3. **Given** a card has low confidence, **When** the review screen renders, **Then** that
   card is visually distinguished from high-confidence cards so the employee notices it
   needs a closer look.

---

### User Story 3 - Handle an item the AI can't identify at all (Priority: P3)

A photographed item has no confident AI match (damaged label, obscure import, poor photo
angle). The employee needs to deal with that one item — enter it manually or exclude it
— without that item blocking confirmation of the rest of the lot.

**Why this priority**: Lots regularly include at least one oddball item; the flow must
degrade gracefully instead of failing the whole scan.

**Independent Test**: Given a lot that includes one item with no catalog match, verify
the review screen shows it as unidentified, and the employee can either exclude it or
enter it manually, then confirm the rest of the lot.

**Acceptance Scenarios**:

1. **Given** an item has no AI match, **When** the review screen renders, **Then** it is
   shown as "unidentified" with options to manually enter details or exclude it from the
   lot.
2. **Given** one item is unidentified and excluded, **When** the employee confirms,
   **Then** the final accepted list contains the remaining items and excludes that one.

---

### Edge Cases

- A lot contains multiple copies of the same game: each physical item MUST appear as its
  own card (not merged into a quantity), since each may need individual attention.
- The employee backgrounds or closes the app mid-review, before confirming: the in-progress
  scan session's results MUST still be there when they return to it (not silently lost).
- A photo is too blurry or poorly lit for any item in it to be identified: those items
  follow the same "unidentified" path as User Story 3, not a hard failure of the whole
  scan.
- The AI identification call itself errors out or times out (not a per-item miss, but a
  failure of the whole scan attempt): the employee MUST see that the scan failed (not an
  endless "processing" state) and be able to try again with a new photo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST let an employee capture exactly one photo per lot scan session
  for v1. Multi-photo and video capture (for large lots that don't fit in one frame) are
  explicitly out of scope for this slice (see Out of Scope).
- **FR-002**: System MUST analyze captured photo(s) and return one detected item per
  physical game, each with a suggested title, platform, variant (when applicable), a
  suggested price, and a confidence level.
- **FR-003**: System MUST present detected items as an itemized card list, one card per
  item.
- **FR-004**: Users MUST be able to select-all or deselect-all cards in a single action.
- **FR-005**: Users MUST be able to toggle an individual card's accepted state
  independently of the bulk select-all/deselect-all action.
- **FR-006**: Users MUST be able to correct a misidentified card by searching the catalog
  and selecting the correct game, updating only that card.
- **FR-007**: System MUST visually distinguish low-confidence cards from high-confidence
  cards.
- **FR-008**: System MUST support items with no AI match, flagging them as unidentified
  and letting the employee manually enter details or exclude them.
- **FR-009**: Users MUST be able to finalize ("confirm") the lot, producing a final list
  of accepted items with their prices; items never shown to the employee MUST NOT be
  silently included.
- **FR-010**: System MUST keep an in-progress scan session's results available if the app
  is backgrounded or closed before the employee confirms.
- **FR-011**: System MUST record which employee performed the scan and confirm actions,
  and when.
- **FR-012**: Confirming a lot MUST produce a priced quote only; it MUST NOT write
  accepted items into RetroStoreManager's existing inventory/POS system in this slice.
  The confirmed (and in-progress) session and its items MUST be persisted in this
  product's own database as a historical quote record — this is distinct from writing
  to sellable inventory, and is what satisfies FR-010/FR-011. Whether/how a confirmed
  quote later becomes trackable store inventory (with sold-status) is an explicit,
  deferred integration decision (see Out of Scope) — not resolved by this persistence.
- **FR-013**: Pricing suggestions MUST be based on title/platform/variant reference
  pricing only for v1; the AI MUST NOT attempt to estimate physical condition.
- **FR-014**: Users MUST be able to manually edit the suggested price (and item details)
  on any card during review — including cards the AI identified correctly — so the
  employee can account for condition or other adjustments by hand.
- **FR-015**: Users MUST be able to discard an in-review session without confirming it
  (e.g., the customer walks away mid-review). A discarded session MUST NOT contribute
  any items to a quote.

### Out of Scope (this slice)

- Multi-photo or video capture per lot (single photo only for v1 — candidate future
  enhancement).
- AI-estimated physical condition / condition-adjusted pricing (title/platform reference
  pricing only for v1; employee can still hand-adjust price per FR-014).
- Writing confirmed items into RetroStoreManager's sellable inventory/POS, and therefore
  tracking whether a quoted item was later actually sold — deliberately deferred because
  doing it before there's a real inventory-intake flow would create records that look
  like stock but can't be tracked as sold/unsold.

### Key Entities

- **Lot Scan Session**: One photographing event for one customer's lot. Has a status
  (in review / confirmed / discarded), the employee who ran it, and timestamps. Persists
  as a historical quote record in this product's own database — not as RSM inventory.
- **Scanned Item**: One physical game detected within a session. Has the AI-suggested
  title/platform/variant/price/confidence, the final accepted title/platform/variant/
  price (once reviewed), and a state (pending / accepted / corrected / excluded /
  unidentified).
- **Catalog Game**: An existing game catalog entry (title, platform, variant, reference
  price) — the source of AI suggestions and the target of manual corrections.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An employee can take a 10-20 item lot from first photo to a fully confirmed,
  priced list in under 5 minutes.
- **SC-002**: At least 70% of items in a typical lot are correctly identified by the AI
  without employee correction.
- **SC-003**: Correcting one misidentified card takes under 15 seconds from noticing the
  error to a corrected, confirmable card.
- **SC-004**: 100% of items in a confirmed lot reflect an explicit employee accept or
  correct action — none are included without having been shown to the employee.

## Assumptions

- The employee has a mobile device with a working camera and the lot-scanner app
  installed.
- A queryable game catalog (titles/platforms/variants/reference pricing) already exists
  and can be searched for corrections (RetroStoreManager's existing game catalog).
- This spec covers the scan-and-review experience and persisting it as a quote record;
  it does not cover inventory intake, trade-in payout, or POS integration (see Out of
  Scope).
- Network connectivity is available at the point of scanning (offline scanning is out of
  scope for this slice).
