---
name: env-readiness-check
description: Check the mirror environment is ready before any test run — required environment variables set, URLs reachable, not production, test tenants work, Swagger reachable, tools installed, leftover qa- data cleaned. Prints a READY / NOT READY table. Use before running tests or when asked if the product/environment is ready.
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
fill in the mirror values." Stop there.

## Checks

| # | Check | How | Pass when |
|---|---|---|---|
| 1 | Settings filled in | Compare against `.env.example` | Every variable in its REQUIRED section has a value. Empty OPTIONAL ones are listed as `⚠️ SKIPPED — waiting for decision`, never FAIL |
| 2 | NOT production | Take the **hostname** of `MIRROR_BASE_URL`, `MIRROR_API_URL`, `MIRROR_SWAGGER_URL` and compare it, exactly and case-insensitively, with each entry in `PROD_HOSTS` | `PROD_HOSTS` is set AND no mirror hostname equals a prod host (`mirror.company.com` vs `app.company.com` is fine — never a substring match). Empty `PROD_HOSTS` = FAIL. **Exact match: STOP everything.** |
| 3 | Mirror UI reachable | `curl -s -o /dev/null -w "%{http_code}" "$MIRROR_BASE_URL"` | 200/3xx |
| 4 | Platform API reachable | same, on `$MIRROR_API_URL` health or docs route | 200 |
| 5 | Swagger reachable | `curl` on `$MIRROR_SWAGGER_URL` | 200 and body is JSON/YAML |
| 6 | Test tenants exist | Log in / call the platform API as `qa-tenant-a` and `qa-tenant-b` (login route from the requirement analysis) | Both succeed. Login route not known yet → `⚠️ SKIPPED` |
| 7 | Tools installed | `node -v` (18+), `npx playwright --version`, and in `automation/`: `npm ci` done, `npm run typecheck` passes | Versions print; typecheck passes |
| 8 | Leftover test data | List endpoints/keys starting with `qa-` older than 24h | None — or list them for cleanup |

## Output

```
ENVIRONMENT READINESS — <date/time>
| # | Check | Result | Note |
|---|---|---|---|
| 1 | Settings | ✅ PASS | 9/9 required set · 6 optional empty (SKIPPED) |
| 2 | Not production | ✅ PASS | |
...
VERDICT: READY  (or: NOT READY — fix #3, #6 first)
```

## Rules

- Check 2 failing = stop and tell the user. Do not run anything else.
- Leftover `qa-` data (check 8): list it and ask before deleting.
  Delete only via the platform API, only `qa-` items, only on the mirror.
- If a check can't run because a value is missing, mark it `⚠️ SKIPPED`
  with the reason — don't mark it PASS.
