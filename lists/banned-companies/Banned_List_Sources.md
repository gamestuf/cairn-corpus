# Banned list sources

The sources [`Banned_Companies.json`](Banned_Companies.json) is built from, with
[`Banned_List_Prompt.md`](Banned_List_Prompt.md). A source is used only if it is listed here. Adding one means a row
here, in the same change as the rebuild.

**Authority** says how far an entry can be trusted:

- **authoritative**: the agency's own list or the statute or regulation itself.
- **secondary**: someone else's compilation. Its entries are kept but marked `secondary`.
- **historical**: no longer in force. Its entries go in `historical`, never in `companies`.

**Last built 2026-10-10**, from the copies whose SHA-256 is given. Several publishers refuse automated
clients (fcc.gov, war.gov, media.defense.gov), so those copies were saved by hand from a browser.

The copies read are kept in [`sources/`](sources/), each named for its source id, so the list can be rebuilt from
exactly what it was built from. All are U.S. Government works. The UC San Diego page is not kept: it is not a
Government work, and its URL is recorded instead.

- [`DOD-1237_release_2021-01-14.html`](sources/DOD-1237_release_2021-01-14.html)
- [`DOD-1237_tranche-1.pdf`](sources/DOD-1237_tranche-1.pdf)
- [`DOD-1237_tranche-4.pdf`](sources/DOD-1237_tranche-4.pdf)
- [`DOD-1237_tranche-5.pdf`](sources/DOD-1237_tranche-5.pdf)
- [`DOD-1237_tranches-2-3.pdf`](sources/DOD-1237_tranches-2-3.pdf)
- [`DOD-1260H_FR-2026-11571.html`](sources/DOD-1260H_FR-2026-11571.html)
- [`DOD-1260H_PL-118-31-sec805.htm`](sources/DOD-1260H_PL-118-31-sec805.htm)
- [`DOW-1286_FY25-Section-1286-List.pdf`](sources/DOW-1286_FY25-Section-1286-List.pdf)
- [`FAR-889_52.204-25.html`](sources/FAR-889_52.204-25.html)
- [`FAR-BYTEDANCE_52.204-27.html`](sources/FAR-BYTEDANCE_52.204-27.html)
- [`FAR-KASPERSKY_52.204-23.html`](sources/FAR-KASPERSKY_52.204-23.html)
- [`FCC-CL_DA-25-1086.pdf`](sources/FCC-CL_DA-25-1086.pdf)
- [`FCC-CL_DA-26-673.pdf`](sources/FCC-CL_DA-26-673.pdf)
- [`FCC-CL_DA-26-996.pdf`](sources/FCC-CL_DA-26-996.pdf)
- [`FCC-CL_PL-118-159-sec1709.htm`](sources/FCC-CL_PL-118-159-sec1709.htm)
- [`FCC-CL_covered-list_2026-10-09.html`](sources/FCC-CL_covered-list_2026-10-09.html)
- [`GSA-PM_prohibited-manufacturers.html`](sources/GSA-PM_prohibited-manufacturers.html)
- [`NDAA-5949_PL-117-263.htm`](sources/NDAA-5949_PL-117-263.htm)

## Consumed

| Id | Source | What it names | Legal effect | Authority | As of | Copy read (SHA-256) |
| --- | --- | --- | --- | --- | --- | --- |
| `FCC-CL` | [FCC Covered List](https://www.fcc.gov/supplychain/coveredlist), cross-checked against Appendix A of [DA 26-996](https://docs.fcc.gov/public/attachments/DA-26-996A1.pdf) | 15 named producers and providers, plus 4 categories of foreign-produced equipment | Secure Networks Act section 2: no FCC funds; no new equipment authorizations. Subsidiaries and affiliates included | authoritative | 2026-10-09 | saved page `84140553c353…`; DA 26-996 `3919ef72eb2b…` |
| `FAR-889` | [FAR 52.204-25](https://www.acquisition.gov/far/52.204-25) (FY2019 NDAA section 889) | Huawei, ZTE, Hytera, Hikvision, Dahua and their subsidiaries and affiliates | Agencies may not buy covered equipment (Part A, 2019-08-13), nor contract with anyone who uses it (Part B, 2020-08-13) | authoritative | NOV 2021 | `ef683bab724b…` |
| `FAR-KASPERSKY` | [FAR 52.204-23](https://www.acquisition.gov/far/52.204-23) (FY2018 NDAA section 1634) | Kaspersky Lab covered entities | Contractors may not provide, or use in performance, Kaspersky products (from 2018-10-01) | authoritative | DEC 2023 | `9ffcf29e22b5…` |
| `FAR-BYTEDANCE` | [FAR 52.204-27](https://www.acquisition.gov/far/52.204-27) | ByteDance Limited (TikTok) | The application may not be on IT used by or for the Government | authoritative | JUN 2023 | `342371ef1ffb…` |
| `GSA-PM` | [GSA Vendor Support Center, Prohibited Manufacturers](https://vsc.gsa.gov/drupal/node/6) | 6 manufacturers | GSA Advantage! rejects catalog products naming them | authoritative | 2026-10-10 | saved page `7969414d8a45…` |
| `DOD-1260H` | [Federal Register 2026-11571](https://www.federalregister.gov/documents/2026/06/10/2026-11571/notice-of-availability-of-designation-of-chinese-military-companies), the Section 1260H list; effect from [FY2024 NDAA section 805](https://www.govinfo.gov/content/pkg/PLAW-118publ31/html/PLAW-118publ31.htm) | 80 Chinese military companies with their named subsidiaries; 10 removals | DoD may not contract with them from 2026-06-30, nor buy goods or services they produced from 2027-06-30 | authoritative | 2026-06-10 | `dec8a0b1baf0…` |
| `NDAA-5949` | [FY2023 NDAA section 5949](https://www.govinfo.gov/content/pkg/PLAW-117publ263/html/PLAW-117publ263.htm) | SMIC, CXMT, YMTC | Agencies may not procure electronic parts that include their semiconductors; **in force from 2027-12-23** | authoritative | 2022-12-23 | `2a74f5e0c828…` |
| `DOW-1286` | [FY2025 Section 1286 lists](https://www.cto.mil/wp-content/uploads/2026/07/FY25-Section-1286-List_402307.pdf) | 130 foreign institutions (China 88, Russia 33, Iran 9) and 6 talent programs | From FY2026, no DoW funding for fundamental research that collaborates with them. A research-security rule, not a procurement ban | authoritative | 2026-06-26 | `4e02351693f0…` |
| `UCSD-NDAA` | [UC San Diego, NDAA Prohibited Manufacturers](https://blink.ucsd.edu/technology/security/ndaa/index.html) | 231 subsidiaries and affiliates of the five section 889 manufacturers | None of its own; it names who section 889's "subsidiary or affiliate" reaches | **secondary**: a university's compilation, last updated 2023-03-14, called provisional by its authors | 2023-03-14 | saved page `d2cc7038099d…` |
| `DOD-1237` | [DoD Section 1237 release, 2021-01-14](https://www.war.gov/News/Releases/Release/Article/2472464/dod-releases-list-of-additional-companies-in-accordance-with-section-1237-of-fy/) and its four tranche PDFs ([1](https://media.defense.gov/2020/Aug/28/2002486659/-1/-1/1/LINK_2_1237_TRANCHE_1_QUALIFIYING_ENTITIES.PDF), [2–3](https://media.defense.gov/2020/Aug/28/2002486689/-1/-1/1/LINK_1_1237_TRANCHE-23_QUALIFYING_ENTITIES.PDF), [4](https://media.defense.gov/2020/Dec/03/2002545864/-1/-1/1/TRANCHE-4-QUALIFYING-ENTITIES.PDF), [5](https://media.defense.gov/2021/Jan/14/2002565154/-1/-1/0/DOD-RELEASES-LIST-OF-ADDITIONAL-COMPANIES-IN-ACCORDANCE-WITH-SECTION-1237-OF-FY99-NDAA.PDF)) | 44 companies | **None now.** Every page is stamped "As of June 3, 2021, the Secretary of Defense has removed the entities listed here"; the Section 1260H list replaced it | **historical** | 2021-06-03 | page `d31b3db5c0b3…`; tranches `66784def54ef…`, `2239702c8303…`, `2710d4f8d8e4…`, `e2ff45f7eca0…` (tranche 5 is an image, read by OCR) |

FY2025 NDAA section 1709, which names DJI and Autel Robotics
([Pub. L. 118-159](https://www.govinfo.gov/content/pkg/PLAW-118publ159/html/PLAW-118publ159.htm)), is consumed through
`FCC-CL`. That is where it takes effect: the FCC added both on 2025-12-22 (DA 25-1086).

## Changes from the list first proposed (2026-10-10)

- **Added**:
  - **FAR 52.204-25, -23 and -27**: the regulations that actually bind contractors. The Covered List binds the FCC's
    programs and equipment authorization, not a contract.
  - **The current Section 1260H list**: the list now in force, with DoD's stated basis for each company.
  - **FY2023 NDAA section 5949**: a semiconductor ban already enacted, in force from 2027-12-23.
- **Reclassified**:
  - The **Section 1237 tranches** are historical. DoD removed every entity on 2021-06-03. 27 of the 44 are on the
    current 1260H list by name, and their entries say so.
  - The **UC San Diego page** is secondary. It is not an agency list, but it is the only source here that names
    the five manufacturers' subsidiaries one by one.
- **Scope made explicit**: the Section 1286 list restricts DoD-funded research, not purchasing. Its entries carry that
  in `reason` and `notes`.

## Recommended, not yet consumed

| Source | Why | Why not yet |
| --- | --- | --- |
| [FASCSA exclusion and removal orders](https://www.sam.gov) (FAR 52.204-30) | Binding orders from the Federal Acquisition Security Council that name products and sources | Published on SAM.gov, which needs a signed-in search; none was found in the Federal Register |
| The FCC's list of affiliates and subsidiaries of Covered List entities | The FCC's own extension of the list to named affiliates | Not in the saved page; the FCC calls it non-exhaustive |
| Commerce Entity List, OFAC SDN List, SAM.gov exclusions, DHS UFLPA Entity List | Restrict exports, sanctions, debarment and imports | Different legal effects and thousands of entries each; each would need its own scope decision before joining a "banned companies" list |
