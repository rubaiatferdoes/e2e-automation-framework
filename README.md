# QA Automation Framework — API Endpoint Builder

AI-assisted QA for our product: a customer creates an API endpoint in the
UI → Swagger docs are generated → they get an API key → they call it.

You run **commands** in Claude Code. Each command hands the work to an
**agent** (a specialist AI worker) that follows **skills** (our written
standards). You review the output before running the next step.

## One-time setup

1. Install [Claude Code](https://code.claude.com) and Node.js 18+.
2. Open a terminal **in this folder** and run `claude`:
   ```powershell
   cd D:\project\e2e-automation-framework
   claude
   ```
   ⚠️ Starting `claude` from any other folder (e.g. `D:\project`) gives
   "Unknown command: /..." — the commands only load from this folder.
3. Copy `.env.example` to `.env` and fill in the **mirror** values (ask your lead).
4. Put requirement documents (PRD, FRD, mockups, notes) in `docs/input/`.
5. Check the setup works: `python scripts/verify-setup.py`

## Run the steps in order

Type each command inside Claude Code. **Read the output and get approval
before moving to the next step.**

| Step | Command | What it does | You get |
|---|---|---|---|
| 0 | `/setup:env-readiness` | Checks the mirror environment is safe and ready | READY / NOT READY table |
| 1 | `/requirements:analyze-requirements` | Reads the requirements, finds gaps, writes questions | `docs/analysis/requirement-analysis.md` |
| 2 | `/test-design:test-planner` | Writes the test plan and test cases | `docs/test-plan.md`, `docs/test-cases.md` |
| 3 | `/test-data:data-factory` | Creates dummy test data | `test-data/*.json` |
| 4a | `/automation-api:api-test-generator` | Writes API tests (TypeScript) | `automation/api/`, `automation/tests/api/` |
| 4b | `/automation-api:api-test-reviewer` | Reviews, fixes, and stabilizes API tests | Review report |
| 4c | `/automation-ui:playwright-generator` | Writes UI + E2E tests (TypeScript, POM) | `automation/ui/`, `automation/tests/ui/`, `tests/e2e/` |
| 4d | `/automation-ui:playwright-reviewer` | Reviews, fixes, and stabilizes UI tests | Review report |
| 5 | `/database:db-validator` | DB checks — **only after the DB type + schema are in `docs/input/db/`** | `automation/db/`, schema check |
| 6 | `/execution:test-executor` | Runs everything, sorts failures, files bugs | `docs/reports/execution-*.md`, `docs/bugs/` |
| 7 | `/reporting:quality-reporter` | Release report for managers | `docs/reports/quality-report-*.md` |
| any | `/change:impact-analysis <CR file or "answers">` | Product sent an update → finds what it affects, updates only that | `docs/changes/CR-*-impact.md` + targeted edits |
| any | `/reporting:report-bug <what happened>` | Writes a bug report in our standard format | `docs/bugs/BUG-*.md` |

Tip: you can point a command at a different file, e.g.
`/test-design:test-planner docs/analysis/v2.md`.

**New here? Read the worked example first:**
[docs/examples/module-keys-walkthrough.md](docs/examples/module-keys-walkthrough.md)
— one module (API keys) taken from requirements to a filed bug, with
sample output at every step.

## Running the tests yourself (after step 4)

```bash
cd automation
npm ci
npm test                          # everything
npm run test:api                  # API only
npm run test:e2e                  # E2E only
npx playwright test --grep TC-014 # one test case
npm run report                    # open the HTML report
```

## Rules you must never break

- **Mirror environment only** — never production.
- **Dummy data only** — never real customer data.
- Tests create their own `qa-` data and delete it.
- Never change a test's expected result just to make it pass — log a bug.

Full rules: [CLAUDE.md](CLAUDE.md). Coding standard:
[.claude/skills/ts-pom-standards/SKILL.md](.claude/skills/ts-pom-standards/SKILL.md).

## Where things live

```
CLAUDE.md               shared rules every agent follows
.claude/commands/       the slash commands you type
.claude/agents/         the specialist agents the commands start
.claude/skills/         our standards (test design, security, POM, bug reports...)
scripts/verify-setup.py checks commands → agents → skills are wired correctly
docs/                   inputs, analysis, test cases, reports, bugs
automation/             the TypeScript test project (created in step 4)
```
