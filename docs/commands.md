# Command Reference

Every command, what it needs, and what it gives you.

You type these inside Claude Code, in the framework folder. Each one
stops when it is done — **you review and approve before running the
next**. Nothing runs automatically after something else.

- **A** = runs as a forked agent, in its own context
- **S** = runs as a skill, in your own conversation, so you can discuss
  the result immediately

---

## Quick reference

| Step | Command | Gives you |
|---|---|---|
| 0 **S** | [`/setup:env-readiness`](#step-0--setupenv-readiness) | READY / NOT READY table |
| 1 **A** | [`/discovery:explore-app`](#step-1--discoveryexplore-app) | Sequences + screenshots |
| 2 **A** | [`/discovery:event-definition`](#step-2--discoveryevent-definition) | Events → API calls → DB tables |
| 3 **A** | [`/discovery:validation-definition`](#step-3--discoveryvalidation-definition) | Field rules + who enforces them |
| 1b **S** | [`/requirements:analyze-requirements`](#step-1b--requirementsanalyze-requirements) | Features and rules from documents *(optional)* |
| 4 **A** | [`/test-design:test-planner`](#step-4--test-designtest-planner) | The test plan |
| 5 **A** | [`/test-design:test-case-designer`](#step-5--test-designtest-case-designer) | Manual test cases |
| 6 **A** | [`/test-data:data-factory`](#step-6--test-datadata-factory) | Dummy datasets |
| 7a **A** | [`/automation-api:api-test-generator`](#step-7a--automation-apiapi-test-generator) | API tests |
| 7b **A** | [`/automation-api:api-test-reviewer`](#step-7b--automation-apiapi-test-reviewer) | Stable API tests |
| 7c **A** | [`/automation-ui:playwright-generator`](#step-7c--automation-uiplaywright-generator) | UI + E2E tests |
| 7d **A** | [`/automation-ui:playwright-reviewer`](#step-7d--automation-uiplaywright-reviewer) | Stable UI tests |
| 8 **A** | [`/database:db-validator`](#step-8--databasedb-validator) | DB checks + filled table mappings |
| 9 **A** | [`/execution:test-executor`](#step-9--executiontest-executor) | Execution report + evidence |
| 10 **A** | [`/healing:investigate-failures`](#step-10--healinginvestigate-failures) | A verdict per failure |
| 11 **A** | [`/reporting:quality-reporter`](#step-11--reportingquality-reporter) | Release verdict |
| any **S** | [`/change:impact-analysis`](#anytime--changeimpact-analysis) | What a change affects |
| any **S** | [`/reporting:report-bug`](#anytime--reportingreport-bug) | A bug report |

---

## Arguments every command understands

### Selecting part of the work

Instead of "everything", point a command at one part. It prints the
resolved list of IDs first, then touches **only** those.

| Select by | Example |
|---|---|
| Sequence | `SEQ-002` · `seq:create-endpoint` |
| Screen | `screen:key-list` |
| Module | `module:keys` · `"API keys"` |
| Flow step (1–6) | `flow:3` |
| Definition IDs | `EVT-012` · `VAL-007` |
| E2E scenario | `E2E-03` |
| Test cases | `TC-014 TC-020` |
| Range | `TC-010..TC-015` |
| Priority | `P0` |
| Type / layer | `E2E` · `Security` · `API` · `UI` · `DB` · `Docs` |
| Folder | `docs/test-cases/keys/` |

They combine with AND: `F-003 P0` means the P0 cases of feature F-003.

### Revising a draft

If the output file is still `DRAFT`, run the command again with feedback
and it revises **in place** — same IDs, no duplicates:

```
/test-design:test-case-designer module:keys feedback: merge TC-013 and TC-015
```

It ends with a short "Changed in this revision" list.

### Working before approval

A command stops if its input is still a draft. Override deliberately:

```
/test-design:test-planner allow-draft
```

---

## Step 0 — `/setup:env-readiness`

> Is the environment ready? A two-minute check that saves hours.

```
/setup:env-readiness
/setup:env-readiness ui
```

| | |
|---|---|
| **Runs as** | **S** — skill `env-readiness-check` |
| **Input** | `.env` only |
| **Output** | A table in chat. **No files written** |

Checks settings are filled, the UI and API respond, Swagger is
reachable, both test tenants can log in, tools are installed, and no
leftover `qa-` data is sitting around.

```
VERDICT: NOT READY — fix #5 first
```

**Two things:**

- `⚠️ SKIPPED` is not a failure. Before step 1, `automation/` does not
  exist, and the optional settings are empty on purpose.
- Check 5 (tenants can log in) saves the most time. Discovery clicks
  through the real UI, so an expired password looks like "the app is
  broken" for an hour.

---

## Step 1 — `/discovery:explore-app`

> Travel a workflow and write down the path.

```
/discovery:explore-app "generate an API key for an endpoint and copy it"
/discovery:explore-app /endpoints
/discovery:explore-app SEQ-002
```

| | |
|---|---|
| **Runs as** | **A** — agent `app-explorer` |
| **Input** | `.env` + a URL or a workflow in words |
| **Output** | `docs/discovery/sequences/SEQ-*.md`<br>`docs/discovery/screens/*.png`<br>`automation/discovery/walks/*.walk.ts`<br>`automation/discovery/artifacts/` *(gitignored)* |

**Refuses to run if** `TEST_ENV_BASE_URL` is empty, or the URL you gave
is not on the test environment host.

It travels one screen at a time — reads the screen, picks the control
that advances the workflow, acts, screenshots, compares. It records
three streams at once (screens, network, console) because traffic only
exists while you are clicking.

On the first run it also creates the `automation/` project and the
discovery harness inside it.

**Two things:**

- **It writes data.** You cannot learn an app read-only. Everything is
  `qa-` named and deleted after, and it asks before the first delete,
  bulk action, or anything that emails a person.
- **Dead ends are the point.** A workflow that cannot be completed is the
  most valuable thing a first walk can find.

---

## Step 2 — `/discovery:event-definition`

> What each click actually does behind the screen.

```
/discovery:event-definition SEQ-002
/discovery:event-definition module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `event-mapper` |
| **Input** | Approved `SEQ-*.md` + the artifacts from step 1 |
| **Output** | `docs/discovery/events/<module>.events.md`<br>Event column filled back into the sequences |

**It does not walk the app again** — it mines step 1's recording.

One user action = one `EVT-`, however many calls it fired. Five types:
`onload`, `click`, `submit`, `traverse`, `background`. The last one
(polling) is the usual hidden cause of flaky tests.

```markdown
### API calls (Tier 1 — observed)
| # | Method | Route                    | Status | ms   |
| 1 | POST   | /api/endpoints/{id}/keys | 201    | 3900 |
| 2 | GET    | /api/endpoints/{id}/keys | 200    | 180  |
Order: POST completes before the list GET fires.
```

**Two things:**

- **Routes are generalised** — `/api/endpoints/{id}/keys`, never the
  literal UUID. These become the `waitForResponse` targets in every
  generated test, so a vague route here is a flaky test later.
- **Tier 2 (DB tables) stays `PENDING`** until step 8. It never guesses
  a table name.

---

## Step 3 — `/discovery:validation-definition`

> Every rule on every field, and who actually enforces it.

```
/discovery:validation-definition screen:generate-key-dialog
/discovery:validation-definition module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `validation-mapper` |
| **Input** | Approved `SEQ-*.md` + field dumps + the events (for API routes) |
| **Output** | `docs/discovery/validations/<module>.validations.md` + a screenshot of each rule firing |

Four passes:

1. **Read the DOM** — `required`, `maxlength`, `type`, `pattern`. Free
2. **Probe each field** — empty, one char, too long, wrong format,
   spaces, duplicate, unicode, one injection string
3. **Check enforcement** — send the same bad value straight to the API
4. **Count and quota limits** — create until rejected, capped by
   `DISCOVERY_QUOTA_PROBE_CAP` (default 25)

| ID | Field | Rule | Message (exact) | Enforced |
|---|---|---|---|---|
| VAL-007 | Key name | required | `Key name is required.` | both |
| VAL-008 | Key name | max length 40 | — (typing stops at 40) | **client** |
| VAL-012 | keys per endpoint | max 5 | `You have reached the limit of 5 keys…` | both |

**The `Enforced` column is the most valuable output in the pipeline.**
`client` or `none` means anyone calling the API bypasses the rule — a
real defect, and it moves the test case to the layer where it catches
something.

**Two things:**

- **It writes data** — the uniqueness and quota probes have to. All
  `qa-` named, all deleted, all listed by name in the file.
- **A 5xx from a probe is a bug report, not a rule.** It stops probing
  that field after the second one.

---

## Step 1b — `/requirements:analyze-requirements`

> Optional. What the product was *supposed* to do.

```
/requirements:analyze-requirements
/requirements:analyze-requirements module:keys
```

| | |
|---|---|
| **Runs as** | **S** — skill `requirement-analysis` |
| **Input** | `docs/requirements/` — PRD, FRD, mockups, notes |
| **Output** | `docs/analysis/requirement-analysis.md` (features `F-`, rules `BR-`, questions `Q-`) |

Runs **alongside** discovery, not instead of it. Discovery records what
the app does; this records what it was meant to do. The test planner
compares them, and every disagreement becomes a question or a suspected
bug — which is where most real defects are found.

If `docs/requirements/` is empty, it says so and stops. That is a valid
state: discovery plus your confirmation is then the only oracle, and the
test plan records that as a risk.

---

## Step 4 — `/test-design:test-planner`

> Decide what gets tested, how deep, and what won't be.

```
/test-design:test-planner module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `test-planner` |
| **Input** | Approved sequences + events + validations (+ the analysis if it exists) |
| **Output** | `docs/test-plan.md` — **one page, the plan only** |

Its main job is the comparison. Where discovery and the documents
disagree it lists the conflict and raises a `Q-`; it never picks a side.

It decides depth from evidence, not feel:

| Evidence | Consequence for the plan |
|---|---|
| A `VAL-` marked `client` | Needs an API-layer negative case |
| A `background` (polling) event | Flakiness risk, called out |
| An event with no API call | Cannot be tested at the API layer |
| DB mapping still `PENDING` | That cell is `Later` |

Output is a scope matrix, every cell In / Out / Later **with a reason**:

| Flow step | UI | API | Docs | DB | Pri | Why |
|---|---|---|---|---|---|---|
| 3. Generate API key | In | In | — | Later | P0 | Show-once is unrecoverable if wrong |

---

## Step 5 — `/test-design:test-case-designer`

> The manual test cases, one file each.

```
/test-design:test-case-designer module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `test-case-designer` |
| **Input** | Approved test plan + **sequences + validations** |
| **Output** | `docs/test-cases/<module>/TC-*.md` + module and overall indexes |

**Events are deliberately not an input.** A manual case describes what a
person does and sees; routes and tables belong to the automation step.

| Source | Becomes |
|---|---|
| A `SEQ-` path | The positive case, in the product's real labels |
| A `VAL-` rule | A negative/boundary case; rules on one field merge into one data-driven case |
| A `VAL-` marked `client`/`none` | A case at the **API layer** |
| A quota `VAL-` | Build to the limit via API, assert the next is rejected |
| A sequence finding | A case that proves the broken behaviour, so a fix is detectable |
| `E2E-01..E2E-07` | One case each |

```yaml
id: TC-013
requirements: [SEQ-002, VAL-008]   # what this case rests on
priority: P2
layer: [API]
status: draft
```

**It cannot improve on an expected result** — no rounding a limit, no
tidying a message. If it cannot state one exactly, that is a `Q-`.

---

## Step 6 — `/test-data:data-factory`

> Dummy data, built from the validation rules.

```
/test-data:data-factory module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `data-factory` |
| **Input** | Approved test cases + validations |
| **Output** | `test-data/valid-data.json`, `negative-data.json`, `security-data.json`, `README.md`<br>Fills `data:` and the DAT column in the test case files |

```json
{ "id": "DAT-024", "field": "keyName", "value": "{repeat:k:41}",
  "expected": "rejected", "rule": "VAL-008", "used_by": ["TC-013"] }
```

Boundaries come straight from the rule: `min 3, max 40` produces records
for 2, 3, 40 and 41.

**Three things:**

- **Patterns are filled at runtime** — `{qa-name}` becomes a unique name
  per worker, `{repeat:k:41}` becomes 41 characters, `{faker:…}` is
  seeded by TC number so it is realistic but identical every run.
- **It never shrinks data to fit.** If the rule says 40 and the column is
  `VARCHAR(30)`, that is a finding, not a reason to use 30.
- **It may only touch `data:` and the DAT column** of an approved test
  case. Everything else in that file is frozen.

---

## Step 7a — `/automation-api:api-test-generator`

> API tests, from the test cases and the recorded routes.

```
/automation-api:api-test-generator module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `api-test-generator` |
| **Input** | Approved API/Docs test cases + `test-data/` + **events** + Swagger URL |
| **Output** | `automation/api/` (clients, schemas, validators), `automation/tests/api/` |

Two separate clients, because they are different APIs: `PlatformClient`
(fixed — create/list/delete endpoints and keys) and
`GeneratedEndpointClient` (created at runtime by each test).

**Where each thing comes from:**

| Thing | Source |
|---|---|
| Expected values | **The test case.** Never the Swagger, never a captured response |
| Routes, payload shapes, call order | The event definitions |
| Input values | `test-data/` |

A three-way disagreement between test case, recorded event and Swagger
is a finding plus a `Q-` — never reconciled by picking one.

It builds the smoke test first and makes it pass before generating the
rest.

---

## Step 7b — `/automation-api:api-test-reviewer`

> Make the API tests trustworthy.

```
/automation-api:api-test-reviewer module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `api-test-reviewer` |
| **Input** | `automation/api/`, `automation/tests/api/` + test cases + events |
| **Output** | Fixes applied in place · review report in chat · `BUG-` if the app is at fault |

Checklist includes: no `expect()` in clients, no raw `request.*` in
tests, typecheck and lint pass, every test title starts with a real TC
ID, schema validation on every response, auth failures return 401, no
test expects a 5xx, routes match the event definitions, every
`client`/`none` validation has an API-layer test.

Then runs them, classifies each failure, fixes script bugs, and applies
the **stability gate**: `--repeat-each=3` fully green.

**Never weakens an assertion to force green.** An application bug gets a
report instead.

---

## Step 7c — `/automation-ui:playwright-generator`

> UI and E2E tests, Page Object Model.

```
/automation-ui:playwright-generator module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `playwright-generator` |
| **Input** | Approved UI/E2E test cases + `test-data/` + **sequences, walk files, artifacts, events, validations** |
| **Output** | `automation/ui/` (locators, pages), `automation/tests/ui/`, `automation/tests/e2e/` |

This is the step with the most inputs, and each earns its place:

| From discovery | Gives it |
|---|---|
| Sequences | The real screens, labels, step order |
| Walk files | A path already proven to work — the skeleton of the test |
| Artifacts (DOM dump) | Verified selectors, not guesses |
| Events | The routes to wait on, and the call order |
| Validations | The exact error messages to assert |

Setup and cleanup go through the **API**, not the UI — faster and less
flaky. Only the step under test uses the UI.

---

## Step 7d — `/automation-ui:playwright-reviewer`

> Make the UI tests trustworthy.

```
/automation-ui:playwright-reviewer module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `playwright-reviewer` |
| **Input** | The UI/E2E tests + test cases + definitions |
| **Output** | Fixes in place · review report · `BUG-` if the app is at fault |

Checks POM layering (selectors only in `locators/`, no `expect()` in
pages, no raw `page.*` in tests), selectors match what discovery
recorded, asserted messages match the `VAL-` rule character for
character, waits are on the recorded routes, no `waitForTimeout`,
cleanup runs on failure.

Same stability gate: three green runs in a row.

---

## Step 8 — `/database:db-validator`

> Confirm the app stored it correctly, and complete the event map.

```
/database:db-validator
/database:db-validator module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `db-validator` |
| **Input** | Read-only DB settings in `.env` + approved events with `PENDING` + validations + test cases |
| **Output** | `docs/analysis/schema-check.md`<br>`automation/db/`, `automation/tests/db/`<br>**Fills the `db:` blocks in the event files** |

### What you need — and it is one thing

A **read-only database user** on the test environment, in `.env`:

```
DB_TYPE=postgres            # postgres | mysql | mssql
DB_URL=<host:port/dbname>
DB_READONLY_USER=<SELECT rights only>
DB_READONLY_PASSWORD=
DB_SCHEMA=                  # empty = driver default (public / dbo)
```

That is the whole ask. **No schema export, no config flag, nothing from
the backend team.** The schema is read from `information_schema`.

### What the access is used for — three jobs

| # | Job | Why it needs the DB |
|---|---|---|
| 1 | **Check the app stored things correctly** | Is the API key **hashed**, not plain text? Does the row belong to `qa-tenant-a`? Are there orphan key rows after a delete? |
| 2 | **Fill the `PENDING` table mappings** | Discovery sees that clicking "Generate" fires `POST /api/endpoints/{id}/keys`, but not which table that wrote to |
| 3 | **Compare the schema with the validation rules** | The UI stops at 40 characters, the API accepted 256, the column is `VARCHAR(30)` — three different limits in one system |

Job 1 is why read-only access is needed **regardless** of how you get the
mapping. A query log can tell you which table was written; only reading
the row can tell you the key was stored hashed.

### How the mapping works — the tracer method

Every record the framework creates carries a unique name,
`qa-seq002-lz3k9a-x7f1`. That string exists nowhere else in the
database, so it works as a tracer dye. Rather than diffing the whole
schema, it goes looking for its own value.

**1. Read the catalog once** — this is where the schema comes from:

```sql
SELECT table_name, column_name, data_type, character_maximum_length, is_nullable
FROM information_schema.columns
WHERE table_schema = :schema;
```

Plus `table_constraints` and `key_column_usage` for primary and foreign
keys.

**2. Baseline** — for an event that changes or removes something, re-read
the known row by primary key first. A create event has nothing to
baseline.

**3. Act** — trigger the event through the platform API with the tracer
record. Never write to the database to set up a check.

**4. Locate** — query the text columns for the literal value, name-like
columns first (`name`, `title`, `slug`, `label`, `key`):

```sql
SELECT 'api_keys' AS tbl, 'name' AS col, id FROM api_keys WHERE name = :tracer
UNION ALL
SELECT 'endpoints', 'name', id FROM endpoints WHERE name = :tracer;
```

Whatever returns is **the table and column the action wrote to**.

**5. Walk the foreign keys** — child rows do not hold the tracer
(`api_keys` holds an `endpoint_id`, not the name), so follow the FK map
from step 1 outward. Two levels is almost always enough.

**6. Diff by primary key** — re-read the located rows before and after
and compare field by field. **This is what answers "where is it
updated":**

| Observed | Verdict |
|---|---|
| Row exists where it did not | INSERT — list the columns it filled |
| Same key, some columns differ | UPDATE — each column `before → after` |
| Row gone | DELETE (hard) |
| Row present, `deleted_at`/`status` now set | Soft delete — recorded as an UPDATE, **and raised**, because hard vs soft delete is usually an open question |

### What lands in the event file

```markdown
### DB effect (Tier 2)
| Table | Operation | Change | Confidence | How we know |
|---|---|---|---|---|
| `api_keys` | INSERT | filled `id, endpoint_id, name, key_hash, created_at` | confirmed | tracer found in `api_keys.name` |
| `endpoints` | UPDATE | `key_count` 0 → 1 · `updated_at` bumped | confirmed | row diff by PK (FK walk) |

Not reachable by tracer: `audit_log` (+1 row, no foreign key to our row)
```

| Confidence | When |
|---|---|
| `confirmed` | Row located by tracer or primary key, diff deterministic |
| `likely` | A table count moved but the row could not be isolated |
| `PENDING` | No DB access yet |

`How we know` matters as much as the data. In six months, "how do we
know this?" is the question someone will ask.

### Why this is safe to run on a shared environment

| Concern | Why it is fine |
|---|---|
| Disturbing other testers | Nothing is locked — readers do not block writers, and every query runs in a read-only transaction |
| Load on the database | `information_schema` is metadata only; row lookups are by key |
| Reading customer data | Only your own tracer and the ids it leads to are ever queried |
| Other traffic skewing results | Impossible — nothing is counted globally, so someone else's click cannot affect your row |

**No quiet maintenance window is needed.** This is the main advantage
over counting rows before and after.

### What it cannot see

Honest limits, so nobody over-trusts the output:

- The SQL statements themselves, or their order inside a transaction
- Writes with no link back to your row — counters, cache tables, audit
  rows without a foreign key. These are listed as **"not reachable by
  tracer"**, never as "no DB effect"
- Asynchronous writes that land after the check
- Whether a column was set by this action or by a database default

If you later want statement-level recording, the cheapest route is not
the database: ask backend to switch on SQL query logging on the test
deployment (one environment variable). The event file then records
`source: app SQL log` instead of `source: tracer`.

### Read-only is enforced three times

1. The database user has `SELECT` rights only
2. `DbClient` opens every query in a read-only transaction
3. `DbClient` rejects any statement that does not start with `SELECT`

If the user turns out to be able to write, the agent **stops** and
reports it as a Sev-1 environment risk.

### The two kinds of check it adds

| Kind | Where | Example |
|---|---|---|
| **Inside existing API/E2E tests** | The existing spec gains a DB step | After `[TC-001]` creates an endpoint via API → row exists, owned by `qa-tenant-a` |
| **Standalone integrity checks** | `tests/db/<area>.db.spec.ts` | No key rows without an endpoint; no plain-text keys; no leftover `qa-` rows older than 24 h |

Both carry the `@db` tag, so `--grep @db` runs everything DB-related,
while `npm run test:db` runs only the standalone project.

### Until you have DB access

Nothing is blocked:

- event definitions keep `db: PENDING` and carry a `Q-`
- a DB step inside an API/E2E test skips **only that step** — the API and
  UI parts still run
- standalone `tests/db/` specs skip whole
- the test plan marks the DB column `Later`, with the reason

That is the intended state, not a failure.

---

## Step 9 — `/execution:test-executor`

> Run the suites and collect the evidence.

```
/execution:test-executor
/execution:test-executor module:keys
/execution:test-executor smoke
```

| | |
|---|---|
| **Runs as** | **A** — agent `test-executor` |
| **Input** | `automation/` + test cases + `.env` |
| **Output** | `docs/reports/execution-<date>.md`<br>trace, video and screenshots in `automation/reports/artifacts/` |

Runs readiness first, then Smoke → API → UI → E2E → DB. A suite name
(`api`, `ui`, `e2e`, `smoke`, `db`) runs just that one; any other
selection resolves to TC IDs and runs those plus the smoke set as a
regression guard.

**It gives each failure a first-pass label and hands off. It does not
file bugs** — one agent should not both run the tests and judge them.

It also does the **parity check**: test cases with no automated test,
automated tests with no valid TC, and approved sequences that reach no
test at all. Any mismatch blocks a "regression green" claim.

---

## Step 10 — `/healing:investigate-failures`

> Every failure leaves with a verdict and an owner.

```
/healing:investigate-failures
/healing:investigate-failures TC-014
/healing:investigate-failures verify
/healing:investigate-failures apply FIX-001
```

| | |
|---|---|
| **Runs as** | **A** — agent `test-healer` |
| **Input** | Latest execution report + its artifacts + test cases + definitions |
| **Output** | `docs/reports/investigation-<date>.md`<br>`docs/bugs/BUG-*.md`<br>`FIX-` proposals — **not applied** |

Reproduces each failure twice, then decides from the trace, video and
network — never from the error message alone:

| Verdict | Outcome |
|---|---|
| Broken test | A `FIX-` with the exact diff, blast radius, and a bug-guard checklist |
| Application issue | A `BUG-` with the evidence attached |
| Stale expectation | Neither — routed to discovery + change-impact |
| Environment / data | Reported, never coded around |

**The one rule:** it may fix a broken **test**; it may never fix a
failing **expectation**. If a change would make an assertion weaker,
broader or conditional, it stops — that is a bug, not a fix.

It cannot add a sleep, a retry, a `try/catch` or a `test.skip` to make a
failure disappear, and it never commits anything.

`verify` applies a diff, runs it, then reverts unless you approve.
`apply FIX-001` applies one you approved.

---

## Step 11 — `/reporting:quality-reporter`

> The release verdict, for people who will not read the detail.

```
/reporting:quality-reporter
/reporting:quality-reporter module:keys
```

| | |
|---|---|
| **Runs as** | **A** — agent `quality-reporter` |
| **Input** | Test cases + discovery + latest execution + latest investigation + bugs |
| **Output** | `docs/reports/quality-report-<date>.md` |

Verdict on the first line — READY / READY WITH RISKS / NOT READY — then
test summary, E2E flow status, coverage, defects, test health, exit
criteria, risks.

**Two things:**

- **Every number comes from the input files.** Missing data is written
  "no data" with the file named — never estimated.
- It states how much of the product has been **discovered at all**.
  "100% of discovered workflows pass" means nothing if only three
  workflows were ever walked, so it prints both numbers.

---

## Anytime — `/change:impact-analysis`

> Find what a change affects, and update only that.

```
/change:impact-analysis docs/requirements/changes/CR-001-key-expiry.md
/change:impact-analysis answers
```

| | |
|---|---|
| **Runs as** | **S** — skill `change-impact` |
| **Input** | A `CR-*.md`, new rows in `answers.md`, a new document version, **or a discovery re-run that found the app changed** |
| **Output** | `docs/changes/CR-*-impact.md` → after approval, targeted edits + the exact follow-up commands |

Two phases. **Phase 1 analyses and stops** — it shows the impact report
and waits. **Phase 2 only runs after you approve.**

It traces down the chain:

```
SEQ- / EVT- / VAL-  →  TC-  →  test data  →  automated tests  →  walk files
```

Nothing is renumbered, nothing is deleted. A changed item keeps its ID
and gains `(changed CR-003)`; a removed one is marked `DEPRECATED`.

---

## Anytime — `/reporting:report-bug`

> A bug report in the standard format.

```
/reporting:report-bug TC-014 revoked key still returns 200
```

| | |
|---|---|
| **Runs as** | **S** — skill `defect-report` |
| **Input** | A TC ID, a failing test, or a description |
| **Output** | `docs/bugs/BUG-<nnn>-<short-name>.md` |

Asks you for anything a developer would need and you have not given it.
Severity is QA's call (Sev-1…Sev-4); priority is proposed and the lead
confirms.

---

## Approval gates — what blocks what

Every step checks its input's status and stops if it is still a draft.

```
discovery (SEQ/EVT/VAL)  →  approve  →  test plan
test plan                →  approve  →  test cases
test cases               →  approve  →  test data, automation
```

| Status lives in | Values |
|---|---|
| Files with front matter — sequences, events, validations, test cases | `status: draft` → `approved` → `deprecated` |
| Documents — test plan, analysis, impact reports, test data README | `> Status: DRAFT` → `> Status: APPROVED — <name>, <date>` |

You approve by editing the line, or by telling Claude:

```
approve docs/discovery/sequences/SEQ-002-generate-api-key.md
approve docs/test-cases/keys/
```

**Approving discovery is the most important click in the pipeline.** It
is the moment an observation stops being "what the app did when we poked
it" and becomes "what the app is supposed to do". After it, test cases
and bug reports are written against those values.

**Approved files are locked.** Re-running a step never rewrites one.
Two narrow exceptions: the data factory fills `data:` and the DAT column
of approved test cases, and the db-validator fills the `db:` block of
approved event files. Everything else goes through
`/change:impact-analysis`.
