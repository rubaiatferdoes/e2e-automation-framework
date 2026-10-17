# CR-<nnn> Impact — <short title>

> Status: DRAFT

| Source | Received | Analyzed | Size |
|---|---|---|---|
| <file / email / answers.md Q-003> | YYYY-MM-DD | YYYY-MM-DD | Small / Medium / Large |

## 1. What changed
| Type | ID | Before | After | Source |
|---|---|---|---|---|
| Modified | BR-007 | Key works until deleted | Key expires after 90 days | CR-<nnn> §2 |
| Added | BR-031 (new) | — | Expiry warning email 7 days before | CR-<nnn> §3 |
| Answered | Q-004 | Open | Cross-tenant → 404 | answers.md |

## 2. Impact
| Level | Affected | Action |
|---|---|---|
| Features / rules | BR-007, BR-031 | Update / add |
| Test cases | TC-014 (modify), TC-040 (new) | Update / add |
| Test data | DAT-012 (used by TC-014) | Regenerate |
| Automated tests | `tests/api/keys.api.spec.ts` › `[TC-014] …` | Update |
| Scope decisions | CROSS_TENANT_STATUS → 404 | Lead updates `CLAUDE.md` + `.env` |

Unaffected: everything else (<n> test cases, <n> automated tests).

## 3. Conflicts and questions
- [CONFLICT] …

## 4. Follow-up commands (after approval)
```
/qa:test-data:data-factory TC-014 TC-040
/qa:automation-api:api-test-generator TC-014 TC-040
/qa:execution:test-executor TC-014 TC-040
```
