# How to verify and consume the corpus

This is the contract between `cairn-corpus` and anything that reads it. Nothing here requires a shared
database, an account, or a call back to the pipeline that produced it.

## 1. Verify the signature

The workflow signs `manifest/manifest.json` with `cosign sign-blob` in keyless mode. There is one signature
per run, over the manifest only.

    cosign verify-blob \
      --certificate manifest/manifest.json.crt \
      --signature  manifest/manifest.json.sig \
      --certificate-identity-regexp '^https://github\.com/gamestuf/cairn-corpus/\.github/workflows/build\.yml@' \
      --certificate-oidc-issuer https://token.actions.githubusercontent.com \
      manifest/manifest.json

Pin the identity regexp. Verifying a signature without checking *who* signed it establishes only that
somebody signed it.

## 2. Verify the artifacts you read

The manifest carries the SHA-256 of every derived file. Re-hash each file you actually read and compare:

```python
import hashlib, json, pathlib

manifest = json.load(open("manifest/manifest.json"))

for reg_id, entry in manifest["documents"].items():
    for path, expected in entry.get("derived", {}).items():
        actual = hashlib.sha256(pathlib.Path(path).read_bytes()).hexdigest()
        if actual != expected:
            raise SystemExit(f"{path}: expected {expected}, got {actual}")
```

Signature + hashes together mean: tampering with any artifact is caught by the hash, and tampering with the
manifest is caught by the signature.

`registry_hash` is the SHA-256 of the registry's canonical form, so you can tell which registry produced a
corpus without trusting a version string.

## 3. Read the artifacts

Per document, under `derived/public/{org}/{doc_id}/{version_slug}/{format_profile}/` — the manifest entry lists the exact paths:

| File | Use |
| --- | --- |
| `chunks.jsonl` | Retrieval units and their metadata |
| `vectors.jsonl` | Dense (1024-d, normalized) and sparse embeddings, keyed by `chunk_id` |
| `nodes.jsonl` / `edges.jsonl` | The graph |
| `text.md` | The whole document, for display or re-chunking |

All are JSON Lines: stream them, do not load them whole.

## 4. Join on `chunk_id`

`chunk_id = sha256(reg_id + "|" + version + "|" + section_anchor + "|" + normalized_text)`, and node ids are
equally derived (`control:3.1.1`, `objective:3.1.1[a]`, `section:REG-D03:v2.13:3.1.1`, `scf:IAC-01:2026.2`,
`term:cui`, `document:REG-N01`). No id is ever assigned by a database.

This is what makes the merge a plain equality join:

- **chunks to vectors** — join `chunks.jsonl` and `vectors.jsonl` on `chunk_id`.
- **this corpus to any other** — a corpus that computes ids with the same formula and the same
  normalization collides exactly when the text is the same. That is the intended behaviour: two
  descriptions of the same requirement are the same chunk.
- **upserts are idempotent** — re-loading a document replaces its points and updates its nodes rather than
  duplicating them.

The normalization is NFC, typographic punctuation folded to ASCII, all Unicode whitespace collapsed to
single spaces, zero-width characters stripped, case preserved. Any independent implementation must match it
exactly; `TextNormalizer` is the reference and `ChunkIdentityTests` pins the behaviour.

## 5. Filter on metadata, not on text

Every chunk carries:

| Field | Use |
| --- | --- |
| `tier` | Always `public` here. Your discriminator after the merge. |
| `visibility` | Who inside the organisation may be shown the chunk: `public` \| `internal` \| `technical` \| `cmmc-program` \| `security`. Always `public` here; non-public tiers carry the others. **Enforce it before a chunk reaches a reader.** The manifest entry carries the same field from the current registry, and wins where the two disagree: a reclassified document is not rebuilt until its source changes, so its chunks can hold the earlier value. |
| `normativity` | `requirement` \| `guidance` \| `example` \| `assertion`, assigned structurally from the registry, never by a model. Filter to `requirement` when the question is about obligations, and to `assertion` when it is about how a named system satisfies them — a system security plan's content is `assertion`, which is a claim to be tested, not an obligation (ADR-0013). |
| `framework_rev` | Filter to avoid mixing revisions in one answer. |
| `authority` | Who published it. |
| `control_ids` / `scf_ids` | Crosswalks. `control_ids` holds NIST SP 800 control ids in canonical form: 800-171 `3.1.1` (Rev 2) or `03.01.01` (Rev 3), 800-172 `3.1.1e` or `03.01.01E`, 800-53 `AC-02(01)` — zero-padded, whatever the source printed; an 800-53 control that 800-171 Rev 2 Appendix E labels `NFO AC-1` is `AC-01`. `scf_ids` holds SCF control ids, `GOV-01` or `GOV-01.1`. Ids found in text are kept only when they are valid in that form, so a section number like `3.15.1` is never one; an id a source file sets that is not valid is kept and reported in the run. |
| `section_anchor` | Stable within-document address. |
| `page` | Page in the source PDF, where one is known. For citation. |
| `ocr_confidence` | For text recognised from a scanned PDF, the OCR engine's mean word confidence (0–100) on the chunk's page. Absent for text read from a text layer. Weigh or filter on it: below 60 the run also reported the page. |
| `terms` | Controlled vocabulary mentioned, from a closed list. |
| `provision` | For a regulation chunk, the provision it is part of, as a canonical citation: `FAR 52.204-21`, `DFARS 252.204-7012`, `DFARS 204.7501`, `32 CFR 170.4`. For a CMMC assessment guide chunk, the practice: `AC.L2-3.1.1`, `AC.L1-b.1.i`, `AC.L3-3.1.2e` — its `control_ids` hold the NIST requirement the practice is (`3.1.1`, `3.1.2e`). Null elsewhere. |
| `provision_title` | That provision's own title — "Safeguarding Covered Defense Information and Cyber Incident Reporting". Carried beside the text, so the regulation's words stay exactly its own. |
| `references` | Provisions the chunk's text cites, in the same canonical form. Filter on it to find every chunk that invokes a clause or a section. |
| `relations` | Typed links the source itself states from this chunk's element, each `{kind, target, target_framework, target_version}`. From the NIST CPRT exports: `maps-to` (an 800-171 r3 or 800-172 requirement to the 800-53 controls it derives from), `related-control` (between 800-53 controls), `incorporated-into` and `moved-to` (where a withdrawn requirement went), `addressed-by`. `target_framework` and `target_version` name the target's document and revision (`SP_800_53`, `5.1.1`) when it is another document, and are absent when the target is in this one. Recorded whether or not the corpus holds the target. Empty elsewhere. |

### Regulations meet at `Provision` nodes

A regulation chunk's section links `IN_SECTION` to a `Provision` node named by its canonical citation —
`provision:dfars_252.204-7012`, `provision:32_cfr_170.4` — and links `REFERENCES` to the `Provision` node of every
provision its text cites. The id comes from the citation alone, so DFARS 252.204-7021 citing 32 CFR 170.4 and 32 CFR
part 170's own § 170.4 name the same node: the cross-document join is already made. A `Provision` with an
`IN_DOCUMENT` edge is held by this corpus; one without is cited but not held.

Block types for regulations are `clause text`, `provision text`, `rule text`, `definition`, `prescription`,
`amendatory instruction`, `regulatory note`, `appendix text`, and for a Federal Register rule's preamble
`preamble`, `comment` and `response`. Section anchors are `{provision}({paragraph})/{type}`:
`252.204-7012(b)/clause text`, `§ 170.4(b) Affirming Official/definition`.

### The graph's section nodes carry an anchor slug

A `Section` node has a `heading_slug` property — the anchor form of that section's own heading,
`access-control-overview` for `### Access Control Overview`. It is how one document cites a
section of another: ORGANIZATION's control JSON names the parts of a system security plan narrative it relies on by
anchor, and the slug is the key those resolve against.

Two things worth knowing if you traverse it. **Every ancestor heading has a node**, not only the headings
that carry prose of their own, so the `IN_SECTION` chain from a leaf to the document root has no gaps.
And `section_anchor` on a chunk is still the heading *path*; the slug is on the node, so nothing about chunk
identity changed.

Edges added for that join: **`REFERENCES`** (a control or objective cites a section of a narrative) and
**`INHERITS_FROM`** (a control takes its implementation from the same control in another system).

## 6. Decide what to do with each outcome

Every document in the manifest carries exactly one `outcome`:

| Outcome | What it means for you |
| --- | --- |
| `new`, `changed` | Re-index this document. |
| `unchanged` | Nothing to do. |
| `skipped` | Deliberately not ingested. The report says why. |
| `missing` | The source was unreachable. **The previous version's artifacts are still current** — keep serving them, and treat the corpus as stale for this document. |
| `error` | The run could not process it. Previous version stands. |
| `quarantined` | A QA gate failed. **The previous version stands and is still correct.** Do not ingest this run's output for this document — there is none. |

`missing` and `quarantined` set `warnings: true` in the report and still exit 0. A run with warnings is a
run you should look at, not a run you should discard.

## 7. What is not in the corpus

Most originals **are** here, under `raw/`, so the manifest's source hash can be checked against the bytes
next to it. What is not here is the original of a document the registry does not clear for redistribution,
or one over the publishing size cap: for those, `source.archive_published` is `false` and the source hash
is a claim about bytes this repository does not contain. Verify those against the object store, or by
re-fetching the URL — which is a different claim ("the source says this now"), and `fetched_at` and `etag`
are what make the difference legible.

`source.origin` says where the bytes actually came from: `url` (the publisher, this run), `archive` (the
copy this corpus archived earlier, because the fetch failed), `fallback` (a hand-committed copy, likewise)
or `local`. Treat the middle two as snapshots of the age given by the previous run's `fetched_at`.

No credentials and no non-public content. A row with any `tier` other than `public` fails the run.
