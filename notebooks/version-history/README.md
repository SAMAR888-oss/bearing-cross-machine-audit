# Version history

Earlier save-points of `bearing_cross_machine_audit.ipynb`, kept for provenance.

Work was append-only: each save-point adds cells to the end of the previous one
and never edits or removes earlier cells. So each file is a strict prefix of the
next, and `checkpoint-04` plus the final notebook contain everything in the
earlier ones. They are here to show how the audit grew, not because they add
results.

| File | Cells | What it added |
|---|---|---|
| `checkpoint-01.ipynb` | 52 | Two-dataset stage (CWRU, Paderborn). Feature pipeline, XGBoost and a TensorFlow model, SHAP importance, t-SNE, confusion matrices, the first Wasserstein gap metric, and CORAL / DANN adaptation baselines. |
| `checkpoint-02.ipynb` | 68 | MFPT added, making three datasets, and the first cross-dataset transfer matrix. |
| `checkpoint-03.ipynb` | 104 | Confidence and uncertainty audit: MC-dropout, expected calibration error, reliability diagrams, gradient-reversal experiments, and a move to PyTorch. Balanced accuracy becomes the reported metric. |
| `checkpoint-04.ipynb` | 140 | Ottawa and HIT-SM added, taking the audit to five datasets and twenty directed pairs. Introduces the overconfidence analysis, the directional gap metrics (nearest-neighbour coverage, class-variance containment), and the TENT test-time adaptation stress test. |

The final notebook adds the remaining 14 cells, which produce the figures and
export the result tables. It introduces no new methods.

Outputs are preserved throughout, so results are visible without re-running
anything.
