# Contribution 2: Add support for the `date_bin` function

**Contribution Number:** 2
**Student:** Mirenge Innocent
**Issue:** https://github.com/trinodb/trino/issues/20428
**Status:** Phase I — In Progress

> 📁 Part of my [Open Source Contribution Log](README.md) · Contribution #2 of 2

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

Same environment as Contribution #1 (already working): macOS, Temurin JDK 25, Maven Wrapper (`./mvnw`). Fork: https://github.com/minnocent12/trino.

### Steps to Reproduce

1. Open a Trino SQL session and run `SELECT date_bin(INTERVAL '15' MINUTE, TIMESTAMP '2020-02-11 15:44:17', TIMESTAMP '2001-01-01');`
2. Observed: function not registered — `date_bin` does not exist.
3. `date_trunc` works but only for fixed calendar units, confirming the gap.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — missing-feature issue (no buggy commit; "reproduction" is confirming the function is absent).
- **My findings:** _(to be completed)_

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

**Match:** `date_trunc` (`timestamp/DateTrunc.java`, `timestamptz/DateTrunc.java`) is the structural model — same parametric-precision, multi-type, registered-in-`SystemFunctionBundle` pattern.

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

- [ ] Basic binning into N-minute buckets
- [ ] Sub-second stride (e.g. 0.5 seconds)
- [ ] `source` exactly on a bin boundary
- [ ] `source` before `origin` (negative floor division)
- [ ] High-precision `TIMESTAMP(p)` with p > 6
- [ ] `TIMESTAMP WITH TIME ZONE` variant
- [ ] Error: zero or negative stride
- [ ] Error / rejection: `INTERVAL YEAR TO MONTH` stride

### Integration Tests

_(to be assessed — datetime scalar functions are typically covered by the unit tests run through Trino's expression engine)_

### Manual Testing

_(to be completed)_

---

## Implementation Notes

### Week 4 Progress

- Selected and self-assigned issue #20428 (staff-approved off-sheet Trino issue).
- Researched the closest sibling, `date_trunc`, and confirmed `date_bin` does not yet exist in the codebase.
- Posted a claim comment proposing a PostgreSQL-compatible signature to address the `syntax-needs-review` label, asking maintainers to confirm before implementation.

### Code Changes

_(to be completed)_

---

## Pull Request

**PR Link:** _(to be created)_

**PR Description:** _(draft to be added)_

**Maintainer Feedback:**
- **2026-06-19:** Self-assigned and posted claim comment proposing the `date_bin(INTERVAL DAY TO SECOND, TIMESTAMP(p), TIMESTAMP(p))` signature; awaiting maintainer confirmation.

**Status:** Awaiting signature confirmation

---

## Learnings & Reflections

### Technical Skills Gained

_(to be completed)_

### Challenges Overcome

_(to be completed)_

### What I'd Do Differently Next Time

_(to be completed)_

---

## Resources Used

- [PostgreSQL `date_bin` documentation](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-BIN) — reference semantics for the function
- [trinodb/trino Issue #20428](https://github.com/trinodb/trino/issues/20428) — the issue being addressed
- Trino's existing `date_trunc` implementation (`core/trino-main/src/main/java/io/trino/operator/scalar/timestamp/DateTrunc.java`) — structural model for parametric datetime functions
