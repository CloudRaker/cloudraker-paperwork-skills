---
name: paperwork-agents
description: |
  Run long-lived CloudRaker document agents via the Paperwork CLI — reviewed, versioned automations with human steps, approval gates, steering, live streaming, and HMAC-triggered automations. Use when a document flow needs planning, sequencing, or humans in the loop.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork agents

An agent is a reviewed, versioned automation: steps done by the agent and steps done by people, plus the actions it may take and where sign-off is required. Runs (`agr_…`) are long-lived — days, up to `expiresAt` (~7 days).

## Start a run

```bash
paperwork agents list-agents                       # what can I run? tasks[], actions[], version
paperwork agents get-agent --id <agentId>          # read BEFORE running: which steps come back as human work

# File-first door
paperwork agents create-agent-run --wait 0 --json '{
  "agent": "<agentId>",
  "files": [{"url": "https://example.com/intake.pdf"}],
  "metadata": {"caseId": "42"}
}'

# Prompt-shaped door — same runtime, instruction leads
paperwork agents invoke-agent --id <agentId> --wait 0 --json '{"input": "Review this NDA for redlines", "files": [...]}'
```

## Poll every 10s — and know that **reading a queued run is what starts it**

```bash
paperwork agents get-agent-run --id agr_… --wait 0   # every 10s
```

A `queued` run is not working yet — its files are still preparing, and **polling the run is what picks it up** once they're ready. A webhook alone leaves it queued forever. So always poll agent runs; once `status` is `waiting`, stop spinning — it's blocked on a person.

Run anatomy: `tasks[]` (step ledger; `executor: "human"` are yours), `approvals[]` (pending sign-offs with proposed `params`/`files`), `waiting` (count of both), `result` + `output.files`, `agent.id` + `input.files` (the immutable recipe for a successor run).

## Unblocking a run

```bash
# Approve / reject a sign-off (reject REQUIRES a note; deciding twice → 409)
paperwork agents decide-agent-run-approval --id agr_… --approval-id <apr> --wait 0 \
  --json '{"decision": "approve"}'    # or {"decision": "reject", "note": "Wrong signer."}
# Edit before approving: add "params" and/or "files" (≤ 1 MiB together)

# Complete a human step (both fields optional; agent steps → 422, unmet deps → 409)
paperwork agents complete-agent-run-task --id agr_… --task-id <tsk> --wait 0 \
  --json '{"note": "Client signed; scan attached.", "files": ["file_…"]}'

# Mid-flight guidance
paperwork agents steer-agent-run --id agr_… --wait 0 --json '{"message": "Prefer the 2025 template."}'
```

Lifecycle: `pause-agent-run` (safe point, keeps work) / `resume-agent-run` / `cancel-agent-run` (permanent, completed work retained). `paused` is not a failure. Nothing an agent imported or filed is ever taken back.

## Observing

- `get-agent-run-timeline --id agr_… [--after <eventId>]` — durable history; page with the last event id.
- `stream-agent-run` — SSE (`timeline` durable + dedupe by id, `live` provisional deltas, `status`); 5-min cap, reconnect with `Last-Event-ID`. Prefer polling in shell sessions; streams are for UIs.
- `get-agent-run-result --id agr_…` — durable terminal result.

## Automations

- `automations run-automation --automation-id aut_…` — fire now, exactly as its schedule would (same thread, approval gate, fire history). Live run in thread → `busy`; parked on approval → `needs_input`.
- `automations trigger-automation --ref <ref>` — fire from your own stack. Auth is a per-automation HMAC header (`x-rk1-trigger-signature: t=<unix>,v1=<hmac-sha256("<t>.<body>", secret)>`), not an API key; requests older than 5 min are rejected; body `{}` or `{"input": <json ≤ 8 KiB>}`.

## Tips

- Space-scoped: `paperwork spaces space-create-agent-run` keeps everything the agent files in that space after the run ends.
- A human step must land before `expiresAt` or the run parks for good — watch the deadline on multi-day runs.

## See also

- [paperwork-webhooks](../paperwork-webhooks/SKILL.md) — `agent_run.*` events for waiting/approval/task notifications
