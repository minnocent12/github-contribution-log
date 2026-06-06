# Contribution 1: Add function to convert strings to title case

**Contribution Number:** 1  
**Student:** Mirenge Innocent  
**Issue:** https://github.com/trinodb/trino/issues/2942  
**Status:** Phase I — Complete

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of my Java background and the kind of data infrastructure work I encounter as a software engineering intern in Home Depot's Supply Chain Technology group. Trino is a distributed SQL query engine used by large organizations to query massive datasets — the same category of tooling used in retail supply chain analytics. A built-in `initcap` / title case function is a small but high-value addition that engineers working with product names, store names, and supplier data deal with constantly.

What makes this issue especially appealing is the technical depth behind a seemingly simple feature. The maintainer has made clear that the implementation must operate directly on Trino's internal `Slice` type — a byte array abstraction — rather than converting to a Java String. This means I'll need to understand how Trino handles string operations at a low level, which is a meaningful learning goal. The issue has been open since 2020, has an active maintainer who gave clear implementation guidance, and has no open pull request — making it a genuinely available contribution with a clear path forward.

---

## Understanding the Issue

### Problem Description

Trino is missing a built-in SQL function to convert strings to title case (also known as `initcap`). Title case means the first letter of each word is capitalized and the remaining letters are lowercase — for example, `'hello world'` becomes `'Hello World'`. Every major SQL engine (PostgreSQL, Oracle, Snowflake, Redshift, SparkSQL) has this function natively, but Trino does not. Users currently have to work around this using a complex `regexp_replace` expression, which is not intuitive and breaks on accented characters in non-English languages.

### Expected Behavior

Trino should support a native `initcap(string)` SQL function that converts any string to title case:

```sql
SELECT initcap('hello world');     -- 'Hello World'
SELECT initcap('the HOME depot');  -- 'The Home Depot'
SELECT initcap('new york city');   -- 'New York City'
```

### Current Behavior

Trino has no `initcap` function. Calling it throws an error. Users must use a workaround like:

```sql
SELECT regexp_replace('hello world', '(\w)(\w*)', x -> upper(x[1]) || lower(x[2]));
```

This workaround is verbose, hard to remember, and fails on strings with accented characters (e.g., Portuguese words like `EscritóRio` get incorrectly capitalized after accent marks).

### Affected Components

- **Core string functions** — where `upper()` and `lower()` are implemented in Trino's Java codebase. The new `initcap` function will be added alongside these.
- **Trino's `Slice` type** — Trino represents strings internally as `Slice` objects (byte arrays) for performance. The implementation must operate directly on the `Slice` rather than converting to a Java `String` and back, as the maintainer has explicitly stated that approach is too slow and adding an external library for a single method is not acceptable in this project.
- **`airlift/slice` library** — the upstream library that defines the `Slice` type. The maintainer linked [airlift/slice#177](https://github.com/airlift/slice/issues/177) as a reference for where the low-level byte manipulation logic may need to be added.

---

## Reproduction Process

### Environment Setup

- **OS:** macOS (Apple Silicon)
- **JDK:** Temurin 25 (Trino requires JDK ≥ 25.0.1 Temurin or Oracle — standard Homebrew OpenJDK 21 does not work)
  - Installed via: `brew install --cask temurin@25`
  - Set: `export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-25.jdk/Contents/Home`
- **Build tool:** Maven Wrapper (`./mvnw`) included in the Trino repo
- **Fork:** https://github.com/minnocent12/trino, branch `add-initcap-function`

### Steps to Reproduce

1. Clone Trino, open a Trino SQL session, and run: `SELECT initcap('hello world');`
2. Observed: `Function 'initcap' not registered` — the function simply does not exist in the engine.
3. For comparison, `SELECT upper('hello world')` and `SELECT lower('HELLO WORLD')` work fine — only title case is missing.

### Reproduction Evidence

- **My findings:** Confirmed that `initcap` is absent from `StringFunctions.java` where all other string scalar functions live. The functions `upper()`, `lower()`, and `length()` all follow the same `@ScalarFunction` annotation pattern. There is no `initcap` entry anywhere in the Trino source. The issue has been open since 2020 with explicit maintainer guidance on how to implement it.

---

## Solution Approach

### Analysis

The root cause is simply a missing feature: `initcap` was never implemented. Trino's string functions live in `core/trino-main/src/main/java/io/trino/operator/scalar/StringFunctions.java`. Every scalar string function follows the same pattern:

1. Annotate with `@ScalarFunction`, `@Description`, `@LiteralParameters`, `@SqlType`
2. Accept a `Slice` parameter (Trino's internal UTF-8 byte array string type)
3. Return a `Slice`

The key constraint set by maintainer `dain` on the original issue: **no Java String conversion**. The implementation must work directly on the `Slice` byte representation using Unicode code point iteration, the same way `SliceUtf8.toUpperCase()` and `SliceUtf8.toLowerCase()` work in the underlying `airlift/slice` library.

### Proposed Solution

Implement `initcap(varchar)` and `initcap(char)` directly in `StringFunctions.java` by iterating over Unicode code points in the `Slice`, applying `Character.toTitleCase(int)` to the first letter of each word and `Character.toLowerCase(int)` to the rest, using the existing `tryGetCodePointAt` / `setCodePointAt` / `lengthOfCodePoint` utilities already used throughout the file.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Add a native SQL `initcap(string)` function to Trino that converts strings to title case, operating directly on Trino's internal `Slice` type without String conversion.

**Match:** The existing `charUpper()` / `charLower()` pair in `StringFunctions.java` shows the exact pattern for adding both a `varchar` and a `char` variant of a string function. The `SliceUtf8` class (in airlift/slice) shows how to iterate code points correctly for Unicode safety.

**Plan:**
1. Add `initcap(@SqlType("varchar(x)") Slice utf8)` after `charUpper()` in `StringFunctions.java` — iterates code points, tracks word boundaries by checking `Character.isLetterOrDigit()`, applies title/lower case per position
2. Add `charInitcap(@SqlType("char(x)") Slice utf8)` immediately after — delegates to `initcap()`
3. Add `testInitcap()` and `testCharInitcap()` in `TestStringFunctions.java` covering: empty string, single word, multiple words, ALL CAPS input, punctuation, hyphens, numbers, Unicode accented characters

**Implement:** Branch `add-initcap-function` in fork https://github.com/minnocent12/trino

**Review:**
- [x] No Java String conversion — pure code point iteration on `Slice`
- [x] Handles invalid UTF-8 bytes gracefully (copies byte through unchanged via `tryGetCodePointAt` returning negative)
- [x] Handles Unicode (e.g., `österreich` → `Österreich`)
- [x] Follows the same annotation pattern as all other scalar functions in the file
- [x] Both `varchar` and `char` type variants provided (matching the `upper`/`lower` precedent)

**Evaluate:** Run `./mvnw test -pl core/trino-main -Dtest=TestStringFunctions#testInitcap+testCharInitcap` — all tests pass.

---

## Testing Strategy

### Unit Tests

- [x] Empty string returns empty string
- [x] Single lowercase word: `'hello'` → `'Hello'`
- [x] Multiple words: `'hello world'` → `'Hello World'`
- [x] ALL CAPS input: `'HELLO WORLD'` → `'Hello World'`
- [x] Mixed case: `'hElLo WoRlD'` → `'Hello World'`
- [x] Punctuation word boundary: `'what!!'` → `'What!!'`
- [x] Hyphen word boundary: `'hello-world'` → `'Hello-World'`
- [x] Multiple spaces: `'hello  world'` → `'Hello  World'`
- [x] Numbers (digit resets word boundary): `'hello2world'` → `'Hello2World'`
- [x] Digit-leading string: `'123abc'` → `'123Abc'`
- [x] Unicode: `'österreich'` → `'Österreich'`
- [x] CHAR type: all above cases via `charInitcap()`

### Manual Testing

Verified the implementation compiles cleanly (`./mvnw install -pl core/trino-main -am -DskipTests`) and all targeted tests pass (`./mvnw test -pl core/trino-main -Dtest=TestStringFunctions#testInitcap+testCharInitcap`). Maven exits with no `FAILURE` or `ERROR` lines.

---

## Implementation Notes

### Week 1 Progress

**Environment setup:** Discovered Trino requires JDK ≥ 25.0.1 Temurin — the standard macOS Homebrew OpenJDK 21 does not meet the requirement. Installed Temurin 25 via `brew install --cask temurin@25`. Had to set `JAVA_HOME` manually in the shell session.

**Issue research:** Read the full comment thread on trinodb/trino#2942. Maintainer `dain` explicitly rejected the approach of converting `Slice` → Java `String` → back because it allocates unnecessarily on the hot path. Also confirmed that adding airlift/slice as a dependency for a single method is not acceptable — the implementation must be self-contained in `StringFunctions.java`.

**Implementation:** Located `charUpper()` in `StringFunctions.java` as the model, then wrote `initcap()` below it. Key decisions:
- Used `tryGetCodePointAt` (not `getCodePointAt`) to handle invalid UTF-8 gracefully rather than throwing
- `startOfWord = !Character.isLetterOrDigit(codePoint)` means any non-alphanumeric character (space, hyphen, punctuation) triggers title-case on the next character — matching PostgreSQL's `initcap` behavior
- Used `Slices.ensureSize` because title-case can expand a code point (e.g., some Unicode characters have a different-width title case form)

### Code Changes

- **Files modified:**
  - `core/trino-main/src/main/java/io/trino/operator/scalar/StringFunctions.java` — added `initcap()` and `charInitcap()`
  - `core/trino-main/src/test/java/io/trino/operator/scalar/TestStringFunctions.java` — added `testInitcap()` and `testCharInitcap()`
- **Branch:** `add-initcap-function`
- **Approach decisions:** Chose to implement entirely in `StringFunctions.java` rather than waiting for `airlift/slice#177` to land. This keeps the PR self-contained and unblocked.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
