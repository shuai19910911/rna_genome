# Unified Gate A Evaluation

**Date**: 2026-05-31

All methods are evaluated on identical masks and sample subsets. For imputation, both global entry-wise Pearson and mean per-sample Pearson are reported.

## Random Mask Test Set

| Method | Global Pearson | Mean Per-Sample Pearson | RMSE | NRMSE | Tissue F1 |
|---|---:|---:|---:|---:|---:|
| PCA | 0.9185 | 0.9056 | 0.4020 | 0.4256 | - |
| M1 | 0.7690 | 0.7267 | 0.6142 | 0.6504 | 0.3313 |
| M2 | 0.7735 | 0.7325 | 0.6087 | 0.6446 | 0.3378 |
| M2-shuffled | 0.7723 | 0.7310 | 0.6091 | 0.6450 | 0.3445 |

## Module Mask Test Set

| Method | Global Pearson | Mean Per-Sample Pearson | RMSE | NRMSE |
|---|---:|---:|---:|---:|
| PCA | 0.7609 | 0.7487 | 0.6776 | 0.7159 |
| M1 | 0.6475 | 0.5738 | 0.7362 | 0.7778 |
| M2 | 0.6460 | 0.5727 | 0.7481 | 0.7904 |
| M2-shuffled | 0.6426 | 0.5679 | 0.7529 | 0.7955 |

## External Random Mask

| Method | Global Pearson | Mean Per-Sample Pearson | RMSE | NRMSE |
|---|---:|---:|---:|---:|
| PCA | 0.9026 | 0.8909 | 0.4491 | 0.4553 |
| M1 | 0.7434 | 0.7045 | 0.6698 | 0.6790 |
| M2 | 0.7436 | 0.7035 | 0.6699 | 0.6792 |
| M2-shuffled | 0.7419 | 0.7019 | 0.6719 | 0.6812 |

## Notes

- Test samples evaluated: 1613.
- External samples evaluated: 260.
- PCA is fit on the valid tissue-labelled training subset used by M1/M2 dataloaders: 2914 samples.
- Random mask count: 3; module mask count: 3.
- This report is intended to replace cross-method comparisons from earlier logs where aggregation rules differed.
