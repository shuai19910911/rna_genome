# Soybean Gate A Results Summary

**Date**: 2026-05-31  
**Selection rule**: M1/M2/M2-shuffled use the checkpoint with best module-block r. PCA values are from `results/baseline_results.json`.

## Main Table

| Method | Epoch | Random-mask r | Module-block r | Tissue macro-F1 | External r |
|---|---:|---:|---:|---:|---:|
| PCA-200 | - | 0.9181 | - | - | 0.9029 |
| M1 | 80 | 0.7314 | 0.5986 | 0.3613 | 0.7057 |
| M2 | 80 | 0.7348 | 0.6036 | 0.3645 | 0.7070 |
| M2-shuffled | 60 | 0.7264 | 0.6030 | 0.3359 | 0.6970 |

## Deltas

| Comparison | Random-mask r | Module-block r | Tissue macro-F1 | External r |
|---|---:|---:|---:|---:|
| M2 - M1 | 0.0034 | 0.0049 | 0.0032 | 0.0013 |
| M2 - M2-shuffled | 0.0084 | 0.0006 | 0.0286 | 0.0100 |
| M2 - PCA-200 | -0.1833 | - | - | -0.1958 |

## Interpretation

1. M2 is only slightly above M1: module-block r improves by 0.0049.
2. M2 and M2-shuffled are effectively tied on module-block r: delta 0.0006.
3. M2 remains far below PCA-200 on random-mask reconstruction: delta -0.1833.
4. Current single-seed results do not establish a robust promoter-gene pairing benefit.

## Required Next Step

Do not claim promoter modality success yet. Next work should either run more seeds for M2/M2-shuffled or move to evaluation/diagnostics that can explain why shuffled promoters are nearly tied with true promoters.
