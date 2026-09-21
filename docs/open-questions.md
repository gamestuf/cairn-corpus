# Open questions

Everything the corpus is waiting on, in one place.

The registry already carries these as per-row `confirm` items, and every run report repeats them — but 21
items spread across 43 rows is a list nobody works through. This is the same information grouped by *what
would unblock it*, so each section has a single kind of answer.

Counts and causes below are from the last real build, `reports/35585084424-1.json`. They are what actually
happened, not an estimate.

## Where the corpus stands

**11 of 43 rows produced artifacts.** 24 errored, 8 were deliberately skipped.

| | rows | |
| --- | ---: | --- |
| Produced artifacts | 11 | `REG-R01` `R02` `R03` `C01` `C02` `C04` `C05` `C06` `F03` `R06` `R07` |
| Errored | 24 | see below |
| Skipped by design | 8 | `C03` superseded · `D07` `D10` register-only · `D08` needs OCR · `D11` optional · `S02` `S03` `S04` pointers |

Every error falls into one of four causes, and each has a different fix:

| cause | rows | what fixes it |
| --- | ---: | --- |
| URL is a landing page, not the document | 11 | the real asset URL |
| Publisher returns HTTP 403 to CI | 9 | a committed copy in `sources/` |
| TLS handshake fails | 2 | a committed copy in `sources/` |
| Local file never supplied | 2 | the file |

None of this is a pipeline defect — every one of the 24 is a counted, attributed line in the report, which
is invariant 2 working. They are inputs the corpus does not yet have.

---

## 1. Decisions only you can make

No external access needed. Each is blocking something concrete.

### 1.1 Redistribution of three non-federal documents

`REG-F01` (SCF catalog), `REG-F02` (Cyber AB CAP), `REG-F03` (Cyber AB CoPC) set `redistribute: false`, so
their original files are archived outside this repository. Their derived text is published as usual.

Forty rows are United States federal government work and carry no copyright. These three come from private
bodies under their own terms, and republishing a document is a claim about its licence that the pipeline
should not make on a publisher's behalf.

**To decide:** whether each publisher's terms permit republishing the original file here. A yes is one
field per row.

> `REG-F02` was held on my initiative, not on instruction — the ask named the SCF and the CoPC. It is the
> same publisher under the same terms as the CAP, which is why. If that reasoning does not hold, it is a
> one-line change.

### 1.2 `REG-R03` and `REG-R07` are one document

Both resolve to FR Doc. 2025-17359 (90 FR 43560), split by normativity:

| row | authority | takes |
| --- | --- | --- |
| `REG-R03` | `authoritative` | amended clause text → **requirement** |
| `REG-R07` | `corroborating` | preamble, comment responses → **guidance** |

That split was already implied by the registry's own fields; nobody had stated it. One fetch serves both,
which the report notes as reused bytes — the same idiom `REG-N01`/`REG-N01b` uses.

**To decide:** whether the split is intended, and whether the two should share a `doc_id` with distinct
`part`s rather than carrying separate ones (`dfars-case-2019-d041` and `90-fr-43560`).

### 1.3 Version slugs for the two living regulations

`REG-R01` and `REG-R02` read `version_slug: final-rule`, which does not say which text a chunk came from.
The eCFR now resolves each part to its own most recent amendment date, reported as `as_of`.

**To decide:** whether to pin `version_slug` to that date. **Changing it reissues the row's chunk ids**, so
it costs a re-embed, not just a re-derive. That is why it is a question and not a cleanup.

### 1.4 `REG-R02`: Part 2002 or Part 2000

The source list said Part 2000 (ISOO classified-information procedures). Part 2002 (Controlled Unclassified
Information) was assumed, because that is the CUI rule and this is a CUI corpus. **Confirm.**

---

## 2. Rows blocked on a real source URL

These 11 rows point at a publication landing page. The fetch succeeds, returns HTML, and the magic-byte
check rejects it — correctly, because a landing page is not the document.

| row | document | current url |
| --- | --- | --- |
| `REG-N01b` | SP 800-171 r2u1 (PDF) | `csrc.nist.gov/pubs/sp/800/171/r2/upd1/final` |
| `REG-N02b` | SP 800-171A (PDF) | `csrc.nist.gov/pubs/sp/800/171/a/final` |
| `REG-N03` | SP 800-171 r3 | `csrc.nist.gov/pubs/sp/800/171/r3/final` |
| `REG-N04` | SP 800-171A r3 | `csrc.nist.gov/pubs/sp/800/171/a/r3/final` |
| `REG-N05` | SP 1318 | `nist.gov/publications/protecting-...` |
| `REG-N06` | SP 1352 | `nist.gov/news-events/news/2026/09/...` |
| `REG-N07` | SP 800-53 r5 (OSCAL) | `csrc.nist.gov/pubs/sp/800/53/r5/upd1/final` |
| `REG-N08` | SP 800-53A r5 (OSCAL) | `csrc.nist.gov/pubs/sp/800/53/a/r5/final` |
| `REG-N09` | CSF 2.0 | `nist.gov/cyberframework` |
| `REG-F01` | SCF catalog | `securecontrolsframework.com/free-content/scf-download` |
| `REG-F02` | CMMC CAP | `cyberab.org/` |

**To do:** follow each landing page to the asset it links and record that url instead.

Two notes that may save time. NIST publishes its PDFs on `nvlpubs.nist.gov` under a predictable path, and
the OSCAL renderings of SP 800-53 live in NIST's own `oscal-content` repository rather than on `csrc` at
all — so `REG-N07` and `REG-N08` may want a url in a different host entirely. **I have not verified any
candidate url**: this session cannot reach those hosts, and a guessed url in the registry is worse than an
honest landing page, because it would fail somewhere less obvious.

`REG-F02`'s url is `cyberab.org/` — the site root. Like `REG-R03` before it, that one was never going to
resolve to a document.

---

## 3. Rows blocked on publisher access

The publisher refuses CI, or the handshake fails. This is what `sources/` and `fallback_path` exist for.

| cause | rows |
| --- | --- |
| HTTP 403 | `REG-S01` `REG-D01` `REG-D02` `REG-D03` `REG-D04` `REG-D05` `REG-D06` `REG-D09` `REG-S05` |
| TLS handshake | `REG-R04` `REG-R05` |

Seven of the nine 403s are `dodcio.defense.gov` — the CMMC model overview, all three assessment guides,
both scoping guides, and the NIST-alignment briefing. That is most of the CMMC document set.

**To do:** fetch each from a browser, commit it under `sources/{reg_id}/`, and point the row at it with
`fallback_path`. See [`sources/README.md`](../sources/README.md) for when a copy belongs there and what
using one costs.

> Once the pending PRs land this gets cheaper over time, not more expensive: an original fetched
> successfully **once** is committed to `raw/` and becomes that row's fallback for every later run. A
> hand-committed copy is then only needed for a document that has *never* been reachable — which is
> exactly this list, and no more of it.

`REG-R05` additionally declares `pdf` while its url ends `.html`, so it has two problems.

---

## 4. Rows blocked on a document nobody has supplied

| row | waiting for |
| --- | --- |
| `REG-N01` | the SEP-authored 800-171 Rev. 2 control JSON, at `sources/input/800-171r2.controls.json` |
| `REG-N02` | the SEP-authored 800-171A objective JSON, at `sources/input/800-171A.objectives.json` |
| `REG-D07` | a canonical publisher url for the CMMC 101 Brief; then set `ingest` back to `chunk` |
| `REG-D10` | a canonical publisher url for the SPRS briefing; then set `ingest` back to `chunk` |

`REG-N01` and `REG-N02` are the two rows the whole 800-171 chunking design is built around — one chunk per
field per control, verified against the paired PDF. They are JSON-primary by design, so they error rather
than falling through to their url, which is a landing page. Until those files exist the corpus has no
800-171 control text at all.

---

## 5. Editions to confirm at ingest

Lower stakes: each row works, but nobody has confirmed it is the current edition.

| row | question |
| --- | --- |
| `REG-N01` | was the JSON produced from Rev 2 Update 1 (2021-01-28 errata)? |
| `REG-C02` | record the clause edition date shown on acquisition.gov |
| `REG-C03` | confirm elimination and effective date |
| `REG-C04` | confirm renumbering and text |
| `REG-S01` | record current change number and date |
| `REG-D07` | record version and date from the document |
| `REG-F02` | a rule-alignment update was announced for Dec 2025 |
| `REG-F03` | a v2.1a has circulated; which is current? |
| `REG-R04` | confirm current revision, and that it is not rescinded or superseded |
| `REG-R05` | identify the specific deviation number(s) covering 204.73 |
| `REG-R07` | confirm the FR page citation |
| `REG-S05` | confirm url, and whether later DoD CIO guidance supersedes |

---

## 6. Deferred engineering

Not registry decisions. Recorded so they are not rediscovered.

**OCR is not wired up.** `REG-D08` (CMMC Overview Briefing, audio) carries
`ingest: transcribe-or-skip` and is skipped with the reason named. See `extractors/ocr/README.md` in the
pipeline.

**The XML extractor is CFR-shaped.** `CfrXmlExtractor` understands the CFR's `DIV1`..`DIV9` vocabulary and
keeps the text of anything else — losing structure, never content. It is currently the *only* XML
extractor, so a row declaring `oscal-xml` would land there. If `REG-N07`/`REG-N08` end up on OSCAL-XML
rather than OSCAL-JSON, that wants its own extractor resolved ahead of it.

**Repository growth.** `raw/` grows monotonically and git keeps every version. The per-file publishing cap
bounds one pathological URL, not the total. `S3ObjectStore` is implemented and opt-in; Git LFS is the other
option. Neither is needed at the registry's present size, and adopting either costs a re-derive and never a
re-embed, because no path feeds `chunk_id`.
