# Unified Gate A Evaluation

**Date**: 2026-05-31

All methods are evaluated on identical masks and sample subsets. For imputation, both global entry-wise Pearson and mean per-sample Pearson are reported.

## Random Mask Test Set

| Method | Global Pearson | Mean Per-Sample Pearson | RMSE | NRMSE | Tissue F1 |
|---|---:|---:|---:|---:|---:|
| PCA | 0.7247 | 0.6697 | 0.6134 | 0.6960 | - |

## Module Mask Test Set

| Method | Global Pearson | Mean Per-Sample Pearson | RMSE | NRMSE |
|---|---:|---:|---:|---:|
| PCA | 0.5869 | 0.5709 | 0.7601 | 0.8257 |

## External Random Mask

| Method | Global Pearson | Mean Per-Sample Pearson | RMSE | NRMSE |
|---|---:|---:|---:|---:|
| PCA | 0.8449 | 0.8430 | 0.5581 | 0.5510 |

## Notes

- Test samples evaluated: 8.
- External samples evaluated: 8.
- PCA is fit on the valid tissue-labelled training subset used by M1/M2 dataloaders: 2914 samples.
- Random mask count: 1; module mask count: 1.
- This report is intended to replace cross-method comparisons from earlier logs where aggregation rules differed.
