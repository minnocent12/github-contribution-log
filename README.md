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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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
