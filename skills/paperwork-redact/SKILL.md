---
name: paperwork-redact
description: |
  Destructively remove PII from documents or audio via the Paperwork CLI — text stripped from the PDF content stream (not a black box), audio beeped/silenced with the transcript rewritten. Use for anonymization, GDPR/privacy scrubbing, or sharing sanitized files.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork redact

`POST /v1/redact` — real removal, not cosmetic. Returns a new file; the original is untouched.

## Quick start

```bash
paperwork redact redact --json '{
  "file": {"id": "<fileId>"},
  "categories": ["ssn", "ein"],
  "mode": "targeted"
}'
```

**Wait vs poll:** a digital-text PDF usually finishes inside the default 60s hold — waiting is fine. For scans and audio/video, add `--wait 0` and poll `paperwork runs get-run --id rdr_… --wait 0` every 10s until terminal.

## Parameters route by media type — not interchangeable

- **Documents** — `mode`: `targeted` (default, per-entity boxes) or `lines` (whole lines).
- **Audio/video** — `style`: `beep` or `silence`.

Sending the wrong medium's parameter is a `400`. When saving a redact config, pass `mimeType` so the right variant is chosen.

## Choosing what to remove

- `categories` — defaults cover names, government ids, addresses, phones, emails, dates of birth.
- `instructions` — free-form guidance layered on top ("also remove account numbers in the footer").

## Verify before shipping

The result reports `output.entities` (per-category removal counts) and `output.skipped` (documents with nothing to redact). **Check the counts** — zero entities on a document you expected to be dirty is a signal to tighten `categories`/`instructions`, and render a page (`paperwork tools render-file-page`) to eyeball the output when stakes are high.

## Tips

- Redaction is destructive and produces a new file with `parentFileId` back to the source — keep the original registered if you may need it.
- Repeated shapes → save a config (`paperwork configs create-redact-config`) and pass `action: "<slug>"`.

## See also

- [paperwork-pipeline](../paperwork-pipeline/SKILL.md) — extract from the original while redacting it, one call
- [paperwork-files](../paperwork-files/SKILL.md) — rendering pages to verify output
