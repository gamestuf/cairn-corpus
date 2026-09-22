# Status

Where the corpus is, against what it is meant to become. Read this first; it links to the detail.

- **[`open-questions.md`](open-questions.md)** — every blocker, grouped by what would unblock it.
- **[`registry-reference.md`](registry-reference.md)** — what each registry field means.
- **[`consuming-artifacts.md`](consuming-artifacts.md)** — the contract for anything reading the corpus.

Figures below are from the last real build, `reports/35585084424-1.json`. They are what happened, not an
estimate. Re-derive them from the newest report in `reports/` when this looks stale.

## What the corpus is for

A verifiable, signed, public corpus of the CMMC and CUI source documents, chunked so that every chunk
carries its provenance, its normativity and a stable id — such that any consumer can merge it with
other content and cite it without having to trust this repository's word for anything.

"Done" is not "every document ingested". It is: **every registry row accounted for, every chunk traceable
to bytes anyone can re-hash, and every gap visible in the report** rather than absent from it.

## The scoreboard

| | target | now | |
| --- | --- | --- | --- |
| Rows accounted for | 43 of 43, exactly one outcome each | **43** | ✅ invariant 1 holds |
| Rows producing artifacts | 38 | **11** | ❌ §1 below |
| …of those, usefully chunked | 38 | **6** | ❌ §1.1 — five produced 1 chunk each |
| Originals committed (`raw/`) | every redistributable row | **0** | ⏳ shipped, unreleased |
| Embeddings (`vectors.jsonl`) | one per chunk, model pinned by hash | **0** of 61 chunks | ❌ §2 below |
| Manifest signed (cosign) | every build | untested | ⏳ no manifest yet to sign |
| QA gates enforced | 110 requirements, 320 objectives, page counts | not reached | ❌ blocked by §1 |
| Graph (`nodes`/`edges`) | per document | ✅ produced | |
| Vector / graph sinks | deliberately **off** | off | ✅ by design |

Five rows are correctly never ingested: `REG-C03` (superseded), `REG-D11` (optional, off unless named), and
`REG-S02` `REG-S03` `REG-S04` (pointers to sources held elsewhere). That is why the target is 38 and not 43.

## 1. The 24 rows that error — the dominant gap

Only 11 of 38 rows that should produce artifacts do. This is the single number that matters, and **none of
it is a pipeline defect**: every failure is a counted, attributed line in the report, which is invariant 2
working. They are inputs the corpus does not have.

| cause | rows | what unblocks it |
| --- | ---: | --- |
| URL is a landing page | 11 | the real asset URL |
| Publisher returns HTTP 403 | 9 | a committed copy in `sources/` |
| TLS handshake fails | 2 | a committed copy in `sources/` |
| Local file never supplied | 2 | the file |

Two of these outrank the rest:

- **`REG-N01` and `REG-N02` have never had their source files.** They are the rows the whole 800-171
  chunking design is built around — one chunk per field per control, verified against the paired PDF. Until
  that machine-readable control text is in place, **the corpus has no 800-171 control text at all**, and the QA gates that
  check for 110 requirements and 320 objectives never run.
- **Seven of the nine 403s are `dodcio.defense.gov`** — the model overview, all three assessment guides,
  both scoping guides, the NIST-alignment briefing. That is most of the CMMC document set behind one host.

Per-row detail: [`open-questions.md` §2–§4](open-questions.md).

### 1.1 "11 rows" overstates it

Of the 11 rows that produced artifacts, five produced **exactly one chunk** — 61 chunks across all 11:

| row | chunks | extractor |
| --- | ---: | --- |
| `REG-F03` CMMC CoPC | 32 | `poppler-pdftotext` |
| `REG-C02` DFARS 252.204-7012 | 7 | `anglesharp-readability` |
| `REG-C04` `REG-C05` | 5 each | `anglesharp-readability` |
| `REG-C01` FAR 52.204-21 | 4 | `anglesharp-readability` |
| `REG-C06` | 3 | `anglesharp-readability` |
| `REG-R01` `REG-R02` `REG-R03` `REG-R06` `REG-R07` | **1 each** | `anglesharp-readability` |

A whole federal rule as one chunk is not a chunking result, it is readability finding a fragment. `REG-R01`
is 32 CFR 170 — the CMMC program rule, tens of thousands of words — reduced to a single chunk, because the
eCFR page it was fetched from is script-rendered and there was nothing in the HTML to extract.

So the real position is **one properly chunked document** (the CoPC, a real PDF) and **five short contract
clauses**, where 3–7 chunks is plausible. Everything else is either absent or a stub.

Those five single-chunk rows are exactly the ones §3 fixes: `REG-R01` and `REG-R02` move to the eCFR's
published XML, and `REG-R03` `REG-R06` `REG-R07` to the Federal Register's document body. That is the
strongest argument for cutting the release, and the clearest thing to check afterwards — **if those five
do not jump from 1 chunk to dozens, the release did not do what it was for.**

## 2. Embeddings are not wired up

The pipeline implements stage 3 and pins the model by file hash. The corpus build **does not pass
`--embedding`**, so no `embedding.json` is in play and no `vectors.jsonl` is written. Stage 3 records
`embedding_skipped` per document, so this is visible rather than silent — but the corpus currently has no
retrieval vectors at all.

Nothing is blocking this except doing it. It needs:

1. An ONNX bge-m3 model file and its tokenizer, reachable from the build.
2. An `embedding.json` naming `model_path`, `model_sha256` and `tokenizer_path`. The hash is enforced — a
   model that hashes to anything else fails the run, which is invariant 6.
3. `--embedding <path>` added to the `args=(…)` line in `.github/workflows/build.yml`.
4. A decision on where the model lives. It is large and not a document, so `raw/` is the wrong place and
   committing it to this repository is probably also wrong.

Dense dimension must be 1024; stage 3.2 refuses anything else.

**Until this is done the corpus is text and graph only.** Anything expecting vectors will find none.

## 3. Shipped but not released

`main` on `cairn-pipeline` carries the originals-publishing, publisher-API and CFR-XML work. The corpus is
pinned to the image built from `v0.1.1`, which predates all of it, so **none of it is running yet**.

Cutting `v0.2.0` is what makes `raw/` fill, the Federal Register and CFR rows resolve through their APIs,
and the archive become the first fallback. See
[`cairn-pipeline/docs/releasing.md`](https://github.com/gamestuf/cairn-pipeline/blob/main/docs/releasing.md).

Expect the first build after that release to produce **one large commit** — every redistributable original
at once. Nothing needs deleting first: the stale flat-layout artifacts and the old manifest went with the
path-layout change, so the next build is a clean first build.

## 4. Decisions waiting on a person

Neither code nor access will resolve these. All are recorded as `confirm` items, so every run report raises
them until they are answered. [Detail](open-questions.md#1-decisions-only-you-can-make).

- Whether the SCF and Cyber AB terms permit republishing their original files (`REG-F01` `REG-F02` `REG-F03`).
- Whether `REG-R03` and `REG-R07` — one Federal Register document split by normativity — should share a
  `doc_id` with distinct `part`s.
- Whether the two living CFR regulations should pin `version_slug` to their resolved amendment date.
  **This reissues chunk ids**, so it costs a re-embed.
- Whether `REG-R02` is Part 2002 or Part 2000.

## 5. Deliberately not done

Not gaps. Recorded so they are not mistaken for oversights.

- **Vector and graph sinks are off.** Qdrant and Neo4j loaders exist and are opt-in. The artifacts on disk
  are the product; a consumer loads them where it wants them.
- **OCR is not enabled.** `REG-D08` carries `ingest: transcribe-or-skip` and is skipped with the reason
  named rather than reported as a failure.
- **`raw/` growth is unbounded.** The per-file cap bounds one pathological URL, not the total, and git
  keeps every version. `S3ObjectStore` is implemented and opt-in; Git LFS is the other option. Neither is
  needed at this size, and adopting either costs a re-derive and never a re-embed, because no path feeds
  `chunk_id`.
- **No internal tier.** A row with any `tier` other than `public` fails the build, by design.
