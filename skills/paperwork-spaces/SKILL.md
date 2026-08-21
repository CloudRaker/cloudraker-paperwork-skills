---
name: paperwork-spaces
description: |
  Persistent, semantically searchable containers for files and runs in the Paperwork API — every capability space-scoped, plus semantic search over the space's files. Use when results must outlive a run, when a corpus needs search, or to organize a multi-document job.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork spaces

Without a space, the API works in a hidden workspace where files are temporary and die with their run. Inside a space, files belong to the space, outlive runs, and are **indexed for semantic search**.

## Lifecycle

```bash
# Mint one space per job, then remember its id for the whole session/project
SPACE=$(paperwork spaces create-api-space --json '{"name": "Acme onboarding", "ttl": 604800}' | jq -r .id)
mkdir -p .paperwork && echo "$SPACE" > .paperwork/space   # gitignore .paperwork/
paperwork spaces update-api-space --space-id "$SPACE" --json '{"ttl": 2592000}'  # reset deadline: now + ttl
paperwork spaces delete-api-space --space-id "$SPACE"                            # deletes files, closes space
```

Before minting a new space, check for a remembered one: `cat .paperwork/space`, verify with `get-api-space`. One job = one space — don't scatter a job's files across spaces, and don't dump unrelated jobs into one.

`ttl` 300–7776000 s (90 days), default one week. Expiry deletes the files — extend proactively for long jobs. Delete is batch-wise on huge spaces: repeat until it stops finding files.

## Everything works space-scoped

Every capability has a `space-*` twin with identical bodies and the same wait/poll discipline: `space-create-file`, `space-parse`, `space-extract` (+ `space-extract-batch`), `space-redact`, `space-fill`, `space-sign`, `space-compose` (+ batch), `space-pipeline`, `space-create-agent-run`, and the file tools (`space-convert-file-to-pdf/markdown`, `space-split-file-pages`, `space-stitch-files`, `space-render-file-page`, `space-redline-file`). Read back with `space-get-file`, `space-list-files`, `space-get-run`, `space-list-runs`, `space-get-agent-run` — space-scoped reads `404` anything belonging elsewhere.

## Semantic search

```bash
paperwork spaces space-search --space-id spc_… --q "termination clause with 60 day notice" --limit 20
```

Each match carries provenance: file plus page/box or timecode when indexed with grounding. Only space files are indexed — hidden-workspace files are not. Files index as they parse, so search right after upload can miss the newest file; poll it to `ready` first.

## When to use a space (opinion)

- **Default to a fresh space at the start of any multi-step job** — it keeps the job's files separate from everything else, and search only works there. Remember the id (`.paperwork/space`) and reuse it for the whole session.
- One-shot extract/parse → skip the space; use the default workspace and let TTL clean up.
- Finished run whose output turned out to matter → don't re-run in a space; promote it with `paperwork runs keep-run` (see paperwork-runs).

## See also

- [paperwork-runs](../paperwork-runs/SKILL.md) — `keep-run` promotes a temporary run into a space
