---
description: Step 5 — DB Validator agent — read-only, DB-agnostic SQL checks (drafts only until the DB type is decided)
argument-hint: [path to schema + test cases]
context: fork
agent: db-validator
background: false
---

Create database validation checks from: $ARGUMENTS

If no path was given, use docs/test-cases.md and any schema file under
docs/input/. If the DB type is not decided yet, write ANSI SQL drafts
only and do not connect. Save everything to db-tests/.
