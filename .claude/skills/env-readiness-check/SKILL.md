---
name: env-readiness-check
description: Check the test environment is ready before any test run — required environment variables set, URLs reachable, test tenants work, Swagger reachable, tools installed, leftover qa- data cleaned. Prints a READY / NOT READY table. Use before running tests or when asked if the product/environment is ready.
allowed-tools: Bash, Read, Glob
---

# Environment Readiness Check

**Goal:** a 2-minute check that saves hours of debugging "failures" that
are really environment problems.

Run each check with Bash. Never print secret values — only say whether
they are set.

## Before the checks — load the settings

Values live in `.env` in the project root (copied from `.env.example`).
Each Bash call starts fresh, so begin EVERY Bash command with:

```bash
set -a; source .env; set +a;
```

No `.env` file → verdict NOT READY: "Copy `.env.example` to `.env` and
fill in the test environment values." Stop there.

## Checks

| # | Check | How | Pass when |
|---|---|---|---|
| 1 | Settings filled in | Compare against `.env.example` | Every variable in its REQUIRED section has a value. Empty OPTIONAL ones are listed as `⚠️ SKIPPED — waiting for decision`, never FAIL |
| 2 | Test environment UI reachable | `curl -s -o /dev/null -w "%{http_code}" "$TEST_ENV_BASE_URL"` | 200/3xx |
| 3 | Platform API reachable | same, on `$TEST_ENV_API_URL` health or docs route | 200 |
| 4 | Swagger reachable | `curl` on `$TEST_ENV_SWAGGER_URL` | 200 and body is JSON/YAML |
| 5 | Test tenants exist | Log in / call the platform API as `qa-tenant-a` and `qa-tenant-b` (login route from the requirement analysis) | Both succeed. Login route not known yet → `⚠️ SKIPPED` |
| 6 | Tools installed | `node -v` (18+), `npx playwright --version`. If `automation/` exists: `npm ci` done, `npm run typecheck` passes | Versions print. `automation/` missing (before step 4) → that part is `⚠️ SKIPPED — automation/ not created yet`, not FAIL. `automation/` exists → typecheck must also pass |
| 7 | Leftover test data | List endpoints/keys starting with `qa-` older than 24h | None — or list them for cleanup |

## Output

```
ENVIRONMENT READINESS — <date/time>
| # | Check | Result | Note |
|---|---|---|---|
| 1 | Settings | ✅ PASS | 7/7 required set · 6 optional empty (SKIPPED) |
...
VERDICT: READY  (or: NOT READY — fix #2, #5 first)
```

## Rules

- Leftover `qa-` data (check 7): list it and ask before deleting.
  Delete only via the platform API, only `qa-` items, only on the test environment.
- If a check can't run because a value is missing, mark it `⚠️ SKIPPED`
  with the reason — don't mark it PASS.
- Check 6's `npm ci`/`typecheck` part only applies once `automation/`
  exists (created in step 4). Running this skill at step 0, before any
  automation code exists, is normal — don't let that alone cause NOT READY.
