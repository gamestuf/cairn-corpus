# Open questions

Everything the corpus is waiting on, in one place, grouped by *what would unblock it*. The registry carries the
same items as per-row `confirm` entries, and every run report repeats them.

Reviewed 2026-10-06 against the registry (59 rows) and the newest build report, `reports/20261006T081442Z.json`:
**41 rows built, 18 skipped by design, none errored.** Two `confirm` items remain, on two rows (updated 2026-10-07). Seven rows were added 2026-10-07 — six DFARS clauses
(`REG-C07`–`C12`) and the DoD CIO CMMC FAQ (`REG-D12`); they build in the next run.

## Still open

### 1. Decisions only you can make

**1.1 The NIST CPRT JSON rows.** Decided 2026-10-08 (below). What is still open is `REG-N01`, SP 800-171 Rev. 2:
it verifies at 100% and is to be activated, but the control crosswalk reads CMMC Level 2 equivalence and Appendix D
from chunks that activating it would suppress, so it waits until the pipeline carries those identities across
(pipeline Project Plan T3.4). Its `confirm` (was the JSON produced from Rev 2 Update 1?) is answered then.

### 2. Waiting on a source

| row | waiting for |
| --- | --- |
| FAR 52.240-93 (not a row yet) | A source for the one clause: acquisition.gov has no page for it while the Overhaul is uncodified, and its text is inside the Part 52 page with every other clause. Pipeline decision P-D8 proposes a `select` naming one provision. |

### 3. Rows not activated yet

Not blockers. `REG-N09` (CSF 2.0) and `REG-N51` (SP 800-53A) are `planned`. `REG-D08` (the CMMC overview briefing,
audio) is `planned` and would need a transcript. `REG-D11` is `planned` and optional. `REG-S03` and `REG-S04` are
pointers, also `planned`.

### 4. Deferred engineering

Not registry decisions. Recorded so they are not rediscovered.

- **Embeddings are not wired up.** No build passes `--embedding`, so no `vectors.jsonl` exists and every row reports
  `embedding_skipped`. See [`status.md` §2](status.md#2-embeddings-are-not-wired-up).
- **A newer eCFR amendment is not flagged.** `REG-R01` and `REG-R02` now name their text by its eCFR `as_of` date. When
  the eCFR reports a later `as_of`, the row still fetches the latest text, but `version_slug` keeps the old date until
  someone updates it. A finding when the two differ would make this visible.
- **Repository growth.** `raw/` grows monotonically and git keeps every version. `S3ObjectStore` is implemented and
  opt-in; Git LFS is the other option. Neither is needed at the present size, and adopting either costs a re-derive,
  never a re-embed, because no path feeds `chunk_id`.

## Closed

### Decided 2026-10-06

| was | decision |
| --- | --- |
| 1.1 Redistribution of `REG-F01`, `F02`, `F03` | **Not redistributed.** `redistribute: false` is correct. Their derived text is published; the originals are archived outside the repository. |
| 1.2 `REG-R03` and `REG-R07` are one document | **The split is correct.** One Federal Register document (FR Doc. 2025-17359, 90 FR 43560), split by normativity: `R03` takes the clause text (requirement), `R07` the preamble and responses (guidance). Each keeps its own `doc_id`. |
| 1.3 Version slugs for the living regulations | **`version_slug` is the eCFR `as_of` date:** `REG-R01` `2024-12-16`, `REG-R02` `2016-12-22`. Only the slug changed, and the slug names directories, so chunk ids are **not** reissued. (The old note said they would be; that was wrong — `version`, not `version_slug`, feeds `chunk_id`.) The next build writes these rows under the new path. |
| 1.4 `REG-R02`: Part 2002 or Part 2000 | **Part 2002**, Controlled Unclassified Information. |

### Decided 2026-10-08

| was | decision |
| --- | --- |
| Rows CI can never fetch: 15 rows on `dodcio.defense.gov`, `esd.whs.mil`, `dodcui.mil` (403) and `acq.osd.mil` (TLS) report `archive_used` at warning on every build | **A dated review.** Each row carries `snapshot_reviewed`; for 90 days after it the finding is info (pipeline ADR-0016, unreleased). All 15 were compared with their publisher on 2026-10-08 and found current. |
| The NIST CPRT JSON rows (`REG-N01`–`N04`, `N10`–`N13`) | **Activate the three that verify at 100%** — SP 800-172 (`REG-N10`) and SP 800-172A Rev. 3 (`REG-N13`) now, SP 800-171 Rev. 2 (`REG-N01`) after pipeline T3.4 — and keep the other five planned, relying on the PDF. Where a JSON row is planned, the CMMC guides suppress against its PDF instead (pipeline ADR-0017). |
| `REG-R05`: the deviation numbers behind the 2026-02-01 Revolutionary FAR Overhaul index row | **DFARS Class Deviation 2026-O0025, Revision 3** is the deviation for DFARS Part 240, where former Subpart 204.73 now sits; added as `REG-R09`, and `REG-R05` is `cancelled`. |
| DFARS Class Deviation 2024-O0013, Revision 1 (`REG-R04`): waiting for a fresh copy | **The archived copy stands.** It is Revision 1, the current revision, and is reviewed with the other 14. |

### Decided 2026-10-07

| was | decision |
| --- | --- |
| CMMC FAQ (`REG-D12`): date, title, page count | **Revision 2.3 (Excerpt), July 2026**, 17 pages, "Cybersecurity Maturity Model Certification Program (CMMC) Frequently Asked Questions" — version set to 2.3, the document's own. |
| DFARS Class Deviation 2024-O0013 (`REG-R04`): is Revision 1 current? | **Yes.** |
| CMMC 101 Brief (`REG-D07`), CMMC Briefing: DoD SPRS (`REG-D10`): urls | **Supplied**: `CMMC-101-Nov2025.pdf` (version 2025-11) and `CMMC-SPRS.pdf`; both rows chunk again from the next build. |
| `REG-F02`: is there a later CAP than v2.0? | **No.** v2.0 (December 2024) is the latest edition. |
| `REG-F03`: CoPC v2.0 or v2.1a? | **v2.0** (December 2024) is the latest; there is no later version. |
| `REG-S05`: is the 2023-12-21 memorandum superseded? | **No.** It is the latest version. |

### Resolved by the corpus itself

Answered by the registry or by the text now held; each is recorded in the row's `notes`.

| was | resolved by |
| --- | --- |
| `REG-N10`: does CMMC Level 3 draw on 800-172's initial revision or Rev 3? | 32 CFR 170 (`REG-R01`) incorporates "NIST SP 800-172, … February 2021" by reference and takes the 24 Level 3 requirements from "NIST SP 800-172 Feb2021": the initial revision. |
| `REG-R07`: confirm the FR page citation | The Federal Register API resolves FR Doc. 2025-17359 to 90 FR 43560, the citation the row records. |
| `REG-R08`: 81 FR 63336 or 81 FR 63324? | 81 FR 63324, the document's first page, which is what a Federal Register citation names; the API resolves FR Doc. 2016-21665 to it. 63336 is a page inside the document. |
| `REG-S01`: record the current change number and date | The file the publisher serves reads "Effective: March 6, 2020" and carries no change notice; `version` 2020-03-06 records it. |
| `REG-C02`, `C03`, `C04`: clause edition, elimination, renumbering | Recorded in each row's `version` and `notes`: 7012 2024-05; 7019 eliminated by the RFO class deviations effective 2026-02-01; 7020 renumbered to 252.240-7997, with an alias. |

### Overtaken by later work

The 2026-09 version of this page listed 24 of 43 rows as erroring. All of those causes are gone:

- **Landing-page urls** (11 rows): every NIST PDF, the SCF workbook and the CAP now fetch the document itself.
  `REG-N07`/`N08` (800-53 and 800-53A as OSCAL) were replaced by the CPRT rows `REG-N50` (active, scoped to the
  controls the corpus's mappings reach) and `REG-N51` (planned), so the OSCAL-XML extractor question went with them.
- **Publisher 403s and TLS failures** (11 rows): all build. A publisher that refuses the pipeline's identity is fetched
  under a browser User-Agent, which is reported. Every original fetched once is archived and becomes the row's
  fallback. `REG-R04` is the one that still uses its archived copy. `REG-R05` became `register-only` (§2).
- **800-171 control text** (`REG-N01`, `N02` never supplied): the 110 requirements and 320 objectives come from the
  PDF rows `REG-N01b` and `REG-N02b`. The QA gates run and pass: the crosswalk holds 110/110 requirements and 320/320
  objectives.
- **OCR** is wired up (Tesseract): `REG-S05`, a scan, builds through it.
- **The five single-chunk regulations**: `REG-R01`/`R02` now come from the eCFR's XML (153 and 150 chunks),
  `REG-R03`/`R06`/`R07` from the Federal Register's document body (43, 421, 111).
