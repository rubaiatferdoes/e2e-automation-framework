# Test Cases — <module>

> Status: DRAFT

> Based on: `docs/analysis/requirement-analysis.md` (version/date)
> Environment: mirror only · Data: dummy only · Prefix: `qa-`

## Summary
| Priority | Positive | Negative | Boundary | E2E | Total |
|---|---|---|---|---|---|
| P0 | | | | | |
| P1 | | | | | |
| P2 | | | | | |
| P3 | | | | | |

## Test cases

| TC ID | Title | Req IDs | Module / Flow | Priority | Type | Layer | Precondition | Steps | Expected result | Data |
|---|---|---|---|---|---|---|---|---|---|---|
| TC-001 | Create endpoint with valid name succeeds | F-001, BR-001 | endpoints / 1 | P0 | Positive | UI+API | Logged in as qa-tenant-a | 1. Open Endpoints page<br>2. Click "Create"<br>3. Enter name<br>4. Click "Save" | Success message shown; endpoint listed; GET /endpoints returns it | DAT-001 |

**Column rules**
- Module / Flow: module slug from the analysis + E2E flow step(s) 1–6;
  for an E2E scenario add its ID, e.g. `keys / 3-6 / E2E-03`
- Type: Positive · Negative · Boundary · E2E · Security
- Layer: UI · API · DB · Docs (combine with `+`)
- Expected result: exact and checkable (message text, status code, field value)

## Data tables (for data-driven cases)

### TC-005 data — invalid endpoint names
| Row | Input | Expected error |
|---|---|---|
| 1 | (empty) | "Name is required" — ASSUMPTION — verify (Q-004) |

## Traceability matrix
| Req ID | Test cases | Covered? |
|---|---|---|
| F-001 | TC-001, TC-005 | ✅ |

## Regression candidates (parked)
| Idea | Why parked |
|---|---|

## Open assumptions
| Assumption | Linked question |
|---|---|

## Change log
| CR | Date | Summary | IDs touched |
|---|---|---|---|
| Baseline | YYYY-MM-DD | First approved test cases | all |
