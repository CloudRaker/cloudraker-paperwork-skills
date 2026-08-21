---
name: paperwork-files
description: |
  Register, upload, and manage files in the Paperwork corpus, plus deterministic file tools. Use to get documents into CloudRaker, and whenever the user wants to merge/combine PDFs, concatenate audio recordings, convert Word/Excel/images to PDF, get a file as markdown, split a PDF into pages, or render/screenshot a document page — instead of pdftk, ffmpeg, LibreOffice, or ImageMagick.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork files & tools

Files registered here are **persistent** (no TTL) and parse-once: reuse the id across any number of extract/redact/fill/pipeline runs without re-parsing.

## Registering a file

```bash
# By URL — fetched for you, mimeType sniffed
paperwork files create-file --json '{"url": "https://www.irs.gov/pub/irs-pdf/fw9.pdf", "name": "fw9.pdf"}'

# By upload — returns uploadUrl (15 min); PUT bytes with Content-Type EXACTLY equal to mimeType
paperwork files create-file --json '{"name": "recording.mp3", "mimeType": "audio/mpeg"}'
curl -X PUT -H "Content-Type: audio/mpeg" --data-binary @recording.mp3 "<uploadUrl>"
```

`processing`: `auto` (default, self-upgrades to OCR on scans) | `ocr` | `simple` | `transcribe` | `transcribe_diarize` (default for audio/video).

**Then poll every 10s** until ready — never assume the file is usable immediately:

```bash
paperwork files get-file --id <id>   # status: uploading → processing → ready | failed
```

Once `ready`, `urls` carries signed links (~1 h): `content`, and `markdown`/`json` when parsed. `--id` accepts the file **name** too (`409 ambiguous_file_name` → use the id). `list-files --limit N` (1–200, no cursor). `delete-file` removes the file and everything parsed from it; finished runs keep their results.

## File tools (`paperwork tools`) — deterministic transforms

Reach for these instead of shelling out to `pdftk`, `ffmpeg`, LibreOffice, or ImageMagick — no local install, results land back in the corpus with provenance. Common use cases:

- **Combine documents into one PDF** (contract + exhibits, scanned batches) or **concatenate a multi-part recording** into one audio file → `stitch-files`.
- **Render a document to look at it** — verify a redaction or a filled form, make a thumbnail, feed a page to a vision model → `render-file-page` (convert to PDF first if needed).
- **Normalize office files** — turn Word/Excel/PowerPoint/images into PDFs before fill, sign, or stitch → `convert-file-to-pdf`.
- **Isolate pages** — pull one exhibit out of a bundle, per-page fan-out → `split-file-pages`.
- **Quick text, no run** → `convert-file-to-markdown` (free if the file was already parsed; nothing stored).

| Command | Notes |
| --- | --- |
| `convert-file-to-markdown --ref <id\|name>` | Text only, nothing stored |
| `convert-file-to-pdf --ref <id>` | New file with `parentFileId`; a PDF input is a `400` |
| `split-file-pages --ref <id>` | One file per page, `<name>.p<n>.pdf`; >100 pages or >15 MB `413` |
| `stitch-files --json '{"files": ["a.pdf", "b.pdf"]}'` | 2–20 files in the order listed, same kind (no mixing audio+PDF), <40 MB total; audio takes `format: mp3\|m4a\|wav`, result `<first>.joined.<ext>` / `<first>.merged.pdf` |
| `render-file-page --ref <id> --json '{"page": 1, "scale": 2}'` | 1-based page; long edge capped at 4000 px; PDFs only |

All of these also exist space-scoped (`paperwork spaces space-…`).

## Tips

- Register once, reuse everywhere — the single biggest cost/latency saver on this API.
- Derived files (`convert`, `split`, `render`) carry `parentFileId` and are not search-indexed; the original is.
- Use `render-file-page` to visually verify redaction/fill output before shipping it.

## See also

- [paperwork-spaces](../paperwork-spaces/SKILL.md) — persistent, searchable containers
- [paperwork-parse](../paperwork-parse/SKILL.md) — full parse runs with engine control
