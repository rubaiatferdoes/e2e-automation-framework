---
module: keys
sequences: [SEQ-002]
events: [EVT-010, EVT-011, EVT-012, EVT-013]
db_mapping: pending        # pending | partial | confirmed
status: draft
discovered: YYYY-MM-DD
---

# Events — API keys `[keys]`

> Observation only. Tier 1 (API calls) is recorded fact. Tier 2 (DB) is a
> claim and must name its evidence.

## Index
| Event | Type | Screen | Trigger | Sequence step | API calls | DB |
|---|---|---|---|---|---|---|
| EVT-012 | click | generate-key-dialog | "Generate" button | SEQ-002 step 4 | 2 | PENDING |

---

## EVT-012 — Generate key

| Field | Value |
|---|---|
| Type | `click` |
| Screen | generate-key-dialog |
| Element | `[data-testid="generate-key-submit"]` (button "Generate") |
| From | SEQ-002 step 4 · [screenshot](../screens/SEQ-002-04-key-shown.png) |
| Observed | Dialog closes, full key shown once with a Copy button |

### API calls (Tier 1 — observed)
| # | Method | Route | Request (shape) | Status | Response (shape) | ms |
|---|---|---|---|---|---|---|
| 1 | POST | `/api/endpoints/{id}/keys` | `{ name: string }` | 201 | `{ id, name, key, createdAt, status }` | 240 |
| 2 | GET | `/api/endpoints/{id}/keys` | — | 200 | `{ items: [{ id, name, maskedKey, status }] }` | 90 |

Order matters: POST completes before the list GET fires. A test waits on
call 2 before asserting the list.

### DB effect (Tier 2)
> PENDING — no read-only DB access yet. Raised as Q-0xx.

<!-- When DB access arrives, db-validator replaces the line above with:
| Table | Operation | Columns filled | Confidence | Evidence |
|---|---|---|---|---|
| `api_keys` | INSERT | `id, endpoint_id, name, key_hash, created_at, status` | confirmed | snapshot diff ×2, row `qa-seq002-...` |
-->

### Findings
| # | What we saw | Why it matters | Raised as |
|---|---|---|---|
| 1 | The POST response contains the full `key` in plain text — expected for show-once, but it is also written to the browser console by the app | Key could leak into logs | BUG-0xx (candidate) |

---

## Shared background calls (this module)
| Route | When | Note |
|---|---|---|
| `GET /api/session` | Every screen load | Noise — ignore in per-event tables |

## Change log
| CR | Date | Change |
|---|---|---|
