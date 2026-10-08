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

Per document, under `derived/public/{org}/{doc_id}/{version_slug}/{format_profile}/` — the manifest entry lists the exact paths. Every folder there is one the manifest names: when a document's version slug changes, the build removes the old
folder once the manifest has moved off it, so walking the tree and reading the manifest find the same documents:

| File | Use |
| --- | --- |
| `chunks.jsonl` | Retrieval units and their metadata |
| `vectors.jsonl` | Dense (1024-d, normalized) and sparse embeddings, keyed by `chunk_id` |
| `nodes.jsonl` / `edges.jsonl` | The graph |
| `text.md` | The whole document, for display or re-chunking |
| `scf-id-map.json` | The SCF workbook (`REG-F01`) only: SCF's legacy ids → 2026.3 ids (see *SCF ids have an edition*) |
| `subset.json` | A row that keeps part of its content (`subset` in the registry) — 800-53, `REG-N50`: the controls kept, each with the sources that put it in scope, those left out, and any in scope the publication does not hold |

The `.jsonl` files are JSON Lines: stream them, do not load them whole.

## 4. Join on `chunk_id`

`chunk_id = sha256(reg_id + "|" + version + "|" + section_anchor + "|" + normalized_text)`, and node ids are
equally derived (`control:3.1.1`, `control:AC-02(01)`, `objective:3.1.1[a]`, `section:REG-D03:v2.13:3.1.1`,
`scf:2026.3:IAC-01`, `term:cui`, `document:REG-N01`). A control node's id is `control:` and the canonical id exactly as
`control_ids` holds it, parentheses included. No id is ever assigned by a database.

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
| `authority` | Who published it, and so whether it may be cited — the publisher's standing only. A Rev 3 publication, a Level 1 guide and 800-53 are `authoritative` like any NIST or DoD CIO document: how much one matters to a question is `relevance`. |
| `relevance` | The part the document plays: `requirement` (states the obligation in force — the clauses, 32 CFR 170, 800-171 Rev 2 and 800-171A, 800-172), `guidance` (how to meet, assess or scope it — the CMMC guides, the rules' preambles), `supporting` (related, comparison or background — 800-171 Rev 3, 800-53, CSF, SCF, briefings) or `example` (a template or sample). Rank and filter on it; it never decides citability. Per document, so a `requirement` document's discussion chunks still say `normativity: guidance`. Also on the manifest entry and the graph's `Document` node. |
| `control_ids` / `scf_ids` | Crosswalks. `control_ids` holds NIST SP 800 control ids in canonical form: 800-171 `3.1.1` (Rev 2) or `03.01.01` (Rev 3), 800-172 `3.1.1e` or `03.01.01E`, 800-53 `AC-02(01)` — zero-padded, whatever the source printed; an 800-53 control that 800-171 Rev 2 Appendix E labels `NFO AC-1` is `AC-01`. `scf_ids` holds SCF control ids, `GOV-02` or `GOV-01.1`, **always in the current SCF edition (2026.3)** — see *SCF ids have an edition* below. Control ids found in text are kept only when they are valid in that form, so a section number like `3.15.1` is never one; an id a source file sets that is not valid is kept and reported in the run. |
| `scf_legacy_ids` | The SCF ids as the source wrote them, where it wrote them in the numbering before 2026.3 — each one's 2026.3 id is in `scf_ids`. Absent otherwise (always absent in this corpus's own documents). Transitional: it goes when the documents that use the old numbers are converted. |
| `attributes` | Facts the source states about the element that are not text to retrieve, as string pairs. On the SCF workbook: a control's `domain`, `cadence`, `weighting` (1–10), `pptdf` and `errata`; an assessment objective's `sdp` (SCF's recommended parameter value), `pptdf`, `rigor`, `notes`, and `origin` (`SCF Created` where it has no external source); an evidence request's `area`; a risk's or threat's `grouping` and `csf_function`; a focal document's `geography`, `column`, `source`, `url` and `strm_url`. Absent where there are none. |
| `section_anchor` | Stable within-document address. |
| `page` | Page in the source PDF, where one is known. For citation. |
| `ocr_confidence` | For text recognised from a scanned PDF, the OCR engine's mean word confidence (0–100) on the chunk's page. Absent for text read from a text layer. Weigh or filter on it: below 60 the run also reported the page. |
| `terms` | Controlled vocabulary mentioned, from a closed list. |
| `provision` | For a regulation chunk, the provision it is part of, as a canonical citation: `FAR 52.204-21`, `DFARS 252.204-7012`, `DFARS 204.7501`, `32 CFR 170.4`. For a CMMC assessment guide chunk, the practice: `AC.L2-3.1.1`, `AC.L1-b.1.i`, `AC.L3-3.1.2e` — its `control_ids` hold the NIST requirement the practice is (`3.1.1`, `3.1.2e`). Null elsewhere. |
| `provision_title` | That provision's own title — "Safeguarding Covered Defense Information and Cyber Incident Reporting". Carried beside the text, so the regulation's words stay exactly its own. |
| `references` | Provisions the chunk's text cites, in the same canonical form. Filter on it to find every chunk that invokes a clause or a section. |
| `relations` | Typed links the source itself states from this chunk's element, each `{kind, target, target_framework, target_version, qualifier}`. From the NIST PDFs' own mapping tables and references: `maps-to` (an 800-171 Rev 2 requirement to the 800-53 Rev 4 controls of its Appendix D and the ISO/IEC 27001:2013 controls beside them; an 800-172 requirement to its Appendix C's 800-53 Rev 5 controls — the `qualifier` says when basic requirements share one cell, or an ISO control does not fully satisfy the NIST one) and `derived-from` (an 800-171 Rev 3 or 800-172 Rev 3 requirement to its Source Controls in 800-53 Rev 5; 800-171A / 800-172A Rev 3 to their Source Assessment Procedures in 800-53A). From the NIST CPRT exports: `maps-to` (an 800-171 r3 or 800-172 requirement to the 800-53 controls it derives from), `related-control` (between 800-53 controls), `incorporated-into` and `moved-to` (where a withdrawn requirement went), `addressed-by`. `target_framework` and `target_version` name the target's document and revision (`SP_800_53`, `5.1.1`) when it is another document, and are absent when the target is in this one. `qualifier`, where present, is what the source says about the link itself — an SCF risk's likelihood, a mapping's form. Recorded whether or not the corpus holds the target. The SCF workbook's kinds are listed in its section below. Empty elsewhere. |

### SCF ids have an edition

SCF 2026.3 renumbered its catalogue and gave 700 of the old numbers to *other* controls: legacy `GOV-01` is 2026.3
`GOV-02`, and 2026.3 `GOV-01` is a new control. An SCF id therefore means nothing without its edition, and the corpus
never reads one without it:

- `scf_ids` holds 2026.3 ids only. A document written in the old numbering declares so in the registry
  (`scf_edition: legacy`); its ids are translated through the SCF id map, and the ids as written stay in
  `scf_legacy_ids`.
- **The SCF id map** is `scf-id-map.json`, beside `REG-F01`'s chunks in
  `derived/public/scf/scf-catalog/2026.3/scf-workbook/`: every 2026.3 id (`ids`), and every legacy id with the 2026.3
  id it resolves to and how (`legacy`: `{"GOV-01": {"id": "GOV-02", "how": "renumbered"}}`; `how` is `renumbered`,
  `merged` or `unchanged`), with the edition, the edition it translates from, and the hash of the workbook it was
  built from. It holds ids only. A build that reads SCF ids in another numbering uses it and records it in its
  manifest under `scf_maps` (edition, path, SHA-256).
- In the graph an SCF node is named by edition: `scf:2026.3:GOV-02`, labelled `ScfControl`, with `scf_id` and
  `scf_edition`. An id written in the old numbering has its own node, `scf:legacy:GOV-01`, with a `RENUMBERED_TO` edge
  to the 2026.3 node; the SCF workbook's graph carries all 1,534 of those edges, so the renumbering itself can be
  traversed.
- SCF-shaped ids found in prose are recorded only where the registry row says its text cites SCF
  (`scf_ids_from_text`), because the same shape is used by other numbering schemes — process codes, form numbers.

### The SCF workbook (`REG-F01`)

The SCF 2026.3 workbook is read into one chunk per element, every one `authority: derived` (never cited as an
obligation; `content_class: informative`). Section anchors are `{id}/{type}`:

| `block_type` | Anchor | `scf_ids` | `control_ids` |
| --- | --- | --- | --- |
| `control` | `GOV-02/control` | its own id | The NIST controls SCF maps it to: 800-53 (`PM-01`), 800-171 Rev 2 (`3.1.1`) and Rev 3 requirements (`03.15.01`), 800-172 (`03.01.17E`), CMMC Level 2 and 3 practices as their NIST requirement |
| `control question`, `control risk`, `possible solutions` | `GOV-02/control question` … | the control's | — |
| `assessment objective` | `GOV-02_A01/assessment objective` | the control's | — |
| `evidence request` | `E-GOV-07/evidence request` | the controls it evidences | — |
| `domain`, `risk`, `threat`, `focal document` | `GOV/domain`, `R-AC-1/risk`, `NT-1/threat`, `general-nist-800-171-r2/focal document` | — | — |

`possible solutions` is `example` normativity (SCF's suggestions by business size, often naming products); the rest
are `guidance`. A control's `relations` carry its mappings and links:

| `kind` | Target |
| --- | --- |
| `maps-to` | An id in another framework. `target_framework` is the framework's identifier in SCF's Focal Documents sheet (`general-nist-800-171-r2`, `general-nist-800-171-r3`, `general-nist-800-53-r5-2`, `usa-federal-dow-cmmc-2-level-2`, `usa-federal-far-52-204-21`, `general-iso-27001-2022`, …). Ids are canonical: `CM.L2-3.4.1`, `AC.L1-b.1.iv`, `CM-08(05)`, 800-171 Rev 3 statement parts `03.15.01.a`, clause paragraphs `252.204-7012(b)(2)(ii)(D)`. `qualifier: nfo` marks an 800-53 control SCF lists under 800-171 Rev 2's Appendix E (`NFO`). ITSP-10-171 targets are 800-171 Rev 3 ids; its focal document `adopts` 800-171 Rev 3 and is `assessed-by` 800-171A Rev 3. |
| `renumbered-from` | A legacy id (`target_framework: scf`, `target_version: legacy`, `qualifier`: how) |
| `evidenced-by`, `compensated-by` | An evidence request (`E-GOV-07`), another control |
| `addresses-risk`, `addresses-threat` | `R-AC-1`, `NT-1`, with `qualifier` the likelihood, `Possible` or `Likely` (SCF's `Unlikely` cells are not recorded) |

An assessment objective's relations: `derived-from` the objective it comes from (800-53A Rev 5 `PM-01a.[01]` under
`general-nist-800-53a-r5`, 800-171A `3.4.9[a]`, 800-171A Rev 3 `A.03.01.01.a[01]`, 800-172A `3.4.1e[c]` under
`general-nist-800-172a`); `maps-to` a CMMC Level 1 objective (`AC.L1-b.1.i[c]`); `reciprocal-of` another SCF
objective. An evidence request `evidences` its controls and the CMMC Level 2 practices SCF names.

**How the frameworks meet.** A `control` chunk's `scf:2026.3:…` node links `IN_SECTION` to the chunk's section, and
every NIST control in its `control_ids` links `MAPS_TO` that node — so `control:3.1.1` reaches the SCF controls the
catalogue maps it to. Any other chunk that carries an SCF id `CITES` the same node. A merged corpus whose documents
cite SCF controls, in either numbering, therefore joins to 800-171 through `scf:2026.3:…` with no further work.

### Mappings to 800-53 are pinned to a revision

A requirement's `maps-to` / `derived-from` relations to 800-53 become graph edges from its control node —
`control:3.1.1` `MAPS_TO` `control:AC-02` (800-171 Rev 2, 800-172), `control:03.01.01` `DERIVED_FROM` `control:AC-02`
(Rev 3) — each with `target_version`, `target_framework`, `qualifier` and `source` (the row) as properties. Rev 2's
targets are **800-53 Rev 4**, as its Appendix D states; the corpus holds Rev 5. Seven of them Rev 5 withdrew
(`AU-08(01)`, moved to `SC-45(01)`, …): the edge still names the Rev 4 control, and the crosswalk adds a derived edge to
the successor. Read `target_version` before treating a Rev 2 edge as a Rev 5 claim. A target the corpus does not hold
by design — ISO/IEC 27001:2013 — is an `ExternalControl` node, `external:iso-iec-27001:2013:A.9.2.1`.

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
that carry prose of their own, so the `IN_SECTION` chain from a leaf to the document root has no gaps. So
does a heading merged into the chunk *before* it — a one-line section takes the next heading with it — and
that node has no `chunk_id` of its own: its `folded_into` property names the chunk that holds its text.
And `section_anchor` on a chunk is still the heading *path*; the slug is on the node, so nothing about chunk
identity changed.

Edges added for that join: **`REFERENCES`** (a control or objective cites a section of a narrative) and
**`INHERITS_FROM`** (a control takes its implementation from the same control in another system).

### The control crosswalk (`crosswalk/public/`)

One file set per build, hashed in the manifest under `crosswalk`, joining NIST SP 800-171 Rev 2's 110 requirements and
800-171A's 320 objectives to what maps them — and Rev 3's requirements and objectives in the relationship list:

| File | Content |
| --- | --- |
| `edges.jsonl`, `edges.csv` | The relationship list, one edge per line: `from`, `from_kind`, `type`, `to`, `to_kind`, `to_framework`, `to_version`, `revision` (`r2`/`r3`), `basis` (`stated`, or `derived` with `via` listing the `edge_id`s it chains), `source_reg_id` and `source` (the row and the place in it), `page`, `resolution`, `qualifier` |
| `hierarchy.json`, `hierarchy.txt` | One tree per Rev 2 requirement: CMMC Level 2 and 1, 800-53 Rev 4 from Appendix D (a withdrawn control shown with its Rev 5 successor), the SCF 2026.3 controls the catalogue maps it to, then its 800-171A objectives, each with its CMMC objective and SCF controls |
| `coverage.json`, `coverage.md` | For each relationship, how many requirements and objectives have an edge, and every gap with its reason; which row supplied each source; the gate; where sources disagree |
| `sp800-53-scope.csv`, `sp800-53-scope.json` | The NIST SP 800-53 controls in scope for CUI, one line each: `control`, `base`, `enhancement`, `held`, `sources` (800-171 Rev 3 source controls and Appendix C `CUI` rows; Rev 2 Appendix D mappings and Appendix E `CUI`/`NFO` rows; the SCF controls that map 800-171, 800-171A, CMMC Level 1 or 2, DFARS 252.204-7012, FAR 52.204-21/-25/-27 or ITSP-10-171), `r3_requirements`, `scf_controls`. The JSON's `notes` list the controls tailored out and where Rev 3's two statements of its source controls disagree |

Edge types: `IS_EQUIVALENT_TO` (a requirement or objective and its CMMC Level 2 practice or objective), `MAPS_TO`
(CMMC Level 1 → requirement; requirement or objective → `scf:2026.3:…` control or objective), `ASSESSED_BY`
(requirement → objective), `DERIVED_FROM` (Rev 2 requirement → its Appendix D 800-53 Rev 4 control, `to_version`
`r4`, and by a derived edge to the Rev 5 successor of one Rev 5 withdrew; Rev 3 requirement → its 800-53 source control),
`MAPS_TO` to `external-control` (Rev 2 requirement → ISO/IEC 27001:2013, resolution `external`), `MAPS_TO` from an
`nfo-control` (`NFO AC-01`: an 800-53 control Rev 2's Appendix E expects of a nonfederal organization → 800-53 Rev 5's
control of the same id, `to_version` `r5`; Appendix E names Rev 4, and the corpus holds Rev 5 only, so a control Rev 5
withdrew is named in the qualifier and reaches its successor by a derived edge), `RENUMBERED_TO`
(`scf:legacy:…` → `scf:2026.3:…`), `EVIDENCED_BY` (SCF control → evidence request). Ids are canonical; SCF ids carry
their edition. Canada's ITSP-10-171 is 800-171 Rev 3 under another cover, so SCF's ITSP-10-171 column is read as Rev 3's:
where it names an SCF control the `NIST 800-171 R3` column does not, the Rev 3 requirement gains a `MAPS_TO` whose
`source` names the ITSP column; 800-171A Rev 3 assesses both.

A merged corpus that holds an organization's system security plans, Policies and Standards builds the same files in
its own lane, adding what those state — `IMPLEMENTED_IN`, `IMPLEMENTED_BY`, `DETAILED_BY`, `SUPPORTED_BY` — on top of
these edges, which it reads unchanged. A plan's NFO control (`NFO AC-1`) is keyed `NFO AC-01`, so it joins Appendix E's
edge to 800-53 Rev 5.

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
