---
name: paperwork-classify
description: |
  Label documents and cut multi-document packets with the Paperwork CLI — `paperwork classify` labels a file (or every page, finding where each document starts), `paperwork split` cuts a PDF into one real file per page range. Use for mailroom routing, picking an extraction schema per document, or unbundling a scanned intake packet before extract.
allowed-tools:
  - Bash(paperwork *)
---

# paperwork classify & split

**Classify decides. Split cuts.** Classify is the only one that calls a model. Split is page arithmetic — no model, no config, no classes.

## The pipeline is three calls

```bash
# 1. label every page and find the seams
paperwork classify --file ./intake-packet.pdf --params '{
  "classes": [
    {"id": "invoice",  "description": "A bill from a supplier with line items and a total due."},
    {"id": "contract", "description": "A signed agreement with clauses and signature blocks."},
    {"id": "other",    "description": "None of the above."}
  ],
  "granularity": "page"
}'
# → clr_…  with output.segments[]

# 2. cut on those seams
paperwork split --params '{"file": {"id": "<fileId>"}, "classifyRunId": "clr_…"}'
# → output.documentIds[]

# 3. extract each child
paperwork extract --params '{"files": [{"id": "file_…"}, …], "schema": {…}}'
```

**Three, not two — on purpose.** Between step 1 and step 2 you can read the segments, fix one, or decide the packet was a single document all along.

**Already know the ranges? Skip step 1.** That path runs no model and bills no classification:

```bash
paperwork split --params '{
  "file": {"id": "<fileId>"},
  "segments": [
    {"startPage": 1, "endPage": 3, "classId": "invoice"},
    {"startPage": 4, "endPage": 5, "classId": "invoice"},
    {"startPage": 6, "endPage": 6}
  ]
}'
```

`--file ./path.pdf` uploads a local file and folds the id in for you. It works on both verbs.

## Classify: descriptions are the accuracy lever

A class is `{id, description}` — no training data, no display name, no examples field. Write the description the way you would brief a new hire; paste your examples into it.

- 2 to 50 classes. `id` matches `^[a-z0-9][a-z0-9_-]{0,63}$`, unique, and is your **only** branch key.
- `description` ≤ 500 chars. Whole config ≤ 16 KB.
- **One class must be the catch-all** (`"other"` by convention). Omit it and the API injects one, echoing the effective list in `config.classes`. A closed set with no exit makes the model guess.

### Two granularities

| `granularity` | Answer | Pages billed |
|---|---|---|
| `document` (default) | one label for the whole file | first + last window only |
| `page` | `pages[]` + derived `segments[]` | every page |

Page mode is what feeds split. **Boundaries come from a per-page `documentStart` flag, not from a change of class** — that is what separates an invoice followed by another invoice, the most common packet shape. `rules.minPages` (default 1) merges anything shorter into the previous segment.

Classify takes any parseable file. `pageRange` defaults to `{start: 1, end: 750}`; a longer file **fails** `page_limit_exceeded` rather than truncating.

### Confidence: threshold it yourself

`confidence` is an integer **0–5**. `>= 4` auto-process, `<= 3` route to a human or sharpen the description. A low score **never** rewrites the label — "unsure" and "sure it is `other`" must stay distinguishable. There is no `minConfidence` field. A segment scores the **minimum** of its pages.

`reasoning` is one sentence of prose. It is not evidence and carries no citations. Do not parse it.

## Split: PDF only, echoes everything

- **PDF only, unencrypted.** Office, image, audio and video fail the run with `unsupported_split_source`. Classify accepts anything; split does not.
- Send **exactly one** of `segments` / `classifyRunId`. Neither → `segments_required`; both → `segments_conflict`. The run must be on the same file and `granularity: "page"`, or you get `classify_run_mismatch` / `classify_run_not_paged`.
- `classId` and `confidence` on `splits[]` are **echoed, never produced**. Split has no opinion about what a page is.
- `splits[].fileId` is authoritative; `output.documentIds[]` is the same ids flat, ready to paste into extract.
- Children carry `parentFileId`, inherit the parent's `metadata`, and gain `metadata.parentRunId`. They land in the parent's space.
- One segment covering the whole file creates nothing — you get the parent's `fileId` back.
- Caps: 100 segments, 50 MB source. 100 matches extract's file cap, so a split packet always fits one downstream call.

**`materialize: false` is the safe first call on a big packet** — it returns the ranges and creates nothing.

## Cost traps

1. **Classify is 5× split per page.** Classify runs the model; split is deterministic byte work.
2. **The children are parsed again.** Child PDFs are byte extractions, not parsed at split time — the first verb that consumes a child parses it. A 400-page packet pays parse 400 + classify 400 + split 400 + ~400 more across the children, then extract. `usage` itemizes `{pages, parse, split}`.
3. **Document mode bills only the pages it read** — the first and last window, not the whole file.

## Wait vs poll

Document-mode classify on a short file finishes inside the default 60s hold. **Page-mode classify and any split of a long packet return `202`** — that is the contract, not an error. Add `--wait 0` and poll `paperwork runs get-run --id clr_… --wait 0`.

A classify run expires with its TTL (24h default). Classify Monday, split Wednesday → `classify_run_mismatch`. Split promptly, or keep the segments client-side and post them explicitly.

## Save the taxonomy

A class list is usually organization-wide: `paperwork configs create-classify-config`, then pass `action: "<slug>"`. Inline fields merge over the saved config.

**There is no split config library.** Split has nothing to configure.

## See also

- [paperwork-extract](../paperwork-extract/SKILL.md) — the verb that consumes the children
- [paperwork-runs](../paperwork-runs/SKILL.md) — polling, TTL, and `keep` (which moves the parent *and* every child)
- [paperwork-files](../paperwork-files/SKILL.md) — registering and downloading files
