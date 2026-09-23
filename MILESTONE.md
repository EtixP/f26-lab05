# Lab 5: work record

What was done for each milestone, where it lives, and how to show it. The full write-up is in
`SMELLS.md`; this file is the index.

| Milestone | Status | Where |
|---|---|---|
| 1. Three smells | Done | `SMELLS.md`, Milestone 1 |
| 2. One small fix | Done, suite green | `src/pricing.ts`, `src/reservationManager.ts`, `src/reportGenerator.ts`; write-up in `SMELLS.md`, Milestone 2 |
| 3. Two proposals, one false positive | Done | `SMELLS.md`, Milestone 3 |

## Milestone 1: three smells

Each smell is in a different part of the module.

| # | Smell | Where | Principle |
|---|---|---|---|
| 1 | Duplicated pricing rules | `ReservationManager.calculatePrice`/`applyDiscounts` and `ReportGenerator.priceOf` | DRY / single source of truth |
| 2 | Dead query cache, read but never written | `src/cache/*`, used only by `ReservationManager.listBookingsForRoom` | YAGNI; code should say what it does |
| 3 | Notification registry for one channel, hard-wired anyway | `src/notifications/notifierFactory.ts`, `ReservationManager` constructor | YAGNI; dependency inversion |

Also found but not recorded, since the lab asks for three:
- `ReservationManager` mixes booking logic with receipt and summary formatting.
- The overlap rule is written three ways: `hasConflict`, `isSlotFree` and `overlapsWindow`.

## Milestone 2: the fix (smell 1)

- Added `src/pricing.ts` with `priceFor(room, start, end)`, which holds the only copy of the
  pricing constants and formula.
- `ReservationManager.calculatePrice` now calls `priceFor` (same public signature).
  `applyDiscounts` and the duplicate constants are removed.
- `ReportGenerator.revenue` now calls `priceFor`. `priceOf`, `durationOf` and the duplicate
  constants are removed.
- **Scope line:** one definition of the pricing rule, with no change to what any caller sees.
  Not touched: switching `revenue` to read `booking.priceCents` (that changes behavior), the
  duplicated overlap check (a separate rule), and the public API.
- **Verification:** `npm run typecheck` is clean, `npm test` passes 39/39, and no test files
  were changed.

## Milestone 3

- **Proposal A (smell 2):** take the cache out of `ReservationManager`. If caching is ever
  needed, add it as a `CachingStorageProvider` decorator that owns invalidation on
  `save`/`update`. Cost: every new write method must remember to invalidate, and the cache
  goes stale when more than one process writes.
- **Proposal B (smell 3):** pass a `NotificationChannel` into the `ReservationManager`
  constructor (defaulting to `EmailChannel`) and delete the registry. Cost: choosing a channel
  by name from config is no longer built in.
- **False positive:** the `StorageProvider` interface with a single implementation. Unlike the
  notifier, it is actually injected and shared: the tests pass one storage instance to both
  `ReservationManager` and `ReportGenerator`. It would become a real problem if the manager
  built its own storage, or if callers came to depend on in-memory-only behavior.

## Showing it in recitation

```
npm install
npm run typecheck
npm test
git diff 3ad9375 -- src/     # the milestone 2 diff, including the new src/pricing.ts
```

## Submission

- [x] AI tools line added to `README.md`.
- [x] Committed and pushed to the fork (`EtixP/f26-lab05`, `main`).
