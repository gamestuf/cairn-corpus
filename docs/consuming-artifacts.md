# How the consuming side verifies and consumes the corpus

This is the contract between `cairn-corpus` and the consuming system that merges it with internal content.
Nothing here requires a shared database or a call back to this pipeline.

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

Per document, under `derived/{reg_id}/{version}/`:

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
- **public to internal** — if the internal side computes ids with the same formula and the same
  normalization, ids collide exactly when the text is the same. That is the intended behaviour: two
  descriptions of the same requirement are the same chunk.
- **upserts are idempotent** — re-loading a document replaces its points and updates its nodes rather than
  duplicating them.

The normalization is NFC, typographic punctuation folded to ASCII, all Unicode whitespace collapsed to
single spaces, zero-width characters stripped, case preserved. An internal implementation must match it
exactly; `TextNormalizer` is the reference and `ChunkIdentityTests` pins the behaviour.

## 5. Filter on metadata, not on text

Every chunk carries:

| Field | Use |
| --- | --- |
| `tier` | Always `public` here. Your discriminator after the merge. |
| `normativity` | `requirement` \| `guidance` \| `example`, assigned structurally from the registry, never by a model. Filter to `requirement` when the question is about obligations. |
| `framework_rev` | Filter to avoid mixing revisions in one answer. |
| `authority` | Who published it. |
| `control_ids` / `scf_ids` | Crosswalks. |
| `section_anchor` | Stable within-document address. |
| `page` | Page in the source PDF, where one is known. For citation. |
| `terms` | Controlled vocabulary mentioned, from a closed list. |

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

`raw/` is archived but never committed. Raw hashes in the manifest are claims about bytes the public
repository does not contain; verify them against the object store, or by re-fetching the URL — which is a
different claim ("the source says this now"), and `fetched_at` and `etag` are what make the difference
legible.

No credentials, no internal-tier content, no `internal-tier registry`. A row with any `tier` other than `public`
fails the run.
