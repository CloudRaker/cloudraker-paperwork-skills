---
name: paperwork-compose
description: |
  Generate PDFs from saved typst templates with schema-validated data via the Paperwork CLI — invoices, contracts, letters, reports; single render or batch. Also covers authoring template bundles (files, lint, preview). Use for any "generate a document/PDF from data" task.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork compose

`POST /v1/compose` — render a saved template with data. Data is validated against the template's JSON Schema **before** anything renders, so a missing field is a `422` with exact instance paths, never a broken document.

## Render

```bash
# One document — synchronous, no run to poll
paperwork compose create --json '{"template": "invoice", "data": {"customer": "Acme", "total": 1240}}'

# Raw bytes straight to disk, nothing stored
paperwork compose create --output invoice.pdf --json '{"template": "invoice", "data": {...}, "output": "raw"}'

# Batch — one call, one PDF per item (or "output": "zip" for one archive)
paperwork compose batch --json '{
  "template": "invoice",
  "items": [
    {"name": "acme-0042", "data": {"customer": "Acme", "total": 1240}},
    {"name": "globex-0043", "data": {"customer": "Globex", "total": 990}}
  ]
}'
```

Batch is all-or-nothing: if any item fails validation, nothing renders (`422` with failing indexes). Item `name` must be a bare file name, unique within the batch.

## Authoring a template

1. **Create the config** — holds the JSON Schema, bundle files, `main` entry point, named examples:
   ```bash
   paperwork configs create-compose-config --json '{"name": "invoice", "config": {...}}'
   ```
2. **Add files to the bundle** (`put-compose-template-file`): `{name, content}` for text sources (≤ 2 MB), `{name, mimeType}` for binary assets → `PUT` the bytes to the returned `uploadUrl` with exactly that `Content-Type`.
3. **Lint** — compile diagnostics only, nothing rendered or stored:
   ```bash
   paperwork compose lint-compose-template --id-or-slug invoice
   ```
   Empty `diagnostics` with no `missing` assets / `unresolved` fonts means it renders.
4. **Preview** — render with a saved example (or `{}` for a fresh template):
   ```bash
   paperwork compose preview-compose-template --id-or-slug invoice --output preview.pdf --json '{}'
   ```

**Always lint → preview → render.** Edit-loop with `list-compose-template-files` / `get-compose-template-file` to read sources back.

## Tips

- Removing the `main` file answers `422` unless the same call names a new entry point with `--main`.
- Replaced bundle files are detached, never deleted — past renders keep working; the run records `templateHash`.
- Compose also exists space-scoped: `paperwork spaces space-compose` stores the PDF in that space.

## See also

- [paperwork-spaces](../paperwork-spaces/SKILL.md) — compose into a persistent, searchable space
