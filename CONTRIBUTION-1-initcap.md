# Contribution 1: Add function to convert strings to title case

**Contribution Number:** 1  
**Student:** Mirenge Innocent  
**Issue:** https://github.com/trinodb/trino/issues/2942  
**Status:** Phase IV — Complete ✓ | PR #29773 open, awaiting maintainer merge (rename discussion ongoing)

> 📁 Part of my [Open Source Contribution Log](README.md) · Contribution #1 of 2

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

- **Commit showing reproduction:** Not applicable — this is a *missing-feature* issue, not a code defect. There is no buggy commit to point to; the "reproduction" is confirming the function does not exist. The fix is the first implementation commit ([e17e199](https://github.com/minnocent12/trino/commit/e17e1990beb4de0911e69442eeb6fe41cf09cccb)).
- **Screenshots/logs:** Running `SELECT initcap('hello world')` in a Trino session returns a `Function 'initcap' not registered` error, while `upper()` and `lower()` resolve normally — confirming title case is the only one of the three missing.
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

Implement `initcap(varchar)` and `initcap(char)` directly in `StringFunctions.java` using `SliceUtf8.toTitleCase()` — a new method added to the `airlift/slice` library (version 2.8) by maintainer `wendigo`, available starting with Trino's airlift 435 upgrade. This keeps the implementation a one-liner that delegates to the officially supported library method, consistent with how `upper()` and `lower()` delegate to `SliceUtf8.toUpperCase()` and `SliceUtf8.toLowerCase()`.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Add a native SQL `initcap(string)` function to Trino that converts strings to title case, operating directly on Trino's internal `Slice` type without String conversion.

**Match:** The existing `charUpper()` / `charLower()` pair in `StringFunctions.java` shows the exact pattern for adding both a `varchar` and a `char` variant of a string function. The `SliceUtf8` class (in airlift/slice) shows how to iterate code points correctly for Unicode safety.

**Plan:**
1. Add `initcap(@SqlType("varchar(x)") Slice utf8)` after `charUpper()` in `StringFunctions.java` — calls `SliceUtf8.toTitleCase(utf8)` directly
2. Add `charInitcap(@SqlType("char(x)") Slice utf8)` immediately after — also calls `SliceUtf8.toTitleCase(utf8)`
3. Add `testInitcap()` and `testCharInitcap()` in `TestStringFunctions.java` covering: empty string, single word, multiple words, ALL CAPS input, punctuation, hyphens, numbers, Unicode accented characters
4. Add documentation to `docs/src/main/sphinx/functions/string.md`, `list.md`, and `list-by-topic.md`

**Implement:** Branch `add-initcap-function` in fork https://github.com/minnocent12/trino

**Review:**
- [x] Uses `SliceUtf8.toTitleCase()` — the officially supported library method, no custom byte logic needed
- [x] No Java String conversion
- [x] Handles Unicode (e.g., `österreich` → `Österreich`)
- [x] Follows the same annotation pattern as all other scalar functions in the file
- [x] Both `varchar` and `char` type variants provided (matching the `upper`/`lower` precedent)
- [x] Documentation added in all three required docs locations

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
- [x] Hyphen (not a Unicode word boundary): `'hello-world'` → `'Hello-world'`
- [x] Multiple spaces: `'hello  world'` → `'Hello  World'`
- [x] Numbers (digit resets word boundary): `'hello2world'` → `'Hello2World'`
- [x] Digit-leading string: `'123abc'` → `'123Abc'`
- [x] Unicode: `'österreich'` → `'Österreich'`
- [x] CHAR type: all above cases via `charInitcap()`

### Integration Tests

A scalar string function like `initcap()` does not require dedicated integration tests — its behavior is fully exercised by the unit tests in `TestStringFunctions`, which run the function end-to-end through Trino's expression evaluation engine (not just the raw Java method). Beyond that, Trino's CI runs the full product-test matrix (Hive, Delta Lake, Iceberg, fault-tolerant execution, etc.) against the change; all relevant suites pass, confirming the new function does not regress query execution across connectors.

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

### Week 2 Progress

**PR opened (June 6):** Opened draft PR #29773 against `trinodb/trino`. 98/99 CI checks passed immediately. Only failure: CLA (Contributor License Agreement) not yet on file.

**CLA process:** Downloaded, signed, and emailed the Trino Foundation Individual CLA to cla@trino.io. On June 15, Martin Traverso (co-founder of Trino Software Foundation, co-creator of Presto and Trino) personally confirmed the CLA was received and added `minnocent12` to the `trinodb/Contributors` GitHub team.

**Documentation added (June 10):** Maintainer `ebyhr` added `needs-docs` and `syntax-needs-review` labels. Added `initcap()` documentation to three files: `functions/string.md` (description + example), `functions/list.md` (alphabetical index), and `functions/list-by-topic.md` (string functions section). GitHub Actions auto-applied the `docs` label confirming the change was recognized.

**Maintainer feedback — use airlift method (June 18):** Maintainer `PlePato` asked `@wendigo` whether our implementation aligned with the airlift work. `wendigo` (who assigned the issue and also authored `airlift/slice` PR #177) replied: "new Slice method can be used." This prompted us to update from our manual code-point loop to `SliceUtf8.toTitleCase()`, available in `airlift/slice` 2.8 (included with Trino's airlift 435 upgrade that `wendigo` himself committed to master on June 16).

**Rebase required (June 18):** When syncing with upstream to get airlift 435, we initially used `git merge` which created a merge commit. Trino's CI check `check-commits-dispatcher` rejected this: "PR requires a rebase. Found: 1 merge commits." Fixed by cherry-picking our 3 commits cleanly onto the latest upstream master and force-pushing.

### Code Changes

- **Files modified:**
  - `core/trino-main/src/main/java/io/trino/operator/scalar/StringFunctions.java` — added `initcap()` and `charInitcap()` using `SliceUtf8.toTitleCase()`
  - `core/trino-main/src/test/java/io/trino/operator/scalar/TestStringFunctions.java` — added `testInitcap()` and `testCharInitcap()`
  - `docs/src/main/sphinx/functions/string.md` — added `initcap()` function entry
  - `docs/src/main/sphinx/functions/list.md` — added `initcap` to alphabetical index
  - `docs/src/main/sphinx/functions/list-by-topic.md` — added `initcap` to string functions topic list
- **Branch:** `add-initcap-function`
- **Key decision:** Initially implemented manually using `tryGetCodePointAt` + `Character.toTitleCase(int)` + `setCodePointAt`. Updated to `SliceUtf8.toTitleCase()` after `wendigo`'s feedback, making the implementation a clean one-liner aligned with the rest of the codebase.

---

## Pull Request

**PR Link:** https://github.com/trinodb/trino/pull/29773

**PR Description:** Adds a native `initcap(string)` SQL function that converts strings to title case (first letter of each word uppercased, the rest lowercased), implemented via `SliceUtf8.toTitleCase()` (airlift/slice 2.8). Includes `varchar` and `char` variants, unit tests in `TestStringFunctions`, and documentation in `functions/string.md`, `list.md`, and `list-by-topic.md`. Closes #2942.

**Status:** Ready for review — all 100 CI checks pass; CLA signed; awaiting maintainer approval

**CLA:** Approved by Martin Traverso (Trino co-founder) on 2026-06-15. Added to `trinodb/Contributors` GitHub team.

**Maintainer Feedback:**
- **2026-06-10:** `ebyhr` added `needs-docs` and `syntax-needs-review` labels → Added documentation to three docs files; `docs` label auto-applied by GitHub Actions
- **2026-06-18:** `PlePato` asked `@wendigo` if implementation aligns with airlift work → `wendigo` replied "new Slice method can be used" → Updated implementation to use `SliceUtf8.toTitleCase()` from airlift/slice 2.8
- **2026-06-18:** CI `check-commits-dispatcher` failed: "PR requires a rebase. Found: 1 merge commits." → Fixed by cherry-picking 3 commits onto latest upstream master
- **2026-06-19:** Re-triggered CI to clear flaky cloud/container suites → all 100 checks pass; marked PR ready for review
- **2026-06-19:** `martint` (Trino co-founder) suggested renaming `initcap` → `title_case` for clarity → Replied with cross-engine compatibility context (PostgreSQL/Oracle/Snowflake/Redshift/Spark all use `initcap`) and offered `title_case` as primary name with `initcap` as alias
- **2026-06-27:** One week since `martint`'s rename suggestion — awaiting direction before implementing rename

---

## Learnings & Reflections

### Technical Skills Gained

- **How Trino processes strings internally:** Trino never uses Java `String` on the hot path — everything is a `Slice` (a byte array wrapper from airlift). Understanding why this matters (garbage collection, allocation cost) gave me insight into how high-performance data systems are built.
- **Unicode-aware string processing:** Learned the difference between bytes, Java `char` (UTF-16), and Unicode code points. Functions that look trivial (capitalize a letter) require code point iteration to handle multi-byte characters correctly.
- **Open source contribution workflow end-to-end:** Forking, feature branching, writing tests that mirror the project's existing patterns, adding docs in the right format, signing a CLA, responding to maintainer feedback, rebasing instead of merging, and navigating CI requirements.
- **Maven multi-module builds:** Learned to use `-pl core/trino-main -am -DskipTests` to build only the relevant module and its dependencies, and how to run targeted tests by class and method name.
- **Git rebase vs merge:** Trino requires a linear history with no merge commits. Experienced the difference firsthand — had to cherry-pick our commits onto upstream master after an accidental merge commit failed the `check-commits-dispatcher` check.
- **Engaging with maintainers professionally:** Learned to comment on issues to claim work, respond to labels quickly, and adapt to feedback without being defensive.

### Challenges Overcome

- **JDK version:** Trino requires JDK ≥ 25.0.1 Temurin specifically — the standard macOS Homebrew OpenJDK 21 silently fails. Solved by installing Temurin 25 and setting `JAVA_HOME` manually.
- **airlift/slice dependency:** The low-level `SliceUtf8.toTitleCase()` method didn't exist in the released library when we started. Initially implemented the logic manually. When `wendigo` merged airlift/slice PR #177 and bumped Trino to airlift 435, we updated to use the new method — making our implementation cleaner and maintainer-approved.
- **Merge commit rejected by CI:** Syncing with upstream via `git merge` created a merge commit that Trino's `check-commits-dispatcher` CI check explicitly rejects. Solved by aborting the rebase and cherry-picking our 3 commits cleanly onto the latest upstream master.
- **CLA process:** Required finding, filling out, digitally signing, and emailing a PDF legal document. Took 9 days from submission to approval. The `recheck` comment didn't auto-trigger the CLA bot — had to close/reopen the PR to force it to re-run after the CLA was processed.

### What I'd Do Differently Next Time

- **Sync with upstream via rebase from the start**, not merge. Any time you need to pull in upstream changes to a feature branch, use `git fetch upstream && git rebase upstream/master` — never `git merge`.
- **Sign the CLA on day one.** It takes days to process and blocks everything. The moment you decide to contribute to a project, check whether they have a CLA and sign it before writing any code.
- **Read the CI requirements before pushing.** Trino's `CONTRIBUTING.md` explicitly says no merge commits. Reading it upfront would have avoided the extra rebase cycle.

---

## Resources Used

- [trinodb/trino CONTRIBUTING.md](https://github.com/trinodb/trino/blob/master/CONTRIBUTING.md) — contribution guidelines, coding standards, commit message format
- [trinodb/trino Issue #2942](https://github.com/trinodb/trino/issues/2942) — the original issue with maintainer guidance from `dain` and `wendigo`
- [airlift/slice PR #177](https://github.com/airlift/slice/pull/177) — `wendigo`'s "Add SliceUtf8.toTitleCase()" PR, the upstream implementation our function delegates to
- [Trino Foundation CLA](https://github.com/trinodb/cla) — CLA information and PDF form
- [airlift/slice SliceUtf8 source](https://github.com/airlift/slice/blob/master/src/main/java/io/airlift/slice/SliceUtf8.java) — reference for understanding how `toUpperCase` and `toLowerCase` work, which guided our initial implementation pattern
