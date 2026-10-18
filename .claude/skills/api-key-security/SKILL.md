---
name: api-key-security
description: Security test checklist for API keys and tenant isolation, mapped to the OWASP API Security Top 10 (2023). Covers key lifecycle (create, show-once, store, revoke, expire), authentication failures, cross-tenant access, rate limits, and safe error messages. Use when designing, automating, or reviewing security tests for the API endpoint builder.
---

# API Key & Tenant Security

**Goal:** a key only ever opens its own customer's endpoints — and stops
working the moment it should.

Test environment only. Attack-style inputs (injection strings, huge
payloads) only against `qa-` data you created.

## 1. Key lifecycle

| # | Check | Expected |
|---|---|---|
| K1 | Create a key | Full key shown once; copy works |
| K2 | Reload the page / list keys | Key is masked (e.g. `sk_...a1b2`) |
| K3 | Key storage (DB, when available) | Hashed — never the plain key |
| K4 | Key in logs, URLs, error messages | Never appears |
| K5 | Revoke a key, then call | 401 within the agreed revocation delay (poll, don't sleep) |
| K6 | Expired key (if expiry exists) | 401 |
| K7 | Rotate key (if supported) | New key works; old key dead per policy |

## 2. Authentication failures — all must return 401 and no data

| Input | Example |
|---|---|
| No key | header missing |
| Empty key | `x-api-key: ` |
| Wrong key | random string of the right length |
| Malformed key | too short, too long, unicode, spaces |
| Key in the wrong place | in query string instead of header (unless the docs allow it) |
| Revoked / expired key | see K5, K6 |

## 3. OWASP API Security Top 10 (2023) — what applies to us

| OWASP | Risk in our product | Test |
|---|---|---|
| API1 Broken Object Level Authorization | Tenant B reaches tenant A's endpoint/key/docs | Use B's key and session on A's resources → rejected (403 or 404 — per product decision) |
| API2 Broken Authentication | Weak key handling | Sections 1 and 2 above |
| API3 Broken Object Property Level Authorization | Response leaks internal fields; request sets fields it shouldn't (e.g. `tenantId`) | Check responses for extra fields; send `tenantId` of tenant B in a create request → ignored or rejected |
| API4 Unrestricted Resource Consumption | No rate limit / huge payloads | Burst calls → 429 with `Retry-After` (if defined); oversize payload → 413/400, never 5xx |
| API5 Broken Function Level Authorization | Customer calls admin-only platform API | Customer session on admin routes → 403 |
| API8 Security Misconfiguration | Open CORS, verbose errors, HTTP allowed | No stack traces/SQL in errors; CORS per requirement; HTTP → HTTPS only |
| API9 Improper Inventory Management | Deleted endpoints still reachable | Deleted endpoint → 404; gone from docs |

## 4. Injection inputs (endpoint name and payloads)

Use the negative data from `test-data/negative-data.json`. Expected for
every one: rejected with a clear 4xx error OR stored safely as plain
text — **never** a 5xx, never executed (no script runs in the UI).

- SQL: `' OR '1'='1`, `'; DROP TABLE x; --`
- XSS: `<script>alert(1)</script>`, `"><img src=x onerror=alert(1)>`
- Path: `../../etc/passwd`

## Rules

- Response code for cross-tenant access (403 vs 404) and the revocation
  delay are **product decisions, not observations**. Watching the app
  return 404 does not settle it — that is exactly the kind of behaviour
  that could be the bug. Use the confirmed answer from product (via the
  requirement analysis or an approved `Q-`); if there is none, mark
  `ASSUMPTION — verify`, raise a `Q-`, and let the test skip itself
  (`ENV.crossTenantStatus`).
- A security failure is at least P1 / Sev-2.
