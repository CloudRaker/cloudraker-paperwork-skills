---
name: paperwork-fill
description: |
  Fill a form PDF via the Paperwork CLI — deterministically from a JSON `values` object, or LLM-drafted from source documents. Use for W-9s, intake forms, applications — any "complete this form" task. Also covers configure-once field curation (inspect + fill configs) and the template library.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork fill

`POST /v1/fill` — fills a form template and returns the completed PDF. Two modes, send **exactly one** of:

- **`values`** — `Record<string, unknown>` keyed by field name. Deterministic, no LLM, fast. Unknown keys → `400 unknown_fields` with the offending list.
- **`files`** — source documents; the LLM drafts one value per field.

## Configure once, fill many

The intended shape: inspect the template once, curate the fields, save them on a fill config, then every fill (values or files) reuses the curated schema. Fill runs never infer labels — inspect at configure time is the only labeling pass.

1. **Register the blank form as a template** (reusable, never parsed):
   ```bash
   paperwork templates create-template --json '{"url": "https://www.irs.gov/pub/irs-pdf/fw9.pdf", "name": "w9.pdf"}'
   ```
2. **Inspect it** — typed response `{ fields, schema, pageBoxes, pageCount, detected, templateHash }`. Each field entry: `{ name, type, label, description?, required?, options?, page, box {x,y,width,height}, ignore? }`. Templates over 32 MB → `413 template_too_large`:
   ```bash
   paperwork templates inspect-template --id <templateId>
   ```
3. **Save a fill config** — curate the inspected fields (fix labels, add descriptions, set `ignore` on fields to skip) and store them with the `templateHash` so staleness is detectable:
   ```bash
   paperwork configs create-fill-config --json '{
     "name": "w9",
     "config": {
       "template": "<templateId>",
       "output": "flattened",
       "fields": [ ...curated entries from inspect... ],
       "templateHash": "<templateHash from inspect>"
     }
   }'
   ```
   Config keys: `template` (a bare template or file id **string**), `instructions`, `output` (`flattened` | `editable`), `fields`, `templateHash`. Unknown keys → 400. PATCH deep-merges — patching `output` leaves `fields` intact.
4. **Fill** — with a saved config, `template` is optional (the saved one applies; an inline `template` wins on conflict):
   ```bash
   # Deterministic: your JSON in, filled PDF out, no LLM
   paperwork fill fill --json '{"action": "w9", "values": {"f1_01[0]": "Acme Corp"}}'

   # Drafted from sources
   paperwork fill fill --json '{"action": "w9", "files": [{"id": "<sourceDocId>"}]}'
   ```

One-off without a config still works: pass `template` (`{"id": …}`, `{"url": …}` — fetched fresh, purged with the run — or any file you own) plus `values` or `files`. With no curated `fields`, the raw field inventory is used as-is.

**Two `template` shapes — the most common 400.** Inline on `POST /v1/fill` it is an **object** (`{"id": …}` or `{"url": …}`). On a saved fill config it is a **bare id string**. Sending the object to `configs create-fill-config` is rejected.

**Wait vs poll:** `values` mode and PDF sources — waiting (default 60s hold) is fine. Scans/audio sources or many files — add `--wait 0` and poll `runs get-run --id flr_…` every 10s.

## Tips

- `values` mode is the API-integration path: map your data to field names once, then every fill is deterministic and auditable.
- Field names are exact PDF field names (e.g. `topmostSubform[0].Page1[0].f1_01[0]`) — take them from inspect or the saved config's `fields`, never guess. In `values` mode unknown names fail loudly (`unknown_fields`).
- If inspect returns a different `templateHash` than the saved config's, the template changed — re-inspect and re-curate before filling.
- Templates reach `ready` the moment bytes land (no parsing stage) — no need to poll them like files.
- Never open the filled PDF to check it — read `output.fields` from `runs get-run --id flr_… --query output.fields`, or render one page (`tools render-file-page`) if you must look.

## See also

- [paperwork-sign](../paperwork-sign/SKILL.md) — send the filled PDF for signature next
- [paperwork-runs](../paperwork-runs/SKILL.md) — polling, outputs, TTL
