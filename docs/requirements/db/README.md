# docs/requirements/db — database access and schema

To switch the database checks on you need **one thing**: a read-only
database user on the test environment, in `.env`. Then run
`/database:db-validator`.

Everything in this folder is optional extra context.

## What read-only access is used for

Three jobs, and only the first is obvious:

| # | Job | What it needs |
|---|---|---|
| 1 | **Check the app stored things correctly** — row exists, right tenant, key hashed not plain | `SELECT` |
| 2 | **Fill the `PENDING` table mappings in the event definitions** — which table each UI action writes to, and which columns it updates | `SELECT` |
| 3 | **Compare the schema with the validation rules** — the UI stops at 40 characters, the API accepted 256, the column is `VARCHAR(30)` | `SELECT` on `information_schema` |

Job 2 is why this matters even before anyone writes a DB test: it is what
completes the picture of what each click actually does.

**QA never writes to the database.** Only the application writes. If the
read-only user turns out to be able to write, the validator stops and
reports it as a Sev-1 environment risk.

## 1. Settings in `.env` (project root)

```
DB_TYPE=postgres            # postgres | mysql | mssql
DB_URL=<test environment DB host:port/dbname>
DB_READONLY_USER=<user with SELECT rights only>
DB_READONLY_PASSWORD=<password>
DB_SCHEMA=                  # empty = driver default (public / dbo)
```

Ask DevOps for a **read-only** user on the **test environment**
database, with `SELECT` on the application tables and on
`information_schema`. No `INSERT`, `UPDATE`, `DELETE`, or DDL. The
framework blocks writes three times over — user rights, a read-only
transaction, and a check that every statement starts with `SELECT` — but
the grant is the real protection.

That is the whole ask. No config flags, no logging to enable, nothing
from the backend team.

## 2. How the table mapping works — and why it is safe

Every record the framework creates has a unique name like
`qa-seq002-lz3k9a-x7f1`. That string exists nowhere else in the
database, so it works as a tracer:

1. Read `information_schema` once — tables, columns, primary keys,
   foreign keys. **This is where the schema comes from**, so nobody
   needs to export one.
2. Perform the action with the tracer record.
3. Search the text columns for that literal value → the table and column
   it landed in.
4. Follow the foreign keys outward to find the child rows (the key row,
   the settings row).
5. Re-read those rows **by primary key** before and after → exactly which
   columns changed, and from what to what.

Because every query is scoped to one tracer or one primary key:

| Concern | Why it's fine |
|---|---|
| Disturbing other people's testing | Nothing is locked. Readers don't block writers, and the queries run in a read-only transaction |
| Load on the database | `information_schema` reads are metadata only; row lookups are by key |
| Reading real customer data | Only your own tracer and the ids it leads to are ever queried |
| Other traffic skewing results | Impossible — nothing is counted globally, so someone else's click cannot affect your row |

No quiet maintenance window is needed.

## 3. Optional files you can add here

Not required — the catalog in step 2 gives the framework the schema. But
these make the output better:

| File | What | Why it helps |
|---|---|---|
| `notes.md` | Soft vs hard delete · how keys are hashed · which table holds tenants · tables that change on every request (session, audit, cache) · anything written asynchronously by a job or trigger | Narrows the search and explains the rows the tracer cannot reach |
| `erd.png` | Diagram of tables and relationships | Faster to read than FK metadata |
| `schema.sql` | Schema export, no data | Useful for review in a PR; the framework does not depend on it |

**Never** add a data dump. Schema only, no customer rows.

## 4. What this method cannot see

Honest limits, so nobody over-trusts the output:

- The SQL statements themselves, or their order inside a transaction
- Writes with no link back to your row — counters, cache tables, audit
  rows without a foreign key. These are listed as **"not reachable by
  tracer"**, never as "no DB effect"
- Asynchronous writes that land after the check
- Whether a column was set by this action or by a database default

If you later want statement-level recording, the cheapest route is not
the database at all: ask backend to switch on SQL query logging on the
test deployment (one environment variable). Then each event's `db:`
block records `source: app SQL log` instead of `source: tracer`.

## 5. Tell the framework the DB is decided

Update the "Database type" row in `CLAUDE.md` (scope table) from
"Unknown — read-only access is coming" to the chosen type.

## 6. Until then

Nothing is blocked. `DB_TYPE` empty means:

- event definitions keep `db: PENDING` and carry a `Q-`
- DB steps inside API/E2E tests skip **only that step**, so the rest of
  the test still runs
- standalone `tests/db/` specs skip whole
- the test plan marks the DB column `Later`, with the reason

That is the intended state right now, not a failure — see the scope
table in `CLAUDE.md`.
