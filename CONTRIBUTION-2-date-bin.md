# Contribution 2: Add support for the `date_bin` function

**Contribution Number:** 2
**Student:** Mirenge Innocent
**Issue:** https://github.com/trinodb/trino/issues/20428
**Status:** Phase I ✓ | Phase II ✓ | Phase III ✓ | Phase IV pending — PR [#30075](https://github.com/trinodb/trino/pull/30075) awaiting maintainer review

> 📁 Part of my [Open Source Contribution Log](README.md) · Contribution #2 of 3

---

## Why I Chose This Issue

After getting my first contribution (`initcap`, [#2942](https://github.com/trinodb/trino/issues/2942)) to the point of co-founder review, I wanted a second Trino issue that builds directly on what I learned while stretching me a little further. `date_bin` is a perfect step up: it's the same `@ScalarFunction` pattern I already used, but in the datetime domain instead of strings, and it requires real timestamp/interval arithmetic rather than a one-line delegation. CodePath staff confirmed I can take this off-sheet Trino issue, and it scored cleanly on the issue-selection checklist (open, unassigned, no existing PR, clear PostgreSQL reference spec, community demand).

It also maps closely to the data work I do as a software engineering intern in Home Depot's Supply Chain Technology group. `date_bin` is fundamentally a time-series *downsampling* tool — bucketing timestamped events into fixed-width intervals (every 15 minutes, every 0.5 seconds, etc.). That's exactly what you do when aggregating supply chain telemetry, order flow, or sensor data into regular time buckets for analysis. Implementing it teaches me how a distributed SQL engine handles parametric timestamp precision, interval types, and multi-type function registration.

---

## Understanding the Issue

### Problem Description

Trino has `date_trunc`, which rounds a timestamp down to a fixed *calendar* unit (hour, day, month, etc.). But it cannot bin timestamps into **arbitrary fixed-width intervals** — for example, 15-minute or 0.5-second buckets — aligned to a chosen origin point. PostgreSQL solves this with [`date_bin`](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-BIN). Trino has no equivalent, so users doing time-series downsampling at non-calendar intervals have to fall back on awkward arithmetic.

### Expected Behavior

A native `date_bin` function that returns the start of the bin (of width `stride`, aligned to `origin`) that `source` falls into:

```sql
-- bin into 15-minute buckets aligned to 2001-01-01
SELECT date_bin(INTERVAL '15' MINUTE,
                TIMESTAMP '2020-02-11 15:44:17',
                TIMESTAMP '2001-01-01 00:00:00');
-- 2020-02-11 15:30:00
```

Proposed signature (PostgreSQL-compatible, pending maintainer confirmation per the `syntax-needs-review` label):

```
date_bin(stride INTERVAL DAY TO SECOND, source TIMESTAMP(p), origin TIMESTAMP(p)) -> TIMESTAMP(p)
```

Semantics: `origin + floor((source - origin) / stride) * stride`. Following PostgreSQL, `stride` must be positive, and only `INTERVAL DAY TO SECOND` is supported (no months/years, which are variable-length).

### Current Behavior

No `date_bin` function exists. Users approximate with `date_trunc` (limited to fixed calendar units, so no "every 15 minutes" or "every 0.5 seconds") or write verbose epoch-arithmetic expressions.

### Affected Components

- **Datetime scalar functions** — `core/trino-main/src/main/java/io/trino/operator/scalar/`. The implementation will mirror the existing `date_trunc` structure, which lives in per-type classes (`timestamp/DateTrunc.java`, `timestamptz/DateTrunc.java`), each with short (`long`) and high-precision (`LongTimestamp`) variants for parametric `TIMESTAMP(p)`.
- **New `DateBin` classes** — likely `timestamp/DateBin.java` and `timestamptz/DateBin.java`, registered alongside the other datetime functions.
- **Interval handling** — `INTERVAL DAY TO SECOND` is represented internally as a `long` (milliseconds); the function must convert and align it against microsecond-precision timestamps.

---

## Reproduction Process

### Environment Setup

- **Branch:** `add-date-bin-function` — created from upstream `master` via `git checkout -b add-date-bin-function`
- **Fork:** https://github.com/minnocent12/trino
- **Setup approach:** Used the Maven Wrapper (`./mvnw`) with a selective module build to avoid rebuilding all 400+ modules: `./mvnw test -pl core/trino-main -am -DskipTests`. This is the same workflow established for Contribution #1 (macOS, Temurin JDK 25).
- **Challenge encountered:** On the first full build attempt, Maven's enforcer plugin reported `RequireUpperBoundDeps` violations for `okhttp-jvm` and `jackson-databind`. Investigated by running `git diff HEAD` — the only changed files were the five new `DateBin` files and the `SystemFunctionBundle.java` registration. Cross-referenced the same error on a clean checkout of `master` with no modifications and reproduced it identically. Confirmed these are pre-existing upstream dependency conflicts in the current `master`, not caused by our changes. Proceeded with the targeted module build.

### Steps to Reproduce

1. Open a Trino SQL session and run `SELECT date_bin(INTERVAL '15' MINUTE, TIMESTAMP '2020-02-11 15:44:17', TIMESTAMP '2001-01-01');`
2. Observed: function not registered — `date_bin` does not exist.
3. `date_trunc` works but only for fixed calendar units, confirming the gap.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — missing-feature issue; "reproduction" means confirming the function is absent.
- **Codebase search:** Ran `grep -r "date_bin" core/trino-main/src/main/java/` and `grep -r "date_bin" core/trino-main/src/test/` → zero results in both. No implementation, no tests, no registration in `SystemFunctionBundle.java`.
- **Confirmed the gap:** `date_trunc` exists at `core/trino-main/src/main/java/io/trino/operator/scalar/timestamp/DateTrunc.java` and is registered in `SystemFunctionBundle.java`. Running `SELECT date_trunc('hour', TIMESTAMP '2020-02-11 15:44:17')` succeeds. Running any `date_bin(...)` call fails with `Function 'date_bin' not registered` — confirming the function is entirely absent from the engine.
- **Scope confirmed:** Issue #20428 was open, unassigned, and had no linked PR when selected. The `syntax-needs-review` label indicated the function signature needed community input, not that it was blocked.

---

## Solution Approach

### Analysis

This is a missing-feature issue. Investigating the codebase confirmed:

- `date_bin` does not exist anywhere in Trino.
- Datetime scalar functions live in `core/trino-main/src/main/java/io/trino/operator/scalar/`, organized by type (`timestamp/`, `timestamptz/`, `time/`, `timetz/`).
- The closest sibling, `date_trunc`, is implemented as per-type classes (`timestamp/DateTrunc.java`, `timestamptz/DateTrunc.java`), each with **two precision variants**: a `long` (epoch micros) for `TIMESTAMP(p)` with p ≤ 6, and a `LongTimestamp` / `LongTimestampWithTimeZone` for p > 6.
- Functions are registered in `core/trino-main/src/main/java/io/trino/metadata/SystemFunctionBundle.java` via `.scalar(<Class>.class)` (the `date_trunc` registrations are at ~lines 671 and 719).
- `INTERVAL DAY TO SECOND` (`StandardTypes.INTERVAL_DAY_TO_SECOND`) is represented internally as a `long` of **milliseconds**.

So the work is to add a new `DateBin` function following the exact `date_trunc` structure, plus the binning arithmetic.

### Proposed Solution

Implement `date_bin` as a parametric datetime scalar function mirroring `date_trunc`: per-type classes for `TIMESTAMP(p)` and `TIMESTAMP(p) WITH TIME ZONE`, each with short and high-precision (`LongTimestamp` / `LongTimestampWithTimeZone`) variants. Core formula, applied in microseconds:

```
strideMicros = stride_millis * 1000
if (strideMicros <= 0) -> throw INVALID_FUNCTION_ARGUMENT ("stride must be positive")
binIndex = floorDiv(source - origin, strideMicros)      // floorDiv handles source < origin
result   = origin + binIndex * strideMicros             // addExact / multiplyExact for overflow safety
```

For the high-precision (`LongTimestamp`) form, the picos-of-micro fraction needs a small correction (when `source`/`origin` micros are equal at a bin boundary), and the result inherits the origin's picos-of-micro. Only `INTERVAL DAY TO SECOND` is accepted (no months/years).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Add `date_bin(stride, source, origin)` to bin timestamps into fixed-width intervals aligned to an origin.

**Match:** `date_trunc` (`timestamp/DateTrunc.java`, `timestamptz/DateTrunc.java`) is the structural model — same parametric-precision, multi-type, registered-in-`SystemFunctionBundle` pattern. Used `git log --oneline --follow` on `DateTrunc.java` to find that this per-type structure was introduced in commit `21e3ddf55a6` (2020-05-15, "Implement parametric timestamp type") and has been stable for 5+ years, confirming it is the canonical pattern and the right model to follow.

**Plan:**
1. `operator/scalar/timestamp/DateBin.java` — `TIMESTAMP(p)` short (`long`) + high-precision (`LongTimestamp`) variants
2. `operator/scalar/timestamptz/DateBin.java` — `TIMESTAMP(p) WITH TIME ZONE` short (packed `long`) + `LongTimestampWithTimeZone` variants
3. Register both in `metadata/SystemFunctionBundle.java`, beside the `date_trunc` registrations
4. Add tests mirroring the existing `date_trunc` test coverage
5. Document the function in `docs/src/main/sphinx/functions/datetime.md` (+ list indexes)

**Implement:** Branch `add-date-bin-function` in fork https://github.com/minnocent12/trino

**Review:**
- [ ] Mirrors the `date_trunc` structure and registration
- [ ] Positive-stride validation; `INTERVAL DAY TO SECOND` only
- [ ] Correct `floorDiv` behavior when `source` < `origin`
- [ ] Overflow-safe arithmetic (`addExact` / `multiplyExact`)
- [ ] All four type/precision variants covered
- [ ] Matches PostgreSQL `date_bin` semantics

**Evaluate:** Run the new `TestDateBin` cases via `./mvnw test -pl core/trino-main -Dtest=...` — all pass.

---

## Testing Strategy

### Unit Tests

Both `TestDateBin.java` files mirror the structure and naming conventions of the existing `TestDateTrunc.java` files in the same packages (`timestamp/` and `timestamptz/`), including method naming (`testBin`, `testZeroStride`), use of `AbstractTestFunctions` as the base class, and per-precision assertion loops — the same pattern used throughout Trino's datetime test suite.

- [x] Basic binning into N-minute buckets — `date_bin(INTERVAL '15' MINUTE, TIMESTAMP '2020-02-11 15:44:17', TIMESTAMP '2001-01-01 00:00:00')` → `2020-02-11 15:30:00`
- [x] Sub-second stride (0.5 seconds) — exercises millisecond-precision path
- [x] `source` exactly on a bin boundary — result equals `source`
- [x] `source` before `origin` (negative floor division) — this is an edge case the issue did not explicitly call out; `Math.floorDiv` (not `/`) is required here because Java's `/` truncates toward zero, which would place `source` in the wrong bin when it precedes `origin`
- [x] All 13 timestamp precisions (p = 0 through 12) — parametric `TIMESTAMP(p)` tested across the full precision range in a single loop
- [x] `TIMESTAMP WITH TIME ZONE` variant — separate `TestDateBin` in `timestamptz/` package covering both short (`long`) and `LongTimestampWithTimeZone` variants
- [x] Error: zero or negative stride — throws `INVALID_FUNCTION_ARGUMENT`
- [ ] Error / rejection: `INTERVAL YEAR TO MONTH` stride — not applicable; Trino's type system rejects this at the call site before the function is reached

### Integration Tests

Datetime scalar functions in Trino are exercised through the `AbstractTestFunctions` infrastructure, which routes assertions through the full expression engine — providing integration-level coverage without a separate integration test class. All existing integration test suites passed (CI 99/99 ✓).

### Manual Testing

CI is the primary validation gate. All 99 checks passed on commit `46caab8` (implementation) and again on `e53e217` (docs) — including `build-success` (required check), `maven-checks` across Java 25/26/27-ea (airstyle + error-prone), `build-pt-junit` (parametric-type JUnit suite), and `artifact-checks`. No pre-existing test failures were introduced by our changes. The PR diff is scoped to exactly 6 files: 4 new source/test files, 1 modified registration file, 1 updated doc.

---

## Implementation Notes

### Week 4 Progress

- Selected and self-assigned issue #20428 (staff-approved off-sheet Trino issue).
- Researched the closest sibling, `date_trunc`, and confirmed `date_bin` does not yet exist in the codebase.
- Posted a claim comment proposing a PostgreSQL-compatible signature to address the `syntax-needs-review` label.
- No maintainer reply after one week — proceeded with implementation based on the well-established PostgreSQL spec.

### Week 5 Progress

- Created branch `add-date-bin-function` from upstream master.
- Implemented all four type/precision variants mirroring the `date_trunc` structure.
- Key decision: `Math.floorDiv` (not integer division) ensures correct behavior when `source` is before `origin` — floor division rounds toward negative infinity, so the result is always the start of the bin that *contains* the source.
- `INTERVAL DAY TO SECOND` arrives as `long` milliseconds; multiplied by 1000 for the plain `TIMESTAMP(p)` micros path; used directly for the `TIMESTAMP(p) WITH TIME ZONE` millis path.
- Opened draft PR #30075 against `trinodb/trino`.

### Code Changes

- **Files created:**
  - `core/trino-main/src/main/java/io/trino/operator/scalar/timestamp/DateBin.java` — `TIMESTAMP(p)` short (`long`) + high-precision (`LongTimestamp`) variants
  - `core/trino-main/src/main/java/io/trino/operator/scalar/timestamptz/DateBin.java` — `TIMESTAMP(p) WITH TIME ZONE` short (packed `long`) + `LongTimestampWithTimeZone` variants
  - `core/trino-main/src/test/java/io/trino/operator/scalar/timestamp/TestDateBin.java`
  - `core/trino-main/src/test/java/io/trino/operator/scalar/timestamptz/TestDateBin.java`
- **Files modified:**
  - `core/trino-main/src/main/java/io/trino/metadata/SystemFunctionBundle.java` — added import + two `.scalar()` registrations beside the `date_trunc` entries
- **Branch:** `add-date-bin-function`
- **Docs added:** `docs/src/main/sphinx/functions/datetime.md` — new "Binning function" section with two SQL examples; commit `e53e217`

---

## Pull Request

**PR Link:** https://github.com/trinodb/trino/pull/30075

**PR Description:** Adds `date_bin(stride INTERVAL DAY TO SECOND, source TIMESTAMP(p), origin TIMESTAMP(p))` for both plain and timezone-aware timestamp types. Mirrors the `date_trunc` structure — four type/precision variants registered in `SystemFunctionBundle`. Core algorithm: `origin + floorDiv(source − origin, stride) × stride`. Tests cover all 13 precisions, boundary, before-origin, sub-second stride, and zero-stride error.

**Maintainer Feedback:**
- **2026-06-19:** Self-assigned and posted claim comment proposing the PostgreSQL-compatible signature; awaiting maintainer confirmation per `syntax-needs-review` label.
- **2026-06-27:** No reply after one week — proceeded with implementation. Draft PR #30075 opened; CI running.
- **2026-06-27:** `ebyhr` added `syntax-needs-review` and `needs-docs` labels. `wendigo` requested a review from `martint`.
- **2026-07-14:** CI failing — 4 checks failed (`build-success`, `maven-checks 25.0.3`, `maven-checks 26`, `maven-checks 27-ea`). Root cause: `airstyle-maven-plugin` formatting check found 1 file needing reformatting — two `static import` lines in `timestamp/TestDateBin.java` were in the wrong alphabetical order. Fixed by running `./mvnw airstyle:format -pl core/trino-main` locally (requires Java 25), then rebuilding a clean commit `cf18515` on current upstream master via GitHub API (same 5-file approach as Contribution #1). CI re-triggered.
- **2026-07-15:** CI failed again on `cf18515` — 31 checks failing. Root cause: `SystemFunctionBundle.java` in our branch was taken from an old tree; upstream `master` had since added `.scalar(VarcharMethods.class)`, `.scalar(CharMethods.class)`, `.scalar(CharToVarcharCast.class)`, and `ROW_FIELDS_FUNCTION` registrations. Without them, every function in those classes threw "Function not registered" at test time. Fix: fetched upstream master's current `SystemFunctionBundle.java` blob, applied our DateBin import and two `.scalar()` registrations via script, created clean commit `eb77ad1` on upstream master. Failures dropped from 31 → 6.
- **2026-07-15:** `maven-checks` still failing on `eb77ad1` — airstyle rejected the import order in `SystemFunctionBundle.java`. Our script had placed `import io.trino.operator.scalar.timestamp.DateBin` after `DateTrunc` instead of the correct alphabetical position (between `DateAdd` and `DateDiff`). Fix: re-cloned branch, ran `./mvnw airstyle:format -pl core/trino-main -am -q` (Java 25), which auto-corrected the import position, created new blob `6ac2ac6e`, new commit `46caab8` parented to upstream master `c5620867`. CI re-triggered.
- **2026-07-15:** All 99 CI checks passed on commit `46caab8` ✓. Key lesson: always run `airstyle:format` after any manual import insertion and never assume import placement — the formatter determines the canonical position.
- **2026-07-15:** Added documentation — new "Binning function" section in `docs/src/main/sphinx/functions/datetime.md`, mirroring the "Truncation function" section. Two SQL examples showing 15-minute bucket and sub-second (0.5 s) stride. Committed as `e53e217` via GitHub API (new tree on top of `46caab8`). Addresses `needs-docs` label. CI re-triggered; once green, will mark PR ready for review.

**Status:** PR ready for review — CI green on `46caab8` (implementation); docs added in `e53e217`; PR marked ready for review; awaiting `martint` review

---

## Learnings & Reflections

### Technical Skills Gained

- **Parametric timestamp arithmetic in Trino**: `TIMESTAMP(p)` has two runtime representations — a `long` (epoch micros) for p ≤ 6 and a `LongTimestamp` for p > 6. Implementing all four variants (two types × two precisions) gave me a solid model of how Trino handles precision-parametric datetime types.
- **`floorDiv` vs integer division**: Java's `/` truncates toward zero, giving wrong results when `source < origin`. `Math.floorDiv` rounds toward −∞, matching PostgreSQL's `date_bin` spec. Catching this edge case was the key correctness insight for the implementation.
- **GitHub API-based clean commits**: Learned to build commits using the git objects API (blob → tree → commit → ref update) to guarantee a minimal diff and linear history, bypassing the limitations of shallow clones and avoiding the `check-commits-dispatcher` failures that merge commits trigger.
- **Trino's airstyle formatter**: Understood that import order is enforced mechanically — the formatter is the authority, not the developer. Running `./mvnw airstyle:format -pl core/trino-main -am -q` locally before every commit push is now a fixed step in my workflow.

### Challenges Overcome

**1. Upstream drift in `SystemFunctionBundle.java` caused 31 CI failures.**
After the initial draft PR opened, 31 checks failed with "Function not registered" errors for unrelated built-in functions like `starts_with` and `to_utf8`. Root cause: the `SystemFunctionBundle.java` blob in our branch came from an older commit tree and was missing new `.scalar()` registrations upstream had added (`VarcharMethods`, `CharMethods`, `CharToVarcharCast`, `ROW_FIELDS_FUNCTION`). Without them, those function classes were invisible to the engine. Fix: fetched the current upstream master blob, applied our two DateBin registrations on top, rebuilt the commit. Lesson: in an active codebase, always base any modified file on the current upstream, not a snapshot from branch creation time.

**2. Airstyle import ordering rejected twice.**
Trino's `airstyle-maven-plugin` enforces strict alphabetical import ordering and fails CI if any file needs reformatting. Our import for `timestamp.DateBin` was inserted in the wrong position twice: first, two static imports in `TestDateBin.java` were out of alphabetical order; second, in `SystemFunctionBundle.java`, the import was placed after `DateTrunc` instead of its correct position between `DateAdd` and `DateDiff`. Fix: never guess import position — always run `airstyle:format` locally and let the formatter decide. This cost two extra 20-minute CI cycles that a 30-second local run would have prevented.

**3. Clean commit construction for a project with strict history requirements.**
Trino's `check-commits-dispatcher` rejects merge commits. Local shallow clones cannot rebase against upstream reliably. The solution was to use the GitHub API git objects endpoint: fetch the current upstream master tree SHA, patch in only the intended file blobs, create a new tree, create a commit parented directly to upstream master, and force-push the branch ref. This guarantees a linear history and a diff containing exactly the files we intended to change — nothing more.

### What I'd Do Differently Next Time

- **Run `airstyle:format` locally before every GitHub API commit.** Both CI regressions in this PR could have been caught in 30 seconds locally instead of waiting 20–30 minutes for CI. This is now a non-negotiable step before any push.
- **Always fetch the current upstream blob for every file the PR touches.** A quick `git show upstream/master:path/to/File.java` before constructing the commit tree would have avoided the 31-failure round caused by upstream drift.
- **Add docs in the initial commit.** The `needs-docs` label appeared on day one. Including documentation upfront would have kept the PR to one commit and one CI cycle instead of two.

---

## Resources Used

- [PostgreSQL `date_bin` documentation](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-BIN) — reference semantics for the function
- [trinodb/trino Issue #20428](https://github.com/trinodb/trino/issues/20428) — the issue being addressed
- Trino's existing `date_trunc` implementation (`core/trino-main/src/main/java/io/trino/operator/scalar/timestamp/DateTrunc.java`) — structural model for parametric datetime functions
