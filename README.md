<div align="center">

# AIrPPG

**Lightweight, multi-region remote photoplethysmography for heart-rate estimation from facial RGB video**

[![Python 3.11](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)](pyproject.toml)
[![OpenCV](https://img.shields.io/badge/OpenCV-Video%20Processing-27354A?logo=opencv&logoColor=white)](src/airppg/)
[![MediaPipe](https://img.shields.io/badge/MediaPipe-Face%20Mesh-27354A)](src/airppg/preprocessing/roi.py)
[![Research](https://img.shields.io/badge/Status-Experimental-167D86)](#project-status)
[![License](https://img.shields.io/badge/License-MIT-27354A)](LICENSE)

**[Results](#experimental-results) · [Reports](#reports-and-visualizations) · [Getting started](#getting-started) · [Methodology](#methodology) · [Limitations](#limitations)**

</div>

---

## Overview

**AIrPPG** investigates whether combining multiple facial regions improves camera-based heart-rate estimation without contact sensors. The system processes RGB face videos, extracts pulse-related color changes from the **forehead, left cheek, and right cheek**, and compares classical remote photoplethysmography (rPPG) methods under a consistent evaluation protocol.

The current implementation is an **experimental, notebook-driven research pipeline**, not a live webcam application or a trained deep-learning system. It includes:

- Face Mesh landmarks, per-frame ROI polygons/masks, and explicit invalid-frame handling.
- Dataset-aware ground-truth parsing and portable, versioned artifacts.
- **Green, CHROM, and POS** signal extraction and frequency-domain HR estimation.
- **Single-region**, **simple-average**, and **quality-weighted** multi-ROI comparisons.
- Per-recording evaluation, aggregate metrics, figures, and detailed experiment reports.

The research uses **UBFC-rPPG**, including both `DATASET_1` and `DATASET_2`. Original videos and ground-truth files must be obtained separately from the dataset provider.

## Experimental results

**Snapshot of the committed fusion experiments:** 49 processed recordings (**39 train / 5 validation / 5 test**), 0 failed across the preprocessing, signal-extraction, and fusion manifests. The test split contains **206 HR windows** across five recordings.

The table uses `outputs/fusion/fusion_aggregate_summary.csv`. MAE, RMSE, and Pearson `r` are **arithmetic means of the per-recording metrics**, not metrics recomputed after pooling every window across recordings.

| Method | ROI strategy | Test MAE ↓ (BPM) | Test RMSE ↓ (BPM) | Pearson r ↑ |
|:---|:---|---:|---:|---:|
| Green | Best single ROI (left cheek) | 13.82 | 17.82 | 0.348 |
| Green | **Average fusion** | **10.24** | **12.09** | **0.519** |
| Green | Quality-weighted fusion | 11.69 | 15.59 | 0.407 |
| POS | Best single ROI (forehead) | 1.80 | 2.33 | 0.423 |
| POS | Average fusion | 1.66 | 2.12 | **0.467** |
| POS | **Quality-weighted fusion** | **1.64** | **2.10** | 0.466 |
| CHROM | Average fusion | 48.66 | 48.81 | -0.284 |
| CHROM | Quality-weighted fusion | 48.32 | 48.48 | -0.319 |

<div align="center">
  <img src="report/assets/task3_test_mae.png" alt="Test-set MAE: single ROI versus fusion strategies for Green, CHROM and POS" width="90%" />
  <sub>Test-set MAE comparison from the <a href="report/task3_fusion_report.html">fusion report</a>.</sub>
</div>

**What the results support**

1. **Average fusion helps Green and POS** compared with their best single-ROI alternatives on this test split. For Green, MAE decreases from 13.82 to 10.24 BPM.
2. **POS is the strongest classical baseline in the current experiment.** Quality weighting changes test MAE only from 1.66 to 1.64 BPM relative to averaging. This very small improvement is **not evidence of a reliable general advantage**.
3. **Hand-crafted quality weighting is not consistently beneficial.** It degrades Green versus simple averaging, and CHROM remains inaccurate under the current pipeline.
4. The `signal_extraction_baseline` reference **already averages ROI signals**. It is a reproduction/reference check, **not an independent single-ROI baseline**.

These are **results for this repository's UBFC split and protocol**, not performance guarantees for arbitrary webcams, motion, lighting, skin tones, or clinical use. Validation errors are substantially higher in parts of the experiment; see the full reports before drawing generalization claims.

**Data:** [Fusion aggregate CSV](outputs/fusion/fusion_aggregate_summary.csv) · [Per-recording fusion metrics](outputs/fusion/fusion_metrics_summary.csv) · [Classical baseline metrics](outputs/signal_extraction/signal_extraction_metrics_summary.csv)

## Methodology

```text
UBFC-rPPG RGB videos + reference heart rate
                 |
                 v
Facial preprocessing (01_preprocessing.ipynb)
   MediaPipe Face Mesh -> forehead / cheeks -> ROI masks
   Frame validity + timestamps + normalized ground truth
                 |
                 v
Pulse signal extraction (02_signal_extraction.ipynb)
   ROI mean RGB -> detrend / band-pass -> Green / CHROM / POS
   Sliding-window FFT -> predicted HR -> aligned ground truth
                 |
                 v
Multi-region fusion (03_fusion.ipynb)
   Single ROI | Average fusion | Quality-weighted fusion
   Quality features, ROI weights, fused HR, evaluation
                 |
                 v
   CSV/JSON/NPZ artifacts + visualizations + HTML reports
```

### Facial ROI extraction

The first notebook reads video metadata and uses **MediaPipe Face Mesh** to locate facial landmarks. Three predefined polygons are converted into pixel masks. Processed frames preserve their original frame indices; invalid frames are recorded with validity flags and failure reasons instead of being silently dropped. Ground truth is normalized separately for UBFC `DATASET_1` and `DATASET_2`.

**Output:** `outputs/preprocessing/preprocessing_dataset_manifest.json`, per-sample `roi_data.npz`, `metadata.json`, `ground_truth.npz` and ROI overlays.

### Pulse signal and heart-rate estimation

Each valid ROI produces a mean RGB trace. The pipeline interpolates missing signal values when needed, detrends, band-pass filters, and evaluates three classical methods:

| Baseline | Approach |
|:---|:---|
| **Green** | Pulse estimation from the green color channel |
| **CHROM** | Chrominance-based pulse projection |
| **POS** | Plane-orthogonal-to-skin pulse projection |

Heart rate is estimated using the dominant spectral peak in sliding FFT windows and aligned with reference HR timestamps.

**Configured experiment:** 30-second windows, 1-second step, 42–210 BPM search range. **Output:** `outputs/signal_extraction/signal_extraction_dataset_manifest.json` and per-sample signals/metrics.

### Multi-region fusion

Fusion consumes **saved signal-extraction artifacts**, so it does not need to process raw videos a second time. It compares:

- **Single ROI:** independent HR from forehead, left cheek and right cheek.
- **Average fusion:** equal-weight averaging of the three ROI signals.
- **Quality-weighted fusion:** per-method, per-window weights derived from valid-frame ratio, signal variability, spectral peak clarity, SNR-like score, and disagreement between ROI estimates.

Weights are nonnegative and normalized to sum to one, with a fallback for invalid or degenerate scores. Signals from overlapping windows are combined to form a continuous fused trace. The output includes per-ROI predictions, fused predictions, feature vectors, weights, metrics and visualizations.

**Output:** `outputs/fusion/fusion_manifest.json`, `fusion_signals.npz`, per-sample metric files and aggregate tables.

## Reports and visualizations

The repository includes two detailed **HTML experiment reports**. Download or open these locally to view their full commentary and figures.

| Report | Contents |
|:---|:---|
| **[Signal Extraction & Classical Baselines](report/task2_signal_extraction_report.html)** | Green/CHROM/POS, train/validation/test metrics, MAE/RMSE/Pearson/SNR interpretation and failure discussion |
| **[Multi-ROI Fusion](report/task3_fusion_report.html)** | Single-region vs average vs quality-weighted strategies, ROI weight behavior, test comparison and limitations |

<div align="center">
  <img src="report/assets/task2_test_baselines.png" alt="Classical rPPG baseline comparison on test recordings" width="48%" />
  <img src="report/assets/task3_test_rmse_pearson.png" alt="Fusion comparison: test RMSE and Pearson correlation" width="48%" />
</div>

Additional artifacts: [Baseline MAE by split](report/assets/task2_mae_by_split.png) · [Example HR traces and ROI weights](report/assets/task3_sample_visualization.png) · [Research plan](docs/task.md) · [Research proposal](docs/rppg_research_proposal.docx)

## Getting started

### Prerequisites

- **Python 3.11** (required by `pyproject.toml`; the project specifies `>=3.11,<3.12`).
- [`uv`](https://docs.astral.sh/uv/) for dependency and environment management.
- [UBFC-rPPG](https://sites.google.com/view/ybenezeth/ubfcrppg) videos and reference signals for preprocessing and signal extraction.

### Install and test

```bash
git clone https://github.com/duc-vu/lightweight-rppg.git
cd lightweight-rppg

uv sync --extra dev
uv run pytest
uv run jupyter lab
```

If an editable installation fails on a Windows path containing non-ASCII characters, the existing project documentation suggests:

```bash
uv sync --extra dev --no-editable
```

### Prepare the dataset

Put the **original UBFC files** under the following structure; raw videos are not bundled with the repository.

```text
datasets/UBFC_DATASET/
├── DATASET_1/
│   └── 10-gt/
│       ├── vid.avi
│       └── gtdump.xmp
└── DATASET_2/
    └── subject1/
        ├── vid.avi
        └── ground_truth.txt
```

Other recording folders follow the same layouts. The preprocessing notebook discovers videos recursively. Keep dataset files on your local machine and comply with the dataset provider's terms.

### Reproduce the experiments

1. [`01_preprocessing.ipynb`](notebooks/01_preprocessing.ipynb) — face landmarks, ROI masks and normalized reference data.
2. [`02_signal_extraction.ipynb`](notebooks/02_signal_extraction.ipynb) — mean RGB, Green/CHROM/POS, sliding-window HR and metrics.
3. [`03_fusion.ipynb`](notebooks/03_fusion.ipynb) — single ROI and fusion experiments, metrics and figures.

The notebooks locate the repository root and add `src/` to the Python import path. They write versioned manifests with relative paths so that later stages can consume earlier artifacts. **RGB extraction requires access to the original videos**; the fusion analysis can run directly from saved signal-extraction artifacts.

Some generated experimental artifacts are already committed under `outputs/`; this makes the repository relatively large. Re-running the earlier notebooks may replace generated outputs.

### Check the stored experiment

```bash
uv run python -c "import json; from pathlib import Path; p=Path('outputs/fusion/fusion_manifest.json'); print(json.loads(p.read_text(encoding='utf-8'))['counts'])"
```

Expected for the currently committed experiment: `{'total': 49, 'processed': 49, 'failed': 0}`.

The test suite in [`tests/test_core.py`](tests/test_core.py) exercises ground-truth parsing, paths/splits, ROI handling, signal processing, baseline shapes, fusion weights and artifact schemas. It is a set of **unit/synthetic-data checks**, not proof of clinical validity.

## Repository map

```text
docs/                        Research proposal and project plan
notebooks/                   The three executable experiment stages
src/airppg/preprocessing/    Dataset discovery, ROI and ground truth
src/airppg/signal_extraction/ RGB extraction and classical rPPG methods
src/airppg/fusion/           ROI quality, strategies and evaluation
src/airppg/signal.py         Signal processing and HR estimation
src/airppg/metrics.py        Ground-truth alignment and metrics
outputs/                     Versioned manifests and generated artifacts
report/                      Experiment reports and figures
tests/                       Unit tests on synthetic fixtures
```

## Research scope

The current repository contains an experimental pipeline for facial ROI extraction, classical rPPG signal processing, and multi-region fusion with reproducible result artifacts. Lightweight learned models and confidence estimation are potential follow-up research directions; no trained neural HR estimator or validated confidence model is included.

## Limitations

- **Limited evaluation:** the current test split contains **five recordings**; the reported means do not include uncertainty intervals or statistical significance tests.
- **Generalization remains open:** the current evaluation uses UBFC-rPPG. Real webcam conditions, demographics, lighting and movement require additional validation.
- **Classical methods only:** DeepPhys, PhysNet, TS-CAN, EfficientPhys and small neural models are mentioned in planning materials but **have not been evaluated here**.
- **Ground truth & protocol:** reference HR is aligned to prediction windows; results depend on the dataset, sampling/splitting policy, FFT windowing and spectral range.
- **Not a medical device:** estimates are for research and education, not diagnosis, treatment or health monitoring decisions.

## License

Source code is available under the [MIT License](LICENSE). The **UBFC-rPPG dataset has its own terms** and is not licensed by this repository's MIT license.
