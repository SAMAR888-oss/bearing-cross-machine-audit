# Cross-Machine Bearing Fault Diagnosis: A Generalization Audit

Bearing fault diagnosis models routinely report >95% accuracy on standard benchmarks. We wanted to know what survives when such a model is moved to a different machine.

So we trained diagnosis models on five bearing benchmarks, evaluated every one on every other, and audited the result along five axes: how far accuracy falls, whether any cheap metric predicts the fall, what the model's own confidence does, whether a modern test-time adaptation method recovers anything, and whether amplitude normalization helps.

The short answer: within-dataset accuracy runs 0.897–0.997, cross-machine transfer averages **0.40** against a chance floor of 0.333, and nothing we could compute from the source data ahead of time predicted which transfers would work.

---

## What we found

**1. Transfer collapses, and it is directional.**

Every within-dataset score exceeds 0.80. Every cross-dataset score averages 0.40. More interestingly, the collapse is asymmetric: MFPT→CWRU reaches 0.625 while CWRU→MFPT manages 0.304, and Paderborn→CWRU reaches 0.616 while CWRU→Paderborn gives 0.345 — even though the distributional distance between each pair is symmetric by construction.

Rows are the training dataset, columns the evaluation dataset. Diagonal is within-dataset 5-fold CV. Chance = 0.333.

| Train \ Test | CWRU | Paderborn | MFPT | Ottawa | HIT-SM |
|---|---|---|---|---|---|
| **CWRU** | 0.997 | 0.345 | 0.304 | 0.337 | 0.596 |
| **Paderborn** | 0.616 | 0.897 | 0.333 | 0.333 | 0.333 |
| **MFPT** | 0.625 | 0.306 | 0.997 | 0.482 | 0.470 |
| **Ottawa** | 0.333 | 0.333 | 0.333 | 0.991 | 0.333 |
| **HIT-SM** | 0.438 | 0.357 | 0.321 | 0.400 | 0.981 |

The failures are structured rather than noisy. Under the hardest transfers the model collapses onto a single class: the Ottawa→CWRU confusion matrix (`fig_confusion.png`, generated near the end of the notebook) shows Normal predicted for every target input. A t-SNE embedding of the standardized features (`fig_tsne.png`) shows the five datasets occupying largely separate regions.

**2. No distributional metric predicts transfer.**

We tested six candidate screens, all computable from unlabeled data before deployment. None reached significance across the twenty directed pairs:

| Screen | Spearman ρ | p |
|---|---|---|
| Mean per-feature Wasserstein distance | +0.10 | 0.69 |
| Per-class Wasserstein decomposition | −0.38 | 0.10 |
| Directional nearest-neighbour coverage | +0.13 | 0.57 |
| Centroid shift | +0.01 | 0.96 |
| Source spread | −0.29 | 0.21 |
| Class-variance containment | +0.12 | 0.62 |

Part of why: a symmetric distance cannot capture an asymmetric phenomenon, and cross-machine scale heterogeneity corrupts the geometric metrics. We had earlier seen a gap metric that looked predictive on three datasets. Scaling to five killed it entirely, which is the main reason we ran the audit at this size.

**3. Overconfidence does track failure — in the worst direction.**

The one signal that correlates with transfer is the model's own miscalibration. Mean predicted-class confidence exceeds balanced accuracy by **0.53** across all twenty transfers, and the excess grows as accuracy collapses: Spearman ρ = **−0.78**, Pearson r = −0.90, p = 1×10⁻⁴. Ottawa-trained models transfer at chance while predicting at essentially unit confidence.

Raw confidence on its own tracks transfer only weakly (ρ = −0.37, p = 0.11). It is the calibration *gap*, not the confidence magnitude, that carries the signal. In-domain predictions sit on the calibration diagonal; foreign-domain predictions pile up in the high-confidence, low-accuracy corner.

**4. Confidence-based adaptation makes it worse.**

If confidence is miscalibrated, a method that optimizes confidence should fail. We tested TENT, which minimizes target prediction entropy over batch-norm affine parameters. Mean Δ balanced accuracy: **−0.019**. Ten of twenty pairs worsened, six improved. Mean confidence rose from 0.93 to 0.94 while accuracy fell.

The mechanism is direct: entropy minimization rewards low-entropy predictions, but under cross-machine shift low entropy coincides with error, so TENT sharpens the miscalibration instead of correcting it. Ottawa→CWRU falls from 0.395 to 0.142 while confidence stays at 0.97. The gains that do occur are pair-specific (Ottawa→MFPT +0.24, HIT-SM→MFPT +0.22), not general.

**5. Amplitude normalization is a conditional remedy.**

Per-window normalization removes amplitude information. It substantially helps pairs that were leaning on an amplitude shortcut (Paderborn→MFPT 0.33→0.63, MFPT→HIT-SM 0.47→0.69) and actively harms pairs where amplitude carries transferable signal (CWRU→MFPT 0.30→0.23, Paderborn→HIT-SM 0.33→0.16). The diagonal barely moves (0.972→0.949), so within-dataset separability is not primarily amplitude-driven.

---

## What this means in practice

| Situation | What the audit says to do |
|---|---|
| New target machine, no target labels | Don't trust distributional gap or model confidence. Get target labels before deploying. |
| High in-domain confidence and calibration | Not evidence of transfer safety. In-domain calibration does not survive shift. |
| Considering confidence-based test-time adaptation | Avoid under strong shift; entropy minimization sharpens miscalibration. |
| Raw-signal model, suspected amplitude mismatch | Try per-window normalization — but only if an amplitude shortcut is plausible. |
| Scale-invariant feature model | Normalization won't help; the residual shift is elsewhere. |
| Direction matters | Validate the specific source→target direction. The reverse can behave completely differently. |

---

## Method

**Datasets.** Five rolling-element bearing benchmarks, recorded on different test rigs under different conditions.

| Dataset | Notes |
|---|---|
| CWRU | Drive-end accelerometer, 12k and 48k sampling |
| Paderborn | Pinned to one operating condition, N15_M07_F10, vibration_1 channel |
| MFPT | Baseline + inner-race + outer-race; baselines 97656 Hz, faults 48828 Hz |
| Ottawa (UORED) | Drive-end vibration channel only (column 0 of 5) |
| HIT-SM | SpectraQuest MFS rig, all shaft speeds pooled |

**Classes.** The five datasets do not share a ball-fault class, so everything is restricted to the three overlapping ones: Normal, inner race (IR), outer race (OR). Chance is 0.333 and all results are balanced accuracy.

**Features.** One identical pipeline for all five datasets, so that preprocessing cannot act as a confounder. Each signal is cut into non-overlapping 2048-sample windows, and nine time-domain statistics are computed per window: max, min, mean, standard deviation, RMS, skewness, kurtosis, crest factor, form factor.

**Models.** XGBoost (200 estimators, max depth 4) on the nine features. For the TENT experiments, which need batch-norm layers, a small three-block CNN on 1024-sample windows reshaped to 32×32. CNN source-only numbers therefore differ from the tree transfer matrix; both fail far below their within-dataset accuracy.

**Protocol.** Diagonal entries are 5-fold stratified cross-validation within a dataset. Off-diagonal entries train on the full source and evaluate on the full target, with standardization fit on the *source*, so transfer is not trivially defeated by raw amplitude scale.

---

## Also in the notebook

Beyond the five axes above, the notebook contains earlier and supporting analyses that shaped the audit but are not part of the headline results:

- **CORAL and DANN** — classical feature-alignment and domain-adversarial adaptation baselines, tried before TENT.
- **MC-dropout** — uncertainty estimation on the neural model, as an alternative trust signal to softmax confidence.
- **Reliability diagrams and expected calibration error** — the in-domain calibration checks that motivated looking at the calibration gap under shift.
- **SHAP importance** — gain-based feature importance for the within-CWRU tree model. Standard deviation dominates (0.448 of total gain), with the remaining eight time-domain statistics contributing progressively less.
- **A TensorFlow model** alongside the later PyTorch CNN, from the earlier two- and three-dataset stages.

These ran across several sessions, so not all of them have saved outputs in the final notebook. See `notebooks/version-history/` for the stage each belongs to.

## Repository

```
notebooks/
  bearing_cross_machine_audit.ipynb    the full audit
  version-history/                     earlier save-points of the same notebook
data/
  README.md                            where to download each dataset
```

The notebook runs top to bottom in this order: dataset loaders → feature extraction → transfer matrix → gap screens → confidence and calibration → TENT → normalization ablation → figures.

## Running it

```bash
git clone https://github.com/<your-username>/bearing-cross-machine-audit.git
cd bearing-cross-machine-audit

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter lab notebooks/bearing_cross_machine_audit.ipynb
```

The datasets are not included — they are large and distributed by their owners. See [`data/README.md`](data/README.md) for download links and the folder layout the loaders expect.

Paths in the notebook currently point at Kaggle mounts (`/kaggle/input/...`), which is where we ran this. To run locally, edit the root constants in the loader cells to point at your copies.

The TENT section needs PyTorch and in practice a GPU; everything else runs on CPU.

## Team

This project was built by four of us, working together:

| | |
|---|---|
| **Lalitaditya Tickoo** | Dept. of Computing Technologies |
| **Md Samar Aazmi** | Dept. of Computing Technologies |
| **Ayushman Mayank** | Dept. of Data Science and Business Systems |
| **Tushar Biswas** | Dept. of Computing Technologies |

SRM Institute of Science and Technology, Kattankulathur, Tamil Nadu, India.

## License

MIT, see [`LICENSE`](LICENSE). The five benchmark datasets are not redistributed here and remain under their original terms.
