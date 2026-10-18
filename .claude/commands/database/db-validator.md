---
description: Step 8 — DB Validator agent — compare the schema with the validations, fill the PENDING table mappings in the event definitions, add read-only DB checks (needs DB type + schema)
argument-hint: [all | module:keys | flow:1 | TC IDs]
context: fork
agent: db-validator
background: false
---

Add database checks for: $ARGUMENTS

If nothing was given, cover all approved test cases with a DB layer.
Needs DB_TYPE in .env and the schema in docs/requirements/db/ — if either is
missing, stop and say what to add. First write
docs/analysis/schema-check.md (schema vs requirements), then build
automation/db/ and the @db checks, and run them read-only.
