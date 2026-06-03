# Gate A Diagnostics

**Date**: 2026-05-31  
**Purpose**: Explain why M2-shuffled nearly matches true M2 and identify evaluation risks before making promoter claims.

## Summary

Current Gate A results are useful for debugging, but not yet sufficient for a biological promoter-modality claim.

Main issues:

1. **Metric mismatch**: PCA baselines report global masked-entry Pearson, while M1/M2 report mean per-sample Pearson. These are not directly comparable.
2. **Test-set checkpointing**: M1/M2 save the best model using test module-block r. `split_val.txt` exists but is not used by `Trainer.run()`.
3. **BioProject/tissue confounding**: 87.2% of BioProjects with tissue metadata are single-tissue; median majority-tissue purity is 1.000.
4. **Promoter evidence is weak**: M2 - M2-shuffled module-block r is only 0.0006.
5. **Promoter embeddings are not pretrained**: `promoter_emb.npy` comes from a randomly initialized local CNN.

## Split / Metadata Diagnostics

| Split | Samples | Valid tissue samples | Invalid/unknown |
|---|---:|---:|---:|
| train | 2927 | 2914 | 13 |
| val | 850 | 844 | 6 |
| test | 1613 | 1613 | 0 |

BioProject/tissue purity:

| Metric | Value |
|---|---:|
| BioProjects with tissue metadata | 494 |
| Single-tissue BioProjects | 431 |
| Single-tissue fraction | 0.872 |
| Mean majority-tissue purity | 0.937 |
| Median majority-tissue purity | 1.000 |

## Metric Comparability

| Component | Current behavior | Risk |
|---|---|---|
| PCA random mask | global entry-wise Pearson over all masked entries | Not comparable to M1/M2 per-sample mean Pearson |
| PCA module masks | global entry-wise Pearson | Not comparable to M1/M2 module-block r |
| M1/M2 random mask | mean of per-sample Pearson correlations | Different weighting of samples/genes |
| External validation | PCA fixed numpy masks; M1/M2 stochastic torch masks | Run-to-run variance and metric mismatch |
| Model selection | best checkpoint selected on test module-block r | Test leakage / optimistic model selection |

## Promoter Embedding Diagnostics

| Item | Value |
|---|---:|
| Embedding shape | [5000, 256] |
| Finite fraction | 1.0000 |
| Mean | 0.0761 |
| Std | 0.0268 |
| Mean row norm | 1.2904 |
| Near-zero variance dims | 0 |

Interpretation: shuffled promoter embeddings can still behave like arbitrary extra per-gene features, especially because M2 also has trainable gene identity embeddings. This explains why shuffled promoters can nearly match true promoters without proving sequence-specific biology.

## Model Result Context

| Metric | M2 | M2-shuffled | Delta |
|---|---:|---:|---:|
| random-mask r | 0.7348 | 0.7264 | +0.0084 |
| module-block r | 0.6036 | 0.6030 | +0.0006 |
| tissue macro-F1 | 0.3645 | 0.3359 | +0.0286 |
| external r | 0.7070 | 0.6970 | +0.0100 |

## Recommended Fixes

1. Build a unified evaluator for PCA/M1/M2/M2-shuffled with identical masks, sample sets, and both global-entry and per-sample Pearson metrics.
2. Use `split_val.txt` for checkpoint selection; reserve `split_test.txt` for final reporting.
3. Save the exact HVG gene list used for `promoter_emb.npy` and verify alignment during M2 loading.
4. Add a gene-embedding-only ablation or remove/freeze `gene_embed` to test whether promoter features add information beyond gene identity.
5. Treat current M2 as an engineering ablation, not evidence for promoter biology.
