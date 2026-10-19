# QA Automation Framework — API Endpoint Builder

AI-assisted QA for our product: a customer creates an API endpoint in the
UI → Swagger docs are generated → they get an API key → they call it.

**The application already exists.** You give the framework a URL and a
test login; it travels the app, records what it finds, and builds the
tests from that.

You run **commands** in Claude Code. Each command hands the work to an
**agent** (a specialist AI worker) that follows **skills** (our written
standards). You review the output before running the next step.

## How it works

```
   App URL + test login
           │
   ┌───────▼────────┐   travel the app, record what it does
   │   DISCOVERY    │   sequences · events · validations
   └───────┬────────┘   ← you approve: observation becomes expectation
           │
   ┌───────▼────────┐   scope, coverage, test cases, data
   │     DESIGN     │
   └───────┬────────┘
           │
   ┌───────▼────────┐   UI, API, E2E, DB tests in TypeScript
   │   AUTOMATION   │
   └───────┬────────┘
           │
   ┌───────▼────────┐   run → investigate every failure → report
   │  RUN & REACT   │   broken test → a fix you approve
   └────────────────┘   broken product → a bug report
```

The one rule that shapes everything: **what the app does is an
observation, not the truth.** Discovery records; you confirm. Anything
that looks wrong is raised as a finding instead of being written down as
correct behaviour.

## One-time setup

1. Install [Claude Code](https://code.claude.com) and Node.js 18+.
2. Open a terminal **in this folder** and run `claude`:
   ```powershell
   cd D:\project\e2e-automation-framework
   claude
   ```
   ⚠️ Starting `claude` from any other folder (e.g. `D:\project`) gives
   "Unknown command: /..." — the commands only load from this folder.
3. Copy `.env.example` to `.env` and fill in the **test environment**
   URL and the two test logins (ask your lead).
4. Optional: put any requirement documents (PRD, FRD, mockups, notes) in
   `docs/requirements/`. The framework works without them — see that
   folder's README for what you lose.
5. Check the setup works: `python scripts/verify-setup.py`

## Run the steps in order

Type each command inside Claude Code. **Read the output and get approval
before moving to the next step.**

| Step | Command | What it does | You get |
|---|---|---|---|
| 0 | `/setup:env-readiness` | Checks the app is reachable and the logins work | READY / NOT READY table |
| 1 | `/discovery:explore-app` | Travels a workflow, screenshots every screen | `docs/discovery/sequences/SEQ-*.md` |
| 2 | `/discovery:event-definition` | Each click/load → its API calls → its DB tables | `docs/discovery/events/*.events.md` |
| 3 | `/discovery:validation-definition` | Probes every field for its rules and messages | `docs/discovery/validations/*.validations.md` |
| 1b | `/requirements:analyze-requirements` | **Optional** — reads requirement documents, if any exist, to compare against | `docs/analysis/requirement-analysis.md` |
| 4 | `/test-design:test-planner` | Decides scope, depth and priority; lists conflicts | `docs/test-plan.md` |
| 5 | `/test-design:test-case-designer` | Writes the manual test cases | `docs/test-cases/<module>/TC-*.md` |
| 6 | `/test-data:data-factory` | Creates dummy test data from the validation rules | `test-data/*.json` |
| 7a | `/automation-api:api-test-generator` | Writes API tests (TypeScript) | `automation/api/`, `automation/tests/api/` |
| 7b | `/automation-api:api-test-reviewer` | Reviews, fixes, and stabilizes API tests | Review report |
| 7c | `/automation-ui:playwright-generator` | Writes UI + E2E tests (TypeScript, POM) | `automation/ui/`, `automation/tests/ui/`, `automation/tests/e2e/` |
| 7d | `/automation-ui:playwright-reviewer` | Reviews, fixes, and stabilizes UI tests | Review report |
| 8 | `/database:db-validator` | DB checks — **only when `DB_ACCESS=readonly`**; with `none`, `@persistence` probes stand in | `automation/db/`, schema check, filled event DB mappings |
| 9 | `/execution:test-executor` | Runs everything, collects evidence | `docs/reports/execution-*.md` |
| 10 | `/healing:investigate-failures` | Diagnoses every failure: broken test or broken product | `docs/reports/investigation-*.md`, `docs/bugs/` |
| 11 | `/reporting:quality-reporter` | Release report for managers | `docs/reports/quality-report-*.md` |
| any | `/change:impact-analysis <CR file or "answers">` | Product sent an update → finds what it affects, updates only that | `docs/changes/CR-*-impact.md` + targeted edits |
| any | `/reporting:report-bug <what happened>` | Writes a bug report in our standard format | `docs/bugs/BUG-*.md` |

Steps 1–3 are one trip through the app recorded three ways: the path, what
each action did, and what each field accepts. Run 1 first; 2 and 3 mine
the same recording.

**Full detail on every command — input, output, arguments, gotchas:**
[docs/commands.md](docs/commands.md)

Tip: every command takes a selection, so you can work on one part:
`/discovery:explore-app seq:create-endpoint`,
`/test-design:test-case-designer module:keys`,
`/healing:investigate-failures TC-014`.

**New here? Read the worked example first:**
[docs/examples/module-keys-walkthrough.md](docs/examples/module-keys-walkthrough.md)
— one module (API keys) from an empty folder to a filed bug, with sample
output at every step.

## Running the tests yourself (after step 7)

```bash
cd automation
npm ci
npm run discover -- walks/create-endpoint.walk.ts   # re-walk the app
npm test                          # everything
npm run test:api                  # API only
npm run test:e2e                  # E2E only
npx playwright test --grep TC-014 # one test case
npm run report                    # open the HTML report
```

## Rules you must never break

- **Test environment only.**
- **Dummy data only** — never real customer data.
- Tests and discovery walks create their own `qa-` data and delete it.
- Discovery may fill in forms and create records; it asks before the
  first delete or anything that emails a person.
- Never change a test's expected result just to make it pass — log a bug.
- A healer may fix a broken test; it may never fix a failing expectation.

Full rules: [CLAUDE.md](CLAUDE.md). Coding standard:
[.claude/skills/ts-pom-standards/SKILL.md](.claude/skills/ts-pom-standards/SKILL.md).

## Where things live

```
CLAUDE.md               shared rules every agent follows
.claude/commands/       the slash commands you type
.claude/agents/         the specialist agents the commands start
.claude/skills/         our standards (discovery, test design, security, POM, healing...)
scripts/verify-setup.py checks commands → agents → skills are wired correctly
docs/discovery/         what we learned by travelling the app
docs/requirements/      requirement documents, if any exist (optional)
docs/                   analysis, test cases, reports, bugs
automation/             the TypeScript project — discovery harness and tests
```
