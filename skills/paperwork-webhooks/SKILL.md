---
name: paperwork-webhooks
description: |
  Saved webhook endpoints for Paperwork run and file events — create, filter, pause, verify ES256 signatures, and debug deliveries. Use when building a production integration that must react to run completion without polling.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork webhooks

Saved delivery targets any run can point at via `webhook: {"id": "whe_…"}`. Runs keep the **reference**, never a URL snapshot — re-pointing or pausing an endpoint takes effect on runs already in flight.

**Opinion:** webhooks are for production services. In an interactive agent session, poll every 10s instead — don't stand up webhook infrastructure for a one-off. And even with webhooks, a `queued` agent run still needs polling: a run that hasn't started has nothing to report.

## Manage endpoints

```bash
paperwork webhooks create-webhook-endpoint --json '{
  "url": "https://example.com/hooks/cloudraker",
  "events": ["run.completed", "run.failed"]
}'
paperwork webhooks update-webhook-endpoint --id whe_… --json '{"disabled": true}'  # pause, keep id
paperwork webhooks list-webhook-endpoints
paperwork webhooks delete-webhook-endpoint --id whe_…   # in-flight runs just stop delivering
```

Omit `events` to receive all types: `processing.started`, `file.processed`, `file.failed`, `run.completed`, `run.failed`, `processing.completed`, `processing.expired`, `space.expired`, and `agent_run.waiting` / `needs_input` / `approval_requested` / `task_ready` / `completed` / `failed`.

## Delivery & verification

At-least-once, up to 5 attempts with backoff — **dedupe on `eventId`**. Every delivery carries `x-rk1-signature`: a compact ES256 JWT binding run id, `eventId`, event `type`, and `bodySha256`, inside a 5-minute `exp` window. Verify against the public keys (no auth needed):

```bash
paperwork webhooks webhook-jwks
```

Compare `bodySha256` against the body you actually received.

## Debugging: "did it arrive?"

```bash
paperwork webhooks list-webhook-deliveries --id whe_…
```

50 most recent attempts, one row per **attempt** (a retried event appears with rising `attempt`), each with event `type`, run, your server's `responseStatus`, and `ok`.

## See also

- [paperwork-runs](../paperwork-runs/SKILL.md) — the polling alternative, and `metadata` for correlating events
