# NADIA First-Pass Quality Analysis — What Curators Had To Fix

**Goal:** Make NADIA's first-pass output publication-ready so curators stop iterating.
**Method:** Reconstructed the delta between NADIA's initial output and the published/curated state from 45 GitHub issue threads (13 approved + 32 closed) in `nf-osi/nadia`.
**Date of analysis:** 2026-06-27

---

## How to read this report

Each issue thread has a predictable shape: (1) NADIA's initial `## NADIA Study Review` body; (2) an auto-`## NADIA Polish Pass` comment; (3) human `/nadia fix:` comments (the GOLD signal); (4) `## NADIA Fix Pass` comments; (5) approval. I extracted every `/nadia fix:` request verbatim and every concrete field change (field, old→new, entity level).

**Caveat on the corpus:** Issues 254–265 are `model-comparison-test` runs (titles prefixed `MODEL-TEST-*`), used to benchmark models, not real curations — they were not iterated to publication. I use their *initial bodies* as evidence of recurring first-pass defects, but I do NOT count them as "curator had to fix" events. Issues 60/64/65/66/32 are dead projects (trashed in Synapse, 0 files) closed administratively.

---

## TAXONOMY — ranked by frequency × severity

| Rank | Category | Freq (issues) | Severity | Priority |
|------|----------|---------------|----------|----------|
| 1 | Dataset entity not publication-ready (schema columns, naming, stable versions) | ~13 | High | **CRITICAL** |
| 2 | File annotations incomplete / set at study level not per-sample | ~11 | High | **CRITICAL** |
| 3 | One paper fragmented into multiple projects (cross-repo dedup failure) | 6+ | High | **CRITICAL** |
| 4 | Non-schema annotations on files (`resourceStatus`, `filename`, `externalAccessionID`) | ~7 | Med-High | **HIGH** |
| 5 | Wrong schema template bound | ~6 | High | **HIGH** |
| 6 | Wrong disease focus / manifestation (germline vs somatic; focus vs manifestation) | ~6 | Med-High | **HIGH** |
| 7 | Study leads wrong format or wrong people (submitter vs author) | ~8 | Med | **HIGH** |
| 8 | Incomplete file enumeration (stub/landing-page only) | ~5 | High | **HIGH** |
| 9 | Project never should have been created (out-of-scope / summary-only / empty / re-analysis) | ~7 | Med | MED |
| 10 | Wrong accession linked to paper (elink false positive / wrong PMID) | ~3 | High | MED |
| 11 | Missing funding / institutions (Acknowledgements not parsed) | ~5 | Med | MED |
| 12 | `externalRepository` = discovery path not actual host (GEO vs ENA/SRA) | ~4 | Low-Med | MED |
| 13 | Wiki not to spec / project name = repo title or synapse ID | ~6 | Low-Med | MED |
| 14 | Empty folders / zip files left in project | ~5 | Low | LOW |
| 15 | Invalid / non-existent bioregistry prefix | 2 | Low | LOW |
| 16 | `alternateDataRepository` missing sibling accessions (GEO↔SRA) | 3 | Low-Med | LOW |

---

## CRITICAL categories (fix these first)

### 1. Dataset entity not publication-ready — schema columns, naming, stable versions
**Frequency:** ~13 issues (216, 35, 28, 11, 10, 9, 25, 186, 80, 290, 124, 59, + every model-test)
**Severity:** High — appears in nearly every iterated thread; usually requires multiple round-trips.

**What goes wrong (three recurring sub-failures):**
- **Column order/completeness.** Dataset views show no identifier or are missing annotation columns. Curator quotes:
  - #216: *"there are a large number of columns missing in the dataset schema"*; *"make sure all of the annotation columns are included"*
  - #28: *"dataset is not showing any of the file annotations"*; *"dataset schemata are not comprehensive and ordered correctly"*
  - #11: *"please make sure both datasets have id, file, and then all annotations in the schema"*
  - #35: *"make sure the dataset is updated to include all annotations, it looks like it's missing 6 props"*
  - #10: *"please make sure the dataset schemas are id, filename, and then all annotations"*
- **Non-human-readable names.** Datasets named `GEO_GSE163028`, `Zenodo_4662239`, `GEO_GSE213789`. Curator quotes:
  - #28: *"dataset name is not human readable"*; #11: *"name the datasets something more human readable/intuitive"* (requested 3×); #216: *"make the names more descriptive"*
  - Fix-pass results show the target form: `single-cell RNA-seq - skin, Cutaneous Neurofibroma (GEO GSE163028)`; `RNA-seq - Schwannoma, MS03 cell line (Mus musculus) (ENA PRJNA1222828)`.
- **Stable versions not minted.** Curators repeatedly had to ask: #216 *"mint stable versions"*; #11 *"mint stable versions of both datasets"*; #9 *"mint a stable version of the dataset (syn74288449)"* (3×); #10 *"mint new stable dataset versions"*.

**Root cause:** Despite Standard 21/checklist and the dynamic-columnIds rule, the first pass is not actually building `columnIds = [id, name, *all annotation keys]`, is using the placeholder `{Repo}_{Accession}` dataset name, and is deferring version minting (correctly, to Phase 3) but Phase 3 is not running on the first pass.

**Recommendation → CLAUDE.md "Per dataset" checklist + `prompts/synapse_workflow.md` + audit Phase 3:**
1. Make the dataset-creation helper in `prompts/synapse_workflow.md` ALWAYS derive `columnIds` from the union of all file annotation keys at the moment the Dataset is finalized, with `id`(ENTITYID) and `name`(STRING) prepended — and have the self-audit (Step 7) hard-fail if a Dataset's columnIds don't contain every key present on its member files.
2. Make the human-readable dataset name a REQUIRED computed field, not optional: `{assay} - {specimenType/tissue}, {tumorType/diagnosis} ({Repo} {Accession})`. Add an audit check that rejects any dataset name matching `^{Repo}_{Accession}$`.
3. Run Phase 3 stable-version minting on the FIRST pass after annotations are finalized (the checklist says "Phase 3, not creation" — but the first pass never reaches Phase 3). Every project's audit entry must mint versions for all `dataset_ids_to_snapshot`.

---

### 2. File annotations incomplete or set at study level (Standard 5 violations)
**Frequency:** ~11 issues (35, 29, 28, 25, 11, 144, 290, 80, 186, 124, 263/265 test)
**Severity:** High — this is the core curation work; when wrong, every file needs re-annotation.

**What goes wrong:**
- **Sample-varying fields set uniformly.** Polish passes repeatedly fixed per-sample fields that NADIA set study-wide:
  - #35: `diagnosis` all `Neurofibromatosis type 1` → per-sample (`Not Applicable` for WT controls); `tumorType` all `Plexiform Neurofibroma` → `Not Applicable` for WT; added per-sample `nf1Genotype` (`+/+` vs `-/-`).
  - #290: `tumorType`/`diagnosis` corrected per specimen from `Sample_list.xlsx` (Neurofibroma→Hybrid PNST, Sarcoma→Synovial Sarcoma, etc.) — 71 files updated.
- **specimenID/individualID = run accession (SRR/ERR).** Direct Standard 5 violation, fixed in #35 (`ERR3568671`→BioSample `SAMEA5986218`), #186 (`SRR32319612`→ENA `sample_title`).
- **Missing optional-but-required fields.** #11 *"lots of optional annotations are missing (e.g. modelSystemName)"*; #25 curator listed `modelSex: Female (still appears as Unknown)`, missing source metadata; #28 *"file annotations are missing important experimental metadata. look to the source metadata and/or publication."*

**Root cause:** The first pass populates annotations from study-level summaries instead of fetching per-sample records (GSM characteristics, ENA filereport per BioSample), and does not run the Tier 1–4 gap-fill before filing.

**Recommendation → `prompts/annotation_gap_fill.md` + CLAUDE.md Standard 5/11:**
1. Make per-sample mapping (run→sample→biological ID) a MANDATORY first-pass step, not a polish step. The first pass must call the ENA filereport with ALL columns / GEO GSM characteristics and assign `specimenID`/`individualID` from BioSample/`sample_title`/GSM — never from SRR/ERR/DRR.
2. Add an audit assertion (Step 7): for any multi-sample study, FAIL if any biological field (`nf1Genotype`, `diagnosis`, `tumorType`, `sex`, `specimenType`) has an identical value across all files. This already exists as a "signal" in Standard 5 — promote it to a blocking audit check.
3. Run the full 4-tier gap-fill on the first pass and require a documented reason for every blank schema field before the issue is filed (Standard 11/14).

---

### 3. One paper fragmented into multiple Synapse projects (cross-repository dedup failure)
**Frequency:** 6+ issues (186, 112, 110, 109, 224, 269, and #37/#28 as consolidation targets)
**Severity:** High — requires manual cross-project entity moves; one paper produced up to **5 separate projects**.

**What goes wrong:** When a single publication deposits data across GEO + SRA/ENA + multiple PRIDE accessions, each connector discovers an accession independently and creates a SEPARATE project. The Mitchell/Clapp NF2 paper (PMID 41616055) fragmented into:
- syn74281986 (GSE292315 + GSE289387), syn74319019 (PRJNA1222828), syn74314703 (PXD060730), syn74314752 (PXD068581) — issues #186, #110, #112 all had to be consolidated into canonical #37.
- The Cabozantinib paper (PMID 33442015): PXD019005/PXD019138 created as orphan project (#109) separate from #28.
- #224: *"tear down the agent-created existing project syn74320111 ... it's been merged with #223"*; #269 closed as duplicate of #271 (same syn74317113).

**Note:** `insdc.sra:PRJNA1222828` is just the BioProject mirror of GSE289387 — NADIA created a whole second project for the same RNA-seq data in a different repository.

**Root cause:** Dedup (Step 0 PMID dedup against state table) only fires when the SAME PMID is re-seen. But the secondary repository-direct passes (PRIDE, ENA) often don't resolve the PMID, so cross-repo deposits of one paper never get grouped. CrossRef/Europe PMC linking is not being used to back-resolve PRIDE/ENA accessions to their paper before project creation.

**Recommendation → CLAUDE.md "Discovery Architecture" + "Deduplication Step 0" + `prompts/repo_apis.md`:**
1. Before creating ANY project from a repository-direct accession, MANDATORY-resolve its publication (the "Before Creating Any Project" section already requires this — enforce it as a hard gate). PRIDE/ENA records carry pubmed/DOI cross-refs; use them.
2. Group accessions by resolved PMID/DOI ACROSS all discovery paths into one publication group BEFORE the NEW/ADD/SKIP decision. A GEO series's `!Series_relation` SRA/BioProject links and a PRIDE project's linked PMID should collapse into one group.
3. Dedup Step 0 must match on DOI too (not just PMID) and must check whether the same paper already has an agent-created project this run, adding the new accession via ADD rather than NEW.
4. Recognize BioProject↔GEO mirroring: if a discovered BioProject is referenced by an already-processed GEO series, treat as the same dataset (alternateDataRepository entry), not a new project.

---

## HIGH categories

### 4. Non-schema annotations set on File entities
**Frequency:** ~7 issues (29, 9, 11, 28, 10, 35, others)
**Severity:** Med-High — creates spurious/duplicate columns in the portal Dataset view.

**What goes wrong:** First pass writes fields onto files that don't belong there:
- `resourceStatus` on files — #9 *"remove resourceStatus annotations from all files"* (3×); #11 *"remove resourceStatus annots from files"*.
- `filename` / custom `name` annotation — #10: *"please remove the filename annotations from all files. I want the name column in the datasets"* (curator had to clarify "name" = system property, not annotation, across 3 comments).
- `externalAccessionID`, `externalRepository`, `study` on files — #29 fix pass removed all three ("Not a file-schema field — belongs on the Dataset entity").

**Root cause:** First pass sets a fixed bundle of fields on files without validating against `fetch_schema_properties()`, contradicting Standard 18 and the "NEVER set on File entities" rule.

**Recommendation → CLAUDE.md Standard 18 + File-Level annotation section + audit:**
1. Hard-enforce: before writing ANY file annotation, intersect the intended key set with the bound schema's properties; drop everything not in the schema.
2. Maintain an explicit `EXCLUDE_COLS = {'resourceStatus','filename','name','externalAccessionID','externalRepository','study','studyId'}` for files and add an audit check that scans every file for these keys and removes them (this is the single most repeated curator request — automate it 100%).

### 5. Wrong schema template bound
**Frequency:** ~6 issues (124, 11, 144, 186, 290, 35)
**Severity:** High — wrong validation rules; requires re-annotation.

**What goes wrong:** Template chosen from paper title/disease context, not actual data modality:
- #124: ChIP-seq data had `processedgeneexpressiontemplate` bound → rebound to `chipseqtemplate`.
- #11: *"GSE120686 is chip seq and can use the epigenomics assay schema"* (curator had to point out the ChIP-seq sub-series; assayTarget needed).
- #186: `rnaSeq`→`RNA-seq` and rebound to `rnaseqtemplate`. #290: spatial imaging vs sequencing templates needed splitting per folder.

**Root cause:** Step 12/schema-binding section requires modality from `library_strategy` but the first pass infers from title.

**Recommendation → CLAUDE.md "Metadata Schema Binding" + Standard 12:** Make `library_strategy`/`library_source` (ENA) or `!Series_library_strategy` (GEO) the REQUIRED input to template selection; forbid title-based selection. For SuperSeries/multi-assay, determine per-accession modality and bind per files-folder. Add audit: cross-check bound template name against the detected `library_strategy` and flag mismatches.

### 6. Wrong disease focus / manifestation
**Frequency:** ~6 issues (124, 144, 290, 35, and model-tests 263/290; also 244 out-of-scope)
**Severity:** Med-High — corrupts portal search/filters.

**What goes wrong:**
- **Spurious extra disease.** #124: `diseaseFocus` had both NF1+NF2 → NF1 removed (NF2-only study); `manifestation` had MPNST removed (schwannoma study only).
- **Focus vs manifestation confusion.** `Disease Focus: Neurofibromatosis type 1, MPNST` (#290 body, #263 test) — MPNST is a tumor type/manifestation, not a disease-focus value.
- **Invalid portal vocabulary.** #144: manifestation `Malignant Peripheral Nerve Sheath Tumor` → `MPNST, Neurofibroma, Plexiform Neurofibroma` ("previous value was not a valid portal term; verified live from syn52694652").
- **Germline vs somatic** (#244 out-of-scope; #35 WT controls).

**Recommendation → CLAUDE.md Standard 13 + "Required Annotations":**
1. Enforce the focus-vs-manifestation distinction in the prompt with an explicit rule: disease-focus values come ONLY from `annotations.disease_focus_values`; tumor types (MPNST, PN, schwannoma) are manifestation/tumorType, never disease focus.
2. Always validate manifestation/diseaseFocus against the LIVE portal table (syn52694652) at first-pass time, not config cache (Standard 1 note — config lags).
3. Don't add a disease just because a gene mutation appears; apply the germline-vs-somatic test (Standard 13) at scoring time.

### 7. Study leads — wrong format or wrong person
**Frequency:** ~8 issues (9, 144, 34, 33, 228, 80, 186, model-tests 263/265/261)
**Severity:** Med — but trivially automatable, so high ROI.

**What goes wrong:**
- **Name format `Lastname Firstname`** persists into the LATEST runs: `Høland Maren, Lothe Ragnhild A` (#144), `Schönung Maximilian, Lipka Daniel B` (#34), `Wang Jiawan` / `Imle Roland` / `Soni, Nishant, Tsankov, Alexander M` (model-tests). Standard 3 requires `Firstname Lastname`.
- **Wrong people / submitter not author.** #9: *"study leads — first author should be Sara Ortega-Bertran and corresponding author should be Eduard Serra"*. #80: `studyLeads: Unknown` → resolved from PubMed. #186: `Not Available` → `Mitchell Dana K, Clapp D Wade` (still wrong format!).

**Recommendation → CLAUDE.md Standard 3 / Annotation Quality:** This is a pure formatting bug — fix once in the PubMed parser. Always combine `f"{ForeName} {LastName}"` and NEVER ship `Lastname Firstname` or `Lastname, F`. Add an audit regex that flags any studyLead where token[0] looks like a surname + initials. When PMID exists, studyLeads = first author + corresponding author from AuthorList, never `Unknown`/`Not Available`.

### 8. Incomplete file enumeration (stub / landing-page only)
**Frequency:** ~5 issues (144, 59, 25, 28, 290)
**Severity:** High — project points at a landing page, not data (Standard 13/file-enumeration-required).

**What goes wrong:**
- #144: project had **1 stub GEO URL pointer** → polish replaced with **158 properly-registered files** (79 CEL.gz + 79 CHP.gz).
- #25: project *"originally missing the GSE213786 sub-series entirely"* — only had 1 raw-counts file, missing all FASTQs and a whole sub-series; polish added 48+12 FASTQs.
- #59: TCIA dataset had no file-level entities; curator: *"we need file-level entities ... I would like to instead just mirror the raw data files"* → 1,210 file entities created.
- #28: *"txt and txt2.zip files ... archived files aren't acceptable, need to be unzipped and uploaded"*.

**Recommendation → CLAUDE.md Standard 13 + `prompts/repo_apis.md`:**
1. Make `get_file_list_*` enumeration mandatory and FAIL the project (don't file as `synapse_created`) if it falls back to a landing-page ExternalLink — flag `file-enumeration-required`.
2. For GEO SuperSeries, expand ALL sub-series (`!Series_relation: SuperSeries of`) and enumerate every GSM's supplementary files + linked SRA runs — don't stop at one counts file.
3. TCIA: always enumerate per-series file entities (Standard 17) on the first pass, not after a curator asks.
4. Reject/unzip archives: zip/tar containing primary data must be expanded (Standard summary-only / no zip-as-final-annotation).

---

## MEDIUM categories

### 9. Project should never have been created
**Frequency:** ~7 issues (235, 213, 246, 244, 228, 33, 34)
**Severity:** Med — wasted review cycle + teardown.

**Examples (curator verbatim):**
- #235: *"tear down ... the data are summary statistics and cannot reasonably be used to reproduce this study's findings"*.
- #213: *"not raw dataset - supp table from a paper. closing."*
- #246: *"looking at OSF there are not any actual data in the repository yet."* (empty repo).
- #244: *"this is a cancer study and not relevant to germline loss of SMARCB1"* (somatic, out of scope).
- #33/#34: closed as duplicates of EXISTING portal studies (syn51133946/57) — dedup missed pre-existing portal coverage.

**Recommendation → CLAUDE.md "Relevance Scoring" reject rules + Deduplication:** These reject criteria already exist (summary-only, stub/empty, re-analysis, somatic out-of-scope). Enforce them at SCORING time before any Synapse write: (a) fetch the file list and reject if all-documents; (b) fetch file count and reject 0-file repos; (c) apply germline-vs-somatic test; (d) extend dedup to query the live portal `studies_table_id`/`datasets_table_id` for the accession AND title, not just the agent state table (issues 33/34 were already in the portal).

### 10. Wrong accession linked to paper (elink false positive / wrong PMID)
**Frequency:** ~3 issues (228, plus 186 cross-refs)
**Severity:** High when it happens — produces a nonsense project.

**Example:** #228 — clinical case report (PMID 41569052, no data) was linked to SRP579342, which actually belongs to an unrelated neuroblastoma study (PMID 41560679). Curator: *"audit and correct this project, lots wrong here"* → *"recommend closing this project."*

**Recommendation → CLAUDE.md "CRITICAL — Verify elink accession ownership" + Standard 7/20:** The ownership-verification step exists but is not running. Make it a hard gate: for every elink/EuropePMC accession, verify the repository record's `PubMedIds`/`study_title` matches the paper. Reject case reports / reviews with no data-availability statement before creating anything (Standard 20).

### 11. Missing funding / institutions (Acknowledgements not parsed)
**Frequency:** ~5 issues (80, 144, 290, 186, others)
**Severity:** Med — Tier 2 gap-fill not run on first pass.

**Examples:** #80 `fundingAgency: Not Applicable` → Wellcome/MRC/EPSRC from GrantList; #144 `Not Applicable` → Norwegian Cancer Society from PMC Acknowledgements; #290 added 2 funders from GrantList; #144 institutions 1→5 from author affiliations.

**Recommendation → CLAUDE.md Standard 11 Tier 2 / Standard 14:** When a PMID exists, fundingAgency and institutions MUST come from GrantList then PMC Acknowledgements on the FIRST pass — the "Not Applicable (External Study)" placeholder is only valid when no PMID/PMC exists. Add audit: if PMID present and fundingAgency == placeholder, flag as incomplete.

### 12. `externalRepository` reflects discovery path, not actual host
**Frequency:** ~4 issues (186, 25, 29)
**Severity:** Low-Med.

**Examples:** #186 `externalRepository: SRA` → `ENA` (files on ENA FTP); #25 curator asked *"why is the externalRepository for the FASTQs 'ENA' instead of 'SRA'?"* (ambiguity both directions — needs a clear rule). Standard 19 already covers this.

**Recommendation:** Derive `externalRepository` from the actual download URL host (`ftp.sra.ebi.ac.uk`→ENA), and document the GEO-vs-SRA-vs-ENA convention explicitly so it's consistent (the #25 question shows the rule is ambiguous to humans too — pick one and state it).

### 13. Wiki not to spec / bad project name
**Frequency:** ~6 issues (216, 59, 186, 28, 60/64/65/66/32 = syn-ID names)
**Severity:** Low-Med.

**Examples:** #216 *"please change the project title to the pub title ... clean up the project wiki"* (project was named with a repo-record title `Spatial transcriptomic analysis of Nf1+/- mouse brain` instead of pub title). #216 wiki footer: *"wiki should have Auto-curated by @nadia-bot and reviewed by NF-OSI"*. Issues 60/64/65/66/32 had project name = the synapse ID itself (no publication resolved, 0 files).

**Recommendation → CLAUDE.md "Project Name" + wiki template:** Enforce project name = publication title (never repo title, never synapse ID). If no publication resolves AND 0 files, do NOT create the project. Fix the wiki footer template once.

---

## LOW categories

### 14. Empty folders / zip files left in project
**Frequency:** ~5 (28, 59, 25, 290, 216) — *"Analysis folder is empty and can be deleted"* (#28), deleted empty `Analysis/` (#25, #290), empty `Source Metadata` (#59). **Recommendation:** audit must delete empty folders before filing (already in checklist — enforce).

### 15. Invalid bioregistry prefix
**Frequency:** 2 (#59 `icrptbia:` → `tcia.collection:`). **Recommendation:** validate every `alternateDataRepository` prefix against the REPO_TO_PREFIX table / Bioregistry before writing; never invent prefixes.

### 16. `alternateDataRepository` missing sibling accessions
**Frequency:** 3 (#25 *"Restore alternateDataRepository inclusion of insdc.sra:SRP398255 ... and add insdc.sra:SRP398257"*; #186 added 5 accessions). **Recommendation → Standard 8:** always expand GEO `!Series_relation` to SRA/BioProject and include all siblings on the first pass.

---

## Cross-cutting observations & top recommendations

1. **The single biggest win is making the self-audit (Step 7) actually run Phases 1–3 on the FIRST pass.** Almost every fix above is already codified as a CLAUDE.md Standard or checklist item — the gap is that the first pass files the issue *before* the audit/gap-fill/version-mint phases execute. Categories 1, 2, 4, 8, 11, 14 would largely vanish if the first pass completed the full audit + Phase 3 before posting.

2. **Convert the most-repeated curator requests into hard, deterministic audit gates** (these need zero LLM judgment):
   - Strip `resourceStatus`/`filename`/custom-`name`/`externalAccessionID`/`externalRepository`/`study`/`studyId` from all file entities (Cat 4).
   - `columnIds = [id, name, *all-file-annotation-keys]` (Cat 1).
   - studyLead name format `Firstname Lastname` regex check (Cat 7).
   - Reject dataset name == `{Repo}_{Accession}` (Cat 1).
   - Reject project with landing-page-only files / 0 files (Cat 8, 13).
   - Delete empty folders (Cat 14).
   - Validate `alternateDataRepository` prefixes (Cat 15).

3. **Publication-first grouping must happen before project creation, across ALL repositories.** Category 3 (fragmentation) is the most expensive to fix manually (cross-project entity moves) and is structural: resolve every repository-direct accession to its PMID/DOI and group before NEW/ADD/SKIP. Extend dedup Step 0 to DOI and to BioProject↔GEO mirroring.

4. **Per-sample metadata is not optional first-pass work.** Categories 2, 5, 6 all stem from populating annotations from study-level summaries and the paper title instead of per-sample repository metadata (ENA filereport per BioSample, GEO GSM characteristics, `library_strategy`). Make per-sample fetch + 4-tier gap-fill mandatory before the issue is filed.

5. **Always validate controlled vocab against the LIVE portal table (syn52694652), not config** — Cat 6 invalid-term fixes (#144) and germline/somatic scoping (Cat 6/9) recur because config lags the portal.
