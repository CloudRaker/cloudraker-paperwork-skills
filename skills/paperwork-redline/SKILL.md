---
name: paperwork-redline
description: |
  Edit a .docx with tracked changes via the Paperwork CLI — session-based redlining: replace/delete/insert text, accept or reject suggestions, export the redlined document. Use for contract markup, negotiated edits, or any Word tracked-changes work.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork redline

Session-based tracked-changes editing on Word documents. Only `.docx` — anything else answers `422`.

## The session loop

```bash
# 1. Open a session on a registered .docx (ttl 60–604800s)
paperwork tools redline-file --ref msa.docx --json '{"ttl": 86400}'   # → sessionId

# 2. Read the document as anchored blocks
paperwork redline get-redline-content --session-id <sid> --view accepted

# 3. Apply edits — tracked changes by default
paperwork redline apply-redline-edits --session-id <sid> --json \
  '{"op": "replace", "anchorText": "thirty (30) days", "replacement": "sixty (60) days"}'

# 4. Export as .docx, tracked changes and all — session stays open
paperwork redline flush-redline-session --session-id <sid> --json '{"fileId": "msa.docx"}'
```

Nothing is stored until flush. The flushed file is `<original>.redlined.docx` with `parentFileId`; `fileId` must be the file the session was opened on (`409 redline_file_mismatch` otherwise).

## Edit operations

`replace` and `deleteText` act on `anchorText`; `fill` applies a list of replacements in one pass; `insertMarkdown` adds paragraphs before/after an anchor or at the end. `tracked: false` writes silently. `?view` on content: `accepted` (default) | `baseline` | `current`.

## Suggestions — ids are POSITIONAL, always re-read first

Tracked-change (`tc`) ids renumber on **every edit**. The safe sequence:

1. `list-redline-suggestions --session-id <sid>` — read ids AND note the session `revision`.
2. Immediately accept/reject, passing that revision so a stale id is a `409 revision_conflict` instead of a silent hit on the wrong change:

```bash
paperwork redline accept-redline-suggestion --session-id <sid> --tc-id tc1 --revision <rev>
paperwork redline apply-redline-suggestions --session-id <sid> --revision <rev> --json '{"accept": ["tc2"], "reject": ["tc5"]}'
```

Never cache tc ids across edits. `list-redline-revisions` shows the full history; `get-redline-session` shows expiry, revision, and counts.

## Tips

- File names resolve in the org workspace only — a file in a space must be flushed by id (space sessions open via `paperwork spaces space-redline-file`).
- Flush is repeatable — checkpoint long editing sessions.

## See also

- [paperwork-files](../paperwork-files/SKILL.md) — convert the flushed docx to PDF for signing
