---
name: paperwork-fill
description: |
  Fill a form PDF's fields from source documents via the Paperwork CLI, with optional human review before finalizing. Use for W-9s, intake forms, applications — any "take these docs and complete this form" task. Also covers the template library and field inspection.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork fill

`POST /v1/fill` — drafts field values from source docs, returns the completed PDF.

## Order of operations

1. **Register the blank form as a template** (reusable, never parsed):
   ```bash
   paperwork templates create-template --json '{"url": "https://www.irs.gov/pub/irs-pdf/fw9.pdf", "name": "w9.pdf"}'
   ```
2. **(Optional) inspect its fields** — names, types, page geometry. First call can be slow; later calls are cached:
   ```bash
   paperwork templates inspect-template --id <templateId>
   ```
3. **Fill from source docs** — for PDF sources, waiting (default 60s hold) is fine; for scans/audio sources or many files, add `--wait 0` and poll `runs get-run --id flr_…` every 10s:
   ```bash
   paperwork fill fill --json '{
     "files": [{"id": "<sourceDocId>"}],
     "template": {"id": "<templateId>"},
     "output": "flattened"
   }'
   ```

`template` also accepts `{"url": "…"}` (fetched fresh, purged with the run) or any file you own. `output`: `flattened` (default, values baked in) or `editable` (form stays fillable).

## Human review

`"review": "required"` parks the run at `status: "needs_input"` with one task per document instead of finishing. Then either:

- Send a person to the ready-made page at `tasks[].url` on the run — the zero-code path; or
- Build your own review: `paperwork runs get-run-task --id <runId>` (drafted values + field inventory + page geometry; `get-run-task-pdf` streams the measured PDF), then submit with `paperwork runs submit-run-task --id <runId>` sending `{"values": {"<fieldName>": "…"}}` using names from the task/inspect payload.

A `needs_input` run is waiting on a human — tell the user, don't poll-spin on it.

## Tips

- Repeated form+shape → save a fill config (`paperwork configs create-fill-config`) and pass `action: "<slug>"`.
- Templates reach `ready` the moment bytes land (no parsing stage) — no need to poll them like files.
- Never open the filled PDF to check it — read the drafted values from `runs get-run-task`, or render one page (`tools render-file-page`) if you must look.
- Field names are exact PDF field names (e.g. `topmostSubform[0].Page1[0].f1_01[0]`); values matching no field are silently discarded.

## See also

- [paperwork-sign](../paperwork-sign/SKILL.md) — send the filled PDF for signature next
- [paperwork-runs](../paperwork-runs/SKILL.md) — task endpoints, polling
