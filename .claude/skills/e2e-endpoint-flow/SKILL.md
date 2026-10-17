---
name: e2e-endpoint-flow
description: The end-to-end journey of the API endpoint builder — create endpoint in UI, auto-generated Swagger docs, generate API key, call the endpoint, revoke/delete, key is dead — with the checkpoints to verify at every step and the standard E2E scenarios. Use when designing, automating, or reviewing E2E tests for this product.
---

# E2E Endpoint Flow

**Goal:** prove a customer can go from zero to a working, secured API —
and that it is fully gone when they delete it.

## The journey and what to check at each step

| # | Step | UI check | API check | Docs check | DB check (optional*) |
|---|---|---|---|---|---|
| 1 | Create endpoint `qa-<tc>-<timestamp>` | Success message; endpoint in list | Platform API lists it with the same name/method/path | — | Row exists, belongs to the right tenant |
| 2 | Swagger docs generated | Docs page opens | Spec downloads and is valid OpenAPI | New path + method appear; auth scheme declared | — |
| 3 | Generate API key | Key shown ONCE; masked afterwards | Key appears in key list (masked) | — | Stored hashed, never plain text |
| 4 | Call endpoint with key | — | 2xx; response matches the expected schema from the test case | Response matches the documented schema | — |
| 5 | Revoke key, then delete endpoint | Both disappear from lists | Platform API no longer lists them | Path removed from docs | Rows deleted or marked deleted; no orphans |
| 6 | Key is dead | — | Call with old key → 401 (poll up to the agreed revocation delay); deleted endpoint → 404 | — | — |

\* DB checks are skipped until the database type is decided (see
`CLAUDE.md` scope table). Mark them `SKIPPED — DB pending`.

## Standard E2E scenarios

| ID | Scenario | Priority | Key expected result |
|---|---|---|---|
| E2E-01 | Happy path: steps 1 → 6 in order | P0 | Every checkpoint above passes |
| E2E-02 | Call with no key / wrong key / malformed key | P0 | 401, no data in response |
| E2E-03 | Call with a revoked key | P0 | 401 within the agreed revocation delay |
| E2E-04 | Tenant B calls tenant A's endpoint with B's key | P1 | Rejected (403 or 404 — per product decision Q-) |
| E2E-05 | Call a deleted endpoint with a valid key | P1 | 404 |
| E2E-06 | Docs match reality: the documented request works exactly as documented | P1 | Same status and schema as the docs say |
| E2E-07 | Two keys for one endpoint, revoke one | P2 | Revoked → 401, other still → 2xx |

When writing test cases, turn each scenario into a TC- row with the
checkpoint table as its expected results.

## How to automate it

Everything is TypeScript + Playwright, following the `ts-pom-standards` skill.

- **UI steps** (create, generate key, view docs) → page objects in `ui/pages/`.
- **API steps** (call endpoint, verify key dead) → API clients in
  `api/clients/`, in the same test.
- **Setup and cleanup through the platform API**, not the UI — faster
  and less flaky. Only the step under test goes through the UI.
- **Cleanup always runs**, even if the test fails — the fixture in
  `fixtures/baseTest.ts` revokes the key, then deletes the endpoint.
- **Key-dead checks poll** — retry the call every 1s up to the agreed
  revocation delay, then fail. Never a fixed sleep.
- Use `qa-tenant-a` for the main flow and `qa-tenant-b` for E2E-04.
