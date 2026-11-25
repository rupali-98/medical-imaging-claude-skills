---
name: data-exploration
description: Explore medical-imaging datasets (MRI NIfTI, training and external sets together) before modelling and answer the data-exploration checklist with numbers and plots - metadata completeness and missing-data statistics, voxel spacing and image-shape distributions (isotropic research vs anisotropic clinical scans), coordinate orientation, affine checks of every image against its mask (also on external datasets), qform/sform disagreement and oblique acquisitions, raw intensity ranges per scanner/dataset with a normalisation proposal and plot, class balance, where segmentation labels sit in the volume, subgroup distributions by sequence, scanner vendor/model, field strength and site, and confounding (a label or diagnosis that comes from one vendor or field strength). Use it whenever the user asks to explore, describe, profile or compare datasets, asks about spacing, orientation, affines, obliquity, intensity ranges, normalisation, label balance or placement, scanner or site variation, subgroup or domain-shift analysis, even if they don't say "skill". For collecting/ingesting a dataset, building the manifest or Table 1 use data-ingestion first.
---

# Medical image data exploration (MRI)

Exploration answers "how do these datasets differ, and what will the model trip over?" It runs on
the manifest from the **data-ingestion** skill, so do ingestion first: integrity, patient IDs and leak
checks belong there, not here.

## Ground rules

- **Read only.** Never change raw data files. No downloads: give the link.
- **The user runs the real pass.** A full exploration run loads every image and label and can take
  minutes: print the command. Quick checks on a few files or tests (`python -m pytest -q`) are fine. If
  a run is stopped by the user, don't restart it; give the command instead.
- **Verified vs sourced.** Header numbers are verified; scanner, site, field strength and diagnosis usually
  come from papers (dataset-level) and must be labelled so. "Unknown" beats a guess.
- Plain words, with numbers and file/case names. Keep the "research use only" line in user-facing text.

## The tool: one exploration script

Look in the user's project for an existing exploration or cohort script and use it. If there is none,
offer to write one (ask first), e.g. `src/<pkg>/explore.py` run as `python -m <pkg>.explore`, that
reads the manifest and writes:

- `files.csv`: one row per image and label. It has the geometry, metadata (with `metadata_source`:
  file, sidecar or paper), obliquity, qform/sform difference, and for labels the voxels per value,
  centroid, margins and affine difference to the image.
- `completeness.csv`: missing values per dataset and field.
- `subgroups.csv`: counts per dataset, field and value.
- `summary.json`: geometry, intensity groups and the normalisation proposal, label balance and
  placement, confounding flags, thresholds.
- `figures/geometry.png`, `intensity.png`, `label_placement.png`.

Useful options: `--exclude <pattern>` to skip very large files, and `--metadata <csv>` for per-file
site, vendor, model, field strength and diagnosis when the user has them.

The manifest must come from a full run (not `--quick`); otherwise there are no intensity percentiles.
Reuse what exists rather than recomputing it: earlier QC reports, and the manifest's `checks.json` for
integrity results. Dataset-level metadata (scanner, site, field strength) comes from the project's
Table 1 (see data-ingestion); keep the two in step.

## Procedure: answer the checklist in this order

Before starting, find out from the project's config or code what the model expects: training voxel
spacing, whether the pipeline resamples or reorients on load, and which normalisation it uses.

1. **Metadata completeness / missing data.** Read `completeness.csv`. For each dataset, list:
   - the fields that are 100 % missing;
   - partial gaps, with counts;
   - which fields are only dataset-level (from a paper, the same for every file) versus per file.

   Say plainly when a dataset has no per-scan metadata at all (e.g. MSD Task04: no ID, side, age, sex,
   diagnosis or scanner). qform/sform code 0 counts as missing orientation.
2. **Voxel spacing (in and out of distribution).** `summary.geometry.<ds>.spacing_mm` gives counts per
   spacing. Compare with the training spacing. If the pipeline doesn't resample, anything else needs
   resampling first: linear for images, nearest-neighbour for labels.
3. **Orientation.** Report the orientation codes and spaces per dataset (e.g. crops in RAS, template
   data in RAS, native data in LAS). Say whether the pipeline reorients on load, and note any format that
   doesn't store orientation reliably (Analyze 7.5).
4. **Variation and subgroups.** Use `subgroups.csv` plus the metadata. Report counts by:
   - sequence;
   - vendor and model;
   - field strength;
   - site;
   - side;
   - diagnosis.

   If a field is unknown per scan, say the subgroup analysis isn't possible from the files, and name
   where the data could come from (the dataset paper, consortium tables such as ADNI's, a `--metadata`
   file). The scanner/site variation then shows only as intensity and geometry differences between
   datasets.
5. **Class balance.**
   - **Classification labels:** class counts per split and per dataset; imbalance ratio.
   - **Segmentation:** the shares in `labels.<ds>.foreground_share_by_value`, the foreground fraction
     per volume (often < 5 %: background-dominated, which is why Dice-type losses are used), and
     L/R counts.
   - Note the difference between files and people.
6. **Intensity range and normalisation.** Use `summary.intensity` and `figures/intensity.png`:
   - p99 range and scale groups (orders of magnitude);
   - p99 by sequence;
   - negative values.

   Proposal: per-scan z-score over non-zero voxels. It is required when scale groups differ (e.g. MSD
   Task04 has three, at about 1e2, 1e3 and 1e5, so no fixed scale fits). Mention alternatives with when
   to use them:
   - percentile clipping (0.5–99.5) before z-score, for bright outliers;
   - N4 bias correction, for inhomogeneity across a whole head;
   - histogram matching, only if a z-score isn't enough.
7. **Label placement.** Use `labels.<ds>`:
   - the relative centroid mean and SD;
   - margins to the volume edge in mm, low and high per axis, plus the minimum margin (0 = the label
     touches the edge, i.e. it is cut off);
   - the world-centroid SD, which is only meaningful in a shared space (e.g. MNI);
   - `figures/label_placement.png`.

   Explain what a placement difference means for the model: a model trained on tight crops centred on
   the structure can lose accuracy when an external set is cropped differently or not at all.
8. **Shapes and spacing plot.** `figures/geometry.png`:
   - left: finest vs coarsest voxel size. Points on the dashed line are isotropic research scans;
     points above it are anisotropic clinical scans (e.g. 0.78 × 0.78 × 2 mm).
   - right: image size vs aspect ratio. Crops sit far below whole heads (e.g. about 0.05 M voxels
     against 5–9 M).

   Report the anisotropic files (`geometry.<ds>.anisotropic`; threshold max/min > 1.5).
9. **Affines of image vs mask, also external.** Use `geometry.<ds>.label_affine_mismatch` (tolerance
   1e-3 mm, plus the shape) and `labels_checked_against_image`. Say how many label/image pairs were
   checked per dataset, not just "none failed".

   Across datasets, the affines aren't meant to match (crops vs template). The check is that each
   dataset's masks match its own images, and that the project saves predictions with the image's
   affine. Check the prediction code for that.
10. **Oblique acquisitions.**
    - `geometry.<ds>.oblique` / `oblique_deg_max`: angle of the voxel axes from the world axes;
      above 1° counts as oblique.
    - `qform_sform_disagree`: two orientation matrices that differ. Different tools pick different
      ones, so masks can shift.

    Template-registered and cropped data should be ≈ 0°. Native clinical scans (e.g. slices angled to an
    anatomical axis) are often oblique. RAS reorientation does not remove obliquity; resampling to an
    axis-aligned grid does.
11. **Labels or diagnoses tied to one vendor or field strength.** Read `summary.confounding`: each entry
    says that a value of `diagnosis` (or a whole `dataset`) comes from only one vendor, field strength or
    site.
    - **Dataset ⇔ vendor/site** is expected with one-site datasets. It means an internal-vs-external
      gap can't be separated into scanner effects and labelling-protocol effects. Say so.
    - **Diagnosis ⇔ scanner** is the dangerous one: the model can learn the scanner instead of the
      disease. Recommend stratified splits or a held-out scanner.

## Report format

- Answer the 11 questions in the user's order, numbered, with numbers and file names. Mark each answer
  **verified** (headers or voxels) or **sourced** (paper, with the link).
- Show the three figures (paths) and say what to look at in each.
- **Problems found:** file, severity, suggested action (the user decides).
- **Commands for the user**, including the full exploration run if it hasn't been run.
- If the user wants it recorded, add a short note to the project's CLAUDE.md (if any) or `docs/`, and
  keep the README and docs numbers the same.
