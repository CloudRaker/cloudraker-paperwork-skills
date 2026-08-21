---
name: paperwork-redline
description: |
  Edit a .docx with tracked changes via the Paperwork CLI — session-based redlining: replace/delete/insert text, accept or reject suggestions, read/add/reply/resolve Word comment threads, export the redlined document. Use for contract markup, negotiated edits, or any Word tracked-changes or comment work.
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
{"op": "fill", "fields": [{"anchorText": "…", "replacement": "…"}, …]}
{"op": "insertMarkdown", "markdown": "**New clause.** …", "position": "end"}
{"op": "insertMarkdown", "markdown": "…", "anchorText": "…", "position": "after"}
```

Note the `fill` op takes `fields`, not `replacements`. Optional on any op: `occurrence` (1-based match picker), `all` (every match), `tracked`, `author`.

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

## Comments — full thread surface, curl until the CLI catches up

The API reads and writes Word comment threads (added 2026-08-21). Comment ids (`cN`) are **stable across edits** — unlike `tc` suggestion ids, they never renumber. CLI ≤0.0.1 predates these endpoints, so call them directly (verified end to end):

```bash
B=https://api.cloudraker.com/v1; A="authorization: Bearer $PAPERWORK_TOKEN"; C="content-type: application/json"

# List — author, date, body, blockId, resolved; replies carry parentId.
# ?openOnly=true returns only unresolved threads.
curl -s "$B/redline/$SID/comments" -H "$A"

# Add, anchored to text (occurrence picks a match when ambiguous;
# author labels Word's review pane, default "CloudRaker")
curl -s -X POST "$B/redline/$SID/comments" -H "$A" -H "$C" \
  -d '{"anchorText":"thirty (30) days","body":"Sixty days is market standard.","author":"Counsel"}'
# → {"revision":1,"result":{"commentId":"c38"},"summary":{…,"comments":39,…}}

# Reply / resolve (whole thread, replies included) / reopen / delete
curl -s -X POST "$B/redline/$SID/comments/c38/reply" -H "$A" -H "$C" -d '{"body":"Agreed, updated."}'
curl -s -X POST "$B/redline/$SID/comments/c38/resolve" -H "$A" -H "$C" -d '{}'
curl -s -X POST "$B/redline/$SID/comments/c38/resolve" -H "$A" -H "$C" -d '{"resolved":false}'
curl -s -X DELETE "$B/redline/$SID/comments/c38" -H "$A"
```

Comments ride along on flush — the exported `.docx` carries every thread, resolved state and all.

Two caveats:

- Parsed markdown (`tools convert-file-to-markdown`, `urls.markdown`) **silently drops comments** — read them through the session endpoints above, never through parse output.
- Against an API that does not yet serve these endpoints (`not_found` on `GET …/comments`), fall back to downloading the file's `urls.content` bytes and reading `word/comments.xml` from the zip locally.

(`files get-file --id` accepts the file **name** as well as the uuid.)

## Tips

- File names resolve in the org workspace only — a file in a space must be flushed by id (space sessions open via `paperwork spaces space-redline-file`).
- Flush is repeatable — checkpoint long editing sessions.
- Flushing does not close or drain the session; keep editing and flush again.
- `PAPERWORK_BASE_URL` (CLI ≤0.0.1): generated commands expect the base to INCLUDE `/v1`; the `files upload` composite appends its own `/v1` and 404s with that base — no single value satisfies both. When overriding the base, upload with `files create-file` + a curl PUT to the returned `uploadUrl` instead of `files upload`.

## See also

- [paperwork-files](../paperwork-files/SKILL.md) — convert the flushed docx to PDF for signing
