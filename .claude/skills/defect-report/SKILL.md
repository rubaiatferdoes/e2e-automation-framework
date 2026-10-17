---
name: defect-report
description: Write a clear, reproducible bug report (BUG-) with severity, priority, steps, expected vs actual, evidence, and a link to the test case and requirement. Use when a test finds an application bug, or when asked to log, write, or file a defect.
---

# Defect Report

**Goal:** a developer can reproduce the bug from the report alone, on
the first try, without asking QA anything.

**Output:** one file per bug: `docs/bugs/BUG-<nnn>-<short-name>.md`
(next free number), using [templates/bug-template.md](templates/bug-template.md).

## Before writing

- [ ] Reproduced at least twice on the mirror environment
- [ ] It's an **application bug** — not a script, data, or environment
      problem (see the classification in `test-executor`)
- [ ] Searched `docs/bugs/` — not already reported

## Severity vs priority (they are different)

**Severity = how bad the impact is** (QA sets it):

| Severity | Meaning | Example |
|---|---|---|
| Sev-1 Critical | Security breach, data loss, core flow blocked for everyone | Tenant B can call tenant A's endpoint |
| Sev-2 Major | Main feature broken, no workaround | Revoked key still works |
| Sev-3 Minor | Feature works wrong but has a workaround | Docs missing an error response |
| Sev-4 Trivial | Cosmetic | Typo in success message |

**Priority = how soon to fix** (product/lead confirms it): P0 now → P3 later.

## Writing rules

- **Title:** `[Area] <what goes wrong> when <condition>`
  e.g. `[API Key] Revoked key still returns 200 after 5 minutes`
- **Steps:** numbered, one action each, starting from a known state.
  Include the exact data used (dummy `qa-` data only).
- **Expected:** quote the requirement or test case (BR-/TC- ID).
- **Actual:** what happened — exact message, status code, response.
- **Evidence:** screenshot, trace, request/response (with keys masked:
  `sk_...a1b2`), timestamp, environment build/version.
- Facts only. No blame, no guessing at the cause in the main sections —
  put suspicions under "Notes".
