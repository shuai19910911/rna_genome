# Next Execution Plan

**Date**: 2026-06-01  
**Current stage**: Gate A validation-protocol repair before rerunning canonical models.

## Current Conclusion

The full unified evaluator has now run successfully. It shows PCA remains much stronger than M1/M2, and true M2 does not yet provide a robust promoter-pairing advantage over M1 or M2-shuffled.

Key decision: do not continue model claims or cross-species expansion until M1/M2/M2-shuffled have been rerun with validation-based checkpointing.

## Completed Today

1. Full unified evaluation completed:
   - `results/unified_eval.json`
   - `docs/2026-05-31-unified-eval.md`
2. Validation protocol implemented:
   - train: `split_train.txt`
   - checkpoint selection: `split_val.txt`
   - final report: `split_test.txt`
3. Training code now writes final summaries:
   - `results/<run>_log.json`
   - `results/<run>_best.pt`
   - `results/<run>_summary.json`
4. `scripts/evaluate_unified.py` no longer uses the warning-prone default `torch.load(..., weights_only=False)` path when the installed PyTorch supports `weights_only=True`.

## Completed After This Plan

The validation-protocol M2 smoke test completed successfully:

```bash
CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 5 --run-name M2-val-smoke
```

Outputs:

- `results/m2-val-smoke_log.json`
- `results/m2-val-smoke_best.pt`
- `results/m2-val-smoke_summary.json`

Protocol checks:

- Console printed `Selection split: val`.
- Log entries include `selection_split: "val"`.
- Summary JSON contains `final_test`.

Best validation checkpoint: epoch 5.

| Metric | Value |
|---|---:|
| validation random-mask r | 0.5048 |
| validation module-block r | 0.3038 |
| validation tissue macro-F1 | 0.3333 |
| external r during validation eval | 0.3359 |

Final test metrics from validation-best checkpoint:

| Metric | Value |
|---|---:|
| test random-mask r | 0.5336 |
| test module-block r | 0.3857 |
| test tissue macro-F1 | 0.1724 |
| external r | 0.3321 |

Resource note: on NVIDIA A100-SXM4-40GB, the user-observed process allocation was about 32.4 GB VRAM.

## Immediate Next Task

Promoter embedding gene-order provenance has been saved to `data/promoter_emb_genes.txt`. The immediate remaining task is to add a loader-side alignment check so M2 refuses to run if current HVG order differs from the embedding order.

## After Gene-Order Audit

Run canonical validation-protocol training in this order:

```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m1 --epochs 80
```

Expected resources:

- GPU: 1 card; suggested A100 40GB or any free >=24GB card.
- VRAM: likely below M2, but use >=24GB; M2 smoke test observed about 32.4GB.
- CPU: 4-8 cores.
- RAM: 32-64 GB.
- Runtime: tens of minutes to a few hours depending on load.
- Outputs to check: `results/m1_log.json`, `results/m1_best.pt`, `results/m1_summary.json`.

```bash
CUDA_VISIBLE_DEVICES=1 python cli.py train m2 --epochs 80
```

Expected resources:

- GPU: 1 card; suggested A100 40GB.
- VRAM: about 32-36 GB based on smoke test, use >=40GB preferred.
- CPU: 4-8 cores.
- RAM: 32-64 GB.
- Runtime: tens of minutes to a few hours depending on load.
- Outputs to check: `results/m2_log.json`, `results/m2_best.pt`, `results/m2_summary.json`.

```bash
CUDA_VISIBLE_DEVICES=2 python cli.py train m2 --epochs 80 --shuffle-promoters
```

Expected resources:

- GPU: 1 card; suggested A100 40GB.
- VRAM: about 32-36 GB based on smoke test, use >=40GB preferred.
- CPU: 4-8 cores.
- RAM: 32-64 GB.
- Runtime: tens of minutes to a few hours depending on load.
- Outputs to check: `results/m2-shuffled_log.json`, `results/m2-shuffled_best.pt`, `results/m2-shuffled_summary.json`.

Then rerun unified evaluation:

```bash
CUDA_VISIBLE_DEVICES=0 python scripts/evaluate_unified.py
```

Do not interpret the new training runs until the final unified evaluation report is regenerated.

## Previous Immediate Task

Run a 5-epoch M2 smoke test under the new validation protocol. Use a noncanonical run name so the existing M2 checkpoint is not overwritten.

```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m2 --epochs 5 --run-name M2-val-smoke
```

Expected resources:

- GPU: 1 card; suggested GPU 0 or any free single card.
- VRAM: 10-16 GB expected; >=24 GB preferred.
- CPU: 4-8 cores.
- RAM: 32-64 GB.
- Runtime: 5-15 min.
- Outputs to check: `results/m2-val-smoke_log.json`, `results/m2-val-smoke_best.pt`, `results/m2-val-smoke_summary.json`.

Success criteria:

- Console prints `Selection split: val`.
- Log entries include `selection_split: "val"`.
- Summary JSON contains `final_test`.

