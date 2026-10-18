---
name: event-definition
description: Turn a recorded UI walk into event definitions — every click, page load, and navigation, the API calls it fires with request/response detail, and the database tables it touches. Use when mapping UI events to API calls or database tables, or when an event definition is requested.
---

# Event Definition — what each UI action actually does

**Goal:** for every event a user can trigger, write down exactly what the
system does behind the screen, so a test can assert on it and a failure
can be traced to a layer.

**Input:** approved `docs/discovery/sequences/SEQ-*.md` and the raw
recordings in `automation/discovery/artifacts/`.
**Output:** `docs/discovery/events/<module>.events.md`, one file per
module (template:
[templates/events-template.md](templates/events-template.md)).

## 1. Two tiers — know which one you are writing

| Tier | Content | Certainty | Where it comes from |
|---|---|---|---|
| **1 — Event + API** | Trigger, element, API calls with method, route, payload, status, response, timing | **Observed fact** | The network recording. Available now |
| **2 — DB effect** | Tables touched, insert/update/delete, which columns | **Claim — needs a source** | Read-only DB diff. `PENDING` until `DB_TYPE` and credentials exist |

Never write a Tier 2 row you cannot evidence. `PENDING` is a correct,
useful answer; a guessed table name is worse than nothing because the
next person will trust it.

## 2. Event types

| Type | Trigger | Typical evidence |
|---|---|---|
| `onload` | A screen or panel renders | GET calls fired without a user action |
| `click` | Button, link, menu item, row | The calls between the click and the next stable screen |
| `submit` | A form is sent | Usually one POST/PUT/PATCH plus a reload GET |
| `traverse` | Navigation between screens, tabs, pagination, back | Route change, often with a new GET |
| `background` | Polling, websockets, retries with no user action | Repeating calls on a timer |

`background` matters more than it looks: a screen that polls every 2 s is
a future source of flaky tests and a reason to use condition-based waits.

## 3. Tier 1 — from the recording to the definition

For each step in the sequence:

1. Take the network slice between this step and the next.
2. Drop noise: static assets, fonts, analytics, source maps. Keep every
   call to the application's own API host.
3. For each remaining call record: method, route **with path parameters
   generalised** (`/endpoints/{id}/keys`, not the literal UUID), request
   body shape, status, response shape, duration.
4. Mask secrets in both directions before writing anything down.
5. Note the ORDER. "POST then GET" versus "GET then POST" is the
   difference between a correct test and a flaky one.

Record the response **shape**, not a dump: field names and types, and
anything unexpected. A response holding fields the screen never shows is
a finding (possible data leak — see `api-key-security` API3).

## 4. Tier 2 — the database effect

Until `DB_TYPE` is set, every event's DB block reads:

```
> PENDING — no read-only DB access yet. Raised as Q-0xx.
```

When read-only access arrives, `db-validator` fills these blocks in by
**snapshot diff**, and only it may edit an approved event file's `db:`
block (CLAUDE.md section 9):

1. Before the action: for each candidate table, record row count and the
   maximum primary key / `updated_at`.
2. Perform the action with a `qa-` record.
3. After: diff. New rows → insert. Changed `updated_at` → update. Fewer
   rows → delete.
4. Narrow to the `qa-` row and list the columns it actually filled.

Snapshot diff is **approximate** — other traffic on a shared test
environment can move the same tables. So:

- Run it on a quiet environment, and repeat each action twice.
- Record a confidence: `confirmed` (same result twice, isolated row
  matched) or `likely` (table moved but the row could not be isolated).
- A table that changes on every action regardless of which action it was
  is noise (session, audit, cache) — list it once per module, not per
  event.

Candidate tables come from the schema in `docs/requirements/db/`. Never
`SELECT *` a table you cannot name a reason for.

## 5. What to flag while mapping

| Observation | Why it matters |
|---|---|
| One click fires the same GET several times | Performance; often a React double-render bug |
| A call returns 4xx/5xx but the UI shows success | Silent failure — the worst class of bug |
| A 5xx anywhere | Always a bug (CLAUDE.md section 6) |
| An id, email, or key in a query string | Leaks into logs and browser history |
| A call with no auth header on an authenticated screen | Possible missing authorisation |
| The screen works but the call was never made (cached) | The test needs a different wait |
| A response field never shown on screen | Possible over-exposure of data |

## 6. How the automation uses this

Event definitions are what let the generators write tests that wait on
the right thing instead of sleeping:

- `waitForResponse('**/endpoints/*/keys')` comes straight from the event.
- An API test for the same route reuses the recorded request/response
  shape as its starting schema (the **test case** still owns the expected
  values — CLAUDE.md section 6).
- A DB check inside a UI test asserts on the Tier 2 table and columns.

So write routes precisely. A vague route is a vague wait, and a vague
wait is a flaky test.

## Rules

- `status: draft` until approved. One file per module; IDs are global and
  permanent.
- Group by event, not by request — one click that fires four calls is ONE
  `EVT-`, with four rows in its API table.
- Every `EVT-` names the sequence and step it came from, so a reader can
  find the screenshot.
- Re-running updates the existing file in place; existing `EVT-` IDs keep
  their numbers.
- Finish with: counts per event type, how many events have a DB mapping
  versus `PENDING`, findings raised, and "Ready for review."
