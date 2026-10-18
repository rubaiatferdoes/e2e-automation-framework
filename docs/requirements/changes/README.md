# docs/requirements/changes — updates from the product team

One file per change: `CR-<nnn>-<short-name>.md` (next free number).
Then run `/change:impact-analysis docs/requirements/changes/CR-<nnn>-<short-name>.md`.

Only written changes count — paste the email or chat message, don't
paraphrase a hallway conversation.

## Changes come from two directions

| Source | What it looks like | What to do |
|---|---|---|
| **Product tells us** | An email, a chat message, a new PRD version, an answer to one of our `Q-` questions | Write a CR file here (template below), then run the impact analysis |
| **The application changed** | A screen was redesigned, a field gained a rule, a route moved — usually found when tests start failing | Re-run discovery for that part first, approve it, *then* run the impact analysis |

The second one has no email to paste, so the discovery diff is the
record: `/discovery:explore-app seq:create-endpoint` updates the
sequence in place and reports what changed since the last walk. That
report is what the impact analysis traces.

```
# the app changed under us
/discovery:explore-app seq:create-endpoint   → approve → /change:impact-analysis
```

A CR file is still worth writing when someone can tell you *why* it
changed — "we renamed the field in build 412" turns an unexplained diff
into a decision with an owner.

## What the impact analysis will trace

Give it the IDs you already know and it finds the rest, down the chain:

```
SEQ- / EVT- / VAL-  →  TC-  →  test data  →  automated tests  →  walk files
```

It writes `docs/changes/CR-<nnn>-impact.md`, waits for your approval, and
only then edits anything. Nothing is renumbered; removed items are marked
`DEPRECATED`, never deleted.

## Template

```markdown
# CR-001: <short title>

| From | Date | Channel |
|---|---|---|
| <name>, Product | 2026-10-21 | email / meeting / chat |

## What changes
<paste the email/chat, or describe the new behaviour in plain words>

## Affects (if known)
<screen, workflow, or SEQ-/VAL-/EVT-/Q-/BR- ID>
```

## Answers to our questions

Discovery and the analysis both raise `Q-` questions. When product
answers one, it goes in `../answers.md` rather than here:

```
/change:impact-analysis answers
```

An answered question often turns an `ASSUMPTION — verify` in a test case
into a real expected result, which is usually a small, well-scoped
change — exactly what the impact analysis is for.
