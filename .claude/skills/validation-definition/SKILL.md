---
name: validation-definition
description: Probe every input field in a discovered UI workflow and record its validation rules — required, length, format, uniqueness, allowed values — with the real error message, whether the rule is enforced on the client, the server, or neither. Use when mapping field validations, input rules, or producing a validation definition.
---

# Validation Definition — what each field accepts and rejects

**Goal:** know every rule on every field, in the product's own words, and
know which rules the server actually enforces. These become the negative
and boundary test cases, and the data factory's rules.

**Input:** approved `docs/discovery/sequences/SEQ-*.md` plus the field
dumps in `automation/discovery/artifacts/`.
**Output:** `docs/discovery/validations/<module>.validations.md`
(template: [templates/validations-template.md](templates/validations-template.md)).

**You have to submit bad data to learn a rule.** That is allowed
(CLAUDE.md section 4) under `qa-` naming, on the test environment, with
cleanup — and only on forms inside the workflow you were asked to cover.

## 1. Read the DOM first — it is free

Before typing anything, record what the page already declares for each
field. Half the rules are usually sitting there:

| Source | Gives you |
|---|---|
| `required`, `aria-required` | Required |
| `maxlength`, `minlength` | Length limits |
| `type` (`email`, `number`, `url`) | Format |
| `pattern` | Regex rule |
| `min`, `max`, `step` | Numeric range |
| `disabled`, `readonly` | Not editable — and a question if it should be |
| Placeholder, helper text, label | The rule as the product words it |

Declared is not enforced. A `maxlength` on the input only stops typing —
the API may still accept 500 characters when called directly. That gap is
exactly what step 3 is for.

## 2. Probe each field

For every field, in this order, submitting between each:

| # | Probe | Learns |
|---|---|---|
| 1 | Leave empty, submit | Required? What is the message? |
| 2 | One character | Minimum length |
| 3 | A long string (`{repeat:k:256}`) | Maximum length — and whether it truncates silently |
| 4 | Wrong format for the type | Format rule and its message |
| 5 | Leading/trailing spaces | Trimmed, rejected, or stored as-is |
| 6 | A value that already exists | Uniqueness, and the message |
| 7 | Unicode and emoji (`qa-日本-🔑`) | Encoding handling |
| 8 | One injection string from `api-key-security` §4 | Rejected cleanly, stored as text — **never** executed, never 5xx |

Record the exact message, character for character, and where it appears
(under the field, in a toast, at the top). Tests assert on these strings,
so "Name is required." and "Name is required" are different answers.

Stop probing a field after two unhandled 5xx responses — you have found
the bug; keep going and you are just making noise.

## 3. Client, server, or neither — the check that matters

For each rule, decide where it lives:

1. Try it in the UI. Blocked before any request → **client**.
2. Send the same bad value straight to the API route from the event
   definition, with the same session. Rejected → **server**.
3. Accepted by the API → **client only**.

| Verdict | What it means | Action |
|---|---|---|
| `both` | Normal and good | Record it |
| `server` | Fine. UI could be friendlier | Note it |
| `client` | **Finding.** Anyone calling the API bypasses it | Raise a suspected bug — this is how bad data gets in |
| `none` | Field claims a rule and nothing enforces it | Raise a suspected bug |

This one column is the highest-value output of the whole step. A
`client`-only rule on a required or length-limited field is a real defect
that no amount of UI testing would ever find.

## 4. Cross-field and form-level rules

Fields are not independent. Also record:

- rules that only fire in combination ("end date must be after start date")
- a submit button that stays disabled until something is true — record
  **what** makes it enable
- rules that depend on state (a name unique per tenant, not globally)
- what happens on double submit — two records, or one?

## 5. Write it down

One row per rule, not per field — a field with three rules gets three
`VAL-` IDs, because each becomes its own test case or data row.

| ID | Screen | Field | Rule | Trigger | Message (exact) | Enforced | Evidence |
|---|---|---|---|---|---|---|---|
| VAL-007 | generate-key-dialog | Key name | required | on submit | `Key name is required.` | both | [png](../screens/VAL-007.png) |
| VAL-008 | generate-key-dialog | Key name | max length 40 | on type (maxlength) | — (typing blocked) | client | [png](../screens/VAL-008.png) |

`Trigger` is `on type`, `on blur`, `on submit`, or `server response` —
the UI generator needs it to know when to assert.

An unknown limit is written `UNKNOWN — probe inconclusive` with a `Q-`,
never a guessed number.

## 6. Clean up

Probing creates records — especially the uniqueness probe. Delete every
`qa-` record you made, report anything left behind, and never touch a
record that is not yours.

## Rules

- `status: draft` until approved. `VAL-` IDs are global and permanent.
- The message is quoted exactly, including the full stop.
- Never record a rule you did not trigger. "Probably 50 characters" is a
  question, not a validation.
- A 5xx from any probe is a bug report, not a validation rule.
- Finish with: rules found per field, how many are `client`-only or
  `none` (list them — these are findings), unresolved limits, and
  "Ready for review."
