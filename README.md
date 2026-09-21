# cairn-corpus

A signed, versioned, retrieval-ready corpus of **public** cybersecurity compliance documents — NIST SP
800-171 / 171A, SP 800-53, CMMC program documents, federal regulation and DFARS clauses, SCF, Cyber AB.

This repository is data only, and it is the public half of the system. The pipeline that produces it,
`cairn-pipeline`, is a private repository; a pinned release of it runs here daily as a container image
published to this repository's own package namespace. Nothing here needs access to that repository —
everything a consumer or a registry contributor needs is in this one.

Everything here is public-tier. There is no `internal-tier registry` in this repository or in `cairn-pipeline` —
internal-tier content lives on the consuming side, in a separate repository. A registry row with any `tier` other than
`public` fails the build.

## Layout

    registry/public.json              the single input: every document and the rules for handling it
    sources/{reg_id}/                 committed copies, used only when a live fetch fails
    derived/public/{org}/{doc_id}/{version}[/{part}]/
                                      text.md, chunks.jsonl, nodes.jsonl, edges.jsonl, vectors.jsonl
    manifest/manifest.json            every document: version, outcome, hashes, counts, model
    manifest/manifest.json.sig        cosign signature over the manifest
    manifest/manifest.json.crt        the signing certificate
    manifest/registry.snapshot.json   the registry as of the last run, for the next run's diff
    reports/{run_id}.{md,json}        per-run outcome table, findings and counts

Artifacts are organized by the issuing organization and the publisher's own document id, with the version
last so revisions sit together — `nist/sp-800-171/r2u1/`, `/r3/`, later `/r4/`. Tier is the root, so
everything in this public repository is under `public/` and anything outside it is wrong at a glance.

`raw/` is **never** committed. Raw bytes are archived to the object store; in CI they are uploaded as a job
artifact with 90-day retention until S3 is enabled.

Open questions — everything the corpus is waiting on, grouped by what would unblock it — are tracked in
[`docs/open-questions.md`](docs/open-questions.md).

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
[`docs/consuming-artifacts.md`](docs/consuming-artifacts.md).

Every chunk carries `tier`, `authority`, `normativity`, `framework_rev`, `control_ids`, `scf_ids`,
`section_anchor`, `page` and `terms`. Filter on those rather than on text: `normativity` in particular is
assigned structurally from the registry and never by a language model, so filtering to `requirement` gives
you obligations and nothing else.

## Builds

`.github/workflows/build.yml` runs daily at 06:00 UTC and on demand. It reads the image pinned in
`.github/pipeline-image`, pulls it, runs it over `registry/public.json`, signs the manifest, and commits
`derived/`, `manifest/` and `reports/` when they change.

The pin is a digest, not a tag: a corpus build is only reproducible if the thing that built it is. It is
written by the pipeline's `publish-to-corpus` workflow when a release is cut — change it by cutting a
release, not by editing the file.

Run a build by hand from the Actions tab. Inputs:

| Input | Meaning |
| --- | --- |
| `only` | Comma-separated registry ids. Dependencies are pulled in automatically. |
| `dry_run` | Validate and print the plan for every row. Writes and commits nothing. |
| `image` | Override the pinned image for one run, to test a candidate before pinning it. |

Until the first `cairn-pipeline` release has been published and delivered, `.github/pipeline-image` does not
exist and the build fails with that reason stated. That is expected for a new corpus, not a fault: there is
no image to run yet.

A build with `warnings: true` — any `missing` or `quarantined` document — opens or updates a single issue
titled **Corpus warnings**. It does not fail the build: a warning is something to look at, not a reason to
discard a corpus.

## When a document cannot be fetched

Some publishers block CI runners or present certificates the runner rejects. For those, commit a copy to
`sources/{reg_id}/` and point the row at it with `fallback_path`. The pipeline still tries the publisher
first and only falls back when that fails — and when it does, the run reports it and the manifest records
`origin: fallback`, so a snapshot is never mistaken for the publisher's current text.

This is not the fix for a row whose `url` returns a landing page instead of the document. That shows up as
`payload is 'html', not 'pdf'`, and the answer there is to correct the url — a fallback would freeze a
document that is perfectly reachable.

## Adding or changing a document

Edit `registry/public.json` and commit that alone; the scheduled build produces the artifacts. Committing
derived files by hand puts files here that no run produced and no manifest describes.

Every field is documented in [`docs/registry-reference.md`](docs/registry-reference.md). To check a change
before the nightly build picks it up, run the **build** workflow with `dry_run: true`: it validates the
registry and prints the plan for every row without writing anything.
