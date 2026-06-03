# Next Execution Plan

**Date**: 2026-05-31  
**Current stage**: Gate A repair / evaluation hardening before more model claims.

## Current Conclusion

The soybean-only pipeline now has M1, M2, and M2-shuffled results, but the current evidence is not strong enough for a promoter-modality claim.

Main blockers:

1. PCA and M1/M2 use different Pearson aggregation rules.
2. M1/M2 currently use test module-block r for checkpoint selection.
3. BioProject and tissue are strongly confounded.
4. M2-shuffled is nearly tied with true M2 on module-block r.
5. Current promoter embeddings come from a randomly initialized local CNN, not a pretrained DNA encoder.

## Short-Term Plan

### Step 1: Implement unified evaluator

Build `scripts/evaluate_unified.py`.

Purpose:

- Load existing checkpoints: `m1_best.pt`, `m2_best.pt`, `m2-shuffled_best.pt`.
- Recreate the same HVG, scaling, split, tissue-label setup.
- Evaluate M1, M2, M2-shuffled under identical masks and identical test samples.
- Evaluate PCA-200 on the same masks and sample subset.
- Report both:
  - global entry-wise Pearson
  - mean per-sample Pearson
- Output:
  - `results/unified_eval.json`
  - `docs/2026-05-31-unified-eval.md`

Resources:

- GPU: optional but recommended for model inference; 1 GPU is enough.
- Suggested GPU: `CUDA_VISIBLE_DEVICES=1` or any free single card.
- VRAM: expected <16 GB for inference, but use >=24 GB card if available.
- CPU: 4-8 cores.
- RAM: 32-64 GB, because PCA and expression matrices can be memory-heavy.
- Runtime: 10-30 min expected.

### Step 2: Fix validation protocol

Modify training/evaluation flow so:

- `split_train.txt` trains.
- `split_val.txt` selects checkpoints.
- `split_test.txt` is held out for final reporting only.

Output target:

- updated trainer or evaluation script supporting train/val/test.
- documented protocol in `docs/2026-05-31-validation-protocol.md`.

### Step 3: Verify promoter embedding alignment

Save and validate the exact HVG order used for promoter embeddings.

Output target:

- `data/promoter_emb_genes.txt`
- loader check that `promoter_emb.npy` rows match current HVG order.

### Step 4: Decide next model ablation

After unified evaluator:

- If true M2 still does not beat shuffled M2, do not continue current promoter branch.
- Then choose one:
  - gene-embedding-only ablation
  - M2 with frozen/removed gene identity embedding
  - trainable promoter encoder with saved gene-order alignment
  - pretrained DNA encoder if available

## Immediate Next Task

Do not run another training job yet. First implement `scripts/evaluate_unified.py`.

After it is implemented, the command to run will be:

```bash
CUDA_VISIBLE_DEVICES=1 python scripts/evaluate_unified.py
```

Expected resources:

- GPU: 1 card, suggested GPU 1 or any free card.
- VRAM: <16 GB expected; >=24 GB preferred.
- CPU: 4-8 cores.
- RAM: 32-64 GB.
- Runtime: 10-30 min.
- Outputs to check: `results/unified_eval.json`, `docs/2026-05-31-unified-eval.md`.
