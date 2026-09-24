# Working in cairn-corpus

**This repository is public.** Everything committed here is visible to anyone. Read §1 before adding a file.

This is data only. The pipeline that builds it is [`cairn-pipeline`](https://github.com/gamestuf/cairn-pipeline),
a private repository; its `DESIGN_GUIDE.md` and `PLAN.md` explain how everything here is produced.

| | |
| --- | --- |
| [`docs/registry-reference.md`](docs/registry-reference.md) | Every field of `registry/public.json` |
| [`docs/consuming-artifacts.md`](docs/consuming-artifacts.md) | The contract for anything reading the corpus |
| [`docs/status.md`](docs/status.md) | Where the corpus stands against what it is meant to become |

## 1. What may be committed here

**Check `redistribute` before adding any document.** The path a copy is filed at answers "may this be
republished?" before anything reads the row:

| | committed | for |
| --- | --- | --- |
| `fallback/{tier}/{org}/{doc_id}/{version}/` | **yes** | rows with `redistribute: true` |
| `fallback-withheld/{tier}/{org}/{doc_id}/{version}/` | **no** — gitignored | rows without it |

Most of what the registry names is United States federal government work and carries no copyright. Three
rows are not: the **SCF catalog** and the Cyber AB's **CAP** and **CoPC**. Their text is derived and
published; their original files must never be committed. The SCF workbook lives at
`fallback-withheld/public/scf/scf-catalog/2026.3/` for local runs and stays there.

The pipeline errors if a non-redistributable row has a copy under `fallback/`, but `git add .` does not —
so check the flag, not your memory.

**Never commit:** credentials, non-public content, licensed originals, internal system names, or paths into
internal infrastructure. That last one includes prose inside `notes` and `corpus_role` fields — it has
happened before, and once pushed it is in the history whether or not the file is later edited.

## 2. The registry is the single input

`registry/public.json` is the only thing edited by hand. Everything else — `derived/`, `manifest/`,
`reports/`, `raw/` — is produced by a run.

- **Do not hand-commit derived artifacts.** Files no run produced and no manifest describes break the
  corpus's central promise.
- **Do not edit `manifest/registry.snapshot.json`.** It is build output and the baseline for the next run's
  change detection; a hand-edit makes the next build treat everything as new.
- A local run pointed at this repository will leave `manifest/`, `reports/` and a rewritten snapshot in the
  working tree. **Revert that residue rather than committing it** unless the run was the real scheduled
  build.
- Every row must be `tier: public`. Any other tier fails stage 0.

## 3. Editing rows

Fields are documented in [`docs/registry-reference.md`](docs/registry-reference.md); update it in the same
change if you alter what a field means.

- **A url must resolve to the document, not to a page about it.** A landing page produces
  `payload is 'html', not 'pdf'`. An index of documents is not a document: set `ingest: register-only` and
  add each real document as its own row — `REG-R05` and `REG-R04` are the worked example.
- **Path slugs are exact.** Convention-based fallback discovery keys on `org`, `doc_id` and `version_slug`,
  so a copy filed under a version the registry does not name is invisible. That is deliberate — guessing
  across versions is how a superseded snapshot gets served as current text.
- **`status: planned` is the phase-in lever.** A planned row is skipped with its reason named, so a row can
  be authored before its parser exists. Promoting to `active` is the sign-off.
- **`version` feeds `chunk_id`; `version_slug` does not.** Changing `version` reissues ids and costs a
  re-embed. Changing `version_slug` only moves directories.
- Prefer a `confirm` item over a guess. It surfaces in every run report until answered.

## 4. Checking a change

```bash
# from the pipeline checkout, against this registry — validates and writes nothing
dotnet run --project src/Cairn.Public -c Release -- \
  run --registry ../Cairn-Corpus/registry/public.json --out <scratch> --dry-run --offline
```

Use a scratch `--out`, never this repository, unless you intend to produce a real build. Read the run
report's **Corpus presence** table to confirm a file is where the registry expects it.

## 5. Ask before proceeding

If two readings of a task would produce materially different work — which repository a change belongs in,
whether a document is a source or an index, whether a fix belongs in the registry or the pipeline — ask
rather than picking one.
