---
name: data-ingestion
description: Audit, collect and ingest a medical-imaging dataset (MRI/CT NIfTI, Analyze, MINC, DICOM) for an MRI/CT deep-learning project (written for hippocampus segmentation), and compare it with candidate external datasets. Answers the data-ingestion checklist with real numbers - size and files, label types and how they were drawn, public access and licence, patient IDs, metadata, coordinate space ("map"), voxel size, generalisability, corrupted files or missing slices - runs the automated integrity pass (truncated .nii.gz, all-zero, NaN/Inf, label values, label-image grid), builds the manifest with a standard folder layout and one label format, records source and patient ID so train/test can't share a person, links sequences and longitudinal sessions per patient, and writes "Table 1" comparing the training data with every external dataset. Use it whenever the user asks to check, audit, ingest, add or describe a dataset, asks about data quality, leakage, patient-level splits, file layout or a manifest, or asks which external datasets could test the model, even if they don't say "skill".
---

# Medical image data collection and ingestion

Before a model is trained or tested on a dataset, someone has to know exactly what is in it: how big it
is, what the labels mean, who the people are, whether any file is broken, and whether a patient could
end up on both sides of a split. This skill answers those questions with numbers from the files
themselves, and turns the answers into a manifest that downstream code reads instead of each dataset's
own paths.

## Ground rules (apply every time)

- **Don't download datasets.** Read what is on disk; for anything else give the link so the user can
  check the data and licence first. Pages and papers about a dataset may be read; its files may not.
- **Raw data is read only.** Never rename, move, convert or delete files in `data/`. Stray or duplicate
  files are reported, and the user decides.
- **Long runs:** quick checks may run, with output in the scratchpad. The real `hippo.manifest` run that
  writes into `data/` is given as a command for the user.
- **Keep file searches inside the project folder.** Web research is for dataset pages and papers only.
- **Never upload data**, and never put patient identifiers in a web query.
- **Separate verified from sourced.** Every answer says whether it came from the files (verified) or from
  a page or paper (sourced, with the link). "Unknown" is a valid answer; a guess is not.

## The tool: the manifest builder (`hippo.manifest` in the reference project)

**Reference project.** The commands, paths and numbers below come from the hippocampus segmentation
project this skill was written for (`src/hippo/manifest.py`, test `tests/test_manifest.py`). If the
user's project has no such module, find its equivalent, or write the same checks as one script and ask
before adding it. Treat the project-specific numbers as a worked example, not as facts about the user's data.

**One script for every dataset.** The manifest builder (`src/<pkg>/manifest.py`, run as
`python -m <pkg>.manifest`, test in `tests/test_manifest.py`) covers the file-level phases (1, 3, 4, 5, 6).
Phases 2 and 7 are research the skill does, not code. Never put a dataset name in the script
(`fastmri_manifest.py`, `msd_ingestion.py`): one script serves every dataset through adapters. If a
project has per-dataset scripts, say so and offer to merge them; don't do it unasked.

`src/hippo/manifest.py` does the file-level work for every dataset that has an adapter:

```bash
PYTHONPATH=src .venv/bin/python -m hippo.manifest --quick --out <scratchpad>/manifest      # headers only, seconds
PYTHONPATH=src .venv/bin/python -m hippo.manifest --out <scratchpad>/manifest \
    --splits data/splits.json data/folds/fold0.json --layout <scratchpad>/standard          # full pass (~4.5 min for 10 GB)
PYTHONPATH=src .venv/bin/python -m hippo.manifest --splits data/splits.json --layout     # the user's real run -> data/manifest/, data/standard/
```

Outputs:
- `manifest.csv`: one row per file.
- `subjects.csv`: one row per `subject_id`, with sessions, sequences and labels. Without real IDs
  (MSD) that is one row per file, so group by `leak_group` instead.
- `labels.csv`: every label scheme in one table.
- `checks.json`: problems, stray files, duplicates, header-text scan, split leaks and a per-dataset summary.

`--quick` reads headers only: the duplicate (MD5), NaN/Inf and empty-slice checks stay empty
until the full run.

The existing MSD QC report (`data/explore/summary.json`, from `hippo.explore`) adds intensity groups and
label-shape warnings; read it rather than repeating it.

## Procedure

Work through the phases in order. Each phase lists the checklist questions it answers.

### 1. Inventory and size: "size of the datasets and files included"

- `du -sh` per dataset and per file kind, plus file counts per pattern (e.g. `ls s*/ | sed 's/^s[0-9]*_//' | sort | uniq -c`).
- Report GB per dataset and the big contributors (high-resolution scans are usually most of it).
- List anything that isn't a dataset file, and say what it is:
  - macOS `._*` and `.DS_Store` files;
  - uncompressed copies of a `.gz` file, which are often 10× larger;
  - Finder duplicates named `"<file> 2"` (the reference code only recognises `x.nii 2`; `x.nii 2.gz`
    shows up as "unknown", so check those by eye);
  - files that match no pattern.

  `checks.json → stray_files` sorts them by kind.

### 2. Identity: public? patient ID? metadata? labels and how drawn? generalisable?

- **Public and licence:** dataset page and `dataset.json` / README. Note whether access needs a login,
  an application (e.g. ADNI) or a data-use agreement.
- **Patient ID:** look at the file names and folders. If there is no subject ID (MSD Task04: files numbered
  001–390), say so and say what the split can and can't guarantee.
- **Metadata:** what is in the files (header text, sidecar JSON/CSV, `dataset.json`), and what exists
  only in the paper. Demographics in a paper are group-level, not per scan.
- **Labels:** the value → name table, whether the names are documented or inferred (MNI-HISUB25's
  subfield names are inferred from the atlas), and the **protocol**:
  - who drew them, and how many raters;
  - the landmarks used (e.g. uncal apex for the head/body boundary);
  - what is included or left out (subiculum, alveus/fimbria, tail end, amygdala);
  - reliability figures (Dice, ICC).

  The protocol decides what a Dice against these labels means. A different protocol shows up as
  "model error" on an external test.
- **Generalisability:** sites, scanner vendors, field strength, population and diagnoses, age range, and
  number of people (not number of files). One site and one population = limited, and say so.

### 3. Technical metadata: "map" (coordinate space), voxel size, orientation

From the headers (`hippo.manifest --quick` fills these columns):
- shape, voxel size, dtype;
- orientation codes (RAS/LAS);
- qform/sform codes;
- origin;
- header text.

Name the space:
- a template (MNI152, with its grid);
- native (each person's own head);
- crops (MSD: origins reset to 0/1, so in no template space).

Flag:
- **anisotropic voxels**, e.g. 0.78 × 0.78 × 2 mm;
- **voxel sizes the model can't take.** This model was trained on 1 mm voxels only. No pipeline step
  resamples, and the API accepts only 0.8–1.25 mm (`api.VOXEL_MM`). Anything else needs resampling to
  1 mm first: linear for images, nearest-neighbour for labels.

### 4. Integrity pass: "corrupted files or missing slices", "truncated, all-zero, NaN/Inf"

Run the full `hippo.manifest` pass (not `--quick`). Every file is loaded completely, so a truncated `.gz` fails here. It
checks:
- unreadable files;
- NaN/Inf;
- all-zero volumes;
- **empty slices inside the volume**: all-zero slices with data on both sides, a hint of missing
  slices;
- label values outside the scheme;
- labels without foreground;
- labels whose stored type is float;
- labels not on their image's grid;
- exact duplicates (MD5).

Read the result before reporting it. These are known false alarms or expected findings:
- **int16 labels with a scale factor load as float.** The check uses the *stored* type; don't
  "fix" this back.
- **All-zero slices at the edges** of MNI-space or skull-stripped scans are background, not missing
  data; only interior gaps count.
- **Partial-coverage slabs** (MNI-HISUB25 native T2: 38–60 coronal slices of 2 mm) are how they were
  acquired. Report them as "partial coverage, by design", and check hippocampus coverage separately if
  that sequence will be used.

### 5. Standard layout and one format: folder hierarchy, labels and metadata in one format

- **Never restructure raw data.** The manifest is the index, and `--layout` adds a BIDS-style symlink
  tree: `data/standard/<dataset>/sub-<id>/ses-<n>/anat/sub-<id>_ses-<n>[_hemi-L]_space-<MNI|native|crop>[_res-1mm]_<T1w|T2w|dseg>.nii.gz`.
  The resolution tag comes from the header, never from a file name.
- **Downstream code filters manifest rows** (`dataset`, `role`, `modality`, `space`, `side`) instead
  of building paths.
- **Adding a dataset = one adapter** in `manifest.py`, registered in `ADAPTERS`, never a new
  `<dataset>_*.py` script. It returns rows built
  with `row(dataset, path, subject, ...)`. It also needs:
  - an entry in `DATASETS`: source, licence, defacing, metadata;
  - its label scheme in `LABEL_SCHEMES` as `value: (name, part, documented)`, where `part` says how the
    value maps onto the whole hippocampus;
  - a test in `tests/test_manifest.py` with a tiny fake copy of its folder layout.
- **Non-NIfTI formats** (Analyze `.hdr/.img`, MINC, DICOM) are converted into a *separate* folder, never
  over the originals. Check left/right after converting Analyze: Analyze 7.5 doesn't store orientation
  reliably. Compare one scan against a viewer, e.g. with an asymmetric landmark.

### 6. Leakage and linking: source and patient ID, longitudinal scans and sequences

- **Every row carries:**
  - `subject_id = "<dataset>:<id>"`;
  - `source_subject_id`;
  - `subject_id_known`;
  - `session`;
  - `leak_group`: the unit a split must keep together.
- **All sequences and sessions of one person share the subject and leak group.** `subjects.csv` shows
  them together, and splits must be made by `leak_group`, never by file.
- **Without IDs, write the assumption down.** MSD's leak group is the possible left/right pair
  `(n + 1) // 2`, an assumption. `--splits` reports leak groups found in more than one split. Report the
  result and don't change the split unless the user asks. (Worked example: in the reference project,
  `data/splits.json` has 22 test+train, 4 test+val and 15 train+val shared groups, a documented limitation.)
- **Cross-dataset identity:** check whether two datasets come from the same cohort before calling one
  "external". Known cases: **OVAL and Learn2Reg "Hippocampus MR" are the MSD Task04 cohort** (same
  Vanderbilt people). Same hospital (the two Henry Ford NITRC sets) → check for patient overlap.
- **Identifiers:** `checks.json → identifier_scan` checks header text and extensions for dates, long
  numbers and name-like keys. Also note:
  - defacing (file name, paper, or a look at a render);
  - skull stripping;
  - whether crops contain a face at all.

  If anything identifying is found, propose an anonymised copy (blank header text, strip extensions)
  in a separate folder. Never edit the originals.

### 7. External datasets and "Table 1": similar datasets, and how similar they are

- **Search:** dataset pages (NITRC, Grand Challenge, consortium sites) and the dataset paper (PubMed
  Central). Read only the pages, never the data. Log-in walls: say what couldn't be read.
- **Start from [references/external_datasets.md](references/external_datasets.md).** It holds the
  last Table 1 for hippocampus MRI and the date it was checked. Don't edit it in place (with a plugin
  install it is replaced on update): write the updated table, with today's date, into the user's project,
  e.g. `docs/external_datasets.md`, and reuse that copy next time.
- **Table 1 has two parts, always these columns.**
  - Include rows for the **training dataset** and the **current external dataset** as references.
  - **A. Data and imaging:**
    - dataset;
    - labelled scans;
    - people (diagnoses, age);
    - scanner and field strength;
    - voxel size;
    - space and format;
    - size.
  - **B. Labels, access, usefulness:**
    - dataset;
    - labels;
    - protocol and raters;
    - patient ID / metadata;
    - access and licence;
    - "useful as an external test?" (with the preparation it would need).
  - Bold the facts that change usability: anisotropic voxels, unusual formats, small labelled counts,
    same cohort as training.
- **After the table, write three things:**
  1. A summary: whether any independent dataset has the training labels (worked example: none has
     MSD's anterior/posterior split, so external tests are whole hippocampus only), and the expected protocol
     gap.
  2. A ranked list of the usable ones, with the reason for each rank.
  3. A "couldn't confirm" list, and links.
- **A single-dataset request** ("point 1 for dataset X") uses the same columns as one table with three
  columns: X, training, current external. Add "what this means for using it", covering resolution,
  format, space, label protocol and the number of labelled scans.

## Report format

Answer the checklist in the user's order, numbered, one or two lines each, with numbers. Then:
- **Problems found:** each with file, severity and what the user might do. Don't do it for them.
- **Commands for the user:** the real `hippo.manifest` run, and any conversion.
- **Verified vs sourced:** what came from the files, and what came from pages or papers (with links).

If the user wants it recorded:
- if the project has a CLAUDE.md, add a short note (manifest script row or Known issues);
- add a line to the README data table, e.g. "evaluated on T1 only, at 1 mm".

Keep the README and `docs/` numbers the same.

## Checklist coverage (for a quick self-check)

| Question | Phase |
|---|---|
| Size and files | 1 |
| Labels present, how identified/drawn | 2 |
| Public? Patient ID? Metadata? | 2 |
| Generalisability | 2 |
| Space ("map") and voxel resolution | 3 |
| Similar external datasets and how similar | 7 |
| Corrupted files, missing slices | 4 |
| Folder hierarchy, one label/metadata format | 5 |
| Technical metadata | 3 (manifest columns) |
| Automated pass for truncated / all-zero / NaN-Inf | 4 |
| Source and patient ID in the manifest, no cross-split leak | 6 |
| Link longitudinal scans and sequences per patient | 6 |
| Table 1 for all external datasets | 7 |
