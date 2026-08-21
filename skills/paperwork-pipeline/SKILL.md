---
name: paperwork-pipeline
description: |
  Run several Paperwork capabilities (extract, redact, fill, sign) over one set of files in a single call — files parse once, steps run in parallel. Use when a task needs more than one capability on the same documents.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork pipeline

`POST /v1/pipeline` — every file parses once, then each step runs over the parsed set **in parallel, not chained**. A step never consumes another step's output. There is no `parse` step — parsing is automatic.

## Quick start

```bash
paperwork pipeline pipeline --json '{
  "files": [{"id": "<fileId>"}],
  "steps": [
    {"extract": {"schema": {"type": "object", "properties": {"business_name": {"type": ["string", "null"]}}}}},
    {"redact": {"mode": "targeted"}}
  ]
}'
```

A step is one of:

- `{"extract": {…}}`, `{"redact": {…}}`, `{"fill": {…}}`, `{"sign": {…}}` — same inline config as the standalone verb.
- `{"action": "<id|slug>", "params": {…}}` — a saved config.

## Always asynchronous — poll every 10s

Pipeline has no wait option — the response is always a `202` with the run id (`plr_…`) and one `{id, capability}` per step. Poll:

```bash
paperwork runs get-run --id plr_… --wait 0   # every 10s until terminal
```

The run's `steps[]` reports each step's status and result under the step ids handed back at create. A pipeline with a `sign` step will sit at `needs_input` until signers finish — report that to the user instead of spinning.

## When to use it vs. separate calls

- **Use pipeline** when steps are independent views of the same documents (extract data AND produce a redacted copy).
- **Use sequential calls** when a step needs another's output (fill from extracted values, sign the filled PDF) — pipeline steps can't see each other.

## See also

- [paperwork-runs](../paperwork-runs/SKILL.md) — polling, keep, TTL
- [paperwork-agents](../paperwork-agents/SKILL.md) — when the flow needs planning, humans, and sequencing
