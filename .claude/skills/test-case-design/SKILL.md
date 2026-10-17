---
name: test-case-design
description: Design manual test cases from an approved requirement analysis using standard techniques (equivalence partitioning, boundary values, decision tables, state transitions, error guessing). Produces one Markdown file per TC- case in a folder per module, plus module and overall indexes with a traceability matrix. Use when writing test cases, a test plan, or checking test coverage.
---

# Test Case Design

**Goal:** the smallest set of test cases that covers every requirement
and every important risk. More cases is not better — better cases are.

**Input:** `docs/analysis/requirement-analysis.md` (must be approved).
**Output:** `docs/test-plan.md` plus one file per test case:

```
docs/test-cases/
├── README.md                          overview — template: templates/overall-index-template.md
└── <module-slug>/
    ├── README.md                      module index — template: templates/module-index-template.md
    └── TC-014-revoked-key-rejected.md one case — template: templates/tc-template.md
```

Templates: [tc-template.md](templates/tc-template.md) ·
[module-index-template.md](templates/module-index-template.md) ·
[overall-index-template.md](templates/overall-index-template.md)

**File rules**
- Folder = the module slug from the analysis (`keys`, `endpoints`, …).
- File name = `TC-<nnn>-<short-title-in-kebab-case>.md`. TC numbers are
  global across modules and never reused.
- The front matter (between `---` lines) holds the fields agents read:
  `id, title, module, flow, e2e, requirements, priority, type, layer,
  data, status`. Keep them exact — the body is for people.
- `status`: `draft` → `approved` → (later, if removed) `deprecated`.
  A deprecated file stays in place.
- After adding or changing cases, update the module `README.md` and the
  overall `docs/test-cases/README.md` (counts, links, traceability).

## Step 1 — Pick the technique for each rule

| If the rule is about... | Use this technique | Example |
|---|---|---|
| A field with valid/invalid groups | **Equivalence partitioning** — one test per group | Name: valid / empty / special chars |
| A limit (length, count, size) | **Boundary values** — min-1, min, max, max+1 | Name max 50 → test 49, 50, 51 |
| Several conditions together | **Decision table** — one test per meaningful combination | Key valid? × Endpoint active? |
| Something with states | **State transition** — each allowed and blocked move | Key: active → revoked → (call) ✗ |
| Known weak spots | **Error guessing** — duplicates, double-click, refresh mid-flow | Create same name twice |
| The whole journey | **E2E scenario** — see the `e2e-endpoint-flow` skill | Create → docs → key → call → delete |

## Step 2 — Write the cases

- One file per test case, from `tc-template.md`.
- Title format: `<action> <condition> <expected outcome>`
  e.g. "Create endpoint with duplicate name shows error".
- Steps: a numbered table, one action per row, each with its own
  expected result. No "etc.".
- Expected result: a concrete, checkable outcome — exact message,
  status code, or visible state. Never "works correctly".
- Every case links to at least one F- or BR- ID.
- Data-driven cases list their input rows in the file's "Test data"
  table; the data factory adds the DAT- IDs. Never real customer data.

## Step 3 — Set priority

| Priority | Meaning | Example |
|---|---|---|
| P0 | Core flow broken = product unusable | Can't create endpoint / key doesn't work |
| P1 | Security or data-isolation risk, or major feature | Other tenant's key accepted |
| P2 | Important validation or error handling | Duplicate name error message |
| P3 | Cosmetic / rare edge | Tooltip text |

## Step 4 — Keep it lean

- Merge similar checks into ONE data-driven case with a data table
  (e.g. all invalid name formats = one case, 6 data rows).
- Max 5–7 negative/boundary cases per feature. Park the rest under
  "Regression candidates" in the module `README.md`.
- Duplicate coverage is a mistake, not thoroughness.

## Step 5 — Self-check before finishing

- [ ] Traceability matrix: every F- and BR- maps to ≥1 TC
- [ ] Every E2E flow step (CLAUDE.md) has ≥1 P0 case
- [ ] Every feature has positive AND negative cases
- [ ] Security cases from the `api-key-security` skill included
- [ ] No case expects a 5xx response
- [ ] Unknowns marked `ASSUMPTION — verify` and linked to a Q- ID
- [ ] A junior QA could run every case without asking questions
- [ ] Every case file's front matter is complete; file name matches its ID
- [ ] Module `README.md` and overall `README.md` list every case, with
      correct counts and links

Finish with a summary: total cases by priority and type, uncovered
requirements (should be zero), open assumptions.
