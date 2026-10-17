# docs/requirements/db — database schema goes here

When the database is decided, add these, then run `/database:db-validator`.

## 1. Files to add here

| File | What | Who gives it |
|---|---|---|
| `schema.sql` | Schema export — tables, columns, types, keys, constraints (no data!) | Backend / DBA (`pg_dump --schema-only`, `mysqldump --no-data`, or SSMS "Script as CREATE") |
| `erd.png` (optional) | Diagram of tables and relationships | Backend / DBA |
| `notes.md` (optional) | Anything not visible in the schema: soft vs hard delete, how keys are hashed, which table holds tenants | Backend |

**Never** add a data dump — the schema only. No customer rows.

## 2. Settings in `.env` (project root)

```
DB_TYPE=postgres            # postgres | mysql | mssql
DB_URL=<test environment DB host:port/dbname>
DB_READONLY_USER=<user with SELECT rights only>
DB_READONLY_PASSWORD=<password>
```

Ask DevOps for a **read-only** user on the **test environment** database. If the
user can write, the DB validator stops and reports it.

## 3. Tell the framework the DB is decided

Update the "Database type" row in `CLAUDE.md` (scope table) from
"Unknown" to the chosen type.
