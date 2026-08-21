---
name: paperwork-runs
description: |
  Read, poll, list, filter, keep, cancel, and delete Paperwork runs — the shared lifecycle behind every capability. Use to check on any run id, find what an integration has been doing, promote results to permanence, or clean up.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork runs

One route reads every kind of run; the id prefix tells you what it is: `exr_` extract, `par_` parse, `rdr_` redact, `flr_` fill, `sgr_` sign, `plr_` pipeline.

## Poll — the standard loop

```bash
# For a quick digital-PDF run, one long-poll is simpler: get-run --wait 60.
# For everything slower (OCR, audio, batches, pipelines): every 10 seconds, --wait 0.
while :; do
  R=$(paperwork runs get-run --id "$ID" --wait 0)
  S=$(echo "$R" | jq -r .status)
  case "$S" in processing|queued) sleep 10 ;; *) break ;; esac
done
```

Statuses: `processing` → `processed` | `failed` | `cancelled` | `expired` | `needs_input`. `needs_input` means a human (fill review task or signer) — surface `tasks[]`/`envelopeUrl` to the user rather than polling on human timescales. `--output inline` returns parse/extract source inline (≤ 1 MiB). Pipeline runs carry `steps[]` keyed by the step ids from create.

## Outputs

Signed URLs are inline on the run body under `output`. The stable alias for one produced file:

```bash
paperwork runs get-run-output --id <runId> --name redacted.pdf --output ./redacted.pdf
```

`409 output_not_ready` is retryable — bytes still settling, ask again in a few seconds.

Download outputs only when the user wants the file itself — never `cat` or open a downloaded PDF/markdown to inspect it. Read the run body (`output.value`, `output.entities`, citations) instead; that's the whole point of the API.

## Listing & filtering

```bash
paperwork runs list-runs --object extract_run --status processed \
  --params '{"metadata.customerId": "cus_42"}' --page-all
```

Filters: `--object`, `--status`, up to 3 `metadata.<key>` pairs (AND). Cursor paging (opaque `cursor`, ≤ 50/page) — or just `--page-all`. The listing is eventually consistent; `get-run` is authoritative. **Attach `metadata` at create** — it's the only way to find your runs later.

## TTL & the three cleanups

Runs and their files purge at `expiresAt` (up to ~7 days; sign runs exempt). An expired run answers `410` in grace, then `404`.

| Intent | Command |
| --- | --- |
| Keep results forever | `runs keep-run --id <id> --json '{"space": {"name": "Q3 invoices"}}'` (or `{"spaceId": "spc_…"}`) — moves files into a real space, indexes them, clears the TTL, returns `dashboardUrl`. Wait for the run to finish first (`409 pipeline_running`); keeping into a second space is `409 already_kept`. |
| Stop the work, files stay until expiry | `runs cancel-run --id <id>` (idempotent) |
| Purge everything now | `runs delete-run --id <id>` (cancels in-flight work first; idempotent `204`) |

**Opinion:** decide keep-or-let-expire as soon as a run finishes. Anything worth showing the user again is worth a `keep-run` before you forget the id.

## See also

- [paperwork-webhooks](../paperwork-webhooks/SKILL.md) — event-driven completion for production
- [paperwork-spaces](../paperwork-spaces/SKILL.md) — where kept runs land
