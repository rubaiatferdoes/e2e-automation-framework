# Standard Validation Baseline — what a well-built form checks

What comparable products (Xano, Supabase, AWS API Gateway, Postman)
check on their inputs. Discovery records what the application **does**;
this file is the one "ought" in discovery — the standard every screen is
also measured against. A missing rule is a **gap**, and a gap is raised,
not recorded and forgotten.

## How to use this baseline — the oracle rules

| Situation | What to do |
|---|---|
| The app lacks a rule the baseline has | A **gap** → a finding (`BUG-` candidate if bad data can get in) or a `Q-` if it is a product decision. Never a `VAL-`, never an expected result |
| The app has a rule the baseline doesn't | Record the row as `Stricter` — someone decided on more. Not a bug; never "fixed" to match |
| The baseline looks wrong for this screen | A `Q-` about the baseline. It is a default, not a law |

- The baseline never creates a `VAL-` row. Only a probe you actually ran
  creates one.
- Items are cited by name, in bold — `**Allowed characters**` — never by
  number.
- Attack technique lives in `api-key-security` (K1–K7, OWASP). This file
  says a well-built form **has** the rule; that skill says how to prove
  it fails.

## 1. Universal rules by field type

### Text — single line, free entry

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Rejects empty with a message naming the field | Submitted empty, stored empty |
| **Minimum length** | Rejects 1–2 character values with the limit stated | Accepted `a`, displayed everywhere |
| **Maximum length** | Blocks typing AND the API rejects the same value | Typing stops at 40; the API stored 256 |
| **Trim whitespace** | Trims, or rejects leading/trailing spaces — one of the two, stated | Stored `"  name  "` as a different value |
| **Allowed characters** | Rejects or escapes control characters; stores the rest as text | Control characters accepted, rendered raw |
| **Uniqueness** | Where the value names a thing, rejects a duplicate — server-side | Two records with one name |
| **Sanitization** | Stored as text, rendered escaped, never executed | Probe string appears interpreted |

### Long text (textarea)

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Enforced when the field is marked required | Skipped because it "has no input box limit" |
| **Server-side max length** | No `maxlength` attribute to lean on — only the server can enforce; it does | Unbounded column, first 1 MB write wins |
| **Sanitization** | Same as text — and the rendered surface is richer | Markdown/HTML passthrough unescaped |

### Number

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Rejects empty (an empty string is not zero) | Empty string stored as `0` |
| **Numeric only** | Rejects `12abc`, `1e999`, `NaN` cleanly | `1e999` → 5xx |
| **Minimum value** / **Maximum value** | Rejects out-of-range with the range stated | Negative count accepted |
| **Integer/step** | Enforces the declared step | `1.5` accepted for a step-1 field |

Absurd values (`1e999`, `-1`) must come back as a clean 4xx. A 5xx here
is a bug, not a rule.

### Date / time

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Rejects empty | — |
| **Valid date** | Rejects `2026-02-30`, `13/45/2026` | Stored as NULL, or 5xx |
| **Range** | Rejects outside the business range (expiry in the past) | Expiry date yesterday, key "valid" |
| **End after start** | Cross-field: rejects an end before the start | Both stored; behavior undefined |
| **Timezone stated** | Says (or asks) which zone an ambiguous date means | Same input, different row depending on browser zone |

### Email

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Rejects empty | — |
| **Format** | Rejects `not-an-email`, `a@b` (no TLD) per stated rule | `a@` accepted |
| **Maximum length** | 320 is the RFC bound; enforce and state it | Unbounded; DB column truncates |
| **Case handling** | States whether `A@B` and `a@b` are the same account | Two accounts, one inbox — or the reverse |

### URL / path

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Rejects empty | — |
| **Format** | Rejects spaces, control characters, `..` | Path traversal accepted into a route |
| **Protocol** | https only, or rejects `javascript:` | `javascript:` stored, rendered as a link |
| **Maximum length** | Enforced server-side | 2 KB URL breaks every downstream client |

### Select / dropdown

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Must choose** | Rejects the placeholder value | Placeholder string submitted as the value |
| **Allowed values only** | The **server** rejects a forged option, not just the UI | `mode=admin` posted directly, accepted |

### Checkbox / toggle

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Boolean only** | Rejects or normalizes a forged `"yes"` / `"on"` / `2` | Truthy string in a boolean column |

### File upload

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Rejects empty when marked | — |
| **Type allowed** | Rejects by content, not just extension | `.exe` renamed `.png` stored |
| **Size limit** | Rejects over the limit with the limit stated | 500 MB accepted; environment stalls |
| **Duplicate name** | States rename or reject | Silent overwrite of another record's file |

### Identifier — a name that acts as a key

(endpoint names, key names, tenant names)

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Required** | Rejects empty | — |
| **Allowed characters** | Restricts to a stated set; rejects separators and control characters | `name/with/slashes` breaks routing |
| **Minimum length** | Enforced with the limit stated | Single-character names |
| **Maximum length** | Enforced by the API too — the classic client-only gap | Typing stops; the API stored 256 |
| **Unique per tenant** | States the scope: per tenant, per parent, global | Collision across tenants |
| **Reserved names** | Rejects `admin`, `api`, `health`, and the framework's own `qa-` | Customer creates `admin` |
| **Case sensitivity** | States whether `Key` and `key` collide | Both exist; lookups pick one |

### Secret / credential (the generated API key)

System-generated, never typed — the rules are about handling:

| Rule | A well-built form… | Typical gap |
|---|---|---|
| **Shown once** | Reveals the full value exactly once, then masks | Full key still in the DOM after reload |
| **Copy works** | The copy affordance copies the real value | Copies the masked `sk_****` |
| **Never logged** | The value appears in no console log or network trace | Response body logged to the browser console |

Key lifecycle (create, store, revoke, expire) is `api-key-security`
K1–K7 — that skill's job. Here, a well-built form has the three above.

## 2. Universal enforcement standard

The same verdict for every rule, whatever its type:

| Verdict | Standard |
|---|---|
| `both` | Normal — record it |
| `server` | Acceptable. The UI could be friendlier — note it, not a finding |
| `client` | **Finding.** Anyone with curl bypasses the rule |
| `none` | **Finding.** The field claims a rule and nothing enforces it |
| Messages | User-safe: no stack traces, SQL, file paths, internal hosts |
| Any 5xx | Always a bug — never a rule, never a limit |

## 3. Message-quality standard

| Quality | What good looks like | What we raise |
|---|---|---|
| Names the field and the problem | `Key name must be at least 3 characters.` | — |
| Same rule, same wording everywhere | Identical string on every screen that states it | Finding — divergent wording is always reportable |
| No message at all | Silent rejection is not a message | Finding — a validation with no message |
| Generic catch-all | `Invalid input` cannot be asserted on and teaches the user nothing | `Q-` — wording is a product decision |
| Reveals that a value exists | Normal for internal tools; a user-enumeration tension on public surfaces | `Q-` + `ASSUMPTION — verify` |

## 4. Minimum probe matrix — which probes apply per type

The skill's eight probes, mapped to field types. A probe marked `no` is
recorded as skipped in the Probe log — skipped on purpose, not forgotten.

| Field type | empty | one char | long | wrong format | spaces | duplicate | unicode | injection |
|---|---|---|---|---|---|---|---|---|
| Text | yes | yes | yes | yes | yes | if **Uniqueness** | yes | yes |
| Long text | yes | yes | yes | yes | yes | if **Uniqueness** | yes | yes |
| Number | yes | yes | absurd values | `12abc`, `1e999` | yes | — | — | yes |
| Date/time | yes | — | — | invalid dates | — | — | — | — |
| Email | yes | yes | 320+ | yes | yes | if **Uniqueness** | yes | yes |
| URL/path | yes | — | yes | yes | yes | — | yes | `..`, `javascript:` |
| Select | placeholder | — | — | forged option | — | — | — | — |
| Checkbox | — | — | — | forged value | — | — | — | — |
| File upload | yes | — | oversize | wrong type | — | duplicate name | unicode filename | double extension |
| Identifier | yes | yes | yes | separators | yes | yes | yes | yes |
| Secret | — | — | — | — | — | — | — | — |

Quota probes (skill section "Count and quota limits") apply to
collections regardless of field type.

## 5. Product overlay — the endpoint-builder specifics

What THIS product's screens should have beyond the universal rules. The
baseline asserts **existence and enforcement, never the number** — the
value is discovered ("a max length exists and the API enforces it too"),
never assumed ("max 40").

### Endpoint creation

- Endpoint name: the identifier rules of section 1, scoped per tenant.
  Exact limits are per-product — `ASSUMPTION — verify` until probed.
- Path: starts with `/`, no trailing slash, no `//`, no query or
  fragment characters, unique among live endpoints.
  `Q-`: may a deleted endpoint's path be reused?
- At least one HTTP method must be selected.
  `Q-`: can an endpoint be edited after creation? (Scope table says OUT
  — confirm before hardening anything that assumes either answer.)

### Auto-generated Swagger docs

- A spec exists the moment the endpoint is created, reflects the real
  path and methods, and stops resolving after the endpoint is deleted.
- Deep spec validity is the `api-contract-check` skill's job —
  cross-reference, don't duplicate.

### API key dialog

- Key name: the identifier rules of section 1 — and **Maximum length**
  enforced by the API too.
- **Shown once**, **Copy works**, masked on reload (K1/K2 for the
  lifecycle).
- An expiry is offered and must be a future date.
  `ASSUMPTION — verify`: not every comparable product offers expiry.

### Quotas

- Keys per endpoint, and endpoints per tenant, each have a limit — and
  the rejection message states the number. (A rejection without a number
  is already a finding; see the skill's quota section.)

### Runtime (calling a generated endpoint)

- Missing, empty, wrong, and revoked keys each get a 401 with a
  user-safe message. The full matrix is `api-key-security` §2.
- Rate limiting returns 429, ideally with `Retry-After`.
  `ASSUMPTION — verify`.
- Cross-tenant access returns 403 or 404 per product decision — never
  hard-coded (CLAUDE.md scope table).

## Where this baseline comes from, and how it changes

Public behavior of Xano, Supabase, AWS API Gateway and Postman, plus the
OWASP API Security Top 10 as mapped in `api-key-security`. Reviewed by
the lead; the review date is recorded in this file's git history.

A discovery run never edits this file — gaps go to `Q-`. Changing the
standard itself (adding, weakening, or **renaming an item**) is the
lead's decision through a normal review; cited names are quoted verbatim
in discovery files, so a rename is a change to those files too.
