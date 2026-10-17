# QA Agent Framework — Command Breakdown

Oct 17, 2026

The QA framework for our API endpoint builder is ready: a QA engineer types a command, a specialist AI agent does that step, and a person approves the result before the next step. It covers the full E2E flow in TypeScript, works one module at a time, and handles product changes without redoing work. All 13 commands are verified.

## At a glance

Each step hands a file to the next, and a person approves it first:

```
 0 Env readiness ──▶ 1 Requirements ──▶ 2 Test planning ──▶ 3 Test data
                                                                 │
 ┌───────────────────────────────────────────────────────────────┘
 ▼
 4 Automation ──▶ 5 DB checks* ──▶ 6 Execution ──▶ 7 Report
   (API + UI       (needs DB         (results        (release
   generators,      type)             and bugs)       verdict)
   then reviewers)

 Every arrow is a review gate.  * waiting on a product decision
```

Step 5 waits until the database type is chosen. Product updates can enter at any point through `/change:impact-analysis`.

## Commands

Run them in order from the framework folder; each one stops for approval.

| Step | Command | What it does | Output (location) |
| --- | --- | --- | --- |
| 0 | `/setup:env-readiness` | Checks the test environment is ready | READY / NOT READY table (in chat) |
| 1 | `/requirements:analyze-requirements` | Finds features, rules, gaps and questions | `docs/analysis/requirement-analysis.md` |
| 2 | `/test-design:test-planner` | Decides the scope, writes test cases | `docs/test-plan.md`, `docs/test-cases/<module>/TC-*.md` (one file per case) |
| 3 | `/test-data:data-factory` | Creates dummy data | `test-data/` |
| 4 | `/automation-api:api-test-generator` then `api-test-reviewer` | Writes and stabilizes API tests | `automation/api/`, `automation/tests/api/` |
| 4 | `/automation-ui:playwright-generator` then `playwright-reviewer` | Writes and stabilizes UI + E2E tests | `automation/ui/`, `automation/tests/ui/`, `automation/tests/e2e/` |
| 5 | `/database:db-validator` | DB checks (after the DB type and schema are ready) | `automation/db/`, `docs/analysis/schema-check.md` |
| 6 | `/execution:test-executor` | Runs tests, sorts failures, files bugs | `docs/reports/`, `docs/bugs/` |
| 7 | `/reporting:quality-reporter` | Release verdict for managers | `docs/reports/` |
| Any | `/change:impact-analysis` | Applies a product change to only what it affects | `docs/changes/` |

Each output folder is created the first time its step runs, so a new project shows only `docs/requirements/`. Setup steps and the full command reference are in the repo `README.md`.

## Example: the API keys module

An illustrative run on one module. Input: PRD section 4 ("up to 5 keys per endpoint, name 3–40 characters, shown once, revocable, sent in the x-api-key header"), one mockup and a client note.

| Step | You type | You get |
| --- | --- | --- |
| 1 | `analyze-requirements module:keys` | 4 features, 5 rules, 5 gaps. First question: how fast must a revoked key stop working? Product answers "60 seconds", which becomes a new rule |
| 2 | `test-planner module:keys` | 8 test cases, e.g. TC-014 "Revoked key rejected within 60 s" (P0) |
| 3 | `data-factory module:keys` | Boundary data for key names: 2, 3, 40 and 41 characters, plus fake keys |
| 4 | `api-test-generator module:keys` | 7 API tests; the UI-only case is left for the UI generator |
| 5 | `test-executor module:keys` | 10 of 11 runs pass. TC-014 fails: the revoked key still works after 60 s, so bug BUG-003 (Sev-2) is filed |

Every link is traceable: BUG-003 → TC-014 → the 60-second rule → product's answer → PRD section 4.4. Only the keys module was touched. The full example, with sample files and code, is in [module-keys-walkthrough.md](module-keys-walkthrough.md) (same folder).

## Working on one part

Add a selection to any command to work on one part instead of everything; other parts are never touched.

| Pick | Example |
| --- | --- |
| A module | `module:keys` |
| A workflow step | `flow:3` |
| Test cases | `TC-014 TC-020` |
| A priority | `P0` |

## Product updates

A change from product updates only what it affects; nothing is rebuilt.

1. Save the email or chat as a change request in `docs/requirements/changes/`.
2. Run `/change:impact-analysis`. It lists every affected rule, test case and test, and waits for approval.
3. Re-run only the commands it lists, for only the affected test cases.

IDs never change, so history stays traceable.

## Database checks

QA never writes to the database: tests create data through the API like a real customer, then read the database to confirm the app stored it correctly.

1. Backend adds the schema (no data) to `docs/requirements/db/`; DevOps adds a read-only login to `.env`.
2. Run `/database:db-validator`. It first compares the schema with the requirements (e.g. PRD says 40 characters, column allows 30), then adds DB checks to the existing tests.
3. Run `npm run test:db`. Until the database is ready, only the DB steps are skipped; everything else still runs.