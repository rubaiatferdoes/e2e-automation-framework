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

Step 5 waits until the database type is chosen. Product updates can enter at any point through `/qa:change:impact-analysis`.

## Commands

Run them in order from the framework folder; each one stops for approval.

| Step | Command | What it does | Main output |
| --- | --- | --- | --- |
| 0 | `/qa:setup:env-readiness` | Checks the mirror is ready and is not production | READY / NOT READY |
| 1 | `/qa:requirements:analyze-requirements` | Finds features, rules, gaps and questions | Requirement analysis |
| 2 | `/qa:test-design:test-planner` | Decides the scope, writes test cases | Test plan + test cases |
| 3 | `/qa:test-data:data-factory` | Creates dummy data | `test-data/` |
| 4 | `/qa:automation-api:api-test-generator` then `api-test-reviewer` | Writes and stabilizes API tests | `automation/` |
| 4 | `/qa:automation-ui:playwright-generator` then `playwright-reviewer` | Writes and stabilizes UI + E2E tests | `automation/` |
| 5 | `/qa:database:db-validator` | DB checks (after the DB type and schema are ready) | `automation/db/` |
| 6 | `/qa:execution:test-executor` | Runs tests, sorts failures, files bugs | Execution report + bugs |
| 7 | `/qa:reporting:quality-reporter` | Release verdict for managers | Quality report |
| Any | `/qa:change:impact-analysis` | Applies a product change to only what it affects | Impact report |

Setup steps and the full command reference are in the repo `README.md`.

## Example: the API keys module

An illustrative run on one module. Input: PRD section 4 ("up to 5 keys per endpoint, name 3–40 characters, shown once, revocable, sent in the x-api-key header"), one mockup and a client note.

| Step | You type | You get |
| --- | --- | --- |
| 1 | `analyze-requirements module:keys` | 4 features, 5 rules, 5 gaps. Blocker question: how fast must a revoked key stop working? Product answers "60 seconds", which becomes a new rule |
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

1. Save the email or chat as a change request in `docs/input/changes/`.
2. Run `/qa:change:impact-analysis`. It lists every affected rule, test case and test, and waits for approval.
3. Re-run only the commands it lists, for only the affected test cases.

IDs never change, so history stays traceable.

## Status and open questions

All 13 commands passed a live check (17 Oct 2026); real test runs start when the mirror environment is ready. Endpoint editing is out of scope for now, and database checks wait for the DB choice.
