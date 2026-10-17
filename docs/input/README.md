# docs/input — put requirement material here

Everything in this folder is read by `/requirements:analyze-requirements`.

## What to add

| Material | Format that works | Name it like |
|---|---|---|
| PRD / FRD | PDF, Markdown (.md), plain text (.txt) | `prd-v1.2.pdf` |
| Mockups / UI screens | PNG or JPG — one screen per image | `mockup-01-create-endpoint.png` |
| Client notes, emails, meeting notes | .md or .txt (copy-paste is fine) | `client-notes-2026-10-17.md` |
| Swagger / API spec (if available) | .json or .yaml | `platform-api.json` |
| Answers to our questions | `answers.md` (template below) | `answers.md` |

Word (.docx) and Excel (.xlsx) files: export to PDF first.

## Tips

- Put a version or date in each file name — the analysis records it.
- Name mockups in flow order (01, 02, ...) so the screens read as a journey.
- Remove real customer data before adding anything.

## answers.md template

When product answers the questions from the analysis, record them here
and run `/change:impact-analysis answers` (not the full analysis
again). Bigger changes from product go in `changes/` — see
`changes/README.md`.

```markdown
# Answers from product

| Q ID | Answer | Answered by | Date |
|---|---|---|---|
| Q-001 | Cross-tenant access returns 404 | <name>, Product | 2026-10-21 |
```
