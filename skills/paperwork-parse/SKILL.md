---
name: paperwork-parse
description: |
  Turn any document or audio/video file into clean markdown plus structured JSON (audio becomes a transcript, optionally diarized) via the Paperwork CLI. Use when the user wants the content of a PDF, scan, office doc, or recording as text.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork parse

`POST /v1/parse` — parsing is the whole job: markdown + structured JSON out, nothing else runs.

## Quick start

```bash
# Digital-text PDF: waiting is fine — one call, result inline
paperwork parse parse --output inline \
  --json '{"file": {"url": "https://www.irs.gov/pub/irs-pdf/fw9.pdf"}}'

# Scan (OCR) or audio: fire with --wait 0, then poll
paperwork parse parse --wait 0 --json '{"file": {"id": "<fileId>", "processing": "transcribe_diarize"}}'
```

**Wait vs poll:** a digital-text PDF finishes inside the default 60s hold — just wait. Scans that need OCR and audio/video take minutes — fire with `--wait 0` and poll `paperwork runs get-run --id par_… --output inline --wait 0` every 10s until `status` leaves `processing` (loop in the `paperwork` skill).

## Output

- `output.markdownUrl` / `output.jsonUrl` — signed, ~1 hour.
- `--output inline` also returns `markdown` and `json` in the body when their combined size ≤ 1 MiB; larger falls back to URLs only.

**Keep the result out of your context.** Do not dump the markdown into the conversation. Write it to a local file (`curl -s "$markdownUrl" -o doc.md`, or redirect the inline body), then `grep`/`head` bounded slices for what you need. If the real goal is specific values, don't parse at all — `extract` with a schema returns just those fields.

## Choosing an engine — `file.processing`

| Value | Use for |
| --- | --- |
| `auto` | Default; self-upgrades to OCR on scans |
| `ocr` | Force OCR |
| `simple` | Text-layer only, fastest |
| `transcribe` | Audio/video |
| `transcribe_diarize` | Audio/video with speaker labels (default for audio/video) |

## Tips

- If you'll do anything else with the document later (extract, redact, fill), register it with `paperwork files create-file` first and parse by id — parse-once-run-many.
- Scanned documents and long audio can take minutes. Keep polling every 10s; don't escalate the interval.
- If you only need markdown text of a file you already own, `paperwork tools convert-file-to-markdown --ref <id>` is cheaper — no run, no storage.

## See also

- [paperwork-extract](../paperwork-extract/SKILL.md) — skip parse entirely when the goal is structured data
- [paperwork-files](../paperwork-files/SKILL.md) — registering files, upload flow, conversions
