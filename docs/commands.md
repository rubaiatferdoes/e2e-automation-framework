# Command Reference

Every command: what it needs, what it gives you.

Type these inside Claude Code, in this folder. Each one stops when it is
done — **you approve before running the next.** Nothing chains
automatically.

**A** = runs as a forked agent · **S** = runs as a skill, in your own
conversation, so you can discuss the result

---

## All 18 at a glance

| Step | Command | Gives you |
|---|---|---|
| 0 **S** | [`/setup:env-readiness`](#0--setupenv-readiness) | READY / NOT READY table |
| 1 **A** | [`/discovery:explore-app`](#1--discoveryexplore-app) | Sequences + screenshots |
| 2 **A** | [`/discovery:event-definition`](#2--discoveryevent-definition) | Events → API calls → DB tables |
| 3 **A** | [`/discovery:validation-definition`](#3--discoveryvalidation-definition) | Field rules + who enforces them |
| 1b **S** | [`/requirements:analyze-requirements`](#1b--requirementsanalyze-requirements) | Features and rules from documents *(optional)* |
| 4 **A** | [`/test-design:test-planner`](#4--test-designtest-planner) | The test plan |
| 5 **A** | [`/test-design:test-case-designer`](#5--test-designtest-case-designer) | Manual test cases |
| 6 **A** | [`/test-data:data-factory`](#6--test-datadata-factory) | Dummy datasets |
| 7a **A** | [`/automation-api:api-test-generator`](#7a--automation-apiapi-test-generator) | API tests |
| 7b **A** | [`/automation-api:api-test-reviewer`](#7b--automation-apiapi-test-reviewer) | Stable API tests |
| 7c **A** | [`/automation-ui:playwright-generator`](#7c--automation-uiplaywright-generator) | UI + E2E tests |
| 7d **A** | [`/automation-ui:playwright-reviewer`](#7d--automation-uiplaywright-reviewer) | Stable UI tests |
| 8 **A** | [`/database:db-validator`](#8--databasedb-validator) | DB checks + filled table mappings |
| 9 **A** | [`/execution:test-executor`](#9--executiontest-executor) | Execution report + evidence |
| 10 **A** | [`/healing:investigate-failures`](#10--healinginvestigate-failures) | A verdict per failure |
| 11 **A** | [`/reporting:quality-reporter`](#11--reportingquality-reporter) | Release verdict |
| any **S** | [`/change:impact-analysis`](#anytime--changeimpact-analysis) | What a change affects |
| any **S** | [`/reporting:report-bug`](#anytime--reportingreport-bug) | A bug report |

---

## Arguments they all accept

**Work on one part.** The command prints the resolved IDs first, then
touches only those.

| Select by | Example |
|---|---|
| Sequence · screen · module | `SEQ-002` · `screen:key-list` · `module:keys` |
| Flow step (1–6) · E2E scenario | `flow:3` · `E2E-03` |
| Definition IDs | `EVT-012` · `VAL-007` |
| Test cases · range | `TC-014 TC-020` · `TC-010..TC-015` |
| Priority · layer | `P0` · `API` · `Security` |

They combine with AND — `F-003 P0` is the P0 cases of feature F-003.

**Revise a draft** — re-run with feedback; it edits in place, same IDs:
`/test-design:test-case-designer module:keys feedback: merge TC-013 and TC-015`

**Use an unapproved input** — add `allow-draft`.

---

## 0 · `/setup:env-readiness`
> Is the environment ready? Two minutes that save hours.

`/setup:env-readiness` · `/setup:env-readiness ui`

- **In** — `.env` only
- **Out** — a table in chat. No files written
- **Note** — `⚠️ SKIPPED` is not a failure; missing `automation/` and empty optional settings are expected early
- **Note** — check 5 (tenants can log in) saves the most time; an expired password looks like "the app is broken"

## 1 · `/discovery:explore-app`
> Travel a workflow and write down the path.

`/discovery:explore-app "generate an API key for an endpoint"`

- **In** — `.env` + a URL or a workflow in words
- **Out** — `docs/discovery/sequences/SEQ-*.md` · screenshots · `automation/discovery/walks/*.walk.ts` · raw artifacts *(gitignored)*
- **Stops if** — `TEST_ENV_BASE_URL` is empty, or the URL is not on the test host
- **Note** — it writes `qa-` data and deletes it, and asks before the first delete or anything that emails a person
- **Note** — records screens, network and console in **one pass**; steps 2 and 3 mine that recording and never re-walk

## 2 · `/discovery:event-definition`
> What each click actually does behind the screen.

`/discovery:event-definition SEQ-002`

- **In** — approved `SEQ-*.md` + the artifacts from step 1
- **Out** — `docs/discovery/events/<module>.events.md` + the Event column filled back into the sequences
- **Note** — one user action = one `EVT-`, however many calls it fired. Types: `onload`, `click`, `submit`, `traverse`, `background`
- **Note** — routes are generalised (`/api/endpoints/{id}/keys`) because they become the `waitForResponse` targets in every generated test
- **Note** — the DB half stays `PENDING` until step 8. It never guesses a table name

## 3 · `/discovery:validation-definition`
> Every rule on every field, and who actually enforces it.

`/discovery:validation-definition screen:generate-key-dialog`

- **In** — approved `SEQ-*.md` + field dumps + the events (for API routes)
- **Out** — `docs/discovery/validations/<module>.validations.md` + a screenshot of each rule firing
- **Note** — five passes: read the DOM, probe each field, **send the same bad value straight to the API**, probe count/quota limits, then compare everything against the standard validation baseline — a gap is a finding, never an expected result
- **Note** — `Enforced: client` or `none` is a defect, not a rule: anyone calling the API bypasses it. That verdict moves the test case to the API layer
- **Note** — it writes data (uniqueness and quota probes must), deletes it, and lists it by name

## 1b · `/requirements:analyze-requirements`
> Optional. What the product was *supposed* to do.

`/requirements:analyze-requirements module:keys`

- **In** — `docs/requirements/` (PRD, FRD, mockups, notes)
- **Out** — `docs/analysis/requirement-analysis.md` — features `F-`, rules `BR-`, questions `Q-`
- **Note** — runs *alongside* discovery, not instead of it. The planner compares the two and every disagreement becomes a question or a suspected bug
- **Note** — empty folder? It says so and stops. Valid state: discovery plus your confirmation is then the only oracle

## 4 · `/test-design:test-planner`
> Decide what gets tested, how deep, and what won't be.

`/test-design:test-planner module:keys`

- **In** — approved sequences + events + validations (+ the analysis, if it exists)
- **Out** — `docs/test-plan.md`. **One page, the plan only** — test cases are step 5
- **Note** — where discovery and the documents disagree it lists the conflict and raises a `Q-`. It never picks a side
- **Note** — depth comes from evidence: a `client`-only rule needs an API-layer case, a polling event is a flakiness risk, a `PENDING` DB mapping makes that cell `Later`

## 5 · `/test-design:test-case-designer`
> The manual test cases, one file each.

`/test-design:test-case-designer module:keys`

- **In** — approved test plan + **sequences + validations**
- **Out** — `docs/test-cases/<module>/TC-*.md` + module and overall indexes
- **Note** — events are deliberately *not* an input: a manual case is what a person does and sees. Routes and tables belong to step 7
- **Note** — real screen names, labels and messages only. It cannot round a limit or tidy a message; if it can't state one exactly, that's a `Q-`

## 6 · `/test-data:data-factory`
> Dummy data, built from the validation rules.

`/test-data:data-factory module:keys`

- **In** — approved test cases + validations
- **Out** — `test-data/*.json` + fills `data:` and the DAT column in the test case files
- **Note** — boundaries come straight from the rule: `min 3, max 40` produces 2, 3, 40, 41
- **Note** — patterns fill at runtime: `{qa-name}`, `{repeat:k:41}`, `{faker:…}` seeded by TC number
- **Note** — it never shrinks data to fit a column. That mismatch is a finding

## 7a · `/automation-api:api-test-generator`
> API tests, from the test cases and the recorded routes.

`/automation-api:api-test-generator module:keys`

- **In** — approved API/Docs test cases + `test-data/` + **events** + the Swagger URL
- **Out** — `automation/api/` (clients, schemas, validators) · `automation/tests/api/`
- **Note** — expected values come from the **test case**; routes, payload shapes and call order come from the **events**. Never from the Swagger — that is under test
- **Note** — test case ≠ event ≠ Swagger is a finding plus a `Q-`, never reconciled by picking one

## 7b · `/automation-api:api-test-reviewer`
> Make the API tests trustworthy.

`/automation-api:api-test-reviewer module:keys`

- **In** — the API tests + test cases + events
- **Out** — fixes applied in place · review report in chat · `BUG-` if the app is at fault
- **Note** — checks layering, typing, TC traceability, schema validation, 401 on every auth failure, and that every `client`/`none` rule has an API-layer test
- **Note** — stability gate: `--repeat-each=3` fully green. It never weakens an assertion to get there

## 7c · `/automation-ui:playwright-generator`
> UI and E2E tests, Page Object Model.

`/automation-ui:playwright-generator module:keys`

- **In** — approved UI/E2E test cases + `test-data/` + **sequences, walk files, DOM artifacts, events, validations**
- **Out** — `automation/ui/` (locators, pages) · `automation/tests/ui/` · `automation/tests/e2e/`
- **Note** — selectors come from the recorded DOM, waits from the event routes, asserted messages from the validations. Nothing is guessed
- **Note** — setup and cleanup go through the API. Only the step under test uses the UI

## 7d · `/automation-ui:playwright-reviewer`
> Make the UI tests trustworthy.

`/automation-ui:playwright-reviewer module:keys`

- **In** — the UI/E2E tests + test cases + definitions
- **Out** — fixes in place · review report · `BUG-` if the app is at fault
- **Note** — POM layering, messages matched character for character, waits on recorded routes, no `waitForTimeout`, cleanup on failure
- **Note** — same three-green-runs gate

## 8 · `/database:db-validator`
> Confirm the app stored it correctly, and complete the event map.

`/database:db-validator module:keys`

- **In** — read-only DB settings in `.env` + approved events with `PENDING` + validations + test cases
- **Out** — `docs/analysis/schema-check.md` · `automation/db/`, `automation/tests/db/` · **fills the `db:` blocks in the event files**
- **Note** — two inputs, both in place before step 0: the **read-only user** in `.env` and the **schema export** in `docs/requirements/db/`. The live schema still comes from `information_schema`; the export is the intended schema it is compared against
- **Note** — it finds each table by tracing the unique `qa-` record, then diffs that row by primary key — so it is safe on a shared environment and needs no quiet window
- **Note** — until `DB_TYPE` is set, DB steps inside tests skip *only that step*; everything else runs

📖 **Full detail** — the tracer method, what it cannot see, safety, and
what to ask DevOps for: [requirements/db/README.md](requirements/db/README.md)

## 9 · `/execution:test-executor`
> Run the suites and collect the evidence.

`/execution:test-executor module:keys` · `/execution:test-executor smoke`

- **In** — `automation/` + test cases + `.env`
- **Out** — `docs/reports/execution-<date>.md` + trace, video and screenshots in `automation/reports/artifacts/`
- **Note** — readiness first, then Smoke → API → UI → E2E → DB
- **Note** — it labels each failure and **hands off. It does not file bugs** — one agent should not both run the tests and judge them
- **Note** — the parity check (cases with no test, tests with no case, sequences with no coverage) blocks any "regression green" claim

## 10 · `/healing:investigate-failures`
> Every failure leaves with a verdict and an owner.

`/healing:investigate-failures` · `… TC-014` · `… verify` · `… apply FIX-001`

- **In** — the latest execution report + its artifacts + test cases + definitions
- **Out** — `docs/reports/investigation-<date>.md` · `docs/bugs/BUG-*.md` (`Status: PROPOSED`) · `FIX-` proposals — **nothing filed, nothing applied: the lead decides on both**
- **Note** — reproduces twice, then decides from trace, video and network — never from the error message alone
- **Note** — verdicts: broken test (`FIX-`) · application issue (`BUG-`) · stale expectation (back to discovery) · environment · data
- **Note** — **it may fix a broken test; never a failing expectation.** No sleeps, no retries, no `test.skip`, no commits

## 11 · `/reporting:quality-reporter`
> The release verdict, for people who will not read the detail.

`/reporting:quality-reporter module:keys`

- **In** — test cases + discovery + latest execution + latest investigation + bugs
- **Out** — `docs/reports/quality-report-<date>.md`
- **Note** — verdict on line one: READY / READY WITH RISKS / NOT READY
- **Note** — every number comes from the files; missing data is written "no data" with the file named, never estimated
- **Note** — it prints how much of the product has been **discovered at all**, so "100% passing" can't hide a thin walk

## Anytime · `/change:impact-analysis`
> Find what a change affects, and update only that.

`/change:impact-analysis docs/requirements/changes/CR-001.md` · `… answers`

- **In** — a `CR-*.md`, new rows in `answers.md`, a new document version, **or a discovery re-run that found the app changed**
- **Out** — `docs/changes/CR-*-impact.md` → after approval, targeted edits + the exact follow-up commands
- **Note** — phase 1 analyses and **stops**. Phase 2 only runs once you approve
- **Note** — traces `SEQ-/EVT-/VAL- → TC- → data → tests → walk files`. Nothing renumbered, nothing deleted; removed items become `DEPRECATED`

## Anytime · `/reporting:report-bug`
> A bug report in the standard format.

`/reporting:report-bug TC-014 revoked key still returns 200`

- **In** — a TC ID, a failing test, or a description
- **Out** — `docs/bugs/BUG-<nnn>-<short-name>.md`
- **Note** — it asks you for anything a developer would need. Severity is QA's call; priority is proposed and the lead confirms

---

## Approval gates

Every step checks its input and stops if it is still a draft.

```
discovery (SEQ/EVT/VAL) → approve → test plan → approve → test cases → approve → data + automation
```

| Where status lives | Values |
|---|---|
| Files with front matter — sequences, events, validations, test cases | `status: draft` → `approved` → `deprecated` |
| Documents — test plan, analysis, impact reports | `> Status: DRAFT` → `> Status: APPROVED — <name>, <date>` |

Approve by editing the line, or by saying so:
`approve docs/test-cases/keys/`

**Approving discovery is the most important click in the pipeline.** It
is the moment an observation stops being "what the app did when we poked
it" and becomes "what the app is supposed to do".

**Approved files are locked.** Two narrow exceptions: the data factory
fills `data:` and the DAT column of approved test cases, and the
db-validator fills the `db:` block of approved event files. Everything
else goes through `/change:impact-analysis`.
