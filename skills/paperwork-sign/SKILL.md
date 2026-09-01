---
name: paperwork-sign
description: |
  Send a PDF for e-signature via the Paperwork CLI — email-verified signers, tamper-evident cryptographic seal, audit trail, envelope management (resend, void, roster). Use for any "get this signed" task.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork sign

`POST /v1/sign` — each signer verifies email with a one-time code and signs by typing their name. The completed document gets a certificate and cryptographic seal, returned as a new file with `output.auditUrl`.

## Quick start

```bash
paperwork sign sign --wait 0 --json '{
  "file": {"id": "<fileId>"},
  "signers": [{"name": "Ada Lovelace", "email": "ada@example.com"}],
  "cc": [{"name": "Grace Hopper", "email": "grace@example.com"}],
  "message": "Please countersign by Friday.",
  "placement": "page"
}'
```

`cc` parties never sign — no OTP, no tag, no signing link. They receive the sealed PDF when the envelope completes.

**Always asynchronous** — you get `202` with `status: "needs_input"` the moment the envelope exists, and it stays there until the last signer completes. **Do not poll a sign run every 10s** — humans sign on human timescales. Fire it, report the envelope to the user, and check back on request (or wire a webhook for `run.completed`). Sign runs are exempt from the run TTL: envelopes wait for their signers.

## Placement

- `page` (default) — signature certificate page appended.
- `tags` — each signer's stamp replaces a `[Signature N]` placeholder in the document; every signer needs one or the run fails.

## Managing the envelope (run id `sgr_…`)

| Task | Command |
| --- | --- |
| Who still has to sign? | `paperwork runs get-run-envelope --id <runId>` — per-signer `status`, `emailVerifiedAt`, `signedAt`; cc parties listed separately, labeled CC |
| Lost/expired invitation | `paperwork runs resend-run-signer --id <runId> --signer-id <id>` — issues a **new** link, kills the old one (not a reminder nudge) |
| Cancel everything | `paperwork runs void-run-envelope --id <runId>` — optional `{"reason": "…"}` lands in the audit trail; `409` once completed |
| Audit trail (JSON, available mid-flight) | `paperwork runs get-run-audit-trail --id <runId>` |

Signing links are signer-held secrets — they never appear anywhere on the sender's API. To get a link to a signer, resend.

## Tips

- Fill first, then sign: chain `paperwork fill` output into `sign`, or do both in one `pipeline` call.
- Deciding twice / voiding a completed envelope answers `409` — the first action stands.

## See also

- [paperwork-fill](../paperwork-fill/SKILL.md) — complete the form before sending it out
- [paperwork-webhooks](../paperwork-webhooks/SKILL.md) — `run.completed` instead of polling humans
