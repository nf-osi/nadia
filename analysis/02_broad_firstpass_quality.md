# NADIA First-Pass Quality Gaps — Corpus-Wide Analysis

**Scope:** All 277 `study-review` GitHub issues in `nf-osi/nadia` (closed + open, fetched 2026-06-27 via `gh issue view ... --json number,title,body,comments,labels`).
**Method:** Each issue dumped to JSON; programmatic harvest + clustering of (1) curator `/nadia fix:` comments, (2) NADIA's own "Items for human review" bullets, (3) labels, (4) self-flagged "approximation / could-not-determine / Unknown" admissions in NADIA curation comments.

## Corpus shape (denominators matter)

| Metric | Count | Notes |
|---|---|---|
| Total `study-review` issues | 277 | every project files one |
| Issues with ≥1 NADIA bot comment thread | 66 (24%) | the "engaged"/curated subset — most signal lives here |
| Issues with ≥1 curator `/nadia fix:` | 46 (17%) | i.e. ~70% of engaged issues required at least one human correction round |
| Issues with `approved` label | 13 (5%) | only 13 have made it through to provisioning |
| Genuine curator `/nadia fix:` comments | 103 | across those 46 issues — avg ~2.2 fix rounds per touched issue |
| Total comments across corpus | 729 | |

**Headline:** When a human actually reviews a NADIA project, it almost always needs corrective rounds — 46 of 66 engaged issues (70%) drew at least one `/nadia fix:`. First-pass output is rarely approve-on-sight. The gaps below are ranked by how often they drive that human correction.

**Labels in use:** `automated` (277), `study-review` (277), `dhart-spore` (18), `approved` (13), `model-comparison-test` (12). Notably `file-enumeration-required` and `ntap-integration-review` are *referenced in text* (file-enumeration-required appears in 21 issues / 8%) but are **not applied as GitHub labels** — the CLAUDE.md Standard 13 / NTAP rule prescribes labels that NADIA never actually sets.

---

# Ranked taxonomy of first-pass quality gaps

Two prevalence lenses are reported:
- **Curator-fix prevalence** = % of the 103 `/nadia fix:` comments that touch the theme (what humans actually had to correct, multi-label).
- **Self-flag prevalence** = % of all 277 issues where NADIA's own text admits the gap.

---

## TIER 1 — The dominant, recurring gaps (a Standard exists but is NOT working)

### 1. Schema / template selection & population is wrong or incomplete — **32% of fixes (33/103); NADIA self-flags schema/template uncertainty in 44 review-bullets**
The single largest driver of human correction. Sub-failures:
- Wrong template bound (RNA-seq schema on WGS data, epigenomics vs transcriptomics).
- Missing required schema fields → "files fail validation because several required fields are missing (e.g. `libraryStrand`, `libraryPrep`)" (#22).
- Curator has to redirect template choice: "try using the ProcessedGeneExpression template with the rawcounts files" (#272); "GSE120686 is chip seq and can use the epigenomics assay schema… assay target where the chip targets can be annotated" (#11).
- Aggregate count matrices break the per-specimen schema model: "Each rawcounts file is a multi-sample count matrix, so `specimenID` (schema requires a single string)…" (#272).

**Root cause:** NADIA selects/populates schema from inference (paper title, disease context) rather than verifying library metadata first, and does not always run the Standard 11 four-tier gap-fill to completion before filing. Aggregate/processed-matrix files have no first-class handling.
**Already supposed to handle?** YES — Standards 1, 6, 11, 12, and the Metadata Schema Binding section all cover this. The rule exists but is not reliably executed. **Strengthen, don't add.** Make template selection a hard gate that reads `library_strategy`/`library_source` and records it in scored metadata (CLAUDE.md → "Metadata Schema Binding" + Standard 12); add explicit handling for multi-sample count matrices to Standard 5 (a matrix file maps to many specimens — `specimenID` cannot be a single value).

### 2. Dataset entity display broken — wrong column order, missing annotation columns, non-human-readable names, stale file versions — **17% of fixes (17/103)**
Extremely repetitive and mechanical:
- "first two cols of datasets should be id, name" / "please make sure both datasets have id, file, and then all annotations" (#11, #223, #220, #251).
- "dataset is not showing any of the file annotations" (#28); "it looks like it's missing 6 props" (#35); "a large number of columns missing in the dataset schema" (#216).
- "datasets still don't show the annotations because they are pointing at the v:1 instead of v:latest of the files" (#27) — **Dataset items reference pinned old file versions**, so annotation fixes don't surface.
- Names: "dataset name is not human readable" (#28); "name the datasets something more human readable/intuitive" (#11, #251).
- `name` column empty because a custom `name`/`filename` annotation was added instead of using the system property (#10, #251).

**Root cause:** `columnIds` not consistently `[id, name, …annotations]`; Dataset `items` pin specific file versions; dataset naming uses generic repo strings; the "no custom name/filename annotation" rule is violated in practice.
**Already supposed to handle?** YES — Project Completion Checklist (id|name|annotations column order, dynamic columnIds, descriptive dataset name), Standards 18 & the "NEVER set on File entities" block. Still wrong repeatedly. **Strengthen:** make the audit (Phase 3) assert column order, assert Dataset `items` use `syn…` without a pinned version (or re-point to latest after file edits), and assert dataset name ≠ `{Repo}_{Accession}`.

### 3. File enumeration incomplete — landing-page stubs, missing ENA runs, un-extracted tar archives — **17% of fixes (18/103); 21 issues carry "file-enumeration-required" text (8%); "landing page" in 31 (11%); tar/needsExtraction in 43 (16%)**
- "where are all the files?" (#23); "the raw data aren't indexed and annotated, dataset doesn't contain raw data" (#267); "a number of ENA runs are missing??" (#184).
- "3 datasets contain landing-page stubs instead of enumerated files" (#22).
- GEO `*_RAW.tar` archives left with `needsExtraction: true` and never expanded (#22, #220 — idat files suspected inside the tar).

**Root cause:** When `get_file_list_*` returns empty, NADIA falls back to a landing-page ExternalLink and logs `synapse_created` anyway. GEO RAW.tar contents are not enumerated. SRA→ENA run resolution misses runs.
**Already supposed to handle?** YES — Standard 13 (file enumeration: a landing-page link is not a complete dataset; must flag, must NOT log as `synapse_created`). The flag (`file-enumeration-required`) is referenced but not enforced as a blocking condition or as a real label. **Strengthen:** make landing-page-only or unextracted-RAW.tar a *blocking* audit error, not a flag; build a GEO RAW.tar member-listing step.

### 4. Out-of-scope / non-reusable projects created (stub, summary-only, reanalysis, somatic-only) → curator tears them down — **7% of fixes (tear-down theme); ~13 distinct teardown requests**
- "this can be closed and torn down. this is a stub dataset ie not raw or otherwise meaningfully…data" (#233).
- "re-analysis of existing geo data. the existing geo data should be indexed if it hasn't been" (#194).
- "this is a cancer study and not relevant to germline loss of SMARCB1" (#244) — somatic-only.
- "the data are summary statistics and cannot reasonably be used to reproduce…or mine" (#235); summary + unavailable targeted panels (#243).
- OSF record with no data yet (#246).

**Root cause:** Scoring/rejection gate is not catching summary-only, reanalysis, stub, and somatic-only cases reliably *before* project creation. These represent wasted Synapse writes + human teardown effort.
**Already supposed to handle?** YES — "Reject these regardless of relevance score" (re-analysis, summary-only, stub/empty) and Standard 13 disease scoping (somatic vs germline). The rejection is supposed to happen *before any Synapse write*. Still leaking through. **Strengthen:** enforce the file-list-content check (all-document = reject) and the OSF/Zenodo zero-file check at scoring time as hard gates, and add a somatic-only screen to the relevance step that keys on "somatic" / "acquired mutation" language plus absence of germline-cohort signals.

---

## TIER 2 — Frequent, high-cost gaps

### 5. Controlled-vocabulary / enum validity failures (invalid values, wrong URI, bad prefix, Unicode mismatch) — **11% of fixes; NADIA self-flags vocab gaps in 33 review-bullets; 65% of issues mention "PR needed / schema extension / not in dictionary"; 37% use "closest enum/approximation"**
- Invalid enum written: "HSC1λ is the schema value, not HSC1L" (#220); "diagnosis = 'Neurofibromatosis type 1' — valid schema enum?" (#220).
- Wrong term URI: "you said the wrong uri for invasive breast carcinoma. please make sure to look these up" (#177).
- Bogus bioregistry prefix invented: "icrptbia:Vestibular-Schwannoma-SEG is not a real bioregistry prefix… change to tcia.collection" (#59).
- Case/typo handling: platform enum `Q Exative HF` typo vs source `Q Exactive` (#28).

**Root cause:** NADIA writes values without verifying character-for-character against the *live* schema enum (Unicode, case), and sometimes invents prefixes. The very high (65%) "PR needed / not in dictionary" rate shows genuine vocabulary gaps too, but a large share are NADIA failing to find an existing valid value.
**Already supposed to handle?** YES — Standards 1, 9, 18, 22 (enums are ground truth, case-sensitive, Unicode-exact, validate field names, flag-don't-drop) and the `alternateDataRepository` prefix table. Repeated violations mean the runtime enum-fetch + `validate_against_enum()` step is being skipped or not trusted. **Strengthen:** make enum validation mandatory and return-value-driven (use the function's exact-case return, never the raw source); add a prefix allowlist check against the CLAUDE.md table so invented prefixes (icrptbia:) are impossible.

### 6. Project-level publication metadata left as placeholders — studyLeads, institutions, fundingAgency — **studyLeads "Unknown/Not Available" in 76 issues (27%); fundingAgency "Not Applicable (External Study)" placeholder in 96 issues (35%); 53% of all issues contain an "Unknown" placeholder**
- "studyLeads is currently 'Unknown' — if the PI can be identified…" (#59); "studyLeads is still 'Not Available' — no PMID linked" (#186).
- Author-name *format* wrong: "studyLeads follows Firstname Middlename/initial Lastname style, not Lastname, firstname" (#223); curator supplied the actual names (#9).
- fundingAgency placeholder used even when a PMID/PMC exists: "Set to placeholder… no grant information is present in the Zenodo deposit" (#243, #186) without consulting PMC Acknowledgements.

**Root cause:** For repository-direct (no-PMID) candidates NADIA gives up on publication resolution; even with a PMID it sometimes leaves placeholders rather than parsing AuthorList/affiliations/GrantList; PMC Acknowledgements Tier-2 funding fallback under-used; author-name reformatting (`<ForeName> <LastName>`) not applied.
**Already supposed to handle?** YES — Standards 3, 11, 14, "Before Creating Any Project — Resolve the Publication First," and the Project Completion Checklist. 27–35% placeholder rates show the publication-resolution + Tier-2 gap-fill is not running to completion. **Strengthen:** make studyLeads/institutions/fundingAgency a blocking checklist item when a PMID/preprint exists; force the PMC Acknowledgements parse before any fundingAgency placeholder; enforce name reformatting in the writer.

### 7. Per-sample / sample-varying fields set at study level (genotype, sex, tumorType, specimenID) — **genotype 6%, tumorType/specimenType 10%, sex 7% of fixes; NADIA self-flags per-sample identifier issues in 17 review-bullets, genotype in 10, sex in 5; sex=Unknown-on-all-files in 26 issues (9%)**
- "NF1_genotype for the WT mice should be +/+. For normal nerve samples, tumorType should be set to Not Applicable" (#272).
- "All 32 ENA files currently have nf1Genotype = Unknown… samples split into plusDOX vs minusDOX" (#177) — uniform value despite distinct groups.
- "sex is wrong on the scRNA-seq files" (#27); "If sex cannot be directly verified, leave it blank and do not extrapolate" (#272, #29) — NADIA both fails to populate AND over-extrapolates in different cases.
- specimenID/individualID as run accessions instead of biosample/sample_title.

**Root cause:** Fields populated once at study level rather than re-derived per file from per-sample metadata; run accessions misused as biological identifiers; and an over-eager inference of `sex` when it should be left blank.
**Already supposed to handle?** YES — Standard 5 (sample-varying fields per-file; run accessions are not specimen IDs) and Checklist ("no sample-varying field has the same value on all files"). Still failing. **Strengthen:** add an audit assertion that fails when a known sample-varying field is uniform across all files in a multi-group study; and an explicit "do not extrapolate `sex`; leave blank if not in source" rule (curator stated this twice — #272, #29).

### 8. Disease scoping — somatic vs germline, normal/control mislabeled with tumor type — **disease-scoping 14 review-bullets; tumorType/diagnosis-ambiguous 11; somatic/germline in fixes 3%**
- "the diagnosis of NF1 should only be set if the biospecimen is from an actual person with NF1. Somatic loss of NF1 (in cancer) does not qualify" (#177).
- Normal Schwann cell line HSC1λ annotated as MPNST — "HSC1λ is normal Schwann cells, not MPNST" (#220).
- "diagnosis changed to Not Applicable because the parent cell line hTERT ipn02.3 2λ is from a non-NF1 [donor]" (#35).

**Root cause:** Disease annotation is driven by the search context, not by whether the sample carries germline disease or is a normal/control.
**Already supposed to handle?** YES — Standard 13 (germline vs somatic) and Standard 12 (normal/control samples must not get tumorType). These are detailed rules that are still violated. **Strengthen:** require a per-sample germline-vs-somatic determination step in the curation comment and an audit check that normal/control cell lines have tumorType unset.

---

## TIER 3 — Recurring but lower-volume gaps

### 9. Wiki template / disclaimer non-compliance — **17% of fixes (18/103)**
"wiki is not following the correct template" (#223, #251, #220); "wiki is missing the disclaimer that is supposed to be there" (#267); "when project is provisioned wiki should have Auto-curated by" (#216). **Standard exists** (wiki template in synapse_workflow.md, ACK markers in Standard 16). **Strengthen:** audit assertion that wiki contains the required disclaimer block and section headers.

### 10. Source Metadata folder empty / unpopulated — **7 fixes + many review notes**
"source metadata folder should be populated" (#186, #251, #267). The folder is created but left empty. **Strengthen:** Checklist already says "no empty folders" — make it blocking and actually deposit fetched SOFT/filereport/record JSON there.

### 11. Consolidation / dedup / cross-project moves — **consolidate 8% of fixes; cross-project move 5%**
"consolidate these 5 projects into one" (#203–207); "check if this should be consolidated with issue 222/224" (#223); "already part of syn54700333" (#24); "Migrate annotations from NADIA project … to manually-curated project … for shared PRIDE PXD043034" (#296). Figshare-articles-share-resource_doi grouping and cross-run PMID dedup are failing — one paper's supplementary files become 5 separate projects.
**Already supposed to handle?** YES — Deduplication "Step 0" cross-run PMID dedup + Figshare resource_doi grouping rule. Still produces duplicate projects. **Strengthen:** enforce Figshare resource_doi grouping and cross-run state-table PMID check as the curator repeatedly cleans these up by hand.

### 12. resourceStatus / name / filename annotation hygiene on files — **9% of fixes; 3 as primary**
"remove resourceStatus annotations from all files" (#9, #11); "remove the filename annotations from all files. I want the name column" (#10, #251). **Standard exists** (the "NEVER set on File entities" block). Recurs because the audit isn't stripping these. **Strengthen:** audit must hard-delete `resourceStatus`, custom `name`, and `filename` from File entities.

### 13. Stable Dataset version not minted / minted on wrong (pinned) file version — **12% of fixes mention stable version**
"once this is correct please mint stable versions of both datasets" (#11, #216, #251); version mint returned 405/failed (#267). **Standard exists** (Phase 3 minting, `dataset_ids_to_snapshot`). **Strengthen:** ensure mint runs *after* file edits and re-points items to latest first.

### 14. libraryPrep / assay specificity beyond template choice — **13% of fixes**
"for libraryPrep, annotate as polyAselection where appropriate" (#272); "Chromium Single Cell 3' Reagent Kit v3 - let's set this as Unknown rather than making this inference" (#29 — over-inference); scRNA vs bulk RNA-seq mislabeled. Overlaps with #1. **Standard exists** (Standard 6, assay specificity section). **Strengthen** the verify-from-library-metadata gate.

### 15. modelSystemName / cell line / strain names missing or not in tools DB — **9% of fixes; 15 review-bullets**
"modelsystemname should be MS03. I will file a PR to add it" (#186); "pull the fly strain names from the paper" (#287); "modelSystemName can also be annotated (cell line names) but will need to be added to the tools DB" (#267). Partly a genuine NF-tools-DB gap, partly NADIA not extracting strain/line names from paper/GSM. **Standard exists** (Standard 12 model-system block). **Strengthen:** extract model-system names from GEO GSM characteristics / SRA BioSample and flag DB-additions explicitly.

### 16. instrument/platform, license, species, PMID-verification — **lower volume tails**
- instrument/platform unresolved (9 review-bullets); use exact model (Standard 2) — still left generic/typo'd.
- License annotation missing (#59, #242) — Standard 16; not always added.
- species occasionally inferred not read (#168 xenograft, #28) — Standard 4.
- No PMID / wrong PMID: "no PMID found" in 31 issues (11%); curator-supplied corrections (#23, #120, #186, #294). Standard 20 (verify PMID belongs to data paper) and the publication-resolution section exist.

---

## What NADIA most often punts to humans (its own "Items for human review", 213 bullets / 43 issues, multi-label)

| Rank | Category | Bullets | Interpretation |
|---|---|---|---|
| 1 | libraryPrep / assay specificity | 44 | should be resolved from library metadata (Std 6/12) — over-escalated |
| 1 | Schema/template choice uncertain | 44 | should be resolved from modality (Std 12) — over-escalated |
| 3 | Controlled-vocab / enum gap | 33 | mix of real gaps (need schema PR) + NADIA failing to find existing value |
| 4 | studyLeads / PI / authors unresolved | 21 | resolvable from PubMed when PMID exists (Std 3/14) — over-escalated |
| 5 | specimenID / per-sample identifier | 17 | Std 5 — should be derived, not punted |
| 6 | modelSystemName / strain not in DB | 15 | partly genuine DB gap |
| 7 | summary-only / reanalysis / stub | 14 | should be rejected pre-creation, not flagged |
| 7 | disease scoping somatic/germline | 14 | Std 13 — judgment OK to flag, but often determinable |
| 9 | repository access / controlled / embargo | 13 | |
| 10 | tumorType/diagnosis ambiguous | 11 | |

**Key meta-finding:** A large fraction of what NADIA escalates to humans is work it is *supposed* to do autonomously per CLAUDE.md Standard 14 ("resolve everything autonomously resolvable before filing the review issue"). studyLeads-with-a-PMID, schema/assay-from-library-metadata, and per-sample identifiers are escalated despite Standards 3, 5, 6, 11, 12, 14 saying they must be resolved first. **Standard 14's bar for escalation is not being respected** — this is the single highest-leverage rule to strengthen/enforce.

---

# Gaps where a Standard already exists but is evidently NOT working (enforce, don't add)

These are the rules that exist and are still violated at scale — the priority list for tightening NADIA's audit/enforcement rather than writing new prose:

| # | Standard / section | Evidence it's not working |
|---|---|---|
| Std 12 / Schema Binding | Template chosen from title not modality | 32% of fixes; 44 review-bullets |
| Checklist: id\|name\|annotation cols, dynamic columnIds, descriptive name | Dataset display broken | 17% of fixes (#11/27/28/35/216/223/251) |
| Std 13 (file enumeration) | Landing-page stubs / un-extracted RAW.tar logged as created | 17% of fixes; 21 issues |
| "Reject regardless of score" (summary/reanalysis/stub) + Std 13 somatic | Out-of-scope projects created then torn down | ~13 teardowns |
| Std 1/9/18/22 (enum ground truth, case, Unicode, prefix) | Invalid enums, bad prefixes, wrong URIs | 11% of fixes; HSC1λ, icrptbia: |
| Std 3/11/14 (publication metadata) | studyLeads/funding placeholders despite PMID | 27% / 35% of issues |
| Std 5 (per-sample fields; run≠specimen) | Uniform genotype/sex/specimenID | genotype+sex+specimen ≈ 25% of fixes |
| Std 12/13 (normal/control; germline vs somatic) | tumorType on normal cell lines; somatic NF1 = NF1 | #220, #35, #177, #244 |
| Std 14 (resolve before escalating) | Over-escalation of resolvable fields | dominant pattern in 213 review-bullets |
| Dedup Step 0 + Figshare resource_doi grouping | 5 projects for one paper | #203-207, #223, #296 |
| "NEVER set on File entities" block | resourceStatus/name/filename on files | 9% of fixes |
| Phase-3 stable version mint | Not minted / pinned-version | 12% of fixes |
| Std 16 (wiki ACK / disclaimer) | Wiki non-compliant | 17% of fixes |
| Labels (file-enumeration-required, ntap-integration-review) | Referenced in text, never applied as GitHub labels | 0 issues labeled |

**Genuinely new-rule-worthy (no existing Standard fully covers):**
- **Aggregate / multi-sample count matrices** (rawcounts.txt.gz): one file = many specimens. Standard 5 assumes one file = one sample; needs an explicit matrix-handling rule (#272).
- **Computational/model artifacts** (PDB/CIF coordinates, Perl scripts, protein models): no schema fits; curator asks NADIA to *propose* a schema (#245). A "non-experimental data product" pathway is missing.
- **GEO RAW.tar member enumeration** (idat/CEL files inside archives): no procedure to list/expand archive members (#22, #220, #223).
- **"Do not extrapolate `sex`"**: curator twice asked to leave blank rather than infer (#272, #29) — Standard 5 says populate per-sample but should add an explicit "leave blank, do not extrapolate" guard for unverifiable demographic fields.

---

# Top concrete recommendations (highest leverage first)

1. **Convert the Project Completion Checklist into a blocking, machine-checked audit gate** (Phase 3). The recurring failures (id|name column order, missing annotation columns, pinned file versions, resourceStatus-on-files, empty Source Metadata folder, missing wiki disclaimer, un-minted stable version) are all mechanically assertable. They recur because the audit *flags* instead of *fixing/blocking*. → CLAUDE.md "Project Completion Checklist" + Phase 3 `apply_audit_fixes.py`.
2. **Make schema/template selection read library metadata as a hard prerequisite** and record the verified `library_strategy`/`library_source` in scored metadata; block creation if modality unverified. Add explicit multi-sample-matrix handling. → Std 12 + "Metadata Schema Binding".
3. **Enforce Standard 14's escalation bar:** before filing the issue, require that studyLeads/institutions/fundingAgency (when PMID exists), schema choice (from library metadata), and per-sample identifiers (Std 5) are *resolved*, not flagged. Add a pre-file lint that rejects review-bullets matching these resolvable categories. → Std 14.
4. **Turn the pre-creation rejection gate into a hard stop** for landing-page-only, all-document/summary-only, zero-file, reanalysis, and somatic-only candidates — these cause both wasted writes and manual teardown. → "Reject these regardless of relevance score" + Std 13.
5. **Mandatory runtime enum validation with exact return value + prefix allowlist** so invalid enums (`HSC1L` vs `HSC1λ`), wrong URIs, and invented prefixes (`icrptbia:`) are structurally impossible. → Std 1/9/18/22 + `alternateDataRepository` table.
6. **Enforce cross-run PMID dedup (Step 0) and Figshare resource_doi grouping** to stop one-paper-many-projects. → Deduplication section.
7. **Actually apply the GitHub labels** the rules prescribe (`file-enumeration-required`, `ntap-integration-review`) so humans can triage; currently text-only.
8. **Author-name reformatter** (`<ForeName> <LastName>`) applied in the writer, and **PMC Acknowledgements funding parse** forced before any fundingAgency placeholder. → Std 3/11.

---

## Appendix — methodology notes & caveats
- Theme clustering is regex-based and multi-label; a single `/nadia fix:` comment commonly spans 3–5 themes (curators batch corrections), so percentages sum >100%. The single-primary-theme pass (priority-ordered) and the multi-label pass are both reported.
- "Self-flag" percentages are over all 277 issues (body + bot comments); the heaviest signal concentrates in the 66 issues that have a NADIA bot comment thread, so those rates understate prevalence *within curated projects*.
- `/nadia fix:` extraction excludes bot-echoed template text (`<description>` placeholder, polish-report quotes); 103 genuine curator commands across 46 issues remain.
- Deep per-thread reconstruction of the published/approved subset was intentionally left to the companion analysis; this report characterizes the corpus at scale.
- Raw harvested data retained at `/tmp/nadia_issues/` (`_review_bullets.txt`, per-issue JSON) for re-querying.
