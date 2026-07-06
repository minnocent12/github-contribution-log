# Contribution 3: Add `to_unixtime_nanos` function

**Contribution Number:** 3
**Student:** Mirenge Innocent
**Issue:** https://github.com/trinodb/trino/issues/6819
**Status:** Phase I ✓ | Phase II ✓ | Phase III pending — awaiting maintainer scope confirmation

> 📁 Part of my [Open Source Contribution Log](README.md) · Contribution #3 of 3

---

## Why I Chose This Issue

After implementing `date_bin` (Contribution #2), I wanted a third issue in the same area of the codebase to keep momentum and deepen my understanding while my other PRs await review. `to_unixtime_nanos` is a natural fit: it lives in the same `operator/scalar/timestamptz/` package I just worked in, follows the same two-precision-variant pattern I used for `date_bin`, and the model file (`timestamptz/ToUnixTime.java`) is almost identical to what I'd produce — so I can focus on correctness rather than discovery.

There's also a practical motivation from my internship at Home Depot's Supply Chain Technology group. Supply chain telemetry and sensor data often carries sub-millisecond timestamps. Once you have high-precision `TIMESTAMP(p)` data in Trino, the existing `to_unixtime()` returns a DOUBLE (seconds), which silently loses precision for very recent epoch values. `to_unixtime_nanos` returning a BIGINT eliminates that rounding loss and enables exact nanosecond-level time arithmetic.

The issue was open, unassigned, had no linked PR, and a Trino maintainer had already informally flagged it as "relatively straightforward."

---

## Understanding the Issue

### Problem Description

Trino's existing `to_unixtime(TIMESTAMP(p) WITH TIME ZONE)` returns a `DOUBLE` representing seconds since the Unix epoch. For high-precision timestamps (`TIMESTAMP(9)`, `TIMESTAMP(12)`), the `DOUBLE` representation loses sub-millisecond precision because 64-bit IEEE 754 floating point only has ~15–16 significant decimal digits, while a current epoch value in nanoseconds already requires 19 digits. Users working with sub-millisecond timestamp data have no way to extract an exact integer epoch value.

### Expected Behavior

A new `to_unixtime_nanos` function that accepts any `TIMESTAMP(p) WITH TIME ZONE` and returns a `BIGINT` of epoch nanoseconds:

```sql
SELECT to_unixtime_nanos(TIMESTAMP '2021-01-01 00:00:00.123456789 UTC');
-- 1609459200123456789
```

No precision loss — nanoseconds fit well within BIGINT range until the year 2262.

### Current Behavior

Running any `to_unixtime_nanos(...)` call fails immediately:

```
Function 'to_unixtime_nanos' not registered
```

The only alternative — `to_unixtime()` — returns `DOUBLE`, which rounds away sub-millisecond digits.

### Affected Components

- **`operator/scalar/timestamptz/`** — the per-type class directory for timezone-aware timestamp scalar functions. New file `ToUnixTimeNanos.java` goes here, mirroring `ToUnixTime.java`.
- **`metadata/SystemFunctionBundle.java`** — the central registry; a `.scalar(ToUnixTimeNanos.class)` call needs to be added beside the existing `ToUnixTime` registration.
- **Two precision variants** — same pattern as all other `timestamptz/` functions:
  - Short form: `long` (packed millis, p ≤ 3) via `DateTimeEncoding.unpackMillisUtc`
  - Long form: `LongTimestampWithTimeZone` (p > 3) with `getEpochMillis()` + `getPicosOfMilli()`

---

## Reproduction Process

### Environment Setup

- **Branch:** `add-to-unixtime-nanos-function` — to be created from upstream `master` once maintainer confirms scope (see open questions in claim comment on issue #6819)
- **Fork:** https://github.com/minnocent12/trino
- **Setup approach:** Same environment as Contributions #1 and #2 (already working): macOS, Temurin JDK 25, Maven Wrapper. Targeted module build: `./mvnw test -pl core/trino-main -am -DskipTests`.
- **Challenge (pre-existing):** Maven enforcer `RequireUpperBoundDeps` violations for `okhttp-jvm` and `jackson-databind` appear on first build — confirmed pre-existing upstream issue in master, not caused by new files. Same issue encountered and resolved in Contribution #2.

### Steps to Reproduce

1. Attempt `SELECT to_unixtime_nanos(TIMESTAMP '2021-01-01 00:00:00.123456789 UTC')` in a Trino session.
2. Observed: `Function 'to_unixtime_nanos' not registered` — the function does not exist.
3. Attempt `SELECT to_unixtime(TIMESTAMP '2021-01-01 00:00:00.123456789 UTC')` — succeeds but returns `1.6094592001234568E9` (DOUBLE), which has rounded the sub-millisecond portion.
4. Ran `grep -r "to_unixtime_nanos" core/trino-main/src/main/java/` → zero results, confirming no implementation exists anywhere.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — missing-feature issue; the function simply does not exist.
- **Codebase search:** `grep -r "to_unixtime_nanos" core/trino-main/src/` → zero results in both `main/java/` and `test/`. No implementation, no tests, no registration in `SystemFunctionBundle.java`.
- **Confirmed gap:** `to_unixtime` exists at `core/trino-main/src/main/java/io/trino/operator/scalar/timestamptz/ToUnixTime.java` (introduced 2020-06-09, commit `62cf1510d7c`). Running it on a sub-millisecond timestamp returns a rounded `DOUBLE`, demonstrating the precision loss.
- **Issue scope confirmed:** Open, unassigned, no existing PR when claimed (2026-07-06).

---

## Solution Approach

### Analysis

The existing `to_unixtime` implementation is minimal — 44 lines, two methods, one class. The new `to_unixtime_nanos` follows the same structure:

| | `to_unixtime` | `to_unixtime_nanos` |
|---|---|---|
| Input | `TIMESTAMP(p) WITH TIME ZONE` | `TIMESTAMP(p) WITH TIME ZONE` |
| Return | `DOUBLE` (seconds) | `BIGINT` (nanoseconds) |
| Short form math | `unpackMillisUtc(ts) / 1_000.0` | `unpackMillisUtc(ts) * 1_000_000L` |
| Long form math | `epochMillis / 1000.0 + picosOfMilli / 1e12` | `epochMillis * 1_000_000L + picosOfMilli / 1_000L` |

Only the return type and arithmetic change. No new infrastructure is needed.

### Proposed Solution

New class `operator/scalar/timestamptz/ToUnixTimeNanos.java`, registered in `SystemFunctionBundle.java`:

```java
@ScalarFunction("to_unixtime_nanos")
public final class ToUnixTimeNanos {
    private ToUnixTimeNanos() {}

    @LiteralParameters("p")
    @SqlType(StandardTypes.BIGINT)
    public static long toUnixTimeNanos(@SqlType("timestamp(p) with time zone") long timestamp) {
        return unpackMillisUtc(timestamp) * 1_000_000L;
    }

    @LiteralParameters("p")
    @SqlType(StandardTypes.BIGINT)
    public static long toUnixTimeNanos(@SqlType("timestamp(p) with time zone") LongTimestampWithTimeZone timestamp) {
        return timestamp.getEpochMillis() * 1_000_000L + timestamp.getPicosOfMilli() / 1_000L;
    }
}
```

**Overflow check:** Current epoch in nanos ≈ 1.75 × 10¹⁸. `Long.MAX_VALUE` ≈ 9.2 × 10¹⁸. Safe until ~year 2262 (positive) and ~year 1677 (negative). No overflow risk for real timestamps.

**Open questions pending maintainer reply:**
1. Should this also support plain `TIMESTAMP(p)` (without time zone), matching a broader scope?
2. Is `to_unixtime_nanos` the preferred name, or should `to_unixtime_micros` be added as a companion?

### Implementation Plan

Using UMPIRE framework:

**Understand:** Add `to_unixtime_nanos(TIMESTAMP(p) WITH TIME ZONE) → BIGINT` returning epoch nanoseconds.

**Match:** `timestamptz/ToUnixTime.java` is the structural model — introduced in commit `62cf1510d7c` (2020-06-09, "Implement variable-precision timestamp with time zone type") and stable for 6+ years, confirming it is the canonical pattern. Used `git log --oneline --follow` to verify history.

**Plan:**
1. `operator/scalar/timestamptz/ToUnixTimeNanos.java` (NEW) — two precision variants, `BIGINT` return
2. `metadata/SystemFunctionBundle.java` (MODIFY) — add import + `.scalar(ToUnixTimeNanos.class)` beside `ToUnixTime` registration
3. `operator/scalar/timestamptz/TestToUnixTimeNanos.java` (NEW) — test all precisions p=0..12, sub-millisecond, negative epoch
4. `docs/src/main/sphinx/functions/datetime.md` (MODIFY) — add function entry beside `to_unixtime`

**Implement:** Branch `add-to-unixtime-nanos-function` in fork https://github.com/minnocent12/trino

**Review:**
- [ ] Mirrors `ToUnixTime.java` structure and `SystemFunctionBundle` registration
- [ ] Correct nanos math: millis × 1,000,000 + picosOfMilli ÷ 1,000
- [ ] Both short (`long`) and long (`LongTimestampWithTimeZone`) precision variants
- [ ] BIGINT overflow safe for real-world timestamps
- [ ] Matches PostgreSQL `extract(epoch)` × 10⁹ semantics for integer nanoseconds
- [ ] Docs added beside `to_unixtime`

**Evaluate:** Run `./mvnw test -pl core/trino-main -Dtest=TestToUnixTimeNanos` — all pass.

---

## Testing Strategy

### Unit Tests

- [ ] All 13 precisions p=0..12 (mirrors pattern from `TestDateBin`, `TestDateTrunc`)
- [ ] Epoch boundary: `TIMESTAMP '1970-01-01 00:00:00 UTC'` → `0`
- [ ] Sub-millisecond precision: `TIMESTAMP '2021-01-01 00:00:00.123456789 UTC'` → `1609459200123456789`
- [ ] Before epoch: negative epoch nanoseconds (e.g., 1969-12-31)
- [ ] `LongTimestampWithTimeZone` path (p > 3) produces same result as computed manually

### Integration Tests

_(to be assessed — `to_unixtime` has no dedicated integration tests; unit tests run through the expression engine are sufficient)_

### Manual Testing

_(to be completed after branch is created)_

---

## Implementation Notes

### Week 6 Progress

- Identified issue #6819 as a backup contribution while waiting for responses on #29773 and #30075.
- Confirmed `to_unixtime_nanos` does not exist in the codebase via `grep`.
- Analyzed `timestamptz/ToUnixTime.java` — introduced 2020-06-09 (commit `62cf1510d7c`), unchanged in structure since then; confirmed as the correct model.
- Posted claim comment on issue #6819 (2026-07-06) with implementation plan and two scope questions.
- Awaiting maintainer reply before creating branch and writing code.

### Code Changes

_(to be completed — pending maintainer scope confirmation)_

---

## Pull Request

**PR Link:** _(to be opened after maintainer replies and branch is created)_

**Maintainer Feedback:**
- **2026-07-06:** Claim comment posted; two scope questions asked (plain TIMESTAMP support, naming preference).

**Status:** Awaiting maintainer reply on issue #6819

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

- [Issue #6819 — Support to_unixtime_nanos](https://github.com/trinodb/trino/issues/6819)
- [Existing model: timestamptz/ToUnixTime.java](https://github.com/trinodb/trino/blob/master/core/trino-main/src/main/java/io/trino/operator/scalar/timestamptz/ToUnixTime.java) — introduced commit `62cf1510d7c` (2020-06-09)
- [Related: Issue #6754 — INTERVAL sub-millisecond precision truncation](https://github.com/trinodb/trino/issues/6754) — the root motivation for needing nanos access
- [Trino datetime functions documentation](https://trino.io/docs/current/functions/datetime.html)
