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

**1. Read the catalog once.** This is where the schema comes from, so
nobody needs to export one:

```sql
SELECT table_name, column_name, data_type, character_maximum_length, is_nullable
FROM information_schema.columns
WHERE table_schema = :schema;
```

Plus `table_constraints` and `key_column_usage` for primary and foreign
keys.

**2. Act** — perform the action through the platform API with the tracer
record. Never write to the database to set up a check.

**3. Locate** — query the text columns for that literal value, name-like
columns first (`name`, `title`, `slug`, `label`, `key`):

```sql
SELECT 'api_keys' AS tbl, 'name' AS col, id FROM api_keys WHERE name = :tracer
UNION ALL
SELECT 'endpoints', 'name', id FROM endpoints WHERE name = :tracer;
```

Whatever returns is the table and column the action wrote to.

**4. Walk the foreign keys** outward to the child rows — `api_keys`
holds an `endpoint_id`, not the name, so it is reached from
`endpoints.id`. Two levels is almost always enough.

**5. Diff by primary key** — re-read those rows before and after and
compare field by field. This is what answers *where is it updated*:

| Observed | Verdict |
|---|---|
| Row exists where it did not | INSERT — list the columns it filled |
| Same key, some columns differ | UPDATE — each column `before → after` |
| Row gone | DELETE (hard) |
| Row present, `deleted_at`/`status` now set | Soft delete — recorded as an UPDATE, **and raised**, since hard vs soft delete is usually an open question |

Because every query is scoped to one tracer or one primary key:

| Concern | Why it's fine |
|---|---|
| Disturbing other people's testing | Nothing is locked. Readers don't block writers, and the queries run in a read-only transaction |
| Load on the database | `information_schema` reads are metadata only; row lookups are by key |
| Reading real customer data | Only your own tracer and the ids it leads to are ever queried |
| Other traffic skewing results | Impossible — nothing is counted globally, so someone else's click cannot affect your row |

No quiet maintenance window is needed. This is the main advantage over
counting rows before and after.

## 2b. What lands in the event file

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

**How we know** matters as much as the data. In six months, "how do we
know this?" is the question someone will ask.

## 2c. Read-only is enforced three times

1. The database user has `SELECT` rights only
2. `DbClient` opens every query in a read-only transaction
3. `DbClient` rejects any statement that does not start with `SELECT`

If the user turns out to be able to write, the agent **stops** and
reports it as a Sev-1 environment risk.

## 2d. The two kinds of check it adds

| Kind | Where | Example |
|---|---|---|
| **Inside existing API/E2E tests** | The spec gains a DB step after the action | `[TC-001]` creates an endpoint via API → row exists, owned by `qa-tenant-a` |
| **Standalone integrity checks** | `tests/db/<area>.db.spec.ts` | No key rows without an endpoint · no plain-text keys · no leftover `qa-` rows older than 24 h |

Both carry the `@db` tag, so `--grep @db` runs everything DB-related,
while `npm run test:db` runs only the standalone project.

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
