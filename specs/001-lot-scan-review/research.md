# Phase 0 Research: Lot Scan & Review

## 1. React Native tooling: Expo (managed) vs. bare RN CLI

**Decision**: Expo, managed workflow, with `expo-camera` for capture.

**Rationale**: Camera capture and the review-screen UI (this slice's whole frontend
item) are both fully supported in Expo without ejecting. Expo also removes the need to
keep local Xcode/Android Studio toolchains current for day-to-day work, and EAS Build
can produce signed binaries without a full local native setup — a meaningful win for a
one-person team maintaining several products already.

**Alternatives considered**: Bare React Native CLI — gives more low-level control, but
none of this slice's requirements need it, and it adds native-toolchain maintenance with
no corresponding benefit yet.

## 2. Azure Function language/runtime

**Decision**: C# / .NET 8, isolated worker model — same as `fn-mystore` and
`api-gamedb`.

**Rationale**: Matches existing RSM backend conventions, CI/CD patterns, and test
stack (xUnit/Moq/FluentAssertions), so there's one set of operational knowledge across
all Azure Functions in the org instead of two.

**Alternatives considered**: Node.js/TypeScript Azure Functions, which would share a
language with the RN client — rejected because it would be the only non-.NET Function
app in the org, duplicating tooling/CI knowledge for marginal benefit.

## 3. AI item identification approach

**Decision**: Send the lot photo to Claude's vision capability to detect/segment
candidate items and propose a title/platform guess per item; resolve each candidate
against the existing game catalog (via `api-gamedb`) with fuzzy matching to attach a
canonical title/platform/variant and reference price. Confidence = a blend of the
vision model's own certainty and the catalog fuzzy-match score.

**Rationale**: No training pipeline or labeled dataset needed to ship v1; other RSM/
Samuel projects already use Claude for similar "identify + structure" tasks (portfolio
AI assistant, StayRecap report pipeline), so this reuses proven integration patterns.

**Alternatives considered**: A custom-trained object detection/classification model —
rejected for v1: far higher upfront cost (data collection, training, hosting) for a
feature whose whole pitch is being a fast, low-effort wedge.

## 4. Catalog access pattern

**Decision**: `fn-lot-scanner` calls `api-gamedb`'s existing HTTP API for catalog
search/lookup (both for AI-match resolution and for the employee's manual-correction
search), rather than connecting directly to `db-gamedb`'s Postgres instance.

**Rationale**: Preserves the service boundary RSM already established — `api-gamedb` is
the one place catalog-query logic (including auth, caching, fuzzy search) lives;
duplicating a direct DB connection in a second service would mean two places to update
catalog-query logic.

**Alternatives considered**: Direct Postgres connection to `db-gamedb` — rejected, same
duplication concern, and couples this new, possibly-separable product's deploy lifecycle
to `db-gamedb`'s schema changes more tightly than necessary.

## 5. Quote-record database sizing

**Decision (for this plan)**: A new, dedicated Postgres Flexible Server (burstable tier)
hosting a `lotscanner` database, separate from `db-gamedb`'s server.

**Rationale**: `lot-scanner` is explicitly being evaluated as a standalone-marketable
feature; keeping its data store on its own server avoids coupling its scaling/maintenance
lifecycle to `db-gamedb`'s, which matters more here than in a feature that will always
ship bundled with the rest of RSM.

**Alternatives considered**: A second database on `db-gamedb`'s existing Flexible Server
— cheaper (one fewer Azure resource bill) and still isolates data via a separate
database, but was rejected for now to keep this product's infra story clean in case it's
spun off. **Flag for Samuel**: this is a real monthly-cost tradeoff (a new Flexible
Server vs. a second DB on an existing one, roughly the difference between ~$0 incremental
and a new ~$15-25/mo line item per the pattern seen on `strata-reports` prod) — worth a
quick gut check before `db-lot-scanner` is actually provisioned, not before the plan.

## 6. Confidence threshold for "low confidence" flagging

**Decision**: Flag any item with blended confidence < 0.7 as low-confidence in the review
UI.

**Rationale**: Reasonable starting default; this is a tunable product constant, not an
architectural decision, and can be adjusted post-launch based on real scan data without
a spec/plan change.

**Alternatives considered**: None evaluated in depth — explicitly deferred as a tuning
knob rather than a design question.
