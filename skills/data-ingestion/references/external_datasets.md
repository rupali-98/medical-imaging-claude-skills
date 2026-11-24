# Table 1: training data and candidate external datasets

Last checked: **2026-11-17**.

Sources:
- **Verified from the files:** MSD Task04 and MNI-HISUB25, which are on disk (hippo.ingestion run).
- **From pages and papers:** all other rows. None of those datasets has been downloaded.
- **"Unknown"** means the sources don't say.

Update this file whenever a row is added or a fact is confirmed, and change the date.

## A. Data and imaging

| Dataset | Labelled scans | People | Scanner and field strength | Voxel size | Space and format | Size |
|---|---|---|---|---|---|---|
| **MSD Task04** (training) | 260 crops (+130 unlabelled) | Healthy adults and adults with psychotic disorders, Vanderbilt | Philips Achieva | 1 × 1 × 1 mm | Crops around the hippocampus (30–43 × 40–59 × 24–47 voxels), origins reset to 0/1, RAS; NIfTI | 28 MB |
| **MNI-HISUB25** (current external) | 25 people, 50 hippocampi | 25 healthy adults | 3 T, one site (MNI) | 1 mm used; 0.4 mm (MNI) and 0.6 mm (native T1) available; native T2 0.4 × 2 × 0.4 mm slab | MNI152 (RAS) and native (LAS); NIfTI; defaced | 9.95 GB |
| **OVAL** (Vanderbilt) | Paper: 195 people (+35 in a second set) | 90 healthy, 105 with psychotic disorders | Philips Achieva | 1 mm (170 sagittal slices) | Unknown (download needs a NITRC login) | Unknown |
| **Henry Ford `hippseg_2011`** | 25 labelled (+25 scored only by the organisers) | 40 temporal lobe epilepsy (age 15–64, mean 39; 13 M / 27 F), 10 without epilepsy (19–54, mean 34; 5 M / 5 F) | GE Signa, 1.5 T and 3 T (split unknown), coronal SPGR | **0.78 × 0.78 × 2 mm** (1.5 T), **0.39 × 0.39 × 2 mm** (3 T) | Native; **Analyze 7.5**; skull-stripped, face removed | 0.6 GB (`HFH.rar`) |
| **HFH temporal lobe epilepsy, multi-modality** | Registry entry: 163 patients (not confirmed) | Temporal lobe epilepsy | Henry Ford; T1, plus FLAIR and SPECT in some studies | Unknown | Skull-stripped T1; one zip per patient (1–8.5 MB) | Unknown |
| **EADC-ADNI HarP** | **135** (68 at 1.5 T, 67 at 3 T); benchmark set 20 scans | ADNI: controls, mild cognitive impairment, Alzheimer's; about 55–90 years | Philips, GE, Siemens; 3D T1 | About 1 mm (check per scan) | AC-PC aligned, native; MINC | Unknown |
| **Learn2Reg "Hippocampus MR"** | 263 train / 131 test | **Same Vanderbilt cohort as MSD** | As MSD | 1 mm | Same crops as MSD | As MSD |

## B. Labels, access, usefulness

| Dataset | Labels | Protocol and raters | Patient ID / metadata | Access and licence | Useful as an external test? |
|---|---|---|---|---|---|
| **MSD Task04** | 1 anterior, 2 posterior (documented) | Head ends at the uncal apex (Woolard & Heckers 2012); leaves out an inner strip (subiculum, medial tail) | **No IDs**; no side, age, sex, diagnosis or scanner | CC BY-SA 4.0, open | (training data) |
| **MNI-HISUB25** | 1 ≈ CA1–3, 2 ≈ subiculum, 3 ≈ CA4–DG (names inferred, not documented); scored as whole hippocampus | Expert manual subfield protocol (Kulaga-Yoskovitz 2015); includes the subiculum | `s01`–`s25`; demographics in the paper | CC BY-NC-SA 3.0, NITRC | In use: whole hippocampus only, **T1 at 1 mm in MNI** |
| **OVAL** | **Head / body / tail** (+ amygdala) | Uncal apex for head/body; body/tail cut by a plane behind the thalamus (Plassard 2021) | Probably subject IDs (unknown) | NITRC login; Creative Commons (type unknown) | **No: same people as MSD** (leak). Might give subject IDs for MSD files, or whole-head scans |
| **Henry Ford `hippseg_2011`** | Whole hippocampus, left and right (values not stated) | Geuze 2005; includes the subiculum, excludes alveus, fimbria, amygdala, temporal horn; one rater checked by two (repeat Dice 0.90) | Subject numbers; gender document on NITRC | NITRC registration; custom licence | **Yes:** whole hippocampus; strong shift (epilepsy atrophy, 2 mm slices, GE). Needs Analyze → NIfTI with a left/right check, resampling to 1 mm, cropping without the MNI atlas |
| **HFH multi-modality** | Hippocampus segmentations (manual or automatic: unknown) | Unknown | Patient codes (e.g. `EP0249`) | NITRC, per-study pages | Maybe: read the study pages; check overlap with `hippseg_2011` (same hospital) |
| **EADC-ADNI HarP** | Whole hippocampus, left and right | Harmonised protocol, 5 qualified tracers, inter-rater ICC 0.95–0.98; units include alveus/fimbria, tail end, subiculum | ADNI IDs, full clinical data | Labels: hippocampal-protocol.net; **images need an ADNI application** | **Yes, strongest:** multi-site, 3 vendors, 1.5 and 3 T, ageing and Alzheimer's; whole hippocampus only |
| **Learn2Reg** | As MSD | As MSD | As MSD | Grand Challenge | **No: it is MSD Task04 again** |

## Summary

- No independent dataset has MSD's anterior/posterior labels. The only head/body one, OVAL, is the MSD
  cohort. External tests are whole hippocampus only.
- Every independent option includes the subiculum, so expect the same label-protocol gap as on
  MNI-HISUB25.
- Ranked:
  1. **EADC-ADNI HarP:** biggest, multi-site, clinical population, standard protocol; needs an ADNI
     application.
  2. **Henry Ford `hippseg_2011`:** easy access; hard shift; only 25 labelled scans.
  3. **HFH multi-modality:** check first.
- Not external: OVAL, Learn2Reg.

## Couldn't confirm

- HFH multi-modality: size and labels.
- OVAL: what the download contains.
- HarP: age range and diagnosis split.
- Henry Ford: label values and the 1.5 T / 3 T split.
- Licence terms behind each login.

## Links

- MSD: http://medicaldecathlon.com/
- MNI-HISUB25: https://www.nitrc.org/projects/mni-hisub25
- OVAL: https://www.nitrc.org/projects/oval/ (paper: https://pmc.ncbi.nlm.nih.gov/articles/PMC8715642/)
- Henry Ford: https://www.nitrc.org/projects/hippseg_2011/ (paper: https://pmc.ncbi.nlm.nih.gov/articles/PMC4501402/)
- HFH multi-modality: https://www.nitrc.org/projects/hfh_t1_hp_seg1/
- HarP: https://www.hippocampal-protocol.net/ and https://adni.loni.usc.edu
  - labels paper: https://doi.org/10.1016/j.jalz.2014.12.002
  - benchmark paper: https://adni.loni.usc.edu/adni-publications/Bocchetta_2015_AlzDem.pdf
- Learn2Reg: https://learn2reg.grand-challenge.org/Datasets/
