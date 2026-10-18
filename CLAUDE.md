# QA Framework — Shared Rules

These rules apply to EVERY agent, skill, and command in this project.
Agents and skills do not repeat them — they rely on this file.

## 1. Product under test

A platform where a customer builds their own API endpoint in a web UI.
The platform then auto-generates Swagger (OpenAPI) docs and issues an
API key, and the customer (or their apps) calls the endpoint with it.
Similar products: Xano, Supabase, AWS API Gateway.

**The application is already running.** We are given a URL and test
logins, not a specification. We learn the product by travelling it.

The E2E flow we own:

```
1. Create endpoint (UI) → 2. Swagger docs auto-generated → 3. Generate API key
→ 4. Call endpoint with key → 5. Revoke key → key is dead → 6. Delete endpoint → endpoint is gone
```

## 2. How we learn the product — and what counts as "correct"

Two sources, and they are not equal:

| Source | What it gives us | Status |
|---|---|---|
| **Discovery** — walking the live app (`docs/discovery/`) | What the product **does**: sequences, events, API calls, field validations | **Observation.** Never automatically correct |
| **Requirements** — PRD/FRD/mockups, if any exist (`docs/requirements/`) | What the product **should** do | Intent. Optional — many screens have none |

**Observation is not truth.** This is the most important rule in this
file. An agent that walks the app records what it saw; it never promotes
that recording into an expected result on its own. An observation becomes
an expected result only when:

- a human approves it at the discovery gate (section 9), or
- a requirement document confirms it.

Where the two disagree, that is a **finding**, not a decision to make:
raise a `Q-` question or a suspected bug and let a person choose.

Agents must actively look for what is wrong instead of recording it as
normal. Always report, never bless:

- a required field with no validation, or a validation with no message
- a 5xx response (always a bug — see section 6)
- an error message that leaks internals (stack trace, SQL, file path)
- an action whose API call fails but whose UI shows success
- two screens that word the same rule differently
- a field that appears to write nothing anywhere

## 3. Current scope decisions (update when they change)

| Topic | Decision | Status |
|---|---|---|
| Customizing/editing an endpoint after creation | OUT of scope | Not confirmed if customers can edit — revisit |
| Database type | Unknown — read-only access is coming | Event DB mapping stays `PENDING` and DB checks are skipped until `DB_TYPE` + schema are provided (see `docs/requirements/db/README.md`) |
| Cross-tenant access response (403 vs 404) | Unknown | Ask product — never hard-code either |
| Key revocation delay (how fast a key stops working) | Unknown | Ask product — poll up to that limit |
| Requirement documents | Optional | If `docs/requirements/` is empty, discovery plus human confirmation is the only oracle. Say so in the test plan |
| Automation language and design | **TypeScript + Playwright** for discovery, UI, API, and E2E in one project (`automation/`); **Page Object Model** for UI, **API Object Model** for API — see the `ts-pom-standards` skill | Team standard |

## 4. Safety rules (never break these)

- Test ONLY on the test environment.
- Use ONLY dummy data. Never real customer data.
- Every test and every discovery walk creates its own data with the `qa-`
  prefix (e.g. `qa-tc014-lz3k9a-w2-x7f1` — test case, time, worker, random)
  and deletes it afterwards.
- **Discovery may write.** To learn a validation rule you have to trigger
  it, so discovery agents may fill forms, submit, and create records —
  under the `qa-` prefix, on the test environment, with cleanup, and
  within the limits in section 5.
- Database access is READ-ONLY. Only the application writes.
- Secrets and URLs come from environment variables (see `.env.example`).
  Never write a key, password, or URL into code or docs.
- Two pre-created test tenants are needed: `qa-tenant-a` and `qa-tenant-b`.
- Screenshots and recordings are evidence: mask API keys, tokens, and
  passwords before committing them (`sk_...a1b2`).

## 5. Discovery rules (walking a live application)

- **Never leave the test environment.** Check the host of every URL
  before navigating; a link to production, a payment provider, or an
  external site ends the walk.
- **Destructive actions need permission.** Create and update freely under
  `qa-`. Before the first delete, bulk action, "reset", "revoke all",
  export, invite, or anything that emails a human, stop and ask — then
  record the answer so you only ask once per action type.
- **Only touch your own data.** Never edit or delete a record that does
  not start with `qa-`, even if the screen offers it.
- **Stay inside the given workflow.** Travel the path you were asked to
  travel. A screen you reach by accident is recorded as "seen, not
  explored" — it is not an invitation to wander the whole product.
- **Clean up at the end of every walk**, even a failed one, and report
  anything you could not delete.
- **Record, don't correct.** If a screen behaves oddly, screenshot it and
  raise it. Never retry quietly until it looks right.

## 6. Testing rules (what "correct" means)

- **Expected results come from approved discovery or from requirements —
  never from a response captured during the test run.** The app's own
  Swagger is generated by the product, so it is something we TEST, not
  something we trust. The same applies to discovery: approved first.
- A 5xx response is **always a bug**. Never write a test that expects 500.
- Never weaken an assertion to make a test pass. Report the bug instead.
- A test that passes on retry is FLAKY — report it, but still investigate
  (flaky tests often hide real timing bugs).
- A healer may fix a broken **test**; it may never fix a failing
  **expectation**. See the `failure-healing` skill.

## 7. ID conventions (traceability chain)

| Prefix | Meaning | Created by |
|---|---|---|
| `SEQ-001` | Sequence — one user workflow through the UI | app-explorer |
| `EVT-001` | Event — a click/onload/traverse, its API calls and DB effect | event-mapper |
| `VAL-001` | Validation — one rule on one field | validation-mapper |
| `F-001` | Feature (only when requirement documents exist) | requirement-analysis |
| `BR-001` | Business rule (only when requirement documents exist) | requirement-analysis |
| `Q-001` | Open question for product/customer | any agent |
| `TC-001` | Test case | test-case-designer |
| `DAT-001` | Test data record | data-factory |
| `BUG-001` | Defect | defect-report |
| `FIX-001` | A healer's proposed change to a test script | test-healer |

The chain runs `SEQ-/EVT-/VAL- → TC- → test script → run result → BUG-/FIX-`.
A test case cites the definitions it came from; a bug cites its test case.

IDs are permanent: never renumbered, never reused. A changed item keeps
its ID and gets `(changed CR-003)`; a removed one is marked
`DEPRECATED (CR-003)`, never deleted. `CR-001` = a change request from
product, handled by `/change:impact-analysis`.

Every automated test name includes its TC ID, e.g.
`[TC-014] create endpoint with valid name succeeds`.
One TC can have several automated tests (UI + API + DB); every automated
test must have at least one TC ID.

## 8. Working on a subset (any step, any time)

Commands accept a selection instead of "everything". Discovery steps take
the sequence, screen, and module rows; later steps take all of them:

| Select by | Example argument | Means |
|---|---|---|
| Sequence | `SEQ-002` or `seq:create-endpoint` | One discovered workflow, end to end |
| Screen | `screen:key-list` | Everything recorded on that screen |
| Module | `module:keys` or `"API keys"` | Every feature in that module (module index in `docs/discovery/README.md`) |
| Workflow / flow step | `flow:3` (1–6, see section 1) | Everything covering that step of the E2E flow |
| Definition IDs | `EVT-012` `VAL-007` | Exactly these definitions and what derives from them |
| E2E scenario | `E2E-03` | The test case(s) built from that scenario |
| Test case IDs | `TC-014 TC-020` | Exactly these |
| Range | `TC-010..TC-015` | TC-010 to TC-015 inclusive |
| Feature / rule | `F-003` or `BR-007` | Every test case whose Req IDs include it |
| Priority | `P0` | Every test case with that priority |
| Type / layer | `E2E`, `Security`, `API`, `UI`, `DB`, `Docs` | Every test case of that type or layer |
| Folder | `docs/test-cases/keys/` | Every file in that folder |

Selections combine with AND: `F-003 P0` = P0 cases of feature F-003.
Agents first print the resolved list of IDs, then work ONLY on those
and leave everything else untouched. Items with `status: deprecated` are skipped.

## 9. Draft, revise, approve

Every document a step produces (discovery definitions, analysis, test
plan, test data README, impact report) starts with a status line:

`> Status: DRAFT` → after the lead's review → `> Status: APPROVED — <name>, YYYY-MM-DD`

**Files with front matter** — sequences, event files, validation files,
and test cases — carry the status in their `status:` field instead:
`draft` → `approved` → `deprecated`. The lead usually approves a whole
module or a whole sequence at once.

The lead approves by editing the line/field, or by telling Claude, e.g.
"approve docs/discovery/sequences/SEQ-002-generate-key.md" or "approve
docs/test-cases/keys/" (Claude sets every file in that folder to
`approved` and writes "Approved by: <name>, <date>" in the folder
`README.md` — never on its own initiative).

**Revising a DRAFT.** If a command runs again and its output file is
still DRAFT — usually with feedback in the arguments, e.g.
`/test-design:test-case-designer module:keys feedback: merge TC-013 and TC-015`
— revise the file IN PLACE: apply the feedback, keep the IDs of items
that stay, delete items the feedback removes, never append duplicates.
End with a short "Changed in this revision" list.

**APPROVED is locked.** Never rewrite an APPROVED file by re-running its
step. Two exceptions, each narrow:

- the data factory fills the `data:` field and the DAT column of approved
  test case files — nothing else;
- the db-validator fills the `db:` block of approved event files when DB
  access arrives — nothing else.

Other changes go through `/change:impact-analysis`. Only an explicit
"rework the approved …" from the lead resets a file to DRAFT.

**Inputs must be approved.** Before using another step's output, check
its status. DRAFT input → stop and say which file needs approval first,
unless the arguments say `allow-draft`.

## 10. Folder contract (who writes what)

| Folder / file | Written by |
|---|---|
| `docs/discovery/sequences/SEQ-*.md` | app-explorer |
| `docs/discovery/events/*.events.md` | event-mapper (db-validator fills the `db:` block later) |
| `docs/discovery/validations/*.validations.md` | validation-mapper |
| `docs/discovery/screens/` | app-explorer, validation-mapper (screenshots) |
| `docs/discovery/README.md` | whichever discovery agent runs first; the others update it |
| `docs/requirements/` | Humans — PRD, FRD, mockups, client notes (optional) |
| `docs/analysis/requirement-analysis.md` | requirement-analysis (optional step), change-impact (updates) |
| `docs/requirements/changes/CR-*.md` | Humans — updates from product |
| `docs/changes/CR-*-impact.md` | change-impact |
| `docs/test-plan.md` | test-planner |
| `docs/test-cases/` (one file per case, a folder per module) | test-case-designer |
| `test-data/` | data-factory |
| `automation/` (shared: config, types, fixtures, utils) | whichever generator runs first; the others reuse it |
| `automation/discovery/` | app-explorer (the walk harness), reused by the other discovery agents |
| `automation/ui/`, `automation/tests/ui/`, `automation/tests/e2e/` | playwright-generator / playwright-reviewer |
| `automation/api/`, `automation/tests/api/` | api-test-generator / api-test-reviewer |
| `docs/requirements/db/` | Humans — DB schema export (no data) |
| `automation/db/`, `automation/tests/db/`, `docs/analysis/schema-check.md` | db-validator |
| `docs/reports/execution-*.md`, `docs/reports/quality-report-*.md` | test-executor, quality-reporter |
| `docs/reports/investigation-*.md` | test-healer |
| `docs/bugs/` | defect-report, test-healer |

## 11. Pipeline (a human approves each step before the next)

```
0. /setup:env-readiness             is the app reachable and are the logins good?

   ── Discovery: learn the product from the running app ──
1. /discovery:explore-app           URL or workflow → sequences + screenshots + first-pass events
2. /discovery:event-definition      events → API calls → DB tables (DB part PENDING until access)
3. /discovery:validation-definition probe every field → validation rules and messages
   (optional) /requirements:analyze-requirements   if PRD/FRD/mockups exist, to compare against

   ── Design ──
4. /test-design:test-planner        scope, coverage, risk → docs/test-plan.md
5. /test-design:test-case-designer  manual test cases from sequences + validations
6. /test-data:data-factory          dummy datasets from the validation rules

   ── Automation ──
7. /automation-api:api-test-generator  → api-test-reviewer
   /automation-ui:playwright-generator → playwright-reviewer
8. /database:db-validator           (only after the DB type is known)

   ── Run and react ──
9.  /execution:test-executor        run suites → report with log, video, screenshots
10. /healing:investigate-failures   every failure → broken test (propose a fix) or bug
11. /reporting:quality-reporter     release verdict
```

**When the application changes** (a screen is redesigned, a field is
added), re-run discovery for that part only:

```
/discovery:explore-app seq:create-endpoint   → approve → /change:impact-analysis
```

**When product sends an update** (at any point after discovery is
approved), don't restart the pipeline:

```
/change:impact-analysis <CR file>   → approve → re-run only the
steps it lists, with only the affected IDs
```

## 12. Writing style

Write so a junior QA can follow without asking: short sentences, tables
and checklists, explain jargon the first time. Unknowns are marked
`ASSUMPTION — verify` or raised as a `Q-` question. Never guess silently.
