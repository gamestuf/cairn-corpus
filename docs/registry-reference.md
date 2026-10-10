# Registry field reference

The registry is the single input. `registry/public.json` in `cairn-corpus` is the real one;
`samples/public.min.json` here is a synthetic offline copy used by the dev loop and the tests.

Every row in this registry is public-tier. A row with any `tier` other than `public` fails stage 0.

## File header

| Field | Required | Meaning |
| --- | --- | --- |
| `$schema` | no | Schema identifier, e.g. `documents.registry.v1`. |
| `registry` | no | Which registry this file is, e.g. `public`. |
| `description` | no | What the file is for. |
| `generated` | no | When the registry was last edited. |
| `enums` | no | Enum domains the registry declares. Where a domain is one the pipeline also knows, the two must agree — a registry that widens an enum without a matching code change is a stage-0 failure. `tier` is exempt: the header declares the whole vocabulary shared across registries (`public`, `internal`, `restricted`) while this file may hold only public rows, and the per-row check is what enforces that. |
| `rules` | no | An array of authored prose rules, echoed into the run report. |
| `documents` | **yes** | The rows. |

## Row fields

### Identity

| Field | Required | Meaning |
| --- | --- | --- |
| `id` | **yes** | Stable registry id, e.g. `REG-N01`. Addresses the document across runs and appears in every artifact path and node id. Must be unique. |
| `document_number` | no | The publisher's own number, e.g. `NIST SP 800-171 Rev. 2`. |
| `title` | **yes** | Human-readable title. |
| `version` | no | Document version. Part of every artifact path and every `chunk_id`. **When absent, artifacts use the literal `unversioned`** and stage 0 reports the row; recording the real version later starts a new version directory, which is the correct outcome. |
| `notes` | no | Free prose about the row: what the document is for in the corpus, how it is meant to be chunked, anything an author needs to know. Echoed into the manifest and read by nobody, which is the point — it is the one place for reasoning that the pipeline does not act on. `source`, `publisher`, `intake`, `chunking`, `published` and `corpus_role` were folded into this field: six documentation-only fields that no code consumed, where one suffices. Dates that `version` does not already carry were kept here verbatim. |
| `aliases` | no | Alternative names. Also matched by the term extractor, so an alias appearing in text becomes a `term:` node. |

### Source

| Field | Required | Meaning |
| --- | --- | --- |
| `url` | conditional | Remote source, and the source of record. |
| `local_path` | conditional | An **authoritative** local source. When set, the row is built from this file and no fetch happens. Used by the JSON-primary rows. |
| `fallback_path` | no | A **committed copy**, used only when the live fetch fails. See below. |
| `snapshot_reviewed` | no | `yyyy-MM-dd`: the date a person last compared the row's stored copy with its `url` in a browser and found it to be the current edition. For 90 days after it, a build from that copy is reported at `info` rather than `warning`. See below. |
| `shares_fetch_with` | no | Registry id whose fetched bytes this row reuses, so a paired PDF is not fetched twice. |
| `format` | **yes** for active chunked rows | What the bytes are — exactly one of `pdf`, `html`, `xml`, `json`, `xlsx`, `docx`, `pptx`, `md`, `audio`. The magic-byte check holds the payload to it. Anything else — prose naming two sources, a processing path such as `pdf-scan` — fails stage 0 (`format_not_a_byte_type`): a verifying PDF is named by `qa.verify_against`, and the path by `format_profile`. |
| `format_profile` | no; **`general` when absent** | How the bytes become text and chunks — one path per row (pipeline ADR-0014). `general` for every format; and `nist-companion`, `cmmc-guide`, `scan` (PDF), `cprt-json`, `org-policy-json`, `org-ssp-json`, `org-reference-json` (JSON), `regulation-html` (HTML), `ecfr-xml` (XML), `scf-workbook` (XLSX), `transcript` (audio). A profile that is not a path for the row's `format` fails stage 0, on every row whatever its `status` or `ingest`, and so does a profile on a row with no `format`. It chooses the chunker and the `--commands` entry that runs the row, and it is the last directory of the row's path. |

A row with `ingest: chunk` needs one of `url`, `local_path` or `shares_fetch_with`. Stage 0 fails otherwise.

### Fallback copies

Some publishers cannot be fetched from CI. On 2026-10-08, 14 rows on `dodcio.defense.gov`, `esd.whs.mil`
and `dodcui.mil` return **HTTP 403** to GitHub's runners, and 2 rows on `acq.osd.mil` fail the **TLS
handshake**. When the live fetch fails, the pipeline uses, in order:

1. **The copy an earlier run archived** under `raw/{tier}/{org}/{doc_id}/{version_slug}/{format_profile}/`.
   Any document fetched successfully once is its own fallback from then on.
2. **A committed copy**: the file the row names in `fallback_path`, if it names one (resolved against the
   registry file's directory, then this repository's root; no row uses it today), otherwise the one file found
   by convention at `fallback/{tier}/{org}/{doc_id}/{version_slug}/{format_profile}/`. That folder must hold
   exactly one file; its name does not matter. This is how a document that has *never* been reachable from CI
   gets in. The path slugs must match the row's, so a copy filed under another version is invisible by design.

```json
"url": "https://dodcio.defense.gov/Portals/0/Documents/CMMC/FAQsv6.pdf"
```

with its copy at `fallback/public/dod-cio/cmmc-faq/2.3/general/FAQsv6.pdf`.

Three rules make this safe to rely on:

1. **The url is always tried first.** A reachable document is never replaced by its snapshot.
2. **Using a fallback is never silent.** It raises a finding naming why the fetch failed: `archive_used` for
   the copy an earlier run archived under `raw/`, `fallback_used` for a committed copy. The finding is a
   **warning**, and sets `warnings: true` on the run, unless the row's `snapshot_reviewed` date is 90 days old
   or less. Then it is **info**, and names the review date (pipeline ADR-0016). A stale review, a date after
   the run, or a 404/410 from the publisher keeps the warning. To renew a review, open the `url` in a
   browser, compare it with the copy, and update the date.
3. **The manifest records `origin`** — `url`, `fallback` or `local` — plus `fallback_reason`. A snapshot of
   unknown age and a document retrieved from the publisher today are different claims, and a consumer must
   be able to tell them apart.

`fallback_path` is deliberately *not* `local_path`. `local_path` is the source of record and skips fetching
entirely; `fallback_path` is a stand-in for something whose source of record is still the publisher.

A fallback is the right answer to a 403 or a TLS failure. It is the **wrong** answer to
`payload is 'html', not 'pdf'` — that means the `url` points at a landing page rather than the asset, and
the fix is to correct the url so the document stays current.

### Shared fetches

Two rows naming the **same `url`** are fetched once and the bytes reused — this is how REG-N01 and REG-N01b
are "one fetch, two registry rows" without a field saying so. `shares_fetch_with` names the relationship
explicitly where the URLs differ.

### Classification

| Field | Values | Meaning |
| --- | --- | --- |
| `tier` | `public` in this registry; the vocabulary is `public`, `private`, `training-info`, `other-info` | Must equal the run's lane. The corpus build runs the `public` lane, so anything else here fails the run (invariant 9). The other two tiers are built locally from their own repositories and never appear in this one (pipeline ADR-0008). Copied to every chunk and to `manifest.tier`. |
| `visibility` | `public` in this registry; the vocabulary, declared in `enums.visibility`, is `public`, `internal`, `technical`, `cmmc-program`, `security` | Who inside the organisation may see the document's content. A different axis from `tier`: tier says which corpus a row is built into, visibility which readers a consumer may show it to. **Absent means `security`**, the narrowest audience, and is reported as a warning. A public-tier row must state `public` — anything else, the default included, fails the run, because a public repository cannot restrict what it shows (pipeline ADR-0015). Copied to every chunk and to its manifest entry. |
| `authority` | **closed set** declared in `enums.authority`: `authoritative-apex`, `authoritative`, `authoritative-site`, `authoritative-delta`, `corroborating`, `derived`, `reference-only`, `never-cite`, `pointer`, `none` | Who publishes it, and whether it may be cited at all — the publisher's standing, **never** how relevant the document is: NIST's 800-171 Rev 3 and 800-53 and DoD CIO's Level 1 guides are `authoritative`. `corroborating` is a recognised third party (SCF, the Cyber AB); `derived` the publisher's own summary or briefing. A value outside the declared set fails the run, because it would otherwise fall into whichever bucket a consumer defaults to. Copied to every chunk and the manifest entry. |
| `relevance` | **closed set** declared in `enums.relevance`: `requirement`, `guidance`, `supporting`, `example` | The part the document plays for the corpus. `requirement`: states the obligation in force. `guidance`: how to meet, assess or scope it. `supporting`: related, comparison or background — another revision, a mapping catalogue, a briefing, a pointer. `example`: a template, form or sample. Required on every row of a registry that declares the vocabulary; a missing or unknown value fails the run. Retrieval ranks and filters on it; citability stays with `authority`, and who may see it with `visibility`. Copied to every chunk, the manifest entry and the graph's `Document` node. |
| `framework_rev` | number or string | Framework revision, e.g. `2` or `"3"`. Both spellings are accepted; it is carried on chunks as a string. Checked by a QA gate (invariant 7). |
| `scf_edition` | `legacy`, or an SCF edition such as `2026.3` | The SCF numbering every SCF id the row cites is written in. SCF 2026.3 renumbered its catalogue and reused old numbers for other controls, so an id is read only in its declared edition: a `legacy` row's ids are translated to 2026.3 through the SCF id map (`scf_ids` holds the 2026.3 id, `scf_legacy_ids` the id as written), and without the map `scf_ids` stays empty rather than hold an id in the wrong numbering. **Required** on `scf-workbook` (`REG-F01`: `2026.3`) and on the profiles that exist to cite SCF ids; any other value fails the run. Absent: the row cites no SCF ids. |
| `subset` | `sp800-53-scope`, or absent | Keep only part of the row's content. `sp800-53-scope` (on `REG-N50`): only the 800-53 controls the corpus's mappings reach — 800-171 Rev 2 and Rev 3's own tables and source controls, the SCF controls mapping 800-171, CMMC, DFARS 252.204-7012, FAR or ITSP-10-171, and the Rev 5 successors of any withdrawn — computed in each build, never a kept list. The kept list is written beside the chunks as `subset.json`. Any other value fails the run. Absent: the whole document. |
| `scf_ids_from_text` | `true`, or absent | The document's prose cites SCF controls by id, so ids found in its text are recorded, read in `scf_edition` (which it requires). Absent: only ids a parser reads from a structured field are recorded — the SCF id shape is shared with process codes and form numbers, and read in the wrong document it joins to a control the text never meant. Ids found and not recorded are counted in the run report. |
| `status` | `active`, `missing`, `cancelled`, `planned`, `superseded` | Only `active` rows are acquired; the rest are `skipped` with the status as the reason. **Defaults to `active` when absent**, and stage 0 reports the row. |
| `ingest` | `chunk`, `register-only`, `pointer`, `optional` | What to do with the row. **Defaults to `chunk` when absent**, and stage 0 reports the row. |

Defaults are assumptions, and invariant 2 says assumptions are visible: every row relying on one produces a
`registry_default_applied` finding naming the field and the value assumed. A row using a value the pipeline
implements but the header does not declare produces `registry_enum_undeclared` — a documentation gap in the
registry, reported rather than fatal.

`ingest` in detail:

- **`chunk`** — fetch, extract, chunk, embed.
- **`register-only`** — record the row; do not fetch. Outcome `skipped`.
- **`pointer`** — the row names a source held elsewhere. Outcome `skipped`.
- **`optional`** — skipped unless named by `--only`.
`ingest` says whether a row takes part, never how it is read. A scanned PDF is **`format: pdf`,
`format_profile: scan`**: its bytes are a PDF and the magic-byte check says so, while the profile keeps the
text extractor away — a scan handed to it returns a few characters and reports success. A row whose profile
has no implementation yet keeps `ingest: chunk`, because it should chunk once its path exists, and is `skipped`
before it is fetched with the path named; every declared path is implemented today but `json/general`, which is
unsupported by decision: a JSON row declares `cprt-json`, `org-policy-json`, `org-ssp-json` or
`org-reference-json` (an organisation's glossary, acronym list or cited-document list), and a `general` one
is skipped with that reason. An `audio` row with
`format_profile: transcript` names a transcript as its source (`url`, or a committed copy): plain text, WebVTT or
SubRip, kept word for word. The pipeline does not transcribe.

A `general` PDF that extracts to fewer than 200 characters a page is refused as `scan_suspected` rather than
published thin. A document that really is that sparse sets `qa.min_chars_per_page`.

### Path slugs

These decide where a document's artifacts live. They are **path-only** — none of them feeds `chunk_id` or a
node id, so changing them costs a re-derive and never a re-embed.

| Field | Meaning |
| --- | --- |
| `org` | The organization that **issued** the document, as a slug: `nist`, `dod-cio`, `dfars`. Who wrote the document, not who may read it. |
| `doc_id` | The publisher's own identifier: `sp-800-171`, `252.204-7012`, `cmmc-assessment-guide-l2`. Adopting the publisher's id rather than inventing one means the path is the string people already search for. |
| `version_slug` | Path-friendly revision: `r2u1`, `2.13`, `2021-11`. Separate from `version`, which feeds `chunk_id`. It must name a revision: `current` on an `ingest: chunk` row fails stage 0 (`version_slug_not_revision`), because the next revision would be written over this one's directory. Pointer, register-only and optional rows may keep it. |
| `source_api` | Resolve this row through a publisher API instead of fetching `url` directly. `{"provider": "federal-register", "document_number": "2025-17359"}`, or `search` conditions in place of a document number. The `ecfr` provider takes `params`: `{"title": "32", "part": "170"}`, plus an optional `date` — omit it and the part is pinned to its own most recent amendment, which is reported as `as_of`. `url` stays set to the page a reader should be sent to — this says how to find the text behind it. Carries **no credential field**: a provider names the environment variable it reads, so no registry file can hold a secret. A search matching anything but exactly one document is an error listing the candidates, never a pick. |
| `redistribute` | `true` asserts that the original file may be republished in this repository, which is where it is then archived. **Absent means no**: republishing a document is a claim about its licence, and the pipeline does not make that claim on a publisher's behalf. A withheld row is still fetched, archived and chunked, and its `derived/` artifacts are published exactly as any other row's — only the original file stays out, and the run report names it. |

Artifacts land at:

```text
derived/{tier}/{org}/{doc_id}/{version_slug}/{format_profile}/
raw/{tier}/{org}/{doc_id}/{version_slug}/{format_profile}/          # published originals
raw-withheld/{tier}/{org}/{doc_id}/{version_slug}/{format_profile}/ # archived outside this repository
fallback/{tier}/{org}/{doc_id}/{version_slug}/{format_profile}/     # committed copies, found by convention
```

Tier is the root so a public corpus is visibly public — anything outside `public/` in this repository is
wrong at a glance, and a merge with any other corpus is a union of disjoint subtrees. Version sits last so
revisions of one document are siblings: `nist/sp-800-171/r2u1`, `/r3`, later `/r4`. The profile is last, so
two forms of one document at one revision sit side by side: `r2u1/cprt-json` and `r2u1/nist-companion`.

**Two rows resolving to the same directory fails stage 0**, like a duplicate id: it means the same form of the
same document twice, and the second would otherwise overwrite the first's artifacts while the run reported
both as ingested.

Rows without these fields fall back to `{tier}/{reg_id}/{version}/{format_profile}` and are reported, so the registry can be
migrated a row at a time.

### Content rules

| Field | Meaning |
| --- | --- |
| `normativity_map` | **Normativity-first**: one entry per normativity, listing the block types that carry it — `{"requirement": ["statement"], "guidance": ["discussion", "800-53 mapping"], "example": []}`. Inverted once at load. The block-type-first spelling (`{"statement": "requirement"}`) is also accepted. Applied structurally, never inferred (invariant 8). An unmapped block type defaults to `guidance`, the conservative choice: labelling guidance as a requirement would invent an obligation the source does not state. Values: `requirement`, `guidance`, `example`, and `assertion` for a claim about how a named system satisfies an obligation — a system security plan's narrative, which derives `content_class: contextual` because it is not the obligation (ADR-0013). For `regulation-html` and `ecfr-xml` rows the block types are the regulation parser's: `clause text`, `provision text`, `rule text`, `definition`, `prescription`, `amendatory instruction`, `regulatory note`, `appendix text`, `preamble`, `comment`, `response`. |
| `sections.include` | Section names or clause numbers to keep. |
| `sections.exclude` | `[{section, covered_by}]`. Dropped, counted, and attributed to the rows that carry the material instead. `covered_by` is one registry id or a list of them — a section can restate more than one document. A row may exclude without an include list; everything else is then kept. |
| `language` | ISO 639-1 code, e.g. `en`. **Optional — `en` is assumed.** Every document this corpus names is published by a US federal body or a US standards organisation, so English is the default rather than something each row restates. A row may name more than one, separated by `/` — `en/fr` for genuinely bilingual material — and a paragraph in any declared language is on-language. Under `language_policy: single` it must name exactly one; `en/fr` is a registry error there. |
| `language_policy` | `keep` (filter not engaged), `drop-other` (off-language paragraphs removed and counted), `report-only` (kept and reported), `single` (the row asserts its language; no identification, no language findings — see below). |
| `multilingual` | `true` where the document genuinely mixes languages. Optional; **false** is assumed, and no row sets it today. It does not switch the filter off: off-language prose is still dropped or reported per `language_policy` either way. What it changes is whether an *ambiguous* paragraph is worth a line in the report — in a monolingual document that is noise, and in a genuinely multilingual one it is the point. Paragraphs with too little prose to judge, and ambiguous ones in a monolingual row, are counted as `paragraphs_language_unjudged` rather than listed. |
| `select` | Which part of its source is this row's. On an organisation export (`org-policy-json`, `org-ssp-json`): the records, as field/value pairs all of which must match, `{"domain_key": "AST"}` (ADR-0012). On a `pdf/general` row: `{"pages": "16,27-28"}`, pages and inclusive ranges, 1-based; every page is still extracted, so `qa` page counts check the whole document. On a `regulation-html` row: `{"provision": "52.240-93"}`, the provision whose heading, and everything beneath it, is the row (ADR-0019). Each path takes only its own key, and stage 0 refuses `select` on any other path, because the row would otherwise be built from the whole source while appearing to select part of it. A selector matching nothing — no record, no heading, a page past the end — is an error finding, not an empty document. Pages compose with `sections`, which drop what shares a selected page. |
| `fields.include` | Allow-list for structured sources. Empty means "everything not dropped". |
| `fields.drop` | Fields removed from structured sources. Every drop is a report line. |

Where a row has an include list, a section matched by neither list is **kept** and reported as `unclassified_section`. A section named on both
lists fails stage 0. See ADR-0004.

#### `language_policy: single`

Two of the four policies identify paragraphs and two do not. `drop-other` and `report-only` run the detector;
`keep` and `single` do not.

The difference between the two that do not is what the row is saying. `keep` says *do not filter this
document* — a reason not to look. `single` says *this document is in the language I declared* — an assertion,
and the pipeline takes it at its word: no identification runs and **no language finding is reported for the
row at all**, including the `paragraphs_language_unjudged` count, because a paragraph nobody asked about is
not an open question.

That assertion is checkable, which is the point of the value existing:

| `language` under `single` | Result |
| --- | --- |
| `en`, `fr`, or any one code | Used as declared; no detection |
| `en/fr`, `fr/en`, or any value naming two | **Stage 0 error `language_not_single`; the run stops** |

The error matters because nothing downstream would catch it. Under `single` no paragraph is ever compared
against the declared language, so the row's assertion is never tested and a corpus would record a language
that is not one. Under `drop-other` or `report-only`, `en/fr` is a legitimate and useful value — both
languages are on-language and only a third is reported (ADR-0011) — so the check applies to `single` only,
where naming two contradicts what the policy asserts.

Use `single` for a body of material known to be in one language, where identification only generates findings
nobody can act on: the whole `training-info` registry is `single`/`en`. Use `report-only` where the language
is an open question.

### QA

**The coverage and verification expectations live in pipeline code**, in
`Cairn.Core/Registry/PipelineExpectations.cs`, not in the registry. The registry states them only in
English — REG-N01's `notes` say "110-control coverage", REG-N02's say "320-objective coverage" — and the
machine-readable form is held alongside the gates that enforce it, so changing what the pipeline enforces is
a pipeline change with a test and a release behind it.

Held there today:

| Row | Expectation | Verified against | Built from |
| --- | --- | --- | --- |
| `REG-N01` | 110 requirements | `REG-N01b` | `input/800-171r2.controls.json` |
| `REG-N02` | 320 assessment objectives | `REG-N02b` | `input/800-171A.objectives.json` |

A row may still carry a `qa` block, which **overrides** the code-side entry. That is the escape hatch for a
revision that changes a count without waiting for a pipeline release.

| Field | Meaning |
| --- | --- |
| `expected_requirements` | Exact count of distinct controls yielding a `statement` chunk. |
| `expected_objectives` | Exact count of `determination` chunks. |
| `min_pages` / `max_pages` | Page-count bounds for paged sources. |
| `verify_against` | Registry id of the PDF row whose extracted text this row's statements are verified against. That row must be `format_profile: nist-companion`, and every companion must be some row's target. |
| `suppress_against` | Registry ids of the NIST rows whose statements a `cmmc-guide` row must not repeat: the `cprt-json` row, or its `nist-companion` PDF while the JSON row is planned (pipeline ADR-0017). **Required on that profile**, and never inferred: a guide quoting 800-171 r2 names the r2 rows. |
| `min_chars_per_page` | Lowers the 200-characters-a-page floor below which a PDF is refused as `scan_suspected`, for a document that really is that sparse. |
| `verify_threshold` | Fraction of statements that must match. Defaults to `1.0`. |

A JSON-primary row whose authored file is absent is marked `error` with the expected path named, and the run
continues — it is not silently rebuilt from its `url`, which points at a landing page rather than the JSON.

A failed gate quarantines the document: the prior version stays current and nothing half-extracted is
published.

### Open questions

`confirm` is a list of free-text items surfaced in every run report and carried into the manifest. They are
open questions about the source, not defects: they never block a run, and they keep appearing until the
registry drops them.

## Worked example

```json
{
  "id": "REG-N01",
  "document_number": "NIST SP 800-171 Rev. 2",
  "source": "NIST",
  "publisher": "csrc.nist.gov",
  "title": "Protecting CUI in Nonfederal Systems and Organizations",
  "local_path": "input/800-171r2.controls.json",
  "version": "Rev. 2",
  "tier": "public",
  "authority": "NIST",
  "framework_rev": 2,
  "status": "active",
  "ingest": "chunk",
  "format": "json",
  "format_profile": "cprt-json",
  "chunking": "One chunk per field per control.",
  "normativity_map": {
    "requirement": ["statement"],
    "guidance": ["discussion", "800-53 mapping"],
    "example": []
  },
  "language": "en",
  "language_policy": "keep",
  "fields": { "drop": ["editor_note"] },
  "qa": { "expected_requirements": 110, "verify_against": "REG-N01b" },
  "confirm": ["Confirm the discussion text tracks r2 and not r3."]
}
```

