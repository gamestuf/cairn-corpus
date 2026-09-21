# cairn-corpus

A signed, versioned, retrieval-ready corpus of **public** cybersecurity compliance documents — NIST SP
800-171 / 171A, SP 800-53, CMMC program documents, federal regulation and DFARS clauses, SCF, Cyber AB.

This repository is data only. The pipeline that produces it is
[`cairn-pipeline`](https://github.com/gamestuf/cairn-pipeline); a pinned release of it runs here daily.

Everything here is public-tier. There is no `internal-tier registry` in this repository or in `cairn-pipeline` —
internal-tier content lives on the consuming side, in a separate repository. A registry row with any `tier` other than
`public` fails the build.

## Layout

    registry/public.json              the single input: every document and the rules for handling it
    derived/{reg_id}/{version}/       text.md, chunks.jsonl, nodes.jsonl, edges.jsonl, vectors.jsonl
    manifest/manifest.json            every document: version, outcome, hashes, counts, model
    manifest/manifest.json.sig        cosign signature over the manifest
    manifest/manifest.json.crt        the signing certificate
    manifest/registry.snapshot.json   the registry as of the last run, for the next run's diff
    reports/{run_id}.{md,json}        per-run outcome table, findings and counts

`raw/` is **never** committed. Raw bytes are archived to the object store; in CI they are uploaded as a job
artifact with 90-day retention until S3 is enabled.

## Using the corpus

Verify before you trust it:

```bash
cosign verify-blob \
  --certificate manifest/manifest.json.crt \
  --signature  manifest/manifest.json.sig \
  --certificate-identity-regexp '^https://github\.com/gamestuf/cairn-corpus/\.github/workflows/build\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  manifest/manifest.json
```

Then re-hash the artifacts you read against the `derived` map in each manifest entry. The full consumer
contract — including the `chunk_id` join and what each outcome obliges you to do — is in
[`cairn-pipeline/docs/consuming-artifacts.md`](https://github.com/gamestuf/cairn-pipeline/blob/main/docs/consuming-artifacts.md).

Every chunk carries `tier`, `authority`, `normativity`, `framework_rev`, `control_ids`, `scf_ids`,
`section_anchor`, `page` and `terms`. Filter on those rather than on text: `normativity` in particular is
assigned structurally from the registry and never by a language model, so filtering to `requirement` gives
you obligations and nothing else.

## Builds

`.github/workflows/build.yml` runs daily at 06:00 UTC and on demand. It pulls a pinned `cairn-pipeline`
image, runs it over `registry/public.json`, signs the manifest, and commits `derived/`, `manifest/` and
`reports/` when they change.

Run it by hand from the Actions tab. Inputs:

| Input | Meaning |
| --- | --- |
| `only` | Comma-separated registry ids. Dependencies are pulled in automatically. |
| `dry_run` | Validate and print the plan for every row. Writes and commits nothing. |

A build with `warnings: true` — any `missing` or `quarantined` document — opens or updates a single issue
titled **Corpus warnings**. It does not fail the build: a warning is something to look at, not a reason to
discard a corpus.

## Adding or changing a document

Edit `registry/public.json` and commit that alone; the scheduled build produces the artifacts. Committing
derived files by hand puts files here that no run produced and no manifest describes.

See [adding a document](https://github.com/gamestuf/cairn-pipeline/blob/main/docs/adding-a-document.md).
