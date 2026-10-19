# Example: one module end to end — API keys

> **This is an illustrative example.** The screens and outputs are
> invented, but every format, ID, file name, and command is exactly what
> the framework uses. Use it to see what "good" looks like at each step.

```
explore-app → event-definition → validation-definition → test-planner
→ test-case-designer → data-factory → api-test-generator → test-executor
→ investigate-failures
```

---

## The starting point

No PRD. No mockups. What we have is what the team actually gave us:

```
TEST_ENV_BASE_URL = <the test environment>
QA_TENANT_A_USER  = <a test login>
```

and a sentence from the product owner: *"check the API key part — a
customer generates a key for an endpoint and calls it."*

That is enough to start.

---

## Step 1 — Travel the app

```
/discovery:explore-app "generate an API key for an endpoint and copy it"
```

The agent scaffolds `automation/discovery/`, logs in as `qa-tenant-a`,
creates its own endpoint to work with (`qa-seq002-lz3k9a-x7f1`), and
walks the workflow one screen at a time — screenshotting each screen and
recording every request.

**Chat summary**
```
Walked: SEQ-002 "Generate an API key for an endpoint" — 4 steps, 5 screenshots
Created and deleted: qa-seq002-lz3k9a-x7f1 (endpoint)

Findings: 2
  1. "Copy key" button appears on the key LIST too, after reload — but the
     key is masked there, so it copies "sk_...a1b2". Looks unintended.
  2. Generating a key took 4.1 s. Slow enough to flake a test later.

Questions: 1
  Q-001  Is the key meant to be shown only once? The list still offers "Copy".
```

**`docs/discovery/sequences/SEQ-002-generate-api-key.md`**

```markdown
---
id: SEQ-002
title: Generate an API key for an endpoint
module: keys
flow: [3]
screens: [endpoint-list, endpoint-detail, generate-key-dialog, key-shown]
events: []
validations: []
walk: automation/discovery/walks/generate-api-key.walk.ts
status: draft
---

## Path
| # | Screen | Action | What happened (observed) | Event | Screenshot |
|---|---|---|---|---|---|
| 1 | Endpoint list | Open /endpoints | Table of endpoints, "Create endpoint" top right | | [png] |
| 2 | Endpoint detail | Click the endpoint name | "API keys" panel, empty state "No keys yet" | | [png] |
| 3 | Generate key dialog | Click "Generate key" | Dialog, one field "Key name", buttons Cancel / Generate | | [png] |
| 4 | Key shown | Type a name, click Generate | Full key shown once, Copy button, "You won't see this again" | | [png] |
```

Note the Event column is empty, and nothing says "works correctly". It
records what was on the screen.

**You check:** the path matches what a real user would do · findings are
raised, not smoothed over · the endpoint it created was deleted.

**Then:** approve the sequence.

---

## Step 2 — What each action really does

```
/discovery:event-definition SEQ-002
```

No second walk — it mines the recording from step 1.

**`docs/discovery/events/keys.events.md`** (one event of four)

```markdown
## EVT-012 — Generate key
| Type | click · generate-key-dialog · `[data-testid="generate-key-submit"]` |
| From | SEQ-002 step 4 |

### API calls (Tier 1 — observed)
| # | Method | Route | Request | Status | Response | ms |
|---|---|---|---|---|---|---|
| 1 | POST | /api/endpoints/{id}/keys | { name: string } | 201 | { id, name, key, status } | 3900 |
| 2 | GET  | /api/endpoints/{id}/keys | — | 200 | { items: [{ id, name, maskedKey, status }] } | 180 |

Order: POST completes before the list GET fires.

### DB effect (Tier 2)
> PENDING — no read-only DB access yet. Raised as Q-002.

### Findings
| 1 | The POST response logs the full key to the browser console | Key can leak into logs | BUG candidate |
```

Three things just became possible: the API tests know the real route, the
UI tests know what to wait for instead of sleeping, and the 3.9-second
POST explains the "slow" finding from step 1.

**You check:** routes have `{id}` not a literal UUID · the DB block says
PENDING instead of a guess · the console-logging finding is raised.

---

## Step 3 — What the fields accept

```
/discovery:validation-definition screen:generate-key-dialog
```

Reads the DOM first, then probes the field, then — the part that matters
— sends the same bad values straight to `POST /api/endpoints/{id}/keys`,
and finally compares everything against the standard validation baseline.

**`docs/discovery/validations/keys.validations.md`**

| ID | Field | Rule | Trigger | Message (exact) | Enforced |
|---|---|---|---|---|---|
| VAL-007 | Key name | required | on submit | `Key name is required.` | both |
| VAL-008 | Key name | max length 40 | on type | — (typing stops at 40) | **client** |
| VAL-009 | Key name | min length 3 | on submit | `Key name must be at least 3 characters.` | both |
| VAL-010 | Key name | unique per endpoint | server | `A key with this name already exists.` | server |

```
Findings: 1
  VAL-008 is enforced in the browser only. The API accepted a 256-character
  key name and stored it. Any API caller writes names the UI can never show.
  → BUG candidate, and a test case the UI alone would never have found.

Standard comparison (5 baseline items apply to this screen)
  **Minimum length**  VAL-009 rejects 2 characters with the limit stated   Match
  **Maximum length**  VAL-008 is client-only; the API stored 256           Gap
  every other row    Match — one row per applicable item, Matches included
```

**You check:** messages quoted exactly, full stops included · the
`client`-only row is flagged, not recorded as a normal rule · the
Standard comparison table is filled — Matches included — and its one
Gap is the same BUG candidate as the enforcement finding, not a new one.

**Then:** approve all three discovery files. **This approval is the
moment observation becomes expectation** — everything downstream now
treats these as the truth.

---

## Step 4 — Decide what to test

```
/test-design:test-planner module:keys
```

Thinks like a principal test engineer: what is worth testing, how deep,
and what we will not test.

**`docs/test-plan.md`** (the keys rows)

| Flow step | UI | API | Docs | DB | Priority | Why |
|---|---|---|---|---|---|---|
| 3. Generate API key | In | In | — | Later (PENDING) | P0 | Show-once is unrecoverable if wrong |
| 5. Revoke key | In | In | — | Later | P0 | A key that outlives revocation is a security hole |

```
No requirement documents exist for this module. Expected results rest on
approved observation — recorded as a risk.

Conflicts: none (nothing to compare against).
Depth decisions:
  - VAL-008 is client-only → the boundary case runs at the API layer, not the UI
  - EVT-012 takes 3.9 s → condition-based waits only; this test is not @smoke
```

**You check:** every "Later" has a reason · the missing-documents risk is
stated rather than hidden.

---

## Step 5 — Manual test cases

```
/test-design:test-case-designer module:keys
```

Input: the test plan, SEQ-002, and the validations. Not the events —
those are for the automation step.

| TC ID | Title | From | Pri | Type | Layer |
|---|---|---|---|---|---|
| TC-012 | Generate key with valid name shows full key once | SEQ-002, VAL-007 | P0 | Positive | UI+API |
| TC-013 | Key name length limits (data-driven) | VAL-008, VAL-009 | P2 | Boundary | API |
| TC-014 | Revoked key rejected within the agreed delay | SEQ-003 | P0 | E2E | API |
| TC-015 | Duplicate key name rejected | VAL-010 | P2 | Negative | API |
| TC-016 | Key is masked after page reload | SEQ-002, Q-001 | P1 | Security | UI |

TC-013 is at the **API layer on purpose** — VAL-008 is client-only, so a
UI test would pass while the hole stayed open. That one decision is the
whole reason step 3 checks enforcement.

**You check:** 5 cases, not 30 · every case cites a SEQ- or VAL- · TC-016
is marked `ASSUMPTION — verify (Q-001)` because the show-once rule is
still open.

---

## Step 6 — Test data

```
/test-data:data-factory module:keys
```

Boundaries come straight from VAL-008/VAL-009 (3–40 characters):

```json
[
  { "id": "DAT-021", "field": "keyName", "value": "ab",            "expected": "rejected", "rule": "VAL-009", "used_by": ["TC-013"] },
  { "id": "DAT-022", "field": "keyName", "value": "abc",           "expected": "accepted", "rule": "VAL-009", "used_by": ["TC-013"] },
  { "id": "DAT-023", "field": "keyName", "value": "{repeat:k:40}", "expected": "accepted", "rule": "VAL-008", "used_by": ["TC-013"] },
  { "id": "DAT-024", "field": "keyName", "value": "{repeat:k:41}", "expected": "rejected", "rule": "VAL-008", "used_by": ["TC-013"] }
]
```

DAT-024 expects `rejected` because the **rule** says 40, even though the
API accepted 256 during discovery. The test will fail, and it should —
that failure is the bug report.

---

## Step 7 — Automation, both layers

```
/automation-api:api-test-generator module:keys
/automation-ui:playwright-generator module:keys
```

Both run, because the cases span both layers: TC-013 and TC-015 are
API-only, TC-012 is UI+API, TC-016 is UI. Shared config, fixtures and
`qa-` cleanup are written once by whichever generator runs first.

### API side

The client method comes from EVT-012's route:

```ts
// automation/api/clients/PlatformClient.ts
async createApiKey(endpointId: string, name: string): Promise<ApiResponse<ApiKey>> {
  return this.post<ApiKey>(`/endpoints/${endpointId}/keys`, { name });
}
```

```ts
// automation/tests/api/keys.api.spec.ts
for (const row of dataFor('TC-013')) {
  test(`[TC-013] ${row.id} key name "${short(row.value)}" is ${row.expected}`,
    { tag: ['@P2', '@VAL-008', '@mod-keys', '@flow-3', '@api'] },
    async ({ platform, qaEndpoint }) => {
      const res = await platform.createApiKey(qaEndpoint.id, row.value);
      expect(res.status).toBe(row.expected === 'accepted' ? 201 : 400);
      expect(res.status).toBeLessThan(500);
    });
}
```

- Route and payload shape from the event definition.
- Expected status from the test case and its data — **not** from what
  discovery observed.
- `toBeLessThan(500)` because a 5xx is always a bug (CLAUDE.md).

### UI side

Locators live apart from the page object, which is what makes FIX-001 in
step 9 a one-line change:

```ts
// automation/ui/locators/key.locators.ts
export const keyLocators = {
  generateOpen:   '[data-testid="generate-key"]',
  nameInput:      '[data-testid="key-name"]',
  generateSubmit: '[data-testid="generate-key-submit"]',
  shownDialog:    '[data-testid="key-shown"]',
};
```

```ts
// automation/tests/ui/keys.ui.spec.ts
test('[TC-012] generate key with valid name shows the full key once',
  { tag: ['@P0', '@SEQ-002', '@mod-keys', '@flow-3', '@ui'] },
  async ({ keyPage, qaEndpoint }) => {
    await keyPage.goto(qaEndpoint.id);
    await keyPage.generate(qaName('tc012'));
    await expect(keyPage.shownDialog).toContainText("won't see this again");
  });
```

- Setup and cleanup go through the API, never the UI — a UI test should
  fail on its own subject, not on someone else's form.
- The wait is condition-based: EVT-012's POST takes ~3.9 s, so no
  fixed timeout appears anywhere.

---

## Step 8 — Run it

```
/execution:test-executor module:keys
```

```
| Suite       | Passed | Failed | Flaky | Skipped |
| API (keys)  | 7      | 1      | 0     | 0       |
| UI (keys)   | 2      | 1      | 1     | 0       |
| Total       | 9      | 2      | 1     | 0       |

Failures (first pass — not diagnosed here)
| TC-013 / DAT-024 | API | Suspected application bug | 201 where the case says 400 |
| TC-012           | UI  | Suspected script bug      | locator timeout on the Generate button |
Flaky
| TC-016 | UI | passed on retry | generate-key took 4.2 s

Next: /healing:investigate-failures module:keys
```

The executor collects evidence and hands off. It does not file bugs —
one agent should not both run the tests and judge them.

---

## Step 9 — Investigate every failure

```
/healing:investigate-failures module:keys
```

Reproduces each twice, then decides — from the trace, video and network,
not the error text.

```
| Test            | Verdict           | Outcome                    |
| TC-013/DAT-024  | Application issue | BUG-003 (Sev-2) — PROPOSED |
| TC-012          | Broken test       | FIX-001 proposed           |
| TC-016 (flaky)  | Broken test       | FIX-002 proposed           |
```

Nothing here is filed or applied. The bug and both fixes come to you
with their evidence; you decide which become real.

**BUG-003** — the API accepts a 41-character key name while the UI stops
at 40 (VAL-008). Exactly the hole step 3 predicted. Evidence: request,
response, the VAL- rule, the TC- step. Written `Status: PROPOSED` —
yours to file or reject.

**FIX-001** — proposed, not applied:

```diff
--- a/automation/ui/locators/key.locators.ts
-  generateSubmit: '[data-testid="generate-key-submit"]',
+  generateSubmit: '[data-testid="key-generate-confirm"]',
```

> Bug guard
> - [x] Changes what is asserted? **No** — only where the button is
> - [x] Same control? Yes — same dialog, same POST /api/endpoints/{id}/keys
> - [x] Definitions to update? Yes — EVT-012 element, SEQ-002 step 4
>
> Blast radius: TC-012, TC-013, TC-016 · **PROPOSED — approve to apply**

**FIX-002** — the flaky test waits for a toast. EVT-012 shows the POST
takes ~4 s, so the proposal waits on the response instead:

```diff
-  await expect(page.keyShownDialog).toBeVisible();
+  await page.waitForResponse('**/endpoints/*/keys');
+  await expect(page.keyShownDialog).toBeVisible();
```

Note what the healer did **not** do: it did not raise the timeout, add a
sleep, or relax the 400 in TC-013 to make the suite green.

---

## What this example shows

| | |
|---|---|
| **No documents needed** | A URL, a login and one sentence produced 5 test cases, 12 test runs across two layers, and a real bug |
| **Observation ≠ truth** | Discovery recorded a 256-character key name; the test still expects 400, so the gap surfaced as BUG-003 |
| **Enforcement check pays** | VAL-008's `client` verdict is what moved TC-013 to the API layer — a UI-only test would have passed |
| **Events kill flakiness** | EVT-012's 3.9 s POST explained the flake and supplied the right wait |
| **Healing has a limit** | Two fixes proposed, one bug drafted for your approval — nothing weakened, nothing applied, nothing committed |
| **Traceability** | BUG-003 → TC-013 → DAT-024 → VAL-008 → SEQ-002 step 4 → a screenshot |

**Next for this module:** approve FIX-001 and FIX-002 → re-run
`/execution:test-executor module:keys` → when the P0s pass,
`/reporting:quality-reporter module:keys`. With `DB_ACCESS=none` the `@persistence` probes check storage instead. When read-only DB access
arrives, `/database:db-validator module:keys` fills EVT-012's PENDING
block and checks the key is stored hashed.
