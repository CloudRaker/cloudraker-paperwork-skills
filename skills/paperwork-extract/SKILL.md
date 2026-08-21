---
name: paperwork-extract
description: |
  JSON-Schema extraction with per-field citations over documents or audio via the Paperwork CLI — invoices, contracts, forms, transcripts to structured JSON with provenance. Use whenever the user wants specific fields or data out of files.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork extract

`POST /v1/extract` — grounded structured extraction. One file or many, schema-driven or inferred.

## Quick start

```bash
paperwork extract extract --json '{
  "file": {"url": "https://www.irs.gov/pub/irs-pdf/fw9.pdf"},
  "schema": {
    "type": "object",
    "properties": {
      "business_name": {"type": ["string", "null"]},
      "tax_classification": {"type": ["string", "null"]}
    }
  },
  "citations": true
}'
```

**Wait vs poll:** for a digital-text PDF, waiting (the default 60s hold) is fine — one call, done. For scans, audio, or multi-file runs, add `--wait 0` and poll `paperwork runs get-run --id exr_… --wait 0` every 10s until terminal (loop in the `paperwork` skill).

**This is the context-saver.** Extract returns just the fields you asked for, with citations — never parse a document and read its text to find values yourself.

## Three ways to shape the output (send at most one)

- **`schema`** — inline JSON Schema in the extraction dialect. Make leaf types nullable (`["string","null"]`) so absent values come back `null`, not hallucinated.
- **`action`** — a saved extract config by id or slug (`paperwork configs create-extract-config`). Inline fields (`instructions`, `unit`, `judge`, …) deep-merge over it, inline wins.
- **neither** — schema is inferred from the documents; steer with `hints` in prose. The finished run reports the inferred shape as `config.schema` — save it as a config for the next run.

## Reading the result

- `output.value` — single document; `output.documents[]` — one entry per file.
- `citations: true` → `output.citations` maps each field to `fileId` + page + bbox (or timecode for audio), or `notFound: true`. Always request citations when correctness matters — a cited value is checkable, an uncited one is a guess.
- `--output inline` adds each source's parsed `markdown`/`json` (≤ 1 MiB combined, else URLs only).

## Batch: same shape over up to 100 files

```bash
# 1. Save the shape once
paperwork configs create-extract-config --json '{"name": "invoice-fields", "config": {"schema": {...}}}'

# 2. Fan out — always async, 202 immediately, one run id per file
paperwork extract extract-batch --json '{
  "action": "invoice-fields",
  "files": [{"id": "…"}, {"id": "…"}],
  "metadata": {"batch": "2026-08-invoices"}
}'
```

There is no batch object — poll each run, or list them together: `paperwork runs list-runs --object extract_run --params '{"metadata.batch": "2026-08-invoices"}' --page-all`. A saved `action` is required for batch; inline schemas are single-call only. The whole batch costs one request against your rate limit.

## Tips

- Register files first and extract by id — re-runs with a tweaked schema then skip parsing entirely.
- `status: "needs_input"` means a human task is parked on the run — surface `tasks[].url`, don't poll-spin.
- Per-file result inside a batch is independent: one `{file, error}` entry doesn't fail the rest.

## See also

- [paperwork-runs](../paperwork-runs/SKILL.md) — polling, metadata filters, keep
- [paperwork-pipeline](../paperwork-pipeline/SKILL.md) — extract + redact/fill/sign over one file set
