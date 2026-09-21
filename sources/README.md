# Hand-committed source copies

Documents whose publishers cannot be fetched from CI. The pipeline uses these **only when the live fetch
fails**, and only after it has tried the copy it archived itself.

Most rows need nothing here. Once a document has been fetched successfully, its original is committed to
`raw/` and becomes the fallback for every later run, so this directory is for documents that have *never*
been reachable — the ones where a copy someone found by hand is genuinely the best evidence available.

    sources/{reg_id}/{filename}

Point a registry row at one with `fallback_path`:

```json
"url": "https://dodcio.defense.gov/.../AssessmentGuideL2v2.pdf",
"fallback_path": "sources/REG-D03/AssessmentGuideL2v2.pdf"
```

## When a copy belongs here

| Failure in the run report | Belongs here? |
| --- | --- |
| `HTTP 403 Forbidden` — publisher blocks the runner | **Yes** |
| TLS handshake failure | **Yes** |
| `payload is 'html', not 'pdf'` | **No** — the url points at a landing page; fix the url |
| `HTTP 404` on a withdrawn document | Only if you have a legitimate copy; otherwise set `status: missing` |

## What using one costs

A committed copy is a snapshot of unknown age. Every run that falls back says so — a `fallback_used`
warning, `warnings: true`, and `origin: fallback` with `fallback_reason` in the manifest. That is
deliberate: a consumer has to be able to tell a document retrieved from the publisher today from one
frozen here months ago.

The archive in `raw/` costs the same warning (`archive_used`, `origin: archive`) and is preferred over a
copy here, because the pipeline fetched it from the publisher and recorded its hash, where this is a file
a person placed.

Re-check these periodically. A publisher that starts working again should go back to being fetched, and
the only way anyone notices is by reading the warnings.

## Licensing

A copy here is republished from this public repository, so put one here only for a document that may be.
Almost every row is United States federal government work and carries no copyright; the SCF catalog and
the Cyber AB's CAP and CoPC are not, and those rows set no `redistribute` flag and must not get a copy
here either — the two directories publish the same bytes and the same rule governs both.

Record provenance — the url it came from and when — in the registry row's `notes`.
