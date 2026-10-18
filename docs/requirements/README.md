# docs/requirements — put requirement material here (optional)

Everything in this folder is read by `/requirements:analyze-requirements`.

**This folder may stay empty.** The application already exists, so the
framework learns how it behaves by travelling it
(`/discovery:explore-app` → `docs/discovery/`). Requirement documents
are the other half of the picture: what the product was *supposed* to
do. When both exist, the test planner compares them and every
disagreement becomes a question or a suspected bug — which is how most
real defects are found.

With no documents at all, expected results rest on approved observation
alone. That works, but it means a behaviour nobody questions becomes
"correct" by default, so anything you can add here is worth adding —
even a one-page email.

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
