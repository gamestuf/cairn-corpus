# Banned list prompt

The prompt that builds [`Banned_Companies.json`](Banned_Companies.json) from the sources in
[`Banned_List_Sources.md`](Banned_List_Sources.md). Give it to an AI assistant that can read files and run code,
together with copies of every source. The Cairn pipeline does not build this list: the sources are web pages, PDF
tables, statutes and Federal Register notices, each structured differently, and reading them needs judgement a
parser cannot hold.

---

## Prompt

You are building a list of companies, institutions and programs that U.S. federal law, regulation or agency action
bars or restricts, for a contractor that must not buy from, use, or collaborate with them. Accuracy matters more than
coverage. Every entry must trace to a source you read, and nothing may come from memory.

### Inputs

1. `Banned_List_Sources.md`: the only sources you may use, each with an id, authority (`authoritative`, `secondary`,
   `historical`) and URL.
2. A folder holding a copy of each source: saved web pages (HTML), PDFs, Federal Register text, public laws.
   Fetch a source yourself only if its copy is missing. Several publishers refuse automated clients, so prefer the
   copies.
3. The previous `Banned_Companies.json`, if any, to report what changed.

### Read each source by its structure

- **FCC Covered List** (`FCC-CL`): a two-column table of covered equipment or services and their date of inclusion.
  - Some rows name a company; four name a **category** (foreign-produced UAS, routers, power inverters, advanced
    robotic devices). Companies go in `companies`; categories go in `categories`, with their exceptions.
  - Two rows can share one date cell. Check every date against the FCC's public notices: each notice's Appendix A
    reproduces the whole list.
  - The section 1709 row names no company itself. The statute names **DJI and Autel Robotics**, so read Pub. L.
    118-159 section 1709(a)(1).
- **FAR clauses** (`FAR-889`, `FAR-KASPERSKY`, `FAR-BYTEDANCE`): take the companies from the clause's definitions
  ("covered telecommunications equipment or services", "Kaspersky Lab covered entity", "covered application").
  Record the clause date and effective dates.
- **GSA Prohibited Manufacturers** (`GSA-PM`): a short list of manufacturer names.
- **Section 1260H** (`DOD-1260H`): use the Federal Register HTML, not the plain text.
  - Each designation is a paragraph followed by a bullet list giving DoD's basis. The paragraph is sometimes styled
    and sometimes not, so match on the structure (paragraph, then bullets), not on a CSS class. A match on the
    class alone once dropped Autel Robotics.
  - Names carry their subsidiaries in parentheses ("and AVIC subsidiaries: …"). Keep the company as `company` and
    the subsidiaries in `affiliates`.
  - The removals paragraph lists companies that are **no longer designated**: those go in `historical`.
  - Strip page markers such as "( printed page 35190)".
- **Section 5949** (`NDAA-5949`): the statute names its companies in its definition of "covered semiconductor
  product or services". It is in force five years after enactment, so set `status: future` until then.
- **Section 1286** (`DOW-1286`): a three-column PDF table (English name, native name, country), with bulleted
  affiliates.
  - Text extraction misaligns the country column. Use word coordinates (`pdftotext -bbox`). A country cell is
    centred vertically in its row, so assign it by the row's vertical extent.
  - A new institution starts at the left margin after a bullet line or after a gap wider than a wrapped line.
  - Check that the institution count equals the number of country cells.
  - Table 2 lists talent programs: `entity_type: talent program`.
- **UC San Diego** (`UCSD-NDAA`): lists grouped under each section 889 manufacturer. Every entry is `secondary`,
  with `parent` set to the manufacturer it is grouped under.
- **Section 1237** (`DOD-1237`): every tranche is stamped as removed on 2021-06-03.
  - All its entries go in `historical`.
  - Tranche 5 is an image; read it by OCR, and say so in the sources file.
  - Note whether each entity is on the current 1260H list by name, checking the 1260H subsidiaries too. Say that
    the match is by name.

### Rules

- **One row per company per source.** Huawei on four lists is four rows, each citing its own source, so a reader
  can see every basis. Fill `also_on` with the other source ids that name the same company.
- **Names as the source prints them.** Do not merge spellings or translate.
- **Country** comes from the source where it states one, with `country_basis: source`.
  - Where it does not, give the country of the company's ultimate owner only if it is beyond doubt, with
    `country_basis: general knowledge` or `owner`.
  - Otherwise give `null`. Never guess.
- **Reason** states the legal basis and what it forbids, in one or two sentences. Quote DoD's stated basis for 1260H
  entries.
- **Notes** carry dates, scope limits (a research-funding rule is not a procurement ban; a section 889
  video-surveillance ban applies only to the listed purposes), and anything a reader could misread.
- **Status**:
  - `in_force`: applies today.
  - `future`: enacted, but not yet in effect.
  - `secondary`: a non-authoritative source.
  - `historical`: removed or superseded.
- **Never** add a company because it is well known to be restricted. If it matters and no listed source names it,
  propose the source in `Banned_List_Sources.md` instead.

### Output

`Banned_Companies.json`:

```json
{
  "title": "...", "generated": "YYYY-MM-DD", "method": "...",
  "fields": { "...": "what each field means" },
  "sources": [ { "id": "FCC-CL", "title": "...", "url": "...", "as_of": "YYYY-MM-DD", "authority": "authoritative" } ],
  "counts": { "FCC-CL": 15 },
  "companies": [
    { "source": "FCC-CL", "company": "Huawei Technologies Company", "country": "China",
      "reason": "...", "notes": "...", "entity_type": "company", "status": "in_force",
      "effective": "2021-03-12", "country_basis": "general knowledge",
      "scope": "...", "affiliates": "...", "parent": "...", "also_on": ["FAR-889", "GSA-PM", "DOD-1260H"] }
  ],
  "categories": [ { "source": "FCC-CL", "category": "...", "effective": "YYYY-MM-DD", "notes": "exceptions" } ],
  "historical": [ { "source": "DOD-1237", "company": "...", "country": "China", "reason": "...",
                    "notes": "...", "status": "historical", "removed": "YYYY-MM-DD" } ]
}
```

The five fields asked for are `source`, `company` (name of company), `country` (country of company, if known),
`reason` (reason for ban, if known) and `notes`. The others make an entry checkable: `status`, `effective`,
`entity_type`, `country_basis`, `scope`, `affiliates`, `parent` and `also_on`.

### Check before handing back

1. **Counts per source**, compared with the last build:

   | Source | 2026-10-10 |
   | --- | --- |
   | `FCC-CL` | 15 companies, 4 categories |
   | `FAR-889` | 5 |
   | `FAR-KASPERSKY` | 1 |
   | `FAR-BYTEDANCE` | 1 |
   | `GSA-PM` | 6 |
   | `DOD-1260H` | 80 designated, 10 removed |
   | `NDAA-5949` | 3 |
   | `DOW-1286` | 130 institutions, 6 programs |
   | `UCSD-NDAA` | 231 |
   | `DOD-1237` | 44 historical |

   Explain every difference from the publisher's own changes.
2. **No `in_force` entry from a historical source**, and no `historical` entry still on a current list unless its
   notes say so.
3. **Every `country` that is not `null` has a `country_basis`.**
4. **Update `Banned_List_Sources.md`**: each source's as-of date and the SHA-256 of the copy read.
5. **Report** what was added, removed and changed since the last build, by source.
