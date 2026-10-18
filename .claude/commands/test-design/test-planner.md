---
description: Step 2 — Test Planner agent — decide test scope, write test plan + test cases (positive/negative/boundary/security/E2E); optionally recheck the test environment
argument-hint: [analysis path] [optional: module:keys | flow:3 | E2E-03 | F-003 to plan only that part] [optional: "recheck" or a test environment URL]
context: fork
agent: test-planner
background: false
---

Create the test scope, test plan, and manual test cases from: $ARGUMENTS

If no analysis path was given, use docs/analysis/requirement-analysis.md.
If the arguments contain "recheck" or a URL, first recheck the test
environment read-only and write docs/analysis/environment-recheck.md.
Write docs/test-plan.md and one file per test case in
docs/test-cases/<module>/, plus the module and overall README indexes.
