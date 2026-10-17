---
name: change-impact
description: Handle a requirement change from the product team without redoing everything — read the change, trace which features, rules, test cases, test data, and automated tests it touches, propose targeted edits, and (after approval) update only those. Use when product sends new information, a change request, answers to questions, or a new PRD version after the analysis was approved.
---

# Change Impact Analysis

**Goal:** a change touches only what it must. Nothing is rebuilt from
scratch, no ID is renumbered, and every change is recorded.

**Input:** one of
- a change request: `docs/input/changes/CR-<nnn>-<short-name>.md`
- new answers added to `docs/input/answers.md`
- a new version of a PRD/FRD/mockup in `docs/input/`

**Output:** `docs/changes/CR-<nnn>-impact.md` (using
[templates/impact-template.md](templates/impact-template.md)), then —
only after approval — targeted edits to the existing files.

## Phase 1 — Analyze (no edits yet)

1. **What changed** — compare the new input with the current
   `docs/analysis/requirement-analysis.md` (and, for a new document
   version, with the old version). List each change as:
   - **Added** — new rule or feature
   - **Modified** — existing F-/BR- with new behaviour (quote old vs new)
   - **Removed** — no longer required
   - **Answered** — closes a Q- question
2. **Trace the impact** down the chain, by searching for the IDs:

| Level | Where to search | How |
|---|---|---|
| Features / rules | `docs/analysis/requirement-analysis.md` | F-/BR- IDs |
| Test cases | `docs/test-cases.md` → "Req IDs" column | rows citing those F-/BR- |
| Test data | `test-data/*.json` → `rule` and `used_by` | records citing those BR-/TC- |
| Automated tests | `automation/tests/` | `grep -rn "TC-014"` in test titles |
| Scope decisions | `CLAUDE.md` scope table, `.env` | 403/404, revocation delay, DB, customization |

3. **Size it** — Small (≤ 3 test cases) · Medium (4–10) · Large (> 10, or
   a scope decision changes).
4. **Show the impact report** in chat (summary) and file (full), then
   STOP and wait for approval. If the change conflicts with something
   already approved, list it as a `[CONFLICT]` question instead of
   choosing a side.

## Phase 2 — Apply (only after the user approves)

Edit in place. Keep every existing ID.

| File | Added | Modified | Removed |
|---|---|---|---|
| requirement-analysis.md | New F-/BR- with the next free number | Update the text; add `(changed CR-<nnn>)` | Mark `DEPRECATED (CR-<nnn>)` — never delete |
| test-cases.md | New TC- rows with the next free number | Update steps/expected result; add `(changed CR-<nnn>)` | Mark the row `DEPRECATED (CR-<nnn>)` — never delete |
| Questions | — | — | Close answered Q- with the answer and source |

Then add one line to the **Change log** at the end of each file you
edited: `| CR-<nnn> | YYYY-MM-DD | <one-line summary> | <IDs touched> |`.

Do NOT edit test data, automation, `CLAUDE.md`, or `.env` yourself.
Instead, finish with the exact follow-up commands for ONLY the affected
IDs, for example:

```
/test-data:data-factory TC-014 TC-031
/automation-api:api-test-generator TC-014 TC-031
/automation-ui:playwright-generator TC-031
/execution:test-executor TC-014 TC-031
```

…and, if a scope decision changed, tell the lead which line of
`CLAUDE.md` / `.env` to update.

## Rules

- Never renumber or reuse an ID — deprecated IDs stay deprecated.
- Never change an expected result without a source (the CR, an answer,
  or a new document version).
- A verbal change is not a change: ask for it in writing (an email or
  chat pasted into a CR file is fine).
- One change request = one impact report, even if small.
