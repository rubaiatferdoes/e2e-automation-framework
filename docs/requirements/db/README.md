# docs/requirements/db — database schema goes here

When the database is decided, add the files below and the `.env`
settings, then run `/database:db-validator`.

## What the database access is used for

Three jobs, and only the first is obvious:

| # | Job | Why it needs the DB |
|---|---|---|
| 1 | **Check the app stored things correctly** | Tests create data through the API like a real customer, then read the database to confirm the row exists, belongs to the right tenant, and holds a hashed key — not a plain one |
| 2 | **Fill the `PENDING` mappings in the event definitions** | Discovery can see that clicking "Generate" fires `POST /api/endpoints/{id}/keys`, but not which table that handler wrote to. Until this runs, every event's DB block reads `PENDING` |
| 3 | **Compare the schema with the validation rules** | The UI stops at 40 characters, the API accepted 256, the column is `VARCHAR(30)` — three different limits in one system is a bug waiting to reach production |

Job 2 is why a read-only user matters even before anyone writes a DB
test: it is what completes the picture of what each UI action does.

**QA never writes to the database.** Only the application writes. If the
read-only user turns out to be able to write, the validator stops and
reports it as a Sev-1 environment risk.

## 1. Files to add here

| File | What | Who gives it |
|---|---|---|
| `schema.sql` | Schema export — tables, columns, types, keys, constraints (no data!) | Backend / DBA (`pg_dump --schema-only`, `mysqldump --no-data`, or SSMS "Script as CREATE") |
| `erd.png` (optional) | Diagram of tables and relationships | Backend / DBA |
| `notes.md` (recommended) | Anything not visible in the schema | Backend |

**Never** add a data dump — the schema only. No customer rows.

### What to put in `notes.md`

This is the file that saves the most time, because it narrows which
tables to look at. Worth asking backend for:

- Which table holds tenants, and how a row is linked to one
- Soft delete or hard delete (is there a `deleted_at`?)
- How API keys are hashed, and in which column
- Tables that change on **every** request — session, audit, cache, job
  queues. These are noise for job 2 and get listed once instead of being
  blamed on each event
- Anything written asynchronously (a job, a trigger, a queue), because a
  snapshot taken straight after the click will miss it

## 2. Settings in `.env` (project root)

```
DB_TYPE=postgres            # postgres | mysql | mssql
DB_URL=<test environment DB host:port/dbname>
DB_READONLY_USER=<user with SELECT rights only>
DB_READONLY_PASSWORD=<password>
```

Ask DevOps for a **read-only** user on the **test environment**
database, with `SELECT` on the application tables. No `INSERT`,
`UPDATE`, `DELETE`, or DDL — the framework enforces this three times
over (user rights, read-only transaction, and a check that every
statement starts with `SELECT`), but the grant is the real protection.

## 3. For job 2, a quiet environment helps

Filling the event mappings works by **snapshot diff**: count rows before
the action, perform it with a `qa-` record, count again, and see what
moved. That is approximate — if someone else is clicking around the test
environment at the same time, the same tables move for unrelated
reasons.

So, when you can:

- run the mapping when nobody else is using the test environment;
- expect the agent to repeat each action twice and record a confidence
  (`confirmed` when the result matched twice and the `qa-` row was
  isolated, `likely` when the table moved but the row could not be);
- treat `likely` rows as a hint, not a fact.

Anything less certain than that stays `PENDING`. A guessed table name is
worse than no table name, because the next person will trust it.

## 4. Tell the framework the DB is decided

Update the "Database type" row in `CLAUDE.md` (scope table) from
"Unknown — read-only access is coming" to the chosen type.

## 5. Until all of this exists

Nothing is blocked. `DB_TYPE` empty means:

- event definitions keep `db: PENDING` and carry a `Q-`;
- DB steps inside API/E2E tests skip **only that step**, so the rest of
  the test still runs;
- standalone `tests/db/` specs skip whole;
- the test plan marks the DB column `Later` with the reason.

That is the intended state right now, not a failure — see the scope
table in `CLAUDE.md`.
