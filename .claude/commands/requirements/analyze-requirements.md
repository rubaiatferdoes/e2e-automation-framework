---
description: Step 1 — Requirement analysis (skill) on PRD/FRD/mockups/client notes — runs in this conversation so you can discuss the gaps
argument-hint: [all | module:keys | "API keys" | flow:3 | path, default docs/requirements/]
skill: requirement-analysis
---

Use the requirement-analysis skill on: $ARGUMENTS

If nothing was given, analyze everything under docs/requirements/.
If a module, flow step, or folder was given, analyze only that part and
add it to the existing analysis without touching other modules.

After the analysis:
1. Show me the top gaps and the questions, most important first.
2. Wait for my approval before any test case work starts.
