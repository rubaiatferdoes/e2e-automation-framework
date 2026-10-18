# docs/discovery — what we learned by travelling the app

> Status: DRAFT

This folder is written by the three discovery commands. It is the
framework's picture of the product: what a user can do, what happens
behind each action, and what each field accepts.

**Everything here is an observation, not a rule.** Nothing becomes an
expected result until a human approves it (CLAUDE.md section 2).

## What is in here

| Folder | Holds | Written by |
|---|---|---|
| `sequences/` | `SEQ-*.md` — one user workflow each, step by step, with a screenshot per screen | `/discovery:explore-app` |
| `events/` | `<module>.events.md` — every click/load/navigation, the API calls it fires, the DB tables it touches | `/discovery:event-definition` |
| `validations/` | `<module>.validations.md` — every field rule, its exact message, and whether the server enforces it | `/discovery:validation-definition` |
| `screens/` | Screenshots referenced by the files above | explore-app, validation-mapper |

Raw recordings (full network logs, DOM dumps) stay in
`automation/discovery/artifacts/` and are not committed.

## Module index

| Module | Slug | Sequences | Events | Validations | Flow steps | Status |
|---|---|---|---|---|---|---|
| Endpoint management | `endpoints` | — | — | — | 1, 6 | Not started |
| Swagger docs | `docs` | — | — | — | 2 | Not started |
| API keys | `keys` | — | — | — | 3, 5 | Not started |
| Endpoint runtime | `runtime` | — | — | — | 4 | Not started |
| Access & tenants | `access` | — | — | — | — | Not started |

Status: `Not started` → `Walked — awaiting approval` → `Approved YYYY-MM-DD`

## E2E flow coverage

One row per step of the journey in CLAUDE.md. A step with no sequence is
a hole in what we know about the product.

| Flow step | Sequences | Covered? |
|---|---|---|
| 1. Create endpoint (UI) | | |
| 2. Swagger docs generated | | |
| 3. Generate API key | | |
| 4. Call endpoint with key | | |
| 5. Revoke key → key is dead | | |
| 6. Delete endpoint → endpoint is gone | | |

## Open questions from discovery

| ID | Question | Raised by | Blocks |
|---|---|---|---|

## How to read a finding

Discovery files end with a Findings table. A finding is something that
looked wrong while walking — it is not yet a bug, because nobody has
confirmed what the product is supposed to do. Review them with the lead:
each one becomes a bug, a question, or an accepted behaviour.

A discovery run that produced no findings usually means the walk was too
shallow, not that the product is perfect.
