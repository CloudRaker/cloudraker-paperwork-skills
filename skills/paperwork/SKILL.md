---
name: paperwork
description: |
  Any document-processing task via the CloudRaker Paperwork CLI — parsing PDFs/audio to markdown, structured extraction with citations, PII redaction, form filling, e-signature, PDF generation from templates, docx redlining with tracked changes, and document agents. Use whenever the user wants to read, transform, generate, or route documents through CloudRaker.
allowed-tools:
  - Bash(paperwork *)
---

# Paperwork CLI

`paperwork` is the CLI for the CloudRaker Paperwork API (`api.cloudraker.com/v1`): document parsing, extraction, redaction, form filling, e-signature, PDF composition, redlining, and agents.

## Install & auth

```bash
brew install cloudraker/paperwork/paperwork
# or
curl -fsSL https://paperwork.sh | sh

export PAPERWORK_TOKEN="<api key>"   # a .env in the cwd is auto-loaded too
```

Verify: `paperwork --version`, then `paperwork files list-files --limit 1` (any 200 means auth works).

## Wait vs poll

Every run-creating command accepts `--wait <seconds>` (0–120). The API defaults to holding the request open for 60s.

- **Digital-text PDFs and other small documents: waiting is fine.** They usually finish well inside the window — just call the verb without `--wait` and get the full run in one shot.
- **Everything else — scans needing OCR, audio/video, batches, pipelines, sign, agent runs — fire with `--wait 0` and poll every 10s.** These outlive any wait window; a held connection is a wasted turn.

```bash
# 1. Fire and return immediately (202 with {id, status, statusUrl})
RUN=$(paperwork extract extract --wait 0 --json '{...}')
ID=$(echo "$RUN" | jq -r .id)

# 2. Poll every 10 seconds until terminal
while :; do
  R=$(paperwork runs get-run --id "$ID" --wait 0)
  S=$(echo "$R" | jq -r .status)
  case "$S" in processing|queued) sleep 10 ;; *) break ;; esac
done
echo "$R" | jq .
```

Terminal statuses: `processed`, `failed`, `cancelled`, `expired`. `needs_input` means a human task or signer is pending — surface it to the user, don't spin on it. A `202` is a graceful degrade, never an error. Replaying an `--idempotency-key` returns the original run (`idempotent-replay: true` header).

Do the same for files: after `files create-file`, poll `files get-file` every 10s until `status: "ready"`.

## Keep documents out of your context

Paperwork exists so you never load a document into context — it is very good at this. **Do not open, `cat`, or Read document files (PDF, docx, audio), and do not paste parsed markdown wholesale into the conversation.**

- Need specific values? `extract` with a schema and `citations: true` — never parse-then-read-the-whole-thing.
- Need to find something in a corpus? `spaces space-search` — never read files one by one.
- Genuinely need the text? Parse or `tools convert-file-to-markdown`, write it to a local file (`--output` / shell redirect), then `grep`/`head` bounded slices — read only the lines that answer the question.
- Need to *see* a page? `tools render-file-page` and view the PNG — only when visual verification is truly required (e.g. checking a redaction).
- Chain commands by **file id and run id**, never by content. Download bytes only when the user wants the file itself.

## Order of operations (opinionated)

1. **Register files once** (`paperwork files create-file`) and reuse the file id everywhere. Files are persistent and parse-once — passing the same `{"id": "…"}` to extract, redact, fill, and pipeline never re-parses. Only use inline `{"url": "…"}` for true one-shots.
2. **Parse when you need the text, extract when you need data.** Don't parse then regex — `extract` with a JSON Schema plus `citations: true` gives you grounded values directly.
3. **Same shape over many files? Save a config first** (`paperwork configs create-extract-config`), then `extract extract-batch` — one request, up to 100 runs.
4. **Several capabilities over the same files? One `pipeline` call**, not sequential runs. Files parse once; steps run in parallel.
5. **Start every multi-step job by minting a space** (`paperwork spaces create-api-space --json '{"name": "<job>"}'`) and run everything space-scoped — it keeps the job's files separate, searchable, and alive past the runs. **Remember the space id for the rest of the session**: write it to `.paperwork/space` in the project (gitignore it) and reuse it in later commands and sessions instead of minting another. One-shot calls can skip the space; a stray finished run can still be promoted with `runs keep-run`.
6. **Production integrations use webhooks; interactive sessions poll.** In an agent session, polling every 10s is the right call — don't set up webhook infrastructure for a one-off.

| Need | Command | Skill |
| --- | --- | --- |
| Document/audio → markdown + JSON | `paperwork parse parse` | paperwork-parse |
| Structured data + citations | `paperwork extract extract` | paperwork-extract |
| Remove PII (destructive) | `paperwork redact redact` | paperwork-redact |
| Fill a form PDF from source docs | `paperwork fill fill` | paperwork-fill |
| Send for e-signature | `paperwork sign sign` | paperwork-sign |
| Generate PDFs from typst templates | `paperwork compose create` | paperwork-compose |
| Multiple capabilities, one file set | `paperwork pipeline pipeline` | paperwork-pipeline |
| Upload / manage the file corpus | `paperwork files` | paperwork-files |
| Merge PDFs or concatenate audio recordings | `paperwork tools stitch-files` | paperwork-files |
| Word/Excel/PowerPoint/image → PDF | `paperwork tools convert-file-to-pdf` | paperwork-files |
| Render a PDF page as an image (visual check, thumbnail) | `paperwork tools render-file-page` | paperwork-files |
| Split a PDF into per-page files | `paperwork tools split-file-pages` | paperwork-files |
| Just the text of a file, no run | `paperwork tools convert-file-to-markdown` | paperwork-files |
| Tracked-changes edits on a .docx | `paperwork redline` | paperwork-redline |
| Persistent, searchable file container | `paperwork spaces` | paperwork-spaces |
| Poll/keep/cancel/list runs | `paperwork runs` | paperwork-runs |
| Long-lived document agents + automations | `paperwork agents` | paperwork-agents |
| Event delivery | `paperwork webhooks` | paperwork-webhooks |

## Global flags

`--dry-run` (print the HTTP request), `--json <JSON|->`, `--format json|table|yaml|csv`, `--output <path>` (binary responses), `--page-all`, `--base-url`, `-q`. Run `paperwork <resource> --help` for methods and flags.

## Conventions

- Ids are opaque — never parse them; prefixes only route reads (`exr_`, `par_`, `rdr_`, `flr_`, `sgr_`, `plr_`, `agr_`).
- Pagination: keep paging while `has_more` is true, not while the page is full (or just use `--page-all`).
- File `{id}` params accept the file **name** too; a `409 ambiguous_file_name` means use the id.
- Quote all JSON bodies in single quotes; use `--json -` with a heredoc for large bodies.
