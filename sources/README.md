# Committed source copies

Documents whose publishers cannot be fetched from CI. The pipeline uses these **only when the live fetch
fails**, never in preference to it.

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

Re-check these periodically. A publisher that starts working again should go back to being fetched, and
the only way anyone notices is by reading the warnings.

## Licensing

Everything here is a public document published by a US federal body or a standards organisation, retained
under its own terms. Record provenance — the url it came from and when — in the registry row's `notes`.
