# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplicated code: the pricing rules are implemented twice, so a pricing change
needs the same edit in two places (shotgun surgery).

**Classic or agent-specific.** Agent-specific. Duplication is a classic smell, but this copy
comes from how the code was generated: the reporting code wrote pricing again instead of
reusing the method that already existed, because the agent lacked context on the rest of the
codebase. The giveaway is that the two copies use the same values under different constant
names (`PREMIUM_MULTIPLIER` vs `PREMIUM_RATE_MULTIPLIER`, `LONG_BOOKING_MINUTES` vs
`LONG_BOOKING_CUTOFF`, and so on).

**Where in the code.** `src/reservationManager.ts`, `calculatePrice` and `applyDiscounts`,
plus its five constants at the top of the file. `src/reportGenerator.ts`, `priceOf` (with
`durationOf`), plus its own five constants.

**The principle it violates.** DRY, or a single source of truth: each business rule should
live in one place.

**What it makes expensive.** Any change to pricing. If the evening discount changes and only
`reservationManager.ts` is edited, `ReportGenerator.revenue` reports amounts customers were
never charged. The only test comparing the two (`reporting.test.ts`, "totals revenue over the
window") passes as long as both copies match today's numbers, so it would not catch them
drifting apart. The same problem shows up today without any edit: `revenue` recalculates
prices from the room's current rate instead of reading `booking.priceCents`, so registering a
room again with a new rate changes the revenue of bookings already made.

### Smell 2

**The smell.** Dead code: a query cache that is created and read but never written to.

**Classic or agent-specific.** Agent-specific. This is infrastructure that looks complete but
does nothing: it has a config object, TTLs, eviction and an `invalidate` method, but nothing
fills it. The cause is generating code that looks plausible without checking that any code
path uses it.

**Where in the code.** `src/cache/queryCache.ts` (`QueryCache`) and `src/cache/cacheConfig.ts`
(`withTtl` and `disabled` are never called). The only consumer is
`ReservationManager.listBookingsForRoom` in `src/reservationManager.ts`, which calls
`cache.get`. Nothing in `src/` calls `cache.set` or `cache.invalidate`, so `get` always misses
and the method always falls through to storage.

**The principle it violates.** YAGNI, and code should say what it does. A reader assumes room
listings are cached, with the staleness that caching brings, when they are not.

**What it makes expensive.** Finishing the feature, or even just reasoning about it. The
obvious one-line fix of calling `cache.set` in `listBookingsForRoom` introduces a bug:
`createBooking` and `cancelBooking` never invalidate `bookings:<roomId>`, so
`formatDailySummary` shows an out-of-date schedule for up to 30 seconds, including cancelled
bookings still listed as confirmed. The constructor also builds the cache itself from
`DEFAULT_CACHE_CONFIG`, so a test cannot turn it off or replace it.

### Smell 3

**The smell.** Speculative generality: a pluggable channel registry built for exactly one
channel, which the manager then hard-wires anyway.

**Classic or agent-specific.** Agent-specific. This is over-engineering by default: extension
points added "in case", without a second use case or a caller that needs them. And the
flexibility never reaches the only client, which suggests the pattern was produced by habit
rather than to meet a requirement.

**Where in the code.** `src/notifications/notifierFactory.ts`: `ChannelName` is the single
literal `'email'`, `registeredChannels` is never called, and a module-level `builders` map is
filled as a side effect of importing the module. The `ReservationManager` constructor in
`src/reservationManager.ts` always calls `createNotificationChannel(DEFAULT_NOTIFIER_CONFIG)`,
so the channel is fixed no matter what the registry allows.

**The principle it violates.** YAGNI, and dependency inversion: `ReservationManager` builds
its own concrete dependency instead of receiving a `NotificationChannel`.

**What it makes expensive.** Adding a second channel (SMS, say) still means editing the
`ChannelName` union, registering a builder, and changing the `ReservationManager` constructor
to pick it. The registry saves none of those edits. Testing is also harder: the manager's
`EmailChannel` cannot be replaced or reached, so no test can check what was actually sent,
and the global `builders` map is shared state that one test can change (via
`registerChannel`) for every test that runs after it.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Smell 1, the duplicated pricing rules. It is the only one of
the three where two copies can already disagree without any other code changing: someone
edits one copy of a rate and the reports go wrong with no error. It also has a fix that
preserves behavior exactly, because both copies run the same arithmetic in the same order.

**What changed.**
- New `src/pricing.ts` with one function, `priceFor(room, start, end)`, and the only copy of
  the five pricing constants. Its body is the existing formula: base rate, premium surcharge,
  long-booking discount, then evening discount, rounding after each step.
- `src/reservationManager.ts`: `calculatePrice` now just calls `priceFor`. It stays public
  with the same signature, because callers of the module may use it. The private
  `applyDiscounts` and the five constants are deleted.
- `src/reportGenerator.ts`: `revenue` calls `priceFor(room, booking.start, booking.end)`. The
  private `priceOf` and `durationOf` and the five constants are deleted.

Every price the service computes comes out exactly as before. What changed is that the rule
now lives in one place.

**What you deliberately did not touch.** The scope line is "one definition of the pricing
rule, with no change to what any caller sees."
- `revenue` still recalculates prices from the room's current rate instead of reading
  `booking.priceCents`. Switching it would fix the repricing problem noted in smell 1, but
  it changes what the report returns whenever a room's rate changes. That is a change in
  behavior and needs its own decision and its own test, not a side effect of a refactor.
- The overlap check that is also written three times (`hasConflict`, `isSlotFree`,
  `overlapsWindow`) is a separate rule with its own edge cases (bookings that touch at an
  endpoint). Folding it in would make this diff about two rules.
- `calculatePrice` was not removed or inlined, because doing that would change the public
  API of `ReservationManager`.
- No tests were edited or added.

**How you know behavior is preserved.** `npm test` (39/39 passing) and `npm run typecheck`
(clean), with no test files changed.
- The four pricing tests in `booking.test.ts` pin one case each: the plain hourly rate
  (12000), the long-booking discount (16200), the premium surcharge (18400) and the evening
  discount (11400).
- `reporting.test.ts` "totals revenue over the window" checks that report revenue equals the
  stored `priceCents`, which now holds because both come from one function.

What the suite would not catch:
- Combinations of rules: premium plus long, or long plus evening. Rounding after each step
  makes the order of the multipliers matter. The order is unchanged, but no test pins it.
- Bookings shorter than an hour or not a whole number of hours, where the first rounding
  step matters.
- Repricing after a room is registered again.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

**The problem.** Smell 2, the dead query cache. `ReservationManager` holds a `QueryCache` it
never writes to. The code it would need to be correct, invalidating on every write, belongs
to storage, but it would have to live in the manager, far from the writes that make entries
stale.

**The decomposition.**
- Take the cache out of `ReservationManager` completely: remove the `cache` field, the
  `QueryCache` import, and the `get` call in `listBookingsForRoom`, which becomes
  `return this.storage.findByRoom(roomId)`. Today that is exactly what the method does anyway.
- If a measured need for caching appears, add it as a `CachingStorageProvider` that
  implements `StorageProvider` and wraps another one (a decorator). It owns the cache and the
  invalidation rule: `save` and `update` pass through to the wrapped storage, then drop the
  `bookings:<roomId>` entry and the "all bookings" entry. The finds read through the cache
  and return copies, as `InMemoryStorageProvider` does today.
- Whoever builds the service decides whether to wrap storage in the cache.
  `ReservationManager` and `ReportGenerator` go on depending only on `StorageProvider` and
  never know a cache exists.
- The staleness rule then lives next to the writes that cause it. `cacheConfig.ts` keeps
  `CacheConfig`, but the unused `withTtl` and `disabled` go.

**One cost.** Invalidation becomes a promise the decorator has to keep for every write
method. Adding a new method to `StorageProvider` (a bulk import, or a delete) means
remembering to invalidate in the decorator too, and forgetting compiles fine. The cache is
also only correct while every write goes through this one process's wrapper: two service
instances sharing a database would each serve stale reads for up to the TTL. Today's code
has neither risk, because it has no cache at all.

### Proposal B (not coded)

**The problem.** Smell 3, speculative generality in notifications. A global, mutable channel
registry exists to choose between channels, but there is only one channel, and
`ReservationManager` bypasses the choice by building `DEFAULT_NOTIFIER_CONFIG` itself. The
flexibility sits in the wrong place: the manager cannot be given a channel.

**The decomposition.**
- Keep `NotificationChannel` (the interface) and `EmailChannel` (the one implementation).
  That is the real seam.
- `ReservationManager`'s constructor takes the channel as a parameter:
  `constructor(storage = new InMemoryStorageProvider(), notifier: NotificationChannel = new EmailChannel())`.
  Existing callers keep working, and tests can pass a recording fake.
- Delete `notifierFactory.ts`: the `builders` map, `registerChannel`, `registeredChannels`,
  `ChannelName` and `NotifierConfig`. The code that builds the service picks the concrete
  channel. That is the only place where "which channel" is decided.
- If a second channel appears and has to be chosen from a config string, add a plain
  `switch` in that startup code, which gives a compile error when a case is missing. A
  registry makes sense only once third-party code needs to add channels.

**One cost.** Selecting a channel by name from configuration is no longer built in. Whoever
constructs the service has to import concrete channel classes. If channel choice is later
meant to come from a deployment config file, that mapping has to be written again, and
until it is, switching channels is a code change rather than a config change. It also
changes the signature of `ReservationManager`'s constructor (compatible, but public API).

### The thing that looks smelly but is fine

**What it is.** `src/storage/storageProvider.ts`, the `StorageProvider` interface, and its
single implementation `InMemoryStorageProvider`. At a glance it is the same pattern as smell
3: an abstraction with exactly one implementation.

**Why it is fine.** Unlike the notification registry, this seam is actually used and
actually injected.
- `ReservationManager`'s constructor accepts a `StorageProvider`, and
  `ReportGenerator`'s constructor requires one.
- The tests rely on that: `tests/fixtures.ts` builds one `InMemoryStorageProvider`, passes
  it to the manager, and every reporting test hands the *same* instance to
  `ReportGenerator`. That is how the two classes see the same bookings without either one
  owning the other.
- The interface is small and matches what callers do: save, update, find by id, room or
  all. It also carries a real contract that callers depend on: finds return copies in
  insertion order, and `save`/`update` throw `StorageError` on a duplicate or missing id.
  Because finds return copies, no caller can change a stored booking except through
  `update`.
- Any persistent version (a file, a database) plugs in without touching either consumer.

**What would flip your verdict.** Two changes would turn it into a real problem:
- If `ReservationManager` built its own `InMemoryStorageProvider` with no way to pass one in
  (the way it does with the notifier and the cache), the interface would be a seam nobody
  can use, and it would become smell 3 again.
- Or, if callers started depending on in-memory-only behavior that the interface does not
  promise: calling `clear()` between requests, or assuming a lookup costs nothing and calling
  `findByRoom` in a loop. (`InMemoryStorageProvider.findByRoom` is a full scan of `findAll`,
  which is cheap only because everything is in memory.) The interface would then hide a cost
  that every real implementation has, and the first database-backed implementation would
  make those callers slow.
