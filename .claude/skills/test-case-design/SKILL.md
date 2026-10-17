---
name: test-case-design
description: Design manual test cases from an approved requirement analysis using standard techniques (equivalence partitioning, boundary values, decision tables, state transitions, error guessing). Produces numbered TC- cases in a fixed table format plus a traceability matrix. Use when writing test cases, a test plan, or checking test coverage.
---

# Test Case Design

**Goal:** the smallest set of test cases that covers every requirement
and every important risk. More cases is not better — better cases are.

**Input:** `docs/analysis/requirement-analysis.md` (must be approved).
**Output:** `docs/test-plan.md` and `docs/test-cases.md`, using
[templates/test-case-template.md](templates/test-case-template.md).

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

- Use the table format from the template. One row = one test case.
- Title format: `<action> <condition> <expected outcome>`
  e.g. "Create endpoint with duplicate name shows error".
- Steps: numbered, one action each, no "etc.".
- Expected result: a concrete, checkable outcome — exact message,
  status code, or visible state. Never "works correctly".
- Every case links to at least one F- or BR- ID.
- Put test data in the case as a reference (`DAT-xxx` or "see data
  table") — never real customer data.

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
  "Regression candidates" at the end of the file.
- Duplicate coverage is a mistake, not thoroughness.

## Step 5 — Self-check before finishing

- [ ] Traceability matrix: every F- and BR- maps to ≥1 TC
- [ ] Every E2E flow step (CLAUDE.md) has ≥1 P0 case
- [ ] Every feature has positive AND negative cases
- [ ] Security cases from the `api-key-security` skill included
- [ ] No case expects a 5xx response
- [ ] Unknowns marked `ASSUMPTION — verify` and linked to a Q- ID
- [ ] A junior QA could run every case without asking questions

Finish with a summary: total cases by priority and type, uncovered
requirements (should be zero), open assumptions.
