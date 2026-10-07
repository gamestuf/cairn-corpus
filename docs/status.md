# Status

Where the corpus is, against what it is meant to become. Read this first; it links to the detail.

- **[`open-questions.md`](open-questions.md)** — every blocker, grouped by what would unblock it.
- **[`registry-reference.md`](registry-reference.md)** — what each registry field means.
- **[`consuming-artifacts.md`](consuming-artifacts.md)** — the contract for anything reading the corpus.

Figures below are from the newest build, `reports/20261006T081442Z.json` (2026-10-06), and the manifest it wrote.
They are what happened, not an estimate. Re-derive them from the newest report in `reports/` when this looks stale.

## What the corpus is for

A verifiable, signed, public corpus of the CMMC and CUI source documents, chunked so that every chunk carries its
provenance, its normativity and a stable id — such that any consumer can merge it with other content and cite it
without having to trust this repository's word for anything.

"Done" is not "every document ingested". It is: **every registry row accounted for, every chunk traceable to bytes
anyone can re-hash, and every gap visible in the report** rather than absent from it.

## The scoreboard

| | target | now | |
| --- | --- | --- | --- |
| Rows accounted for | 59 of 59, exactly one outcome each | **59** | ✅ invariant 1 holds |
| Rows producing artifacts | every active `chunk` row | **41 of 41** | ✅ none errored |
| Chunks | — | **26,889** — 2,168 normative, 7,659 contextual, 17,062 informative | |
| Originals committed (`raw/`) | every redistributable row that built | **38 of 38** | ✅ `REG-F01`–`F03` withheld by decision |
| QA gates | 110 requirements, 320 objectives | **110/110, 320/320** | ✅ crosswalk gate, every check passes |
| Graph (`nodes`/`edges`) | per document | **55,253 nodes, 110,944 edges** | ✅ |
| Control crosswalk (`crosswalk/public/`) | every build | ✅ produced | |
| Embeddings (`vectors.jsonl`) | one per chunk, model pinned by hash | **0** | ❌ §2 |
| Manifest signed (cosign) | every build | **not the current one** | ⚠️ §3 |
| Vector / graph sinks | deliberately **off** | off | ✅ by design |

The 18 skipped rows are skipped by design, each with its reason in the report:

- 15 `planned`: the CPRT JSON rows `REG-N01`–`N04` and `N10`–`N13`, plus `N09`, `N51`, `D08`, `D10`, `D11`, `S03` and `S04`.
- 2 `register-only`: `REG-D07` and `R05`, which wait on a url.
- 1 pointer: `REG-S02`.

The informative share is mostly the SCF workbook (`REG-F01`, 13,452 chunks), whose own content is `derived`.

## 1. What changed since the last version of this page

That version was written against a 43-row registry, with 11 rows producing artifacts and 24 erroring. Since then:

- **Every error cause is gone.** Real asset urls replaced the landing pages. Publisher refusals are fetched under a
  browser identity or from the archive. The regulations come through the eCFR and Federal Register APIs.
  [`open-questions.md`](open-questions.md#overtaken-by-later-work) has the detail.
- **800-171 is in the corpus.** The 110 requirements and 320 objectives come from the NIST PDFs (`REG-N01b`,
  `REG-N02b`), so the QA gates run, and pass.
- **The registry grew to 59 rows**, adding:
  - Rev 3 and 800-172 (`N03b`, `N04b`, `N10b`–`N13b`);
  - 800-53 Rev 5 as a CPRT export (`N50`), scoped to the controls the corpus's mappings reach (5,300 chunks);
  - the SCF workbook (`F01`);
  - the Federal Register preambles (`R06`–`R08`);
  - DoD and ISOO notices (`S06`–`S10`).
- **The control crosswalk** is published under `crosswalk/public/`.
- **Each row carries `relevance`** (`requirement`, `guidance`, `supporting`, `example`) beside `authority` and
  `visibility` (decided 2026-10-06). It reaches chunks in the next build.

## 2. Embeddings are not wired up

The pipeline implements stage 3 and pins the model by file hash. No build passes `--embedding`, so no `embedding.json`
is in play and no `vectors.jsonl` is written. Every row reports `embedding_skipped`, so this is visible rather than
silent, but **the corpus has no retrieval vectors**.

Nothing blocks this except doing it. It needs:

1. An ONNX bge-m3 model file and its tokenizer, reachable from the build.
2. An `embedding.json` naming `model_path`, `model_sha256` and `tokenizer_path`. The hash is enforced — a model that
   hashes to anything else fails the run (invariant 6).
3. `--embedding <path>` added to the `args=(…)` line in `.github/workflows/build.yml`, and to local builds.
4. A decision on where the model lives. It is large and not a document, so `raw/` is the wrong place.

The dense dimension must be 1024; stage 3.2 refuses anything else.

## 3. The current build is local, unreleased and unsigned

- **The pinned image is `v0.2.1`** (`.github/pipeline-image`, pinned 2026-09-24). The pipeline's `main` is 30 commits
  ahead and unreleased. That includes:
  - the PDF parsers for the NIST companions and CMMC guides;
  - the SCF workbook;
  - the crosswalk;
  - the 800-53 scope;
  - `relevance`.
- **The newest artifacts come from local builds of `main`**, not from CI. They are staged in the working tree, not
  committed.
- **Signing happens only in CI.** The last signed build is `36973639960-1` (2026-10-02). Its `manifest.json.sig` and
  `.crt` were removed in the 2026-10-03 cleanup, so the manifest on disk now has no signature. A consumer following
  [`consuming-artifacts.md` §1](consuming-artifacts.md#1-verify-the-signature) cannot verify it until a CI build signs
  one.

Cutting a release and pinning it, then letting CI build, is what makes the published corpus match what local builds
produce, and signs it. See [`cairn-pipeline/docs/releasing.md`](https://github.com/gamestuf/cairn-pipeline/blob/main/docs/releasing.md).

## 4. Decisions waiting on a person

[Detail](open-questions.md#still-open).

- Per CPRT JSON row: accept its wording differences from the PDF, or hold it (`REG-N01`–`N04`, `N10`–`N13`).
- Editions:
  - `REG-F02`: is there a December 2025 CAP?
  - `REG-F03`: CoPC v2.0 or v2.1a?
  - `REG-S05`: is the memo superseded?
  - `REG-R04`: is Rev 1 still current?
- Sources:
  - `REG-R05`: the deviation numbers;
  - `REG-D07` and `D10`: their urls.

Closed 2026-10-06: redistribution of `F01`–`F03` (no), the `R03`/`R07` split (correct), the regulations'
`version_slug` (the eCFR `as_of` date), `R02` (Part 2002).

## 5. Deliberately not done

Not gaps. Recorded so they are not mistaken for oversights.

- **Vector and graph sinks are off.** Qdrant and Neo4j loaders exist and are opt-in. The artifacts on disk are the
  product; a consumer loads them where it wants them.
- **The SCF, CAP and CoPC originals are not redistributed** (decided 2026-10-06); their derived text is.
- **800-53 is scoped**, not whole: only the controls the corpus's mappings reach, listed with their sources in
  `crosswalk/public/sp800-53-scope.csv`.
- **`raw/` growth is unbounded.** The per-file cap bounds one pathological url, not the total, and git keeps every
  version. `S3ObjectStore` is implemented and opt-in; Git LFS is the other option. Neither is needed at this size, and
  adopting either costs a re-derive and never a re-embed, because no path feeds `chunk_id`.
- **No internal tier.** A row with any `tier` other than `public` fails the build, by design.
