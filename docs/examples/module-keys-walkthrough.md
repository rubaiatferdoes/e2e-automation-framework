# Example: one module end to end — API keys

> **This is an illustrative example.** The requirement text and outputs
> are invented, but every format, ID, file name, and command is exactly
> what the framework uses. Use it to see what "good" looks like at each step.

```
analyze-requirements module:keys → test-planner module:keys → data-factory module:keys
→ api-test-generator module:keys → (api-test-reviewer) → test-executor module:keys
```

---

## The starting point

The endpoints module was analyzed last week (used F-001..F-003,
BR-001..BR-006, TC-001..TC-011). Product sends:

**`docs/requirements/prd-v1.2.pdf` — §4 API Keys**
> 4.1 A customer can generate up to 5 API keys per endpoint.
> 4.2 Each key needs a name (3–40 characters).
> 4.3 The key is shown once after generation.
> 4.4 A customer can revoke a key. Revoked keys stop working.
> 4.5 Callers send the key in the `x-api-key` header.

**`docs/requirements/mockup-05-generate-key.png`** — a "Generate key" dialog,
and a key list with a **"Copy key"** button on every row.

**`docs/requirements/client-notes-2026-10-15.md`**
> "Client mentioned keys should probably expire, maybe after 90 days?"

---

## Step 1 — Analyze only the keys module

```
/requirements:analyze-requirements module:keys
```

Runs in your chat. Reads all input, analyzes only key-related material,
continues numbering after the endpoints module.

**Chat summary**
```
Analyzed: module keys (PRD §4, mockup-05, client-notes-2026-10-15)
Skipped:  PRD §2–3 (endpoints — already approved), §5 (docs)

Features: 4 (F-004..F-007)   Rules: 5 (BR-007..BR-011)   Gaps: 5   Blocker questions: 1

Top gaps:
 [MISSING]  How fast must a revoked key stop working? (flow step 5)
 [CONFLICT] PRD 4.3 "shown once" vs mockup-05 "Copy key" button on every row
 [UNCLEAR]  Key expiry "maybe 90 days" — client note only, not in PRD
 [MISSING]  Response when tenant B uses its key on tenant A's endpoint (403 or 404?)
 [MISSING]  Is the key stored hashed?

Blocker: Q-005 — revocation delay (can't write the "key is dead" test without it)
```

**Added to `docs/analysis/requirement-analysis.md`**

```
### Module: API keys [keys]
#### F-004: The system shall generate an API key for an endpoint when the customer gives it a valid name.
| BR-007 | Max 5 keys per endpoint            | PRD §4.1 |
| BR-008 | Key name required, 3–40 characters | PRD §4.2 |
#### F-005: The system shall show the full key only once after generation.
| BR-009 | Full key shown once, then masked   | PRD §4.3 |
#### F-006: The system shall reject calls made with a revoked key.
| BR-010 | Revoked key → request rejected     | PRD §4.4 |
#### F-007: The system shall authenticate endpoint calls with the x-api-key header.
| BR-011 | Key sent in x-api-key header       | PRD §4.5 |
```

| ID | Priority | Question | Unblocks |
|---|---|---|---|
| Q-005 | Blocker | How many seconds until a revoked key stops working? | F-006, flow 5 |
| Q-006 | High | Is "Copy key" on the list a mistake? PRD says shown once | BR-009 |
| Q-007 | High | Cross-tenant key use: 403 or 404? | F-007, E2E-04 |
| Q-008 | High | Do keys expire? Client said "maybe 90 days" | F-006 |

**You check:** every rule cites PRD §4 · the conflict is a question, not
a guess · module index shows "keys: Analyzed — awaiting approval".

**Then:** send Q-005..Q-008 to product, approve the analysis. Product
answers Q-005 ("within 60 seconds") → add it to `docs/requirements/answers.md`
→ `/change:impact-analysis answers` → Q-005 closed, new rule
**BR-012: Revoked key rejected within 60 s**.

---

## Step 2 — Test cases for the keys module

```
/test-design:test-planner module:keys
```

Resolves: `module:keys → F-004..F-007, BR-007..BR-012 · flow steps 3, 4, 5 · E2E-02, E2E-03, E2E-04, E2E-07`

**Written to `docs/test-cases/keys/`** — one file per case, plus the
module `README.md` index (table below) and the overall
`docs/test-cases/README.md`.

| TC ID | Title | Req IDs | Module / Flow | Pri | Type | Layer |
|---|---|---|---|---|---|---|
| TC-012 | Generate key with valid name shows full key once | F-004, BR-009 | keys / 3 | P0 | Positive | UI+API |
| TC-013 | Key name length limits (data-driven) | BR-008 | keys / 3 | P2 | Boundary | API |
| TC-014 | Revoked key rejected within 60 s | F-006, BR-012 | keys / 5 / E2E-03 | P0 | E2E | API |
| TC-015 | 6th key on one endpoint is rejected | BR-007 | keys / 3 | P2 | Boundary | API |
| TC-016 | Key is masked after page reload | BR-009 | keys / 3 | P1 | Security | UI |
| TC-017 | Tenant B's key can't call tenant A's endpoint | F-007 | keys / 4 / E2E-04 | P1 | Security | API |
| TC-018 | Missing / empty / wrong / malformed key → 401 | F-007, BR-011 | keys / 4 / E2E-02 | P0 | Negative | API |
| TC-019 | Two keys, revoke one, the other still works | F-006 | keys / 5 / E2E-07 | P2 | E2E | API |

One of those files, `docs/test-cases/keys/TC-014-revoked-key-rejected.md`:

```markdown
---
id: TC-014
title: Revoked key rejected within 60 s
module: keys
flow: [5]
e2e: E2E-03
requirements: [F-006, BR-012]
priority: P0
type: E2E
layer: [API]
data: []
status: draft
---
# TC-014: Revoked key rejected within 60 s

## Precondition
- Logged in as `qa-tenant-a`; one `qa-` endpoint with one active key

## Steps
| # | Action | Expected result |
|---|---|---|
| 1 | Call the endpoint with the key | 200 |
| 2 | Revoke the key | Key shows as revoked |
| 3 | Call again with the same key, retrying every second | 401 within 60 s; no data in the response |
```

- TC-017 is marked `ASSUMPTION — verify (Q-007)` — 403 vs 404 is still open.
- Key expiry is NOT tested — it's an open question (Q-008), not a requirement.

**You check:** 8 cases, not 30 · every rule covered · P0 for flow steps
3, 4, 5 · boundaries merged into one data-driven case (TC-013).

---

## Step 3 — Test data for the keys module

```
/test-data:data-factory module:keys
```

Resolves: `module:keys → TC-012..TC-019`

**`test-data/negative-data.json`** (boundaries for BR-008: 3–40 characters)
```json
[
  { "id": "DAT-021", "field": "keyName", "value": "", "expected": "rejected", "rule": "BR-008", "used_by": ["TC-013"] },
  { "id": "DAT-022", "field": "keyName", "value": "ab", "expected": "rejected", "rule": "BR-008", "used_by": ["TC-013"] },
  { "id": "DAT-023", "field": "keyName", "value": "abc", "expected": "accepted", "rule": "BR-008", "used_by": ["TC-013"] },
  { "id": "DAT-024", "field": "keyName", "value": "kkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkk", "expected": "accepted", "rule": "BR-008", "used_by": ["TC-013"] },
  { "id": "DAT-025", "field": "keyName", "value": "kkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkk", "expected": "rejected", "rule": "BR-008", "used_by": ["TC-013"] }
]
```

**`test-data/security-data.json`** (fake keys for TC-018)
```json
{ "id": "DAT-030", "field": "x-api-key", "value": "not-a-real-key", "expected": "401", "rule": "BR-011", "used_by": ["TC-018"] }
```

`docs/test-cases/keys/TC-013-key-name-length-limits.md` now has
`data: [DAT-021, DAT-022, DAT-023, DAT-024, DAT-025]` in its front matter.

**You check:** 2, 3, 40, 41 characters all present (min−1, min, max,
max+1) · no real keys anywhere.

---

## Step 4 — API automation for the keys module

```
/automation-api:api-test-generator module:keys
```

Resolves: `module:keys → API cases TC-012, 013, 014, 015, 017, 018, 019`
(TC-016 is UI-only → left for the Playwright generator).

**Client method (API Object Model)**
```ts
// automation/api/clients/PlatformClient.ts
async revokeApiKey(endpointId: string, keyId: string): Promise<ApiResponse<void>> {
  return this.delete<void>(`/endpoints/${endpointId}/keys/${keyId}`);
}
```

**The test — reads like the test case**
```ts
// automation/tests/api/keys.api.spec.ts
test('[TC-014] revoked key rejected within 60 s',
  { tag: ['@P0', '@F-006', '@mod-keys', '@flow-5', '@E2E-03', '@slow'] },
  async ({ platform, endpointCaller, qaEndpoint }) => {
    test.skip(!ENV.keyRevocationMaxMs, 'Waiting for product: revocation delay (Q-005)');
    test.setTimeout(ENV.keyRevocationMaxMs + 60_000);
    const key = await platform.createApiKeyFor(qaEndpoint);

    // Step 1 — prove the key works first, so a broken key can't pass this test
    const before = await endpointCaller.call(qaEndpoint, key.value);
    expect(before.status).toBe(200);
    expectMatchesSchema(before.body, 'endpoint-response');

    // Step 2 — revoke, and the platform shows it as revoked
    await platform.revokeApiKey(qaEndpoint.id, key.id);
    expect((await platform.getApiKey(qaEndpoint.id, key.id)).body.status).toBe('revoked');

    // Step 3 — the same key stops working within the agreed delay
    await expect.poll(() => endpointCaller.status(qaEndpoint, key.value),
      { timeout: ENV.keyRevocationMaxMs }).toBe(401);
  });
```

- It proves the key **worked** (200 + schema check) before proving it
  **stopped** working — otherwise a key that never worked would pass.
- It is slow by design (up to the 60 s delay), so it sets its own timeout,
  is tagged `@slow`, and is not part of `@smoke`.
- No selectors or raw HTTP calls in the test — those live in the client.
- `qaEndpoint` is a fixture: it creates `qa-tc014-lz3k9a-w2-x7f1` before the
  test and deletes it after, even if the test fails.
- It polls up to the agreed 60 seconds — no fixed sleep.

**Summary**
```
Created 7 tests for 7 TC IDs · smoke (E2E-01 API part) passed
TC-013 runs 5 times (DAT-021..025)
Next: /automation-api:api-test-reviewer module:keys
```

Recommended before execution: `/automation-api:api-test-reviewer module:keys`
(3 green runs in a row = STABLE).

---

## Step 5 — Run the keys module

```
/execution:test-executor module:keys
```

Readiness → READY · typecheck ✓ · runs `npx playwright test --grep @mod-keys`,
then the `@smoke` set as a guard.

**`docs/reports/execution-2026-10-22.md` (excerpt)**
```
Selection: module:keys → TC-012..TC-019

| Suite        | Passed | Failed | Flaky | Skipped |
| API (keys)   | 10     | 1      | 0     | 0       |
| Smoke guard  | 3      | 0      | 0     | 0       |

Failures
| Test   | Type            | Evidence                                               |
| TC-014 | Application bug | Revoked key still returned 200 after 60 s (polled 60×) |

Bugs raised: BUG-003 [API Key] Revoked key still works after 60 seconds — Sev-2

Parity (keys only)
 TC-016 (UI, masked key) — no automated test yet → /automation-ui:playwright-generator TC-016
```

7 tests, but TC-013 runs once per data row (5) → 11 runs: 10 passed, 1 failed.

`docs/bugs/BUG-003-revoked-key-still-works.md` holds steps, expected
(quoting BR-012 / TC-014), actual (200 OK), request/response with the key
masked (`sk_...a1b2`), and timestamps.

---

## What this example shows

| | |
|---|---|
| **Scope** | Only the keys module was touched — endpoints IDs, cases, data, tests unchanged |
| **Traceability** | BUG-003 → TC-014 → BR-012 → answer to Q-005 → PRD §4.4 |
| **Questions turn into tests** | Product's answer to Q-005 → BR-012 → TC-014 → found BUG-003 |
| **Unknowns stay visible** | Expiry (Q-008) and 403-vs-404 (Q-007) are tracked, not guessed |
| **Coverage gaps get caught** | Parity spotted that TC-016 still needs UI automation |

**Next for this module:** `/automation-ui:playwright-generator TC-016` →
after the fix, `/execution:test-executor TC-014` → when all P0/P1 pass,
`/reporting:quality-reporter module:keys`.
