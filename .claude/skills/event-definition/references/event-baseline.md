# Standard Event Baseline — what a well-built action does

What a competently built web application does when a user acts. Event
discovery records what the application **does**; this file is the one
"ought" in that step — the standard every action is also measured
against. A missing behaviour is a **gap**, and a gap is raised, not
recorded and forgotten.

## Rule or Decision — read this first

This baseline differs from the validation one in an important way.
Almost every field rule is near-universal: a maximum length either
exists or is missing. Action behaviour is not like that. Some items are
defects when absent; others are legitimate design choices where the
**silence** is the problem, not the choice.

So every item below carries one of two marks:

| Mark | Meaning | A gap becomes |
|---|---|---|
| **Rule** | A well-built app always does this | A finding (`BUG-` candidate) |
| **Decision** | Reasonable apps differ, but someone must have chosen | A `Q-` — and never a bug |

Treating a Decision as a Rule is how a QA framework starts filing bugs
against deliberate design. Do not do it.

## How to use this baseline — the oracle rules

| Situation | What to do |
|---|---|
| The app lacks a **Rule** item | A finding, `BUG-` candidate if data or security is at risk. Never an `EVT-`, never an expected result |
| The app lacks a **Decision** item | A `Q-`: what was intended here? |
| The app does more than the baseline | Record as `Stricter`. Not a bug, never "fixed" to match |
| The baseline looks wrong for this screen | A `Q-` about the baseline. It is a default, not a law |

- The baseline never creates an `EVT-` row. Only an action you actually
  recorded creates one.
- Items are cited by name, in bold — `**One write per submit**` — never
  by number.
- Attack technique lives in `api-key-security`. This file says a
  well-built action **has** the behaviour; that skill says how to prove
  it fails.

## 1. Universal rules by event type

### `onload` — a screen or panel renders

| Item | Mark | A well-built app… | Typical gap |
|---|---|---|---|
| **No write on read** | Rule | Opening a screen fires no POST/PUT/PATCH/DELETE | A GET-shaped action mutates; prefetch corrupts data |
| **Server is the source** | Rule | After a write elsewhere, the screen refetches rather than trusting stale local state | List shows a record that no longer exists |
| **No duplicate fetch** | Rule | One render fires each GET once | The same GET three times per render |
| **Bounded list** | Rule | Collection endpoints page or cap | 10,000 rows in one response |
| **Loading state** | Rule | Anything over ~1s shows a pending state | Blank screen, user clicks again |
| **Call depth** | Decision | Independent calls run in parallel, not chained | A four-deep waterfall; slow and brittle |

### `click` — read-only (open a dialog, sort, filter, page)

| Item | Mark | A well-built app… | Typical gap |
|---|---|---|---|
| **No write** | Rule | Sorting or filtering writes nothing | A filter persists state server-side unasked |
| **Where sorting happens** | Decision | Client-side or server-side, consistently | Sorts the current page only, looks like data loss |
| **Something happens** | Rule | Every enabled control does something observable | A button that fires nothing and shows nothing |

### `submit` — a write

The highest-value section. Most real defects live here.

| Item | Mark | A well-built app… | Typical gap |
|---|---|---|---|
| **One write per submit** | Rule | One user submission creates exactly one record | Double-click creates two rows |
| **Guarded while in flight** | Rule | The control disables or otherwise blocks resubmission | Nothing stops a second click |
| **Success means success** | Rule | The success message appears only after a 2xx | Toast says "Saved", the call returned 400 |
| **Failure is shown** | Rule | A non-2xx produces a visible, specific message | Silent failure — the worst class of defect |
| **State reflects the server** | Rule | After the write, what is shown matches what was stored | The list shows the value you typed, not the value saved |
| **No forged fields accepted** | Rule | Owner, tenant, id, role sent by the client are ignored or rejected | Posting another user's id assigns their record |
| **Optimistic UI** | Decision | If the UI updates before the server confirms, a rollback path exists | Update shown, write failed, nobody rolls back |
| **Retry safety** | Decision | A resubmitted write is idempotent, or clearly is not | Network retry silently duplicates |

### `traverse` — navigation, tabs, paging, back

| Item | Mark | A well-built app… | Typical gap |
|---|---|---|---|
| **URL reflects state** | Rule | The address changes, so the page is linkable and Back works | Deep link impossible; Back leaves the app |
| **Back does not resubmit** | Rule | Returning to a form does not repeat the write | Back creates a second record |
| **In-flight requests** | Decision | Abandoned requests are cancelled, or at least never surface as errors | Navigating away shows a failure toast |

### `background` — polling, websockets, retries with no user action

| Item | Mark | A well-built app… | Typical gap |
|---|---|---|---|
| **Stated interval** | Rule | Polls on a known, regular interval | Unbounded loop, growing traffic |
| **Stops on leave** | Rule | Polling stops when the screen is left | Timers accumulate per visit |
| **No retry storm** | Rule | A failed poll backs off | A down backend gets hammered |
| **Pauses when hidden** | Decision | Stops or slows on a hidden tab | — |

A `background` event found here has a direct testing consequence:
**condition-based waits only.** A test that asserts between two polls on
a screen that refreshes itself is flaky by construction.

## 2. HTTP semantics standard

| Item | Mark | Standard |
|---|---|---|
| **GET is safe** | Rule | A GET never changes state. This breaks caches, prefetch and crawlers when violated |
| **Method matches intent** | Rule | POST creates, PUT/PATCH updates, DELETE removes |
| **DELETE is idempotent** | Rule | A second DELETE returns 404 or 204 — never 500 |
| **Create status** | Decision | 201 (ideally with `Location`) or 200 with the body — consistently |
| **Delete status** | Decision | 204 or 200 — consistently; and a follow-up GET returns 404 |
| **Client error status** | Rule | 4xx, specific: 400 bad input, 401 unauthenticated, 403 forbidden, 404 absent, 409 conflict |
| **Never 5xx** | Rule | A 5xx on any input is always a bug, never a documented behaviour |
| **One error envelope** | Rule | Every error shares one body shape across the API |

## 3. Request and response hygiene

| Item | Mark | A well-built app… | Typical gap |
|---|---|---|---|
| **Auth on every call** | Rule | Authenticated screens send credentials on every request | One route quietly unauthenticated |
| **No secrets in URLs** | Rule | Tokens, keys, passwords and emails never appear in a path or query string | Leaks into logs, history, referrers |
| **No over-exposure** | Rule | Responses carry only fields a client needs | Password hashes, internal ids, other tenants' data |
| **Consistent shape** | Decision | The same resource looks the same in list and detail, or a documented subset | Fields appear and vanish between views |
| **Payload bounded** | Rule | Oversized requests are rejected with 413/400, never a 5xx | A 50 MB body takes the service down |

## 4. Storage effects

### When a read-only database user exists

| Item | Mark | A well-built app… |
|---|---|---|
| **One row per create** | Rule | Writes exactly one row for one submission |
| **No orphans** | Rule | Deleting a parent leaves no rows behind in child or join tables |
| **Audit columns filled** | Decision | If the schema has `created_by` / `created_date`, they are populated |
| **Soft or hard delete** | Decision | One or the other, stated — not discovered by accident |
| **No write on GET** | Rule | A read fires no INSERT or UPDATE |

### When there is no database access — the substitutes

Most of the above has an observable proxy through the API alone. Use
these when Tier 2 is `PENDING`, and record `source: black-box probe`.

| Question | Probe |
|---|---|
| Is the column shorter than the UI allows? | **Round-trip:** create at the maximum length, then GET in a *fresh request*. A shorter value came back → silent truncation |
| Soft delete or hard? | **Delete-and-recreate:** delete, then create again with the same unique value. Rejected as duplicate → the row is still there |
| Orphans after a parent delete? | **Cascade probe:** create parent and children, delete the parent, call each child's id directly. 200 → orphan, still reachable |
| Is a secret stored in plain text? | **Exposure sweep:** after show-once, check every list, detail, error and export for the plain value. Proves non-exposure, not hashing |
| Did the write actually persist? | **Write confirmation:** compare what was sent, what the write echoed, and what a later GET returns. Any mismatch is a finding |
| Does a double submit duplicate? | **Idempotency probe:** submit twice, count through the list endpoint |

These substitute for *some* of the database layer, never all of it.
Encryption at rest, exact tables and columns, and writes with no API
surface stay unverifiable — say so rather than implying coverage.

## 5. Timing and reliability

| Item | Mark | Standard |
|---|---|---|
| **Pending state** | Rule | Any action over ~1s shows the user something is happening |
| **Slow actions recorded** | Rule | Anything over ~5s is recorded as a flakiness risk, with its timing |
| **Concurrent edits** | Decision | Two edits to one record: last-write-wins or a 409 conflict — stated either way |
| **Timeouts surface** | Rule | A timed-out call produces a message, not a stuck spinner |

## 6. Minimum check matrix per event type

Which of the checks above apply to which event type. A check marked `no`
is recorded as skipped in the event file — skipped on purpose, not
forgotten.

| Check | onload | click (read) | submit | traverse | background |
|---|---|---|---|---|---|
| No write on read | yes | yes | — | yes | yes |
| One write per submit | — | — | yes | — | — |
| Success/failure shown correctly | — | — | yes | — | — |
| Duplicate fetch | yes | yes | yes | yes | yes |
| Auth on every call | yes | yes | yes | yes | yes |
| No secrets in URLs | yes | yes | yes | yes | yes |
| Over-exposure | yes | yes | yes | — | yes |
| Status codes correct | yes | yes | yes | yes | yes |
| Never 5xx | yes | yes | yes | yes | yes |
| Storage effect / substitutes | — | — | yes | — | — |
| URL reflects state | — | — | — | yes | — |
| Stops on leave | — | — | — | — | yes |
| Pending state | yes | — | yes | yes | — |

## 7. Product overlay — the endpoint-builder specifics

What THIS product's actions should do beyond the universal items. The
baseline asserts **existence and behaviour, never the number**.

### Creating an endpoint

- The create action writes once, and the endpoint appears in the list
  from a server refetch, not from local state.
- Its Swagger entry exists from the moment of creation. (Spec validity
  itself is `api-contract-check` — cross-reference, don't duplicate.)

### Generating an API key

- The full key is returned exactly once, by the create call alone.
- **Never afterwards:** no later GET, list, error or export carries the
  plain value. This is the exposure sweep of section 4, and it is a Rule.
- The key value appears in no console log and no URL.

### Calling a generated endpoint

- Missing, empty, wrong, malformed and revoked keys each return 401 with
  a user-safe body. The full matrix is `api-key-security` §2.
- Rate limiting, if present, returns 429 — ideally with `Retry-After`.
  `ASSUMPTION — verify`: not every comparable product rate-limits.

### Revoking a key

- Revocation takes effect within the agreed delay — a **Decision** until
  product answers, so the test polls up to that limit and never sleeps.
- The key-dead check must run **before** the endpoint is deleted, or a
  404 hides whether revocation worked.

### Deleting an endpoint

- Its path stops resolving, it leaves the list, and it leaves the docs.
- No key rows outlive it (`**No orphans**`, or the cascade probe when
  there is no database access).

## Where this baseline comes from, and how it changes

The HTTP specifications for method safety and status semantics, the
OWASP API Security Top 10 as mapped in `api-key-security`, and the
public behaviour of comparable products. Reviewed by the lead; the
review date is recorded in this file's git history.

A discovery run never edits this file — gaps go to `Q-` or a finding.
Changing the standard itself (adding, weakening, or **renaming an
item**) is the lead's decision through a normal review; cited names are
quoted verbatim in event files, so a rename is a change to those files
too.
