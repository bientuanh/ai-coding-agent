# Sector Strength MVP

## Status

`Completed`

## Goal

Build a fast MVP to score stock sector strength by trading day using price movement, volume behavior, and breadth. Persist daily sector scores in MongoDB and expose public chart-ready APIs so frontend can draw multi-line sector trends over time.

## Scope

### In Scope

- Daily sector scoring job (run once after market close)
- Daily-only persistence model (1 record per sector per trading day)
- Public read APIs for chart data with timeframe filters
- Timeframe support in API: 1D, 1W, 1M, multi-month
- Timeframe aggregation strategy: derive 1W/1M/multi-month on read from stored 1D data
- Partial-data computation when some stocks are missing
- Coverage ratio returned for transparency (availableSymbols/totalSymbols)
- Basic tests for validator, controller, service

### Out Of Scope

- Intraday/realtime scoring
- Dynamic sector management UI
- Advanced factor optimizer/backtesting engine
- Automated notifications based on score thresholds

## Current Context

- Existing stock groups are already defined in src/constants/stock.ts.
- Existing stock helpers and calculation patterns exist in src/utils/stock.ts.
- Existing SSI fetching/retry/cache behavior exists in src/services/stock.service.ts and src/utils/cache.ts.
- Existing script pattern uses connect -> run -> disconnect in src/scripts/daily-task.ts.
- Existing API contracts use standardized success/error response wrappers from src/utils/response.ts.
- Existing architecture follows Type -> Model -> DAO -> Validator -> Controller -> Route.

## Assumptions And Decisions

1. MVP stores only daily snapshots (1D) in DB for simplest implementation.
2. 1W/1M/multi-month views are computed on API read from 1D data.
3. Chart read endpoints are public.
4. If some symbols fail to fetch, sector score is still computed from available symbols.
5. API response includes coverage ratio and constituent counts for confidence display.
6. Score is normalized to 0-100.
7. Job runs once per day after market close.

## Implementation Steps

### Step 1: Define score contract and formulas

- Status: `Completed`
- Files: `docs/plans/sector-strength-mvp.md`, `src/constants/stock.ts` (new scoring constants)
- Changes:
  - Finalize v1 score components and default weights.
  - Define output metrics persisted per sector/day:
    - score (0-100)
    - momentumScore
    - volumeScore
    - breadthScore
    - coverageRatio
    - totalSymbols
    - availableSymbols
  - Define threshold policy for low coverage (minimum symbols required to produce score).
- Validation: Formula spec is explicit and implementable without ambiguity.
- Done when: Scoring formula, fields, and thresholds are documented and frozen for MVP v1.

### Step 2: Add type, model, and DAO

- Status: `Completed`
- Files: `src/types/sectorStrength.ts`, `src/models/sectorStrength.model.ts`, `src/models/daos/sectorStrength.dao.ts`
- Changes:
  - Add TypeScript contract for sector strength snapshots.
  - Add Mongoose model with unique key on (sector, date).
  - Add indexes for query patterns: sector + date, date only.
  - Add DAO methods:
    - upsertSectorStrengthSnapshot
    - getSectorStrengthSeriesByRange
    - getLatestSectorStrengthBySector
- Validation: Typecheck passes and DAO methods return typed results.
- Done when: Daily snapshots can be upserted idempotently and queried by sector/date range.

### Step 3: Implement scoring service with partial-data fallback

- Status: `Completed`
- Files: `src/services/sectorStrength.service.ts`, optional helper updates in `src/utils/stock.ts`
- Changes:
  - Iterate sectors from stock constants.
  - Fetch per-symbol data for target trading date.
  - Compute per-symbol metrics and aggregate sector metrics.
  - Apply partial-data policy when some symbols are unavailable.
  - Return sector snapshot payloads ready for DAO upsert.
- Validation: Unit tests cover full-data and partial-data scenarios.
- Done when: Service computes stable scores for all sectors and reports coverage metrics.

### Step 4: Add daily script and package command

- Status: `Completed`
- Files: `src/scripts/sector-strength-daily.ts`, `package.json`
- Changes:
  - Add script entrypoint using existing connect/run/disconnect pattern.
  - Execute daily calculation for all sectors and persist with upsert.
  - Add npm script command for manual run and cron integration.
- Validation: Manual run succeeds and rerun for same day does not duplicate data.
- Done when: Daily job can be triggered reliably and idempotently.

### Step 5: Expose public chart API

- Status: `Completed`
- Files: `src/validators/sectorStrengthSchema.ts`, `src/controllers/sectorStrength.controller.ts`, `src/routes/sectorStrength.routes.ts`, `src/index.ts`
- Changes:
  - Add query validation for sectors, date range, timeframe, and optional pagination.
  - Add public GET endpoints for chart series and latest sector scores.
  - Derive 1W/1M/multi-month responses from 1D snapshots.
  - Keep response shape consistent with existing success/error wrappers.
- Validation: Endpoint tests verify filters, aggregation windows, ordering, and response contract.
- Done when: FE can query series for multiple sectors and render chart directly.

### Step 6: Add focused tests and run verification

- Status: `Completed`
- Files: `src/test/validators/sectorStrengthSchema.test.ts`, `src/test/controllers/sectorStrength.controller.test.ts`, `src/test/services/sectorStrength.service.test.ts`
- Changes:
  - Add validator tests for valid/invalid query/body data.
  - Add controller tests for success and error paths.
  - Add service tests for formula and partial-data handling.
- Validation:
  - npm run typecheck
  - npm run lint
  - npm run test -- sectorStrength
- Done when: New feature tests pass and quality checks are green.

## Risks And Open Questions

- SSI data delay after market close may produce incomplete daily data at runtime.
- Large sector universe can increase API calls and runtime duration.
- Aggregating 1W/1M on read can become expensive for very long ranges if not optimized.
- Default weights for momentum/volume/breadth were set for MVP and can be tuned in a later iteration.

## Acceptance Criteria

- [x] Daily sector strength snapshots are stored in DB with unique (sector, date).
- [x] Re-running daily job for same date does not create duplicates.
- [x] Public API returns chart-ready series for multiple sectors.
- [x] API supports 1D, 1W, 1M, and multi-month views.
- [x] Missing symbol data does not block sector scoring; coverage ratio is returned.
- [x] Typecheck, lint, and focused tests pass.

## Change History

- 2026-09-30: Created initial MVP plan for sector strength scoring feature.
- 2026-09-30: Implemented MVP end-to-end (model/dao/service/script/api/tests) and wired feature into app routes and scripts.
