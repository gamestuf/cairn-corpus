# Registry field reference

The registry is the single input. `registry/public.json` in `cairn-corpus` is the real one;
`samples/public.min.json` here is a synthetic offline copy used by the dev loop and the tests.

There is no `internal.json` in either repository. Internal-tier content lives on the on-prem side, in
GitLab. A row with any `tier` other than `public` fails stage 0.

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
| `source` | no | Publishing body as the registry groups them, e.g. `NIST`, `DoD CIO CMMC`. |
| `publisher` | no | Where it is published, e.g. `csrc.nist.gov`. |
| `version` | no | Document version. Part of every artifact path and every `chunk_id`. **When absent, artifacts use the literal `unversioned`** and stage 0 reports the row; recording the real version later starts a new version directory, which is the correct outcome. |
| `published` | no | Publication date, free text (`2020-02 (updated 2021-01-28)` is a real value). |
| `corpus_role` | no | What the document is for in the corpus. Carried to the manifest; not used for processing. |
| `intake` | no | How the row is picked up, e.g. `scheduled`. |
| `notes` | no | Operator notes. Echoed into the manifest. |
| `aliases` | no | Alternative names. Also matched by the term extractor, so an alias appearing in text becomes a `term:` node. |

### Source

| Field | Required | Meaning |
| --- | --- | --- |
| `url` | conditional | Remote source, and the source of record. |
| `local_path` | conditional | An **authoritative** local source. When set, the row is built from this file and no fetch happens. Used by the JSON-primary rows. |
| `fallback_path` | no | A **committed copy**, used only when the live fetch fails. See below. |
| `shares_fetch_with` | no | Registry id whose fetched bytes this row reuses, so a paired PDF is not fetched twice. |
| `format` | **yes** for active chunked rows | `pdf`, `html`, `json`, `oscal-json`, `docx`, `xlsx`, `xml` — or prose naming more than one, e.g. `json (SEP-authored control file) + pdf (verification, pages)`. Recognised tokens are extracted in order: **the first is the format the row is chunked from**, and the rest are companions it is verified against. The magic-byte check accepts a payload matching any declared token. |

A row with `ingest: chunk` needs one of `url`, `local_path` or `shares_fetch_with`. Stage 0 fails otherwise.

### Fallback copies

Some publishers cannot be fetched from CI: at the time of writing, 9 registry rows return **HTTP 403**
(the publisher blocks the runner) and 2 fail the **TLS handshake**. `fallback_path` points at a copy of the
document committed to this repository, which the pipeline uses *only when the live fetch fails*:

```json
"url": "https://dodcio.defense.gov/.../AssessmentGuideL2v2.pdf",
"fallback_path": "sources/REG-D03/AssessmentGuideL2v2.pdf"
```

Paths resolve against the registry file's directory first, then this repository's root — so the convention
is `sources/{reg_id}/{filename}` at the top level.

Three rules make this safe to rely on:

1. **The url is always tried first.** A reachable document is never replaced by its snapshot.
2. **Using a fallback is never silent.** It raises a `fallback_used` warning naming why the fetch failed,
   and sets `warnings: true` on the run.
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
| `tier` | `public` | Anything else fails the run (invariant 9). |
| `authority` | **closed set** declared in `enums.authority`: `authoritative-apex`, `authoritative`, `authoritative-site`, `authoritative-delta`, `corroborating`, `derived`, `reference-only`, `never-cite`, `pointer`, `none` | Who publishes it, and whether it may be cited at all. A value outside the declared set fails the run, because it would otherwise fall into whichever bucket a consumer defaults to. Copied to every chunk. |
| `framework_rev` | number or string | Framework revision, e.g. `2` or `"3"`. Both spellings are accepted; it is carried on chunks as a string. Checked by a QA gate (invariant 7). |
| `status` | `active`, `missing`, `cancelled`, `planned`, `superseded` | Only `active` rows are acquired; the rest are `skipped` with the status as the reason. **Defaults to `active` when absent**, and stage 0 reports the row. |
| `ingest` | `chunk`, `register-only`, `pointer`, `transcribe-or-skip`, `optional` | What to do with the row. **Defaults to `chunk` when absent**, and stage 0 reports the row. |

Defaults are assumptions, and invariant 2 says assumptions are visible: every row relying on one produces a
`registry_default_applied` finding naming the field and the value assumed. A row using a value the pipeline
implements but the header does not declare produces `registry_enum_undeclared` — a documentation gap in the
registry, reported rather than fatal.

`ingest` in detail:

- **`chunk`** — fetch, extract, chunk, embed.
- **`register-only`** — record the row; do not fetch. Outcome `skipped`.
- **`pointer`** — the row names a source held elsewhere. Outcome `skipped`.
- **`optional`** — skipped unless named by `--only`.
- **`transcribe-or-skip`** — needs OCR. The OCR extractor is not wired up, so these are `skipped` with that
  reason stated. See `extractors/ocr/README.md`.

### Path slugs

These decide where a document's artifacts live. They are **path-only** — none of them feeds `chunk_id` or a
node id, so changing them costs a re-derive and never a re-embed.

| Field | Meaning |
| --- | --- |
| `org` | The organization that **issued** the document, as a slug: `nist`, `dod-cio`, `dfars`. Means the same thing in the public and internal trees — who wrote it, not who may read it. |
| `doc_id` | The publisher's own identifier: `sp-800-171`, `252.204-7012`, `cmmc-assessment-guide-l2`. Adopting the publisher's id rather than inventing one means the path is the string people already search for. |
| `version_slug` | Path-friendly version: `r2u1`, `2.13`, `current`. Separate from `version`, which feeds `chunk_id` — a living clause reads `current` in the tree while its identity stays `unversioned`. |
| `part` | Separates renditions of one document at one revision: `controls-json` vs `pdf`. Only needed where two rows would otherwise collide. |
| `source_api` | Resolve this row through a publisher API instead of fetching `url` directly. `{"provider": "federal-register", "document_number": "2025-17359"}`, or `search` conditions in place of a document number. `url` stays set to the page a reader should be sent to — this says how to find the text behind it. Carries **no credential field**: a provider names the environment variable it reads, so no registry file can hold a secret. A search matching anything but exactly one document is an error listing the candidates, never a pick. |
| `redistribute` | `true` asserts that the original file may be republished in this repository, which is where it is then archived. **Absent means no**: republishing a document is a claim about its licence, and the pipeline does not make that claim on a publisher's behalf. A withheld row is still fetched, archived and chunked, and its `derived/` artifacts are published exactly as any other row's — only the original file stays out, and the run report names it. |

Artifacts land at:

```
derived/{tier}/{org}/{doc_id}/{version_slug}[/{part}]/
raw/{tier}/{org}/{doc_id}/{version_slug}[/{part}]/          # published originals
raw-withheld/{tier}/{org}/{doc_id}/{version_slug}[/{part}]/ # archived outside this repository
```

Tier is the root so a public corpus is visibly public — anything outside `public/` in this repository is
wrong at a glance — and so the on-prem merge is a union of two disjoint subtrees. Version sits last so
revisions of one document are siblings: `nist/sp-800-171/r2u1`, `/r3`, later `/r4`.

**Two rows resolving to the same directory fails stage 0**, like a duplicate id. The second would otherwise
overwrite the first's artifacts while the run reported both as ingested. Give one a `part`.

Rows without these fields fall back to `{tier}/{reg_id}/{version}` and are reported, so the registry can be
migrated a row at a time.

### Content rules

| Field | Meaning |
| --- | --- |
| `chunking` | Human description of the intended chunking. The chunker actually used is chosen from the content and recorded in the manifest as `chunker`. |
| `normativity_map` | **Normativity-first**: one entry per normativity, listing the block types that carry it — `{"requirement": ["statement"], "guidance": ["discussion", "800-53 mapping"], "example": []}`. Inverted once at load. The block-type-first spelling (`{"statement": "requirement"}`) is also accepted. Applied structurally, never inferred (invariant 8). An unmapped block type defaults to `guidance`, the conservative choice: labelling guidance as a requirement would invent an obligation the source does not state. |
| `sections.include` | Section names or clause numbers to keep. |
| `sections.exclude` | `[{section, covered_by}]`. Dropped, counted, and attributed to the row that carries the material instead. |
| `language` | ISO 639-1 code, e.g. `en`. |
| `language_policy` | `keep` (filter not engaged), `drop-other` (off-language paragraphs removed and counted), `report-only` (kept and reported). |
| `fields.include` | Allow-list for structured sources. Empty means "everything not dropped". |
| `fields.drop` | Fields removed from structured sources. Every drop is a report line. |

A section matched by neither list is **kept** and reported as `unclassified_section`. A section named on both
lists fails stage 0. See ADR-0004.

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
| `verify_against` | Registry id of the PDF row whose extracted text this row's statements are verified against. |
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
  "chunking": "One chunk per field per control.",
  "normativity_map": {
    "requirement": ["statement"],
    "guidance": ["discussion", "800-53 mapping"],
    "example": []
  },
  "language": "en",
  "language_policy": "keep",
  "fields": { "drop": ["internal_note"] },
  "qa": { "expected_requirements": 110, "verify_against": "REG-N01b" },
  "confirm": ["Confirm the discussion text tracks r2 and not r3."]
}
```
