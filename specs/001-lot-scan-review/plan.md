# Implementation Plan: Lot Scan & Review

**Branch**: `001-lot-scan-review` | **Date**: 2026-10-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-lot-scan-review/spec.md`

## Summary

An employee photographs a customer's game lot in a React Native app; an Azure Function
sends the photo to an AI vision model, cross-references candidate matches against the
existing RetroStoreManager game catalog (`api-gamedb`), and returns one item per detected
game with a suggested title/platform/variant/price/confidence. The app shows an itemized
card review screen (select/deselect-all, per-card correct/exclude/price-override); on
confirm, the session and its items are persisted as a quote record in a new
PostgreSQL database on `db-gamedb`'s existing server — no write to RSM's sellable
inventory in this slice.

## Technical Context

**Language/Version**: TypeScript (React Native / Expo) for the client; C# / .NET 10
(isolated worker) for the Azure Function — newer than `fn-mystore`/`api-gamedb`'s
current .NET 8, a deliberate choice for this new product (see research.md).

**Primary Dependencies**: Expo (managed RN workflow) + `expo-camera` for capture;
Azure Functions .NET 10 isolated worker; Anthropic Claude (vision) for item detection/ID;
HTTP client to the existing `api-gamedb` service for catalog lookup/fuzzy match;
Npgsql/EF Core 10 for Postgres access.

**Storage**: PostgreSQL — a new `lotscanner` database on `db-gamedb`'s existing Azure
Flexible Server (no new server; see research.md for the tradeoff this accepts).

**Testing**: Jest + React Native Testing Library (client); xUnit + Moq + FluentAssertions
(function app), matching `fn-mystore` conventions.

**Target Platform**: iOS 15+ and Android 8+ (Expo's current minimums) via one RN
codebase; Azure Functions (Linux) for the backend.

**Project Type**: Mobile app + backend API (Option 3: mobile + API).

**Performance Goals**: AI identification results returned within ~10s of photo upload
for a typical 10-20 item lot, leaving the remaining budget of the 5-minute SC-001 target
for review/correction.

**Constraints**: Online-only (no offline scanning, per spec Assumptions). Must reuse the
existing game catalog via `api-gamedb`'s HTTP API rather than a direct DB connection
(preserves RSM's existing service boundary).

**Scale/Scope**: Single-employee-at-a-time usage per store; typical lot 10-20 items,
design should not break at 50; low overall request volume for v1 (one store's foot
traffic, not multi-tenant SaaS scale yet).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Result |
|---|---|---|
| I. Honest Automation, Human Confirmation | No API path finalizes a quote without an explicit confirm call; AI output is always staged as "pending" until reviewed. | PASS |
| II. Correction Speed Is the Product | Select-all/deselect-all and per-item toggle are client-local state changes, not one network round-trip per card; corrections are a single `PATCH` per item, not a full re-scan. | PASS |
| III. Shared Code First (React Native) | Single Expo/RN codebase for iOS + Android; no native modules planned (camera covered by `expo-camera`). | PASS |
| IV. Vertical Slices, One at a Time | This plan produces exactly one frontend item (RN scan+review app), one function-app item (AI-identify + quote API), one database item (Postgres quote-record schema). | PASS |
| V. Azure-Native, Consistent with RetroStoreManager | Backend is an Azure Function (.NET 10 — newer runtime, same Azure-native pattern); storage is a database on `db-gamedb`'s existing Azure Postgres Flexible Server; catalog reused via `api-gamedb`, not reimplemented. | PASS |

No violations — Complexity Tracking table omitted.

## Project Structure

### Documentation (this feature)

```text
specs/001-lot-scan-review/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── lot-scan-api.md
└── tasks.md             # Phase 2 output (/speckit-tasks — not created here)
```

### Source Code (future repos — not created by this plan)

Per the constitution's Product Constraints, `lot-scanner` holds specs only.
Implementation lives in three new sibling repos under `retrostoremanager/`, created when
a slice moves to `/speckit-implement`, mirroring the `web-mystore` / `fn-mystore` /
`db-gamedb` split:

```text
app-lot-scanner/            # React Native (Expo) client — THE frontend work item
├── app/                    # Expo Router screens: capture, review, confirm
├── src/
│   ├── components/         # ItemCard, SelectAllBar, ConfidenceBadge, etc.
│   ├── api/                # typed client for fn-lot-scanner's HTTP API
│   └── state/              # scan-session local/offline-tolerant state
└── __tests__/

fn-lot-scanner/              # Azure Function (.NET 10 isolated) — THE function-app work item
├── src/
│   ├── Functions/          # HTTP triggers: create-scan, get-scan, patch-item, confirm
│   ├── Services/           # AI-identify orchestration, api-gamedb client, pricing
│   ├── Models/
│   └── Repos/              # Postgres access (EF Core)
└── tests/

db-lot-scanner/               # Postgres schema/migrations for the new `lotscanner`
│                              # database on db-gamedb's existing server — THE database
│                              # work item
├── migrations/
└── schema/                   # LotScanSession, ScannedItem DDL
```

**Structure Decision**: Mobile + API split (Option 3), realized as three repos
(`app-lot-scanner`, `fn-lot-scanner`, `db-lot-scanner`) to match this slice's three work
items and RSM's existing multi-repo convention. None of these repos exist yet; they are
created at implementation time for this slice.

## Complexity Tracking

*No Constitution Check violations — not applicable.*
