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

**Decision**: C# / .NET 10, isolated worker model. Samuel chose .NET 10 over matching
`fn-mystore`/`api-gamedb`'s current .NET 8, deliberately starting this new product on
the newer runtime rather than carrying the older LTS forward.

**Rationale**: Same Azure Functions isolated-worker model and test stack
(xUnit/Moq/FluentAssertions) as the rest of RSM, so operational knowledge still
transfers — only the .NET version differs. Starting fresh repos on the newer runtime
avoids inheriting a version upgrade as unplanned future work.

**Alternatives considered**: .NET 8 (match `fn-mystore`/`api-gamedb` exactly) — would
keep all Azure Functions on one identical runtime, but was explicitly not what Samuel
wanted for this new repo. Node.js/TypeScript Azure Functions (shares a language with the
RN client) — rejected, would be the only non-.NET Function app in the org.

## 3. AI item identification approach

**Decision**: Send the lot photo to Claude's vision capability to detect/segment
candidate items and propose a title/platform guess per item; resolve each candidate
against the existing game catalog (via `api-gamedb`) with fuzzy matching to attach a
canonical title/platform/variant and reference price. Confidence is primarily the
catalog fuzzy-match score (a well-understood, deterministic signal), with the vision
model's own self-reported certainty folded in only as a secondary signal where
available.

**Rationale**: No training pipeline or labeled dataset needed to ship v1; other RSM/
Samuel projects already use Claude for similar "identify + structure" tasks (portfolio
AI assistant, StayRecap report pipeline), so this reuses proven integration patterns.
Note: Claude's vision API does not emit a calibrated numeric confidence score, and
self-reported LLM confidence is known to be poorly calibrated — leaning on fuzzy-match
score as the primary signal avoids depending on a capability that may not hold up. A
short technical spike (run real lot photos through the vision call, see what signal is
actually usable) is worth doing early in `/speckit-tasks` for this function-app item,
before committing to the exact confidence formula.

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

**Decision**: A new `lotscanner` database on `db-gamedb`'s existing Azure Postgres
Flexible Server — no new server.

**Rationale**: Avoids a new recurring Azure line item (roughly $15-25/mo per the
pattern seen on `strata-reports` prod) for a single-database workload; a separate
database still gives `lot-scanner` its own schema/migrations, independent of
`db-gamedb`'s tables, which is enough isolation for this slice.

**Alternatives considered**: A dedicated new Flexible Server, which would fully decouple
`lot-scanner`'s infra lifecycle from `db-gamedb`'s in case this product is ever spun off
— rejected by Samuel as premature cost for a feature still being validated. Revisit if
`db-gamedb`'s server becomes a scaling bottleneck shared across products, or if
`lot-scanner` is actually spun out as its own product.

## 6. Confidence threshold for "low confidence" flagging

**Decision**: Flag any item with blended confidence < 0.7 as low-confidence in the review
UI.

**Rationale**: Reasonable starting default; this is a tunable product constant, not an
architectural decision, and can be adjusted post-launch based on real scan data without
a spec/plan change.

**Alternatives considered**: None evaluated in depth — explicitly deferred as a tuning
knob rather than a design question.
