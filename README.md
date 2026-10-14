# ENG 2440 Assignment 2 — Whole-Tumor Segmentation in Multimodal MRI

Author: Yintong Wang (25011183)
Report: `YintongWang_25011183_ENG2440_A2.pdf` (submitted on Moodle; LaTeX source in `report/`)

A patch-based 3D U-Net that segments the whole tumor in four-channel brain MRI (MSD Task01 BrainTumour), comparing uniform and tumor-aware patch sampling, with sliding-window inference and evaluation on the untouched test split. A learning exercise, not a clinical tool.

## 1. Data setup

The dataset is **not** included in this repository.

1. Download `Task01_BrainTumour.tar` (~7.6 GB) from
   https://msd-for-monai.s3-us-west-2.amazonaws.com/Task01_BrainTumour.tar
2. Extract it and place `assignment2_split.csv` (from Moodle) beside the notebook.
3. Open `Assignment2_Student_25011183.ipynb` and change **only** the cell marked `DATA_ROOT`.

```
ENG2440_Assignment2/
|-- Task01_BrainTumour/{imagesTr, labelsTr, dataset.json}
|-- assignment2_split.csv
|-- Assignment2_Student_25011183.ipynb
```

Disk space: the notebook writes a preprocessed cache of the train and validation cases to `cache/` (about 13 GB, generated automatically on the first run, git-ignored). Test cases are not cached.

## 2. Environment

```bash
conda env create -f environment.yml
conda activate eng2440-a2
```

Versions used for the submitted run (installed in the `eng2440-a2` environment):

| Package | Version |
|---|---|
| Python | 3.11.16 |
| torch | 2.14.1 |
| nibabel | 5.4.2 |
| numpy | 2.4.6 |
| scipy | 1.17.1 |
| pandas | 3.0.6 |
| matplotlib | 3.11.2 |
| tqdm | 4.70.1 |
| jupyter / nbconvert | 1.1.1 / 7.17.1 |

`environment.yml` also lists MONAI (1.6.1 is installed), but **no MONAI code is used** in this project.

## 3. Hardware and runtime

| Item | Value |
|---|---|
| Machine | MacBook with Apple M4 Max, 36 GB unified memory |
| Device | `mps` (selected automatically: CUDA, then MPS, then CPU) |
| Full notebook run, top to bottom | about 22 minutes (21 min 36 s), run with `jupyter nbconvert --execute` |
| Individual stages | not timed separately; a short test showed about 0.35 s per training step (batch 2, 64³ patches), and sliding-window inference took a few seconds per test volume |

## 4. Reproducibility

- Random seed: `0` (`SEED` in the configuration cell). `seed_everything(SEED)` seeds Python, NumPy and PyTorch.
- Both samplers start from identical initial weights: the seed is reset inside `train()` immediately before the model is built (checked in cell E1).
- Both samplers are scored on the same 73 fixed validation patches, drawn once with `np.random.seed(SEED)`.
- Determinism note: GPU computation on MPS is not fully repeatable, so re-running training with the same seed gives slightly different numbers. For example, an earlier run of the same code gave a final validation Dice of 0.774 for the uniform sampler, against 0.702 in the submitted run. All numbers in the report come from the submitted (executed) notebook.
- Test-set inference is repeatable for a given set of weights: recomputing the test metrics from the saved weights reproduced the reported table exactly.

## 5. Exact run order

Run the notebook once from top to bottom (all cells in order):

1. Configuration (`DATA_ROOT`, seed) and imports / device selection.
2. Setup check (split file, 484 cases, files exist).
3. Section A: task definition, geometry of three cases, modalities and three-plane views.
4. Section B: case-level split checks and tumor-volume distribution per split.
5. Section C: geometry check, interpolation demonstration, normalization, crop/pad and augmentation.
6. Cache cell (first cell of Section D): writes the cropped, normalized train/validation cases to `cache/`.
7. D1 dataset and loader, D2 model, D3 training with uniform patches (saves `outputs/uniform.pt`), D4 training with tumor-aware patches (saves `outputs/tumor_aware.pt`), D5 curves.
8. Section E: controlled comparison; E3 selects `final_model` using validation results only.
9. Section F: F1 sliding-window helpers, F2 test-set inference and metrics (run once), F3 volume errors, F4 connected-component postprocessing.
10. Section G: three-plane error analysis and limitations.

Command-line alternative (macOS; `caffeinate -i` prevents sleep during the run):

```bash
caffeinate -i jupyter nbconvert --to notebook --execute \
  --ExecutePreprocessor.timeout=-1 --output executed.ipynb Assignment2_Student_25011183.ipynb
```

Note on the submitted notebook: the outputs of cells G1 to G3 were produced in a separate process that reloaded `outputs/uniform.pt` (the model selected in E3), repeated the test inference of F1 to F3, and then ran the G1 to G3 cells. The recomputed F2 table matched the full run exactly.

## 6. Repository contents

| Path | Purpose |
|---|---|
| `Assignment2_Student_25011183.ipynb` | Fully executed notebook (all code is in the notebook) |
| `report/` | LaTeX source of the report (`report.tex`), its figures, and a built `report.pdf` |
| `src/` | Package placeholder (`__init__.py` only) |
| `outputs/` | Created by the notebook (checkpoints are git-ignored) |
| `environment.yml` | Environment |
| `ai_use_declaration.txt` | AI-use declaration |
| `ai_generated_code_log.txt` | Log of AI-generated or AI-supplied code |

No MRI volumes, masks, extracted patches or caches are stored in the repository.

## 7. Third-party software and attribution

- PyTorch 2.14.1, nibabel 5.4.2, NumPy 2.4.6, SciPy 1.17.1, pandas 3.0.6, matplotlib 3.11.2.
- MONAI is installed in the environment but is not used.
- Dataset: Medical Segmentation Decathlon Task01 BrainTumour (Antonelli et al., *Nature Communications* 13:4128, 2022; Medical Segmentation Decathlon Organizers, 2018), derived from BraTS (Menze et al., *IEEE TMI* 34(10), 2015). License: CC-BY-SA 4.0.
