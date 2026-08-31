# CloudRaker Paperwork Skills

Agent skills for the [CloudRaker Paperwork API](https://docs.cloudraker.com) and its `paperwork` CLI, following the [Agent Skills](https://agentskills.io) format. Available as a plugin for Claude Code and OpenAI Codex.

## Install the CLI

```bash
brew install cloudraker/tap/paperwork
# or
curl -fsSL https://paperwork.sh | sh

paperwork auth login   # sign in with your CloudRaker account
```

`paperwork auth login` runs a browser device-code sign-in (`--no-browser` for headless) and keeps the token refreshed in a local file store. For CI or agents without a browser, use an API key instead: create one at [app.cloudraker.com/admin/api-keys](https://app.cloudraker.com/admin/api-keys) and `export PAPERWORK_TOKEN="<your API key>"` (it overrides the stored login).

## Install the skills

```bash
npx skills add cloudraker/cloudraker-paperwork-skills
```

Or as a Claude Code plugin:

```
/plugin marketplace add cloudraker/cloudraker-paperwork-skills
/plugin install paperwork
```

## Catalog

| Skill | What it covers |
| --- | --- |
| [`paperwork`](./skills/paperwork) | Umbrella: install, auth, the fire-with-`--wait 0`-and-poll-every-10s doctrine, order of operations, capability routing |
| [`paperwork-parse`](./skills/paperwork-parse) | Documents/audio → markdown + JSON; engine selection (auto/ocr/simple/transcribe/diarize) |
| [`paperwork-extract`](./skills/paperwork-extract) | JSON-Schema extraction with per-field citations; saved configs; 100-file batches |
| [`paperwork-classify`](./skills/paperwork-classify) | Document/page classification against a class list; page mode feeds deterministic PDF split |
| [`paperwork-fill`](./skills/paperwork-fill) | Form-PDF filling from source docs or a `values` object; template library; fill configs |
| [`paperwork-redact`](./skills/paperwork-redact) | Destructive PII removal from documents and audio |
| [`paperwork-sign`](./skills/paperwork-sign) | E-signature envelopes: signers, resend, void, audit trail |
| [`paperwork-compose`](./skills/paperwork-compose) | PDF generation from typst templates: render, batch, bundle authoring, lint, preview |
| [`paperwork-pipeline`](./skills/paperwork-pipeline) | Several capabilities over one file set, parsed once |
| [`paperwork-files`](./skills/paperwork-files) | File corpus + tools: upload, convert, split, stitch, render |
| [`paperwork-redline`](./skills/paperwork-redline) | Tracked-changes editing on .docx: sessions, edits, suggestions, flush |
| [`paperwork-spaces`](./skills/paperwork-spaces) | Persistent, semantically searchable containers; space-scoped everything |
| [`paperwork-runs`](./skills/paperwork-runs) | The shared run lifecycle: polling, filtering, keep, cancel, delete, TTL |
| [`paperwork-agents`](./skills/paperwork-agents) | Long-lived document agents: approvals, human tasks, steering, streaming, automations |
| [`paperwork-webhooks`](./skills/paperwork-webhooks) | Event delivery, ES256 verification, delivery debugging |

## The house style these skills teach

1. **Wait for PDFs, poll everything else.** Digital-text PDFs finish inside the default 60-second hold — just wait. Scans, audio, batches, pipelines, and agent runs go out with `--wait 0` and are polled every 10 seconds until terminal. A `202` is a graceful degrade, never an error.
2. **Documents never enter the agent's context.** No opening or `cat`-ing PDFs/docx/audio, no pasting parsed markdown into the conversation — extract with a schema, search the space, or grep bounded slices of a saved artifact. That context reduction is what Paperwork is for.
3. **Mint a space per job and remember its id** (e.g. `.paperwork/space`) — files stay separate, searchable, and reusable across the session.
4. **Register files once, reuse the id everywhere.** Parse-once-run-many is the biggest cost and latency saver on this API.
5. **Repeated shapes become saved configs; repeated files become batches; multi-capability jobs become pipelines.**
6. **Humans run on human timescales.** `needs_input` and `waiting` are surfaced to the user, not polled.
7. **Decide keep-or-expire the moment a run finishes.** `runs keep-run` promotes results into a space; everything else purges at its TTL.

## Docs

- API & capability guides: [docs.cloudraker.com](https://docs.cloudraker.com)
- CLI reference: `paperwork <resource> --help`
