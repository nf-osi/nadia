# NADIA First-Pass vs Published State — Synapse Ground-Truth Audit

**Date:** 2026-06-27
**Auditor:** allawayr (via Synapse REST + GitHub issue threads)
**Corpus:** 13 published (`approved`) NADIA projects — issues 290, 216, 144, 124, 80, 59, 35, 29, 28, 25, 11, 10, 9
**Method:** For each project, compared (a) NADIA's documented first-pass values (issue body annotation block) against current published Synapse annotations, and (b) Dataset entity **version 1** (NADIA's original) against current version, plus full enumeration of file-level annotations, schema bindings, and folder structure. Project entities carry no version history (always v1), so project-level "original" is taken from the issue body; Dataset entities **do** have version history and were diffed directly.

---

## Project ID resolution

| Issue | Project | Title (short) |
|---|---|---|
| 290 | syn74276737 | Spatially resolved transcriptomics of PNSTs |
| 216 | syn74319981 | Nf1 mutation disrupts oligodendroglial plasticity (mouse brain) |
| 144 | syn74316142 | Gene expression profiling of MPNSTs and neurofibroma |
| 124 | syn74315329 | BRD4 ChIP in NF2 |
| 80 | syn74287631 | crossMoDA challenge (VS MRI) |
| 59 | syn74287077 | Vestibular Schwannoma MRI with expert segmentations (TCIA) |
| 35 | syn74280931 | RAC1 disruption prevents plexiform NF (E-MTAB-8398) |
| 29 | syn74278763 | Human cutaneous neurofibroma matrisome scRNA-seq |
| 28 | syn74278723 | Cabozantinib pNF phase 2 trial (proteomics) |
| 25 | syn74278246 | CDK4/6 + ERK1/2 inhibition in pNF |
| 11 | syn74309473 | HuR/ELAVL1 drives MPNST |
| 10 | syn74288246 | Malignant progression in NF1 (scRNA-seq) |
| 9 | syn74288412 | MEK/BET/CDK triple combination in MPNST |

---

# PART A — Ranked taxonomy of what curators had to change, with recommendations

Ranked by frequency across the 13 projects. Each item notes whether the change was **documented** in the GitHub thread or a **SILENT** fix made directly in Synapse (silent fixes are the highest-value finds — NADIA gets no feedback signal for them). **Standard** maps to the numbered CLAUDE.md standard.

### Rank 1 — Author-name format wrong on first pass (12/13 with a PMID) · Standard 3 · DOCUMENTED
NADIA wrote `studyLeads` as `Lastname F` / `Lastname Firstname` (PubMed XML raw form) on nearly every project: `Pan Yuan` → `Yuan Pan`, `Høland Maren, Lothe Ragnhild A` → `Maren Høland, Ragnhild A Lothe`, `Fisher MJ, Clapp DW` → `Michael J Fisher, D Wade Clapp`, `Mund L` → `Julie A Mund, D Wade Clapp` (also wrong *count* — NADIA gave only one author). Curators reformatted to `Firstname [Middle] Lastname` on **every project that had a PMID** (290, 216, 144, 124, 80, 35, 28, 25, 10). The three already-correct ones (29, 11, 9) were cases where NADIA happened to ingest a pre-formatted name.
**Recommendation:** This is the single most frequent fix. NADIA already knows the rule (Standard 3 explicitly says combine `<ForeName>` + `<LastName>`) but the first-pass code clearly reads `Author/CollectiveName` or the abbreviated `Initials` form. Fix the PubMed parser to always build `f"{ForeName} {LastName}"` from the structured XML fields, never from `Initials`, and never truncate the author list to one name (issue 35 lost the corresponding author entirely).

### Rank 2 — Missing/incomplete cross-repository accessions (9/13) · Standards 8 & 19 · DOCUMENTED
NADIA's `alternateDataRepository` on first pass listed only the single discovery accession; curators added the linked SRA/BioProject/ENA/PRIDE siblings on 9 projects:
- 35: `arrayexpress:E-MTAB-8398` → +`insdc.sra:ERP117622`, +`bioproject:PRJEB34683`
- 29: `geo:GSE163028` → +`insdc.sra:SRP297646`, +`bioproject:PRJNA684465`
- 28: `pride.project:PXD019138` → +`pride.project:PXD019005` (a second PRIDE deposit)
- 25: `geo:GSE213789, GSE213787` → corrected to `GSE213786, GSE213787` + `insdc.sra:SRP398255, SRP398257` (note NADIA also had the **wrong GEO accession** GSE213789)
- 80: +`tcia.collection:Vestibular-Schwannoma-MC-RC` (the canonical TCIA host, not just the Zenodo subset)
- 216, 10, 9: NADIA used `insdc.sra:PRJNA…`/`insdc.sra:PRJEB…` for BioProject IDs; curators corrected the **prefix** to `bioproject:` (a PRJNA/PRJEB ID is a BioProject, not an SRA study).
- 59: NADIA emitted an invalid prefix `icrptbia:Vestibular-Schwannoma-SEG`; corrected to `tcia.collection:`.
**Recommendation:** (a) For every GEO series, parse `!Series_relation` to harvest the linked SRA (`SRPxxxxx`) and BioProject (`PRJNAxxxxx`) accessions and add all of them. (b) Fix the prefix logic: `PRJNA`/`PRJEB`/`PRJDB` → `bioproject:`, only `SRP`/`ERP`/`DRP` → `insdc.sra:`. NADIA's `REPO_TO_PREFIX` maps `ENA → insdc.sra` unconditionally, which is wrong for project-level accessions. (c) Remove `icrptbia` from any prefix table; TCIA collections are always `tcia.collection:`.

### Rank 3 — Per-sample fields set at study level on first pass (≥7/13) · Standards 5, 11, 12, 13 · DOCUMENTED
The documented "Fixes Applied" tables show NADIA's first pass set a single uniform value across all files for fields that vary per sample, then curators re-derived per-file:
- **35:** `specimenID`/`individualID` were all ERR run accessions → changed to per-sample BioSamples (SAMEA…); `nf1Genotype`, `tumorType`, `diagnosis` were uniform → split per genotype group (WT controls became `Not Applicable`).
- **290:** `tumorType`/`diagnosis`/`nf1Genotype`/`nf2Genotype` differentiated per section (Schwannoma vs Plexiform NF vs Hybrid PNST vs Synovial Sarcoma).
- **144:** per-patient `sex`, `age`, recurrent vs primary `tumorType` (Standard 22 relapse handling) extracted from supplementary clinical table.
- **10:** `tumorType` split MPNST / Plexiform NF / Atypical NF along the progression series.
Published state now shows correct per-file variation everywhere sampled (verified: 144 has 30 distinct specimenIDs, 2 sexes, 22 ages; 35 has per-genotype tumorType; etc.).
**Recommendation:** NADIA's first pass should never copy a study-level value to all files for the Standard-5 field set. Enforce the gap-fill tier walk (Standard 11) at *creation* time, not only in the Polish/audit re-run. The fact that all of these were caught only on a second polish pass means first-pass file annotation is effectively study-level by default.

### Rank 4 — Run accessions used as specimen/individual ID (≥4/13) · Standard 5 · DOCUMENTED
Explicitly documented in 35 ("All files used ERR run accessions for both specimenID and individualID"). Same antipattern visible in the original of 29/10/9 (ENA discovery). Curators replaced with BioSample (SAMEA/SAMN) or GSM identifiers.
**Recommendation:** Hard rule already in Standard 5 ("Run accessions SRR/ERR/DRR identify runs, not specimens"). First-pass code must map run → `sample_accession`/`sample_alias` from the ENA filereport before writing `specimenID`/`individualID`. This is a deterministic lookup, not reasoning — it should never reach the human.

### Rank 5 — DOI missing on first pass when only a data-deposit DOI was known (4/13) · "Resolve publication first" · DOCUMENTED
216, 144, 124 had `doi: —` on first pass (GEO discovery, no DOI in GEO record); curators added the publication DOI (`10.1038/s41593-…`, `10.1016/j.ebiom.…`, `10.1093/noajnl/…`). 80 had the **TCIA dataset DOI** as the project `doi`; curators moved it to `alternateDataRepository` and set the *publication* DOI as project `doi`.
**Recommendation:** When a PMID is resolved, always fetch the publication DOI from the PubMed `ArticleIdList` (`IdType=doi`) and set it as project `doi`. Never put a repository/dataset DOI in the project `doi` field — that belongs in `alternateDataRepository` as `doi:` (Standard 16 / prefix table).

### Rank 6 — `diseaseFocus` / `manifestation` scoping corrections (5/13) · Standard 13 · DOCUMENTED
- 290: +`Neurofibromatosis type 2` and +`Plexiform Neurofibroma` (cohort spans NF1+NF2 PNSTs).
- 124: NADIA over-broadened to `NF1 + NF2` and listed `MPNST, Schwannoma, VS` as manifestations; curators narrowed to `NF2` focus / `Schwannoma, VS` (NF2-null schwannoma cell lines, not MPNST).
- 144: manifestation broadened from single `MPNST` → `MPNST, Neurofibroma, Plexiform Neurofibroma` (cohort includes benign).
- 216: manifestation `Neurofibroma` → `Behavioral` (this is a mouse brain/motor-learning study with no tumor; NADIA's default-to-Neurofibroma was wrong).
**Recommendation:** 216 is the clearest first-pass error — NADIA assigned a tumor manifestation to a behavioral neuroscience study by disease-context default. First pass should derive `manifestation` from the abstract's actual phenotype, and where the study has no tumor manifestation, use the non-tumor vocabulary term (`Behavioral`) rather than defaulting to `Neurofibroma`.

### Rank 7 — Funding agency placeholder left on first pass (≥3/13) · Standard 11 Tier 2 · DOCUMENTED
80 was `Not Applicable (External Study)` → curators filled `Wellcome Trust, MRC, EPSRC, Royal Academy of Engineering` from PubMed GrantList. Several others (290) expanded a single funder to the full list.
**Recommendation:** Standard 11 already mandates GrantList → PMC Acknowledgements before the placeholder. The placeholder should never survive when a PMID exists with a populated GrantList. Make first-pass funding extraction mandatory whenever PMID is present.

### Rank 8 — Dataset entity renaming (most/all) · `prompts/synapse_workflow.md` naming · DOCUMENTED
NADIA's v1 dataset names were bare `{Repo}_{Accession}` (e.g. `Zenodo_4662239`, `GEO_GSE120685`); curators renamed to descriptive `{assay} - {tissue/model}, {disease} ({source})` form (e.g. `single-cell RNA-seq - skin, Cutaneous Neurofibroma…`). Visible in v1-vs-current name diff on 80 and in the documented reports.
**Recommendation:** Generate the descriptive dataset name at creation from `{assay} - {specimenType/model}, {manifestation} ({Repo Accession})`. NADIA already computes all those fields; wiring them into the dataset name avoids a guaranteed rename.

---

## SILENT, UNDOCUMENTED defects still present in the PUBLISHED state (highest value)

These are *not* described in any GitHub comment and remain wrong in the live approved projects — NADIA has no feedback signal for them today.

### S1 — Dataset entities missing the `id` and `name` system columns entirely (6/13) · `columnIds` order rule · SILENT
Datasets where current `columnIds` do **not** begin with `id|name`, and in several cases omit them completely:
- **syn74288449 (issue 9):** columns start `assay, dataSubtype, dataType…` — `id` and `name` are **absent**. The Datasets tab shows no entity link and no filename column. Confirmed by direct `/column` lookup.
- Same pattern: syn74320534 + syn74320535 (216), syn74457818 + syn74457821 (59), syn74278752 + syn74445161 (28), syn74309550 + syn74309551 (11), syn74301107 (10's second dataset).
- 7 of the 19 datasets are correct (`id|name` first): 290, 144, 124, 80, 35, 29, 25 (both), 10's first dataset.
NADIA's own polish reports for these projects *claim* "system columns (id, name) first" — but the live entity does not reflect it. This is the clearest silent gap: the documented fix was not actually applied.
**Recommendation:** This violates the explicit CLAUDE.md rule "Dataset column order is id | name | annotations." The audit/Phase-3 code must (a) verify `id` (ENTITYID) and `name` (STRING) columns exist and are first **by reading back the stored entity**, not by trusting the build step, and (b) fail the completeness check if absent. A post-write read-back assertion would have caught all 6.

### S2 — `filename` column + `resourceStatus` on File entities, both surviving to published state (issue 10) · Standards (no resourceStatus/filename/name on files) · SILENT
syn74288411 (issue 10, dataset 1) carries a custom **`filename`** column (col order `id, filename, name, …`) — a duplicate of the system `name` column. Separately, **File entities** in syn74288246 carry a **`resourceStatus`** annotation (the only project of 13 where files are not clean). Neither is mentioned in the thread. Standard explicitly forbids both on files.
**Recommendation:** The audit's `EXCLUDE_COLS` check (`resourceStatus`, `filename`, `name`) must run against the actual stored File annotations and Dataset columnIds, not just the in-memory annotation dict before write. Issue 10's project — the largest/most complex (160 + 59 files, processed h5ad formats) — slipped both checks, suggesting the exclusion logic isn't applied when annotations come from Tier-4 file inspection (h5ad `obs` columns), which is exactly where a stray `filename`/`resourceStatus` key would originate.

### S3 — Dataset stable-version minting is inconsistent (multiple) · Standard "stable version in Phase 3" · PARTIALLY SILENT
Version counts vary widely and several datasets that are `resourceStatus=approved` still sit at relevant snapshots, while two approved-project datasets show `ds.resourceStatus=None` (syn74320534/535 issue 216, syn74457818 issue 59, syn74309550/551 issue 11, syn74301107 issue 10, syn74288449 issue 9). For a published project, every Dataset entity should carry `resourceStatus` and a minted stable version; these don't.
**Recommendation:** Phase 3 must set `resourceStatus` on the Dataset entity (not only the project) and mint a version for **every** dataset in `dataset_ids_to_snapshot`, then read back to confirm. The current behavior minted versions on some datasets in a multi-dataset project but not their siblings.

### S4 — Wrong primary GEO accession on first pass (issue 25) · Standard 7 · SILENT (the *original error* was silent; correction documented)
NADIA's first pass recorded `geo:GSE213789` but the real accession is `GSE213786` (curators corrected it, and it appears in the thread only as the corrected value). An elink/parse off-by error produced a non-existent-for-this-paper accession.
**Recommendation:** Standard 7 ownership verification (`esummary db=gds` → check `PubMedIds`) would have rejected GSE213789 if it doesn't belong to PMID 37406085. Enforce ownership verification on **every** GEO accession before writing it, including super-series/sub-series resolution.

---

# PART B — Per-project diff tables (NADIA original → published)

Legend: **D** = documented in thread, **S** = silent fix. Level: P=project, DS=dataset, F=file.

### Issue 290 — syn74276737 (Spatial transcriptomics PNST, Zenodo)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Juliane Bremer, Pamela Franco | Juliane Bremer, Dieter Henrik Heiland | P | D |
| diseaseFocus | NF1, MPNST | NF1, MPNST, NF2 | P | D |
| manifestation | MPNST | MPNST, Plexiform Neurofibroma | P | D |
| fundingAgency | German Cancer Consortium | +Else Kröner-Fresenius, +German Ministry… | P | D |
| Dataset columnIds | 8 | 24 (id\|name first — correct) | DS | D |
| per-file tumorType/genotype | (uniform) | per-section (Schwannoma/pNF/Hybrid/Synovial) | F | D |

### Issue 216 — syn74319981 (Nf1 mouse brain, GEO+ENA)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Pan Yuan, Monje Michelle | Yuan Pan, Michelle Monje | P | D |
| manifestation | Neurofibroma | Behavioral | P | D |
| alternateDataRepository | insdc.sra:PRJNA1096676 | bioproject:PRJNA1096676 | P | D |
| doi | — | 10.1038/s41593-024-01654-y | P | D |
| Dataset `id`/`name` cols | present (v1) | **absent / not first** | DS | **S (S1)** |
| Dataset resourceStatus | — | None (should be set) | DS | **S (S3)** |

### Issue 144 — syn74316142 (MPNST/NF expression array, GEO)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Høland Maren, Lothe Ragnhild A | Maren Høland, Ragnhild A Lothe | P | D |
| manifestation | MPNST | MPNST, Neurofibroma, Plexiform Neurofibroma | P | D |
| doi | — | 10.1016/j.ebiom.2023.104829 | P | D |
| per-file sex/age/tumorType | (study-level) | per-patient (30 IDs, 2 sex, 22 ages) | F | D |

### Issue 124 — syn74315329 (BRD4 ChIP, NF2)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Doherty Joanne, Kissil Joseph L | Joanne Doherty, Joseph L Kissil | P | D |
| diseaseFocus | NF2, NF1 | NF2 | P | D |
| manifestation | MPNST, Schwannoma, VS | Schwannoma, VS | P | D |
| doi | — | 10.1093/noajnl/vdac072 | P | D |
| project name | (repository-ish title) | full publication title | P | D |

### Issue 80 — syn74287631 (crossMoDA, Zenodo/TCIA)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Unknown | Jonathan Shapey, Tom Vercauteren | P | D |
| pmid | — | 34711849 | P | D |
| doi | 10.7937/TCIA… (dataset DOI) | 10.1038/s41597-021-01064-w (pub DOI) | P | D |
| alternateDataRepository | zenodo.record:4662239 | +tcia.collection:Vestibular-Schwannoma-MC-RC | P | D |
| **Dataset items** | **2 (landing-page-only)** | **354 file entities** | DS | D (Std 13 file-enum) |
| fileFormat | zip | DICOM (suffix stripped) | F | D |
| specimenID | crossmoda (uniform) | crossmoda_training / _validation | F | D |

### Issue 59 — syn74287077 (VS MRI segmentations, TCIA)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| alternateDataRepository | icrptbia:Vestibular-Schwannoma-SEG (invalid prefix) | tcia.collection:Vestibular-Schwannoma-SEG | P | D |
| structure | 1 dataset, Analysis folder, landing-page link | 2 datasets (MR + Radiotherapy), per-DICOM file entities, Analysis folder removed | DS/F | D |
| Dataset `id`/`name` cols | — | **absent / not first** (both datasets) | DS | **S (S1)** |
| pmid/doi | — | still unset | P | (open — no pub) |

### Issue 35 — syn74280931 (RAC1/plexiform NF, ArrayExpress)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Mund L (single, abbreviated) | Julie A Mund, D Wade Clapp | P | D |
| alternateDataRepository | arrayexpress:E-MTAB-8398 | +insdc.sra:ERP117622, +bioproject:PRJEB34683 | P | D |
| specimenID/individualID | ERR run accessions (uniform-ish) | per-sample SAMEA BioSamples | F | D |
| nf1Genotype/tumorType/diagnosis | uniform | per-genotype-group (WT→Not Applicable) | F | D |

### Issue 29 — syn74278763 (cNF matrisome scRNA-seq, GEO)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | (correct already) | Jean-Philippe Brosseau, Lu Q Le | P | — |
| alternateDataRepository | geo:GSE163028 | +insdc.sra:SRP297646, +bioproject:PRJNA684465 | P | D |

### Issue 28 — syn74278723 (Cabozantinib pNF proteomics, PRIDE)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Fisher MJ, Clapp DW | Michael J Fisher, D Wade Clapp | P | D |
| alternateDataRepository | pride.project:PXD019138 | +pride.project:PXD019005 | P | D |
| Dataset `id`/`name` cols | — | **absent / not first** (both datasets) | DS | **S (S1)** |

### Issue 25 — syn74278246 (CDK4/6+ERK in pNF, GEO)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Flint Alyssa C, Rhodes Steven D | Alyssa C Flint, Steven D Rhodes | P | D |
| alternateDataRepository | geo:GSE213789, GSE213787 (**213789 wrong**) | geo:GSE213786, GSE213787 + SRP398255/57 | P | D (orig error **S, S4**) |

### Issue 11 — syn74309473 (HuR/ELAVL1 MPNST, GEO+SRA)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| project annotations | (mostly correct) | unchanged | P | — |
| Dataset `id`/`name` cols | — | **absent / not first** (both datasets) | DS | **S (S1)** |
| Source Metadata folder | — | **empty folder present** | struct | **S** |

### Issue 10 — syn74288246 (NF1 malignant progression scRNA-seq, ENA+ArrayExpress)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | Katarzyna J. Radomska, Piotr Topilko | Katarzyna J Radomska, Piotr Topilko | P | D |
| alternateDataRepository | insdc.sra:PRJEB77277 | bioproject:PRJEB77277 | P | D |
| Dataset `filename` column | — | **present (duplicate of name)** | DS | **S (S2)** |
| File `resourceStatus` | — | **present on File entities** | F | **S (S2)** |
| 2nd Dataset `id`/`name` cols | — | absent / not first | DS | **S (S1)** |

### Issue 9 — syn74288412 (MEK/BET/CDK in MPNST, ENA)
| Field | NADIA original | Published | Lvl | |
|---|---|---|---|---|
| studyLeads | (correct already) | Sara Ortega-Bertran, Eduard Serra | P | — |
| alternateDataRepository | insdc.sra:PRJEB83680 | bioproject:PRJEB83680 | P | D |
| Dataset `id`/`name` cols | — | **absent entirely** (syn74288449) | DS | **S (S1)** |

---

## Summary of highest-value (silent) finds

1. **S1 — 6/19 published datasets lack `id`/`name` system columns** (one missing them entirely). NADIA's reports *claim* the fix; the live entity contradicts it. No GitHub feedback exists for this. → Add a read-back assertion in Phase 3.
2. **S2 — issue 10 has `filename` dataset column + `resourceStatus` on files**, both surviving approval. → Apply `EXCLUDE_COLS` against stored state, especially for Tier-4 / h5ad-derived annotations.
3. **S3 — Dataset-level `resourceStatus` + stable version not consistently set** on multi-dataset projects.
4. **S4 — wrong GEO accession (GSE213789)** on first pass — Standard 7 ownership check would have caught it.
5. **Empty Source Metadata folders** persist (issue 11, and 216 at audit time) despite the no-empty-folder rule.

The documented fixes cluster on **author-name formatting (Rank 1)** and **cross-repository accession completeness (Rank 2)** — both fully deterministic from data NADIA already fetches, and both should be eliminated at first pass rather than deferred to human/polish review.
