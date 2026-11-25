# Medical imaging data-prep skills for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Two [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) to run before you train a model on medical images:

- **data-ingestion**: audits a dataset and builds a manifest. Covers size, labels, licence, patient IDs, voxel size and orientation, integrity (truncated, all-zero, NaN/Inf, label grid), patient-level leak checks, and a "Table 1" of external test sets.
- **data-exploration**: answers the pre-modelling checklist with numbers and plots. Covers missing metadata, spacing and shapes, orientation and affines, obliquity, intensity ranges with a normalisation proposal, label balance and placement, scanner/site subgroups and confounding.

Covers MRI today: NIfTI volumes and fastMRI-style raw k-space (HDF5). Analyze, MINC and DICOM rules are written but untested. CT is untested.

## Install

### As a plugin (recommended)

In Claude Code:

```
/plugin marketplace add rupali-98/medical-imaging-claude-skills
/plugin install medical-imaging-data-prep@medical-imaging-skills
```

### Manually

```bash
git clone https://github.com/rupali-98/medical-imaging-claude-skills.git
cp -r medical-imaging-claude-skills/skills/* ~/.claude/skills/
```

Copy into a project's `.claude/skills/` instead to scope the skills to one repo.

## Use

Ask in plain words, for example "audit this dataset", "check for patient leakage between splits" or "explore spacing and intensity for these MRI sets". The skills load on their own.

## Repository layout

```
.claude-plugin/
  plugin.json           plugin manifest
  marketplace.json      lets `/plugin marketplace add` find the plugin
skills/
  data-ingestion/
    SKILL.md
    references/external_datasets.md
  data-exploration/
    SKILL.md
```

## Note

The skills don't depend on any particular codebase. They describe what the manifest and exploration scripts must check and output; Claude uses the scripts your project already has, or offers to write them. Dataset facts (MSD Task04, MNI-HISUB25, OVAL and others) appear only as examples, and `references/external_datasets.md` is a worked hippocampus MRI Table 1.

The skills never download data and never modify raw files.

## Disclaimer

Research use only. Not a medical device and not for clinical decision-making. Check each dataset's licence and data-use agreement yourself before using it.

## Contributing

Issues and pull requests are welcome, especially tested rules for DICOM, CT, MINC or Analyze data.

## License

[MIT](LICENSE)
