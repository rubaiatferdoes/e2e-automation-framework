---
description: Step 3 — Data Factory agent — valid, negative, and security dummy datasets mapped to TC IDs
argument-hint: [all | TC-014 TC-020 | TC-010..TC-015 | module:keys | flow:3 | F-003 | P0 | E2E | path to test cases]
context: fork
agent: data-factory
background: false
---

Generate dummy test data using: $ARGUMENTS

If no path was given, derive field rules from docs/test-cases.md and
docs/analysis/. If a schema/field rules file exists, use it.
Save datasets to test-data/ (valid-data.json, negative-data.json,
security-data.json, README.md).
