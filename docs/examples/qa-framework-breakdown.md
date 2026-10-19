# QA Agent Framework — Command Breakdown

Oct 18, 2026

The QA framework for our API endpoint builder works from the running
application, not from documents. You give it a URL and a test login; it
travels the app, records what it finds, and builds the tests from that.
A QA engineer types a command, a specialist AI agent does that step, and
a person approves the result before the next step. All 18 commands are
verified.

## At a glance

```
 0 Readiness ──▶ 1 Explore ──▶ 2 Events ──▶ 3 Validations
                     (travel the app, record what it does)
                                                 │
        ┌────────────────────────────────────────┘
        ▼   ← you approve here: observation becomes expectation
 4 Test plan ──▶ 5 Test cases ──▶ 6 Test data
                                       │
        ┌──────────────────────────────┘
        ▼
 7 Automation ──▶ 8 DB checks* ──▶ 9 Execution ──▶ 10 Investigate ──▶ 11 Report
   (API + UI        (needs DB        (run, collect    (broken test →     (release
   generators,       access)          evidence)        a fix you         verdict)
   then reviewers)                                     approve;
                                                       broken product
                                                       → a bug)

 Every arrow is a review gate.  * waiting on read-only DB access
```

Requirement documents are **optional** and plug in beside step 1: when
they exist, the test planner compares them against what the app actually
does, and every disagreement becomes a question or a suspected bug.
Product updates can enter at any point through `/change:impact-analysis`.

## The idea that shapes everything

The application is the only thing that can tell us how it behaves — and
it is also the thing we are testing. So:

> **What the app does is an observation, not the truth.**

Discovery records; a person confirms. Until that approval, nothing is an
expected result. Anything that looks wrong — a missing validation, a
silent failure, a rule the browser enforces and the server doesn't — is
raised as a finding instead of being written down as correct behaviour.

Without that rule, a framework that learns from the app would simply
certify the app's own bugs as correct.

## Commands

Run them in order from the framework folder; each one stops for approval.

| Step | Command | What it does | Output |
| --- | --- | --- | --- |
| 0 | `/setup:env-readiness` | App reachable, logins work, tools installed | READY / NOT READY (in chat) |
| 1 | `/discovery:explore-app` | Travels a workflow, screenshots every screen, records the traffic | `docs/discovery/sequences/` |
| 2 | `/discovery:event-definition` | Each click/load → the API calls it fires → the DB tables it touches | `docs/discovery/events/` |
| 3 | `/discovery:validation-definition` | Probes every field; checks whether the server enforces the rule or only the browser | `docs/discovery/validations/` |
| 1b | `/requirements:analyze-requirements` | Optional — reads requirement documents, if any exist | `docs/analysis/` |
| 4 | `/test-design:test-planner` | Decides scope, depth, priority; lists conflicts | `docs/test-plan.md` |
| 5 | `/test-design:test-case-designer` | Writes the manual test cases | `docs/test-cases/<module>/` |
| 6 | `/test-data:data-factory` | Dummy data from the validation rules | `test-data/` |
| 7 | `/automation-api:*` and `/automation-ui:*` | Writes and stabilizes API, UI and E2E tests | `automation/` |
| 8 | `/database:db-validator` | Fills the PENDING table mappings, adds read-only DB checks | `automation/db/`, `docs/analysis/schema-check.md` |
| 9 | `/execution:test-executor` | Runs the suites, collects log, video, screenshots | `docs/reports/execution-*.md` |
| 10 | `/healing:investigate-failures` | Diagnoses every failure: broken test or broken product | `docs/reports/investigation-*.md`, `docs/bugs/` |
| 11 | `/reporting:quality-reporter` | Release verdict for managers | `docs/reports/quality-report-*.md` |
| Any | `/change:impact-analysis` | Applies a change to only what it affects | `docs/changes/` |

Each output folder is created the first time its step runs, so a new
project starts with almost nothing. Setup and the full command reference
are in the repo `README.md`.

## Example: the API keys module

An illustrative run on one module, starting from nothing but a URL, a
login, and the sentence *"check the API key part."*

| Step | You type | You get |
| --- | --- | --- |
| 1 | `explore-app "generate an API key"` | SEQ-002, 4 steps, 5 screenshots. Finding: a "Copy key" button on the masked list. Question: is the key really shown only once? |
| 2 | `event-definition SEQ-002` | 4 events. `POST /api/endpoints/{id}/keys` takes 3.9 s — explains a future flaky test. DB mapping PENDING |
| 3 | `validation-definition screen:generate-key-dialog` | 4 rules. VAL-008 (max 40) is enforced **in the browser only** — the API accepted 256 characters |
| 4–5 | `test-planner` then `test-case-designer` | 5 test cases. The boundary case runs at the API layer because of VAL-008 |
| 9–10 | `test-executor` then `investigate-failures` | 12 runs: 9 pass, 2 fail, 1 flaky → BUG-003 (Sev-2) drafted, two fixes proposed — all three await the lead |

BUG-003 traces back through TC-013 → DAT-024 → VAL-008 → SEQ-002 step 4
→ a screenshot. Only the keys module was touched.

The full example, with sample files and code, is in
[module-keys-walkthrough.md](module-keys-walkthrough.md).

## Why the enforcement check matters

Step 3 does something a normal UI test suite never does: it takes each
rule the browser enforces and sends the same bad value straight to the
API. In the example that single check found a real defect — the UI stops
at 40 characters, the API stores 256 — and it moved the test to the layer
where it would actually catch something. A UI-only test would have passed
forever while the hole stayed open.

## Working on one part

Add a selection to any command; other parts are never touched.

| Pick | Example |
| --- | --- |
| A workflow | `seq:create-endpoint` |
| A screen | `screen:key-list` |
| A module | `module:keys` |
| Test cases | `TC-014 TC-020` |
| A priority | `P0` |

## When the application changes

A redesigned screen does not mean rewriting the suite.

1. Re-run discovery for that part only: `/discovery:explore-app seq:create-endpoint`.
2. Approve the updated sequence.
3. `/change:impact-analysis` lists every affected event, validation, test
   case, data record and automated test — and waits.
4. Re-run only the commands it lists.

IDs never change, so history stays traceable.

## Failures get investigated, not re-run

Every failure leaves with a verdict and an owner: broken test,
application issue, stale expectation, environment, or data. A broken test
gets an exact diff with its blast radius — **proposed, never applied** —
and a checklist proving the fix doesn't hide a bug. A healer may fix a
broken test; it may never fix a failing expectation. It cannot weaken an
assertion, add a sleep, skip a test, or commit anything.

That limit is the point. A self-healing suite that quietly repairs itself
around a regression is worse than no suite at all.

## Database checks

QA never writes to the database: tests create data through the API like a
real customer, then read the database to confirm the app stored it
correctly.

1. DevOps adds a read-only login to `.env`. That is the whole ask — the
   schema is read from `information_schema`, so nobody has to export one
   and backend is not involved.
2. `/database:db-validator` compares the schema with the validation rules
   (e.g. the UI allows 40 characters, the column is `VARCHAR(30)`), fills
   the PENDING table mapping on every event — it traces the unique `qa-`
   record to find the table and column, then diffs that row by primary
   key to see exactly what changed — and adds the DB checks.
3. Until then, only the DB parts are skipped; everything else runs.
