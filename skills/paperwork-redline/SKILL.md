---
name: paperwork-redline
description: |
  Edit a .docx with tracked changes via the Paperwork CLI — session-based redlining: replace/delete/insert text, accept or reject suggestions, export the redlined document. Use for contract markup, negotiated edits, or any Word tracked-changes work. Also covers what to do about Word comments (the API counts them but cannot read or write them).
allowed-tools:
  - Bash(paperwork *)
---

# paperwork redline

Session-based tracked-changes editing on Word documents. Only `.docx` — anything else answers `422`.

Every command and output below was run against a real negotiated contract. Follow the sequence exactly; the two known failure modes and their workarounds are documented inline — do not abandon on the first 400.

## The session loop (verified end to end)

```bash
# 0. Get the file in (skip if already registered)
paperwork files upload ./msa.docx --name msa.docx --wait
# → {"id": "72494de3-…", "status": "ready", …}

# 1. Open a session (ttl 60–604800s). File NAME works as the ref.
paperwork tools redline-file --ref msa.docx --json '{"ttl": 86400}'
# → {"id": "rdl_01M0JY…", "revision": 0, "fileId": "72494de3-…"}

# 2. Inspect — the summary tells you what you are dealing with
paperwork redline get-redline-session --session-id $SID
# → "summary": {"comments": 38, "paragraphCount": 114, "trackedChanges": 22}

# 3. Read the document as anchored blocks (p0, p1, …)
paperwork redline get-redline-content --session-id $SID --view accepted

# 4. Apply edits — see the KNOWN BUG below before running this
# 5. Accept/reject suggestions — see "Suggestions" below
# 6. Export — session stays open; flush is repeatable (checkpoint long jobs)
paperwork redline flush-redline-session --session-id $SID --json '{"fileId": "msa.docx"}'
# → new file "msa.redlined.docx" with parentFileId; the original is untouched
```

Nothing is stored until flush. `fileId` on flush accepts the original file's name or id; anything else is `409 redline_file_mismatch`.

## Edit operations

`replace` and `deleteText` act on `anchorText` (exact text as it appears in a content block); `fill` applies a list of replacements in one pass; `insertMarkdown` adds paragraphs before/after an anchor or at the end. `tracked: false` writes silently. `?view` on content: `accepted` (default) | `baseline` | `current`.

A successful edit answers with what it did — use `blockIds` and `trackedChangeIds` to verify placement:

```json
{"revision": 1,
 "result": {"op": "replace", "replacements": 1, "blockIds": ["p3"], "trackedChangeIds": ["tc0", "tc1"]},
 "summary": {"paragraphCount": 114, "comments": 38, "trackedChanges": 24}}
```

A tracked `replace` creates TWO changes: a `del` of the old text and an `ins` of the new.

### KNOWN BUG (CLI ≤0.0.1): `apply-redline-edits` rejects every documented body

The CLI's embedded schema for the edit body collapsed to `{"op"}` only, so this **fails locally** before any request is sent:

```bash
paperwork redline apply-redline-edits --session-id $SID \
  --json '{"op":"replace","anchorText":"…","replacement":"…"}'
# → 400 validationError: anchorText: Unknown property. Valid properties: ["op"]
```

Do NOT retry variations: `--params` puts the extra fields in the query string (server 400s), and there is no flag to skip local validation. **The server itself is fine.** Workaround — send the exact same body with curl:

```bash
curl -sf -X POST "https://api.cloudraker.com/v1/redline/$SID/edits" \
  -H "authorization: Bearer $PAPERWORK_TOKEN" -H "content-type: application/json" \
  -d '{"op":"replace","anchorText":"thirty (30) days","replacement":"sixty (60) days"}'
```

Same body shapes for the other ops:

```json
{"op": "deleteText", "anchorText": "…exact text…"}
{"op": "fill", "replacements": [{"anchorText": "…", "replacement": "…"}, …]}
{"op": "insertMarkdown", "markdown": "**New clause.** …", "position": "end"}
{"op": "insertMarkdown", "markdown": "…", "anchorText": "…", "position": "after"}
```

Every other redline command (open, content, suggestions, accept, reject, bulk apply, revisions, flush) works through the CLI as documented.

## Suggestions — ids are POSITIONAL, always re-read first

Tracked-change (`tc`) ids renumber on **every edit**. Observed live: after one new `replace`, the fresh del/ins pair became `tc0`/`tc1` and every pre-existing change shifted down by two. The safe sequence:

1. `list-redline-suggestions --session-id $SID` — read ids AND note the session `revision`. Each entry carries `author`, `date`, `blockId`, `kind` (`ins`|`del`), and a `snippet` — enough to decide accept/reject without opening the document.
2. Immediately accept/reject, passing that revision so a stale id is a `409 revision_conflict` instead of a silent hit on the wrong change:

```bash
paperwork redline accept-redline-suggestion --session-id $SID --tc-id tc0 --revision 1
paperwork redline apply-redline-suggestions --session-id $SID --revision 2 --json '{"accept": ["tc0"], "reject": ["tc5"]}'
```

3. Accepting/rejecting bumps `revision` and renumbers again — re-list before the next batch. Never cache tc ids across ANY mutation.

`list-redline-revisions` shows the full history; `get-redline-session` shows expiry, revision, and counts.

## Comments — the API counts them but cannot read, write, or resolve them

The session summary reports a `comments` count, and that is the **entire** comment surface of the API today:

- No endpoint lists comment text, no edit op adds a comment, and accept/reject only touches tracked changes.
- Parsed markdown (`tools convert-file-to-markdown`, `urls.markdown`) **silently drops comments** — verified: a doc with 38 substantive reviewer comments produced markdown with zero of them.

If the task needs the comments (contract review threads usually carry the reasoning), do NOT thrash on the API — download the bytes and read `word/comments.xml` locally:

```bash
URL=$(paperwork files get-file --id msa.docx --format json --query 'urls.content' | tr -d '"')
curl -s "$URL" -o /tmp/doc.docx
python3 - <<'EOF'
import zipfile
from xml.etree import ElementTree as ET
W = '{http://schemas.openxmlformats.org/wordprocessingml/2006/main}'
z = zipfile.ZipFile('/tmp/doc.docx')
for c in ET.fromstring(z.read('word/comments.xml')).findall(W+'comment'):
    text = ' '.join(t.text or '' for t in c.iter(W+'t'))
    print(f"[{c.get(W+'id')}] {c.get(W+'author')} ({c.get(W+'date')}): {text}")
EOF
```

(`files get-file --id` accepts the file **name** as well as the uuid.)

If asked to ADD a comment: say plainly that the Paperwork API cannot, and offer the alternatives — an `insertMarkdown` tracked insertion (visible in the review pane like any suggestion), or local docx tooling outside this API.

## Tips

- File names resolve in the org workspace only — a file in a space must be flushed by id (space sessions open via `paperwork spaces space-redline-file`).
- Flush is repeatable — checkpoint long editing sessions.
- Flushing does not close or drain the session; keep editing and flush again.

## See also

- [paperwork-files](../paperwork-files/SKILL.md) — convert the flushed docx to PDF for signing
