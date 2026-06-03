# Findings: 2026-05-29 Master Execution Plan

## Consolidated Findings

### Baseline Strength
- PCA-200 is the strongest current imputation baseline.
- Internal random-mask r = 0.9181.
- External rnadata r = 0.9029.
- This makes random 15% mask reconstruction a poor primary task for claiming deep-model superiority.

### M1 Status
- M1 trained successfully and produced `results/m1_log.json` plus `results/m1_best.pt`.
- Best epoch 80 metrics:
  - random-mask r = 0.7314
  - module-block r = 0.5986
  - tissue macro-F1 = 0.3613
  - external r = 0.7057
- M1 is currently below PCA-200 and should not be framed as outperforming classical baselines.

### M2 Status
- Old M2 run failed because online promoter CNN encoding required ~38 GB allocation and caused CUDA OOM.
- Current code path in `cli.py` / `src/model/m2.py` uses precomputed promoter embeddings, so M2 should be re-tested from the unified CLI.
- There is no valid M2 result file yet.

### Promoter Embedding Caveat
- `data/promoter_emb.npy` was generated from a lightweight project-local CNN, not a pretrained DNA foundation model.
- Unless replaced or justified, M2 results should be interpreted as a promoter-derived feature ablation, not strong evidence for pretrained promoter biological priors.

### Execution Priority
- The immediate blocking task is M2 smoke test and shuffled-promoter negative control.
- Cross-species work should wait until soybean-only claims and evaluation tasks are stabilized.

## Decisions

| Date | Decision | Rationale |
|---|---|---|
| 2026-05-29 | Use `.planning/2026-05-29-master-execution-plan` as active plan | Historical plans are stale relative to current results |
| 2026-05-29 | Treat PCA-200 as the reference baseline | It is empirically strongest on current metrics |
| 2026-05-29 | Do not claim M1 success over classical methods | M1 is below PCA and tissue classifier baselines |
| 2026-05-29 | Require shuffled promoter control for M2 | Needed to test whether promoter pairing matters |
| 2026-05-29 | Update plan docs after every task | User requested project execution to be plan-driven |

### CLI Smoke Test Bug: Missing Model Hyperparameters
- Date: 2026-05-30
- The first M2 smoke test failed before training because `argparse.Namespace` lacked `n_heads`.
- Root cause: `cli.py` model constructors used `args.n_heads`, `args.d_ff`, and `args.dropout`, but the parser did not define those fields.
- Fix: added CLI defaults for these fields from `src.config.DEFAULTS`.
- Related cleanup: GPU selection now follows `CUDA_VISIBLE_DEVICES=... python cli.py ...`; `--gpu` is optional legacy compatibility.

### M2 Smoke Test Completed
- Date: 2026-05-30
- Command: `CUDA_VISIBLE_DEVICES=0 python cli.py train m2 --epochs 5`
- Outputs: `results/m2_log.json`, `results/m2_best.pt`
- Best epoch: 5
- Metrics: random-mask r=0.5302, module-block r=0.3424, tissue macro-F1=0.1830, external r=0.3329.
- Conclusion: the precomputed promoter embedding M2 code path is runnable. Full training and shuffled-promoter control remain required before any model comparison.

### Full M2 Training Completed
- Date: 2026-05-31
- Command: `CUDA_VISIBLE_DEVICES=1,2 python cli.py train m2 --epochs 80`
- Outputs: `results/m2_log.json`, `results/m2_best.pt`
- Best epoch: 80
- Metrics: random-mask r=0.7348, module-block r=0.6036, tissue macro-F1=0.3645, external r=0.7070.
- Compared with M1 epoch 80: random-mask r +0.0034, module-block r +0.0049, tissue macro-F1 +0.0032, external r +0.0013.
- Conclusion: M2 runs successfully and slightly exceeds M1, but the improvement is small. Shuffled-promoter control is required before attributing gains to promoter-gene pairing.

### M2 Shuffled-Promoter Negative Control Completed
- Date: 2026-05-31
- Command: `CUDA_VISIBLE_DEVICES=1 python cli.py train m2 --epochs 80 --shuffle-promoters`
- Outputs: `results/m2-shuffled_log.json`, `results/m2-shuffled_best.pt`
- Best epoch by module-block r: 60
- M2-shuffled metrics: random-mask r=0.7264, module-block r=0.6030, tissue macro-F1=0.3359, external r=0.6970.
- True M2 minus shuffled: random-mask r +0.0084, module-block r +0.0006, tissue macro-F1 +0.0286, external r +0.0100.
- Conclusion: Shuffled promoter control is nearly tied with true M2 on module-block r. Current evidence does not support a strong promoter-gene pairing benefit.

### Gate A Diagnostics Completed
- Date: 2026-05-31
- Outputs: `results/gate_a_diagnostics.json`, `docs/2026-05-31-gate-a-diagnostics.md`
- Metric comparability risk: high. PCA baseline uses global entry-wise Pearson over masked entries; M1/M2 use mean per-sample Pearson, so direct numeric comparison is not clean.
- Model-selection risk: M1/M2 select best checkpoints on test module-block r. `split_val.txt` exists but is not used by `Trainer.run()`.
- Metadata confounding: 431/494 BioProjects with tissue metadata are single-tissue; single-tissue fraction = 0.872; median majority-tissue purity = 1.000.
- Promoter evidence: M2 - M2-shuffled module-block r = +0.0006, so promoter-gene pairing benefit is not established.
- Promoter embedding caveat strengthened: `promoter_emb.npy` is generated by a randomly initialized local CNN and has no saved companion HVG gene-order file.
- Recommended decision: do not claim promoter modality success; build unified evaluator and validation-based checkpointing before more claims.

### Unified Evaluator Implemented
- Date: 2026-05-31
- Script: `scripts/evaluate_unified.py`
- Purpose: evaluate PCA/M1/M2/M2-shuffled on identical masks, same valid tissue-labelled test subset, and both global-entry and per-sample Pearson metrics.
- Smoke test: CPU PCA-only with 8 samples passed and wrote `results/unified_eval_smoke.json` plus `docs/2026-05-31-unified-eval-smoke.md`.
- Cache: `data/sample_maps_cache.json` avoids repeated slow parquet metadata scans.
- Next required result: full `results/unified_eval.json` and `docs/2026-05-31-unified-eval.md` from the user-run command.



### Full Unified Evaluation Completed
- Date: 2026-06-01
- User command: `CUDA_VISIBLE_DEVICES=0 python scripts/evaluate_unified.py`
- Outputs: `results/unified_eval.json`, `docs/2026-05-31-unified-eval.md`
- Random mask test set: PCA global/per-sample Pearson = 0.9185/0.9056; M1 = 0.7730/0.7316; M2 = 0.7754/0.7349; M2-shuffled = 0.7674/0.7257.
- Module mask test set: PCA = 0.7609/0.7487; M1 = 0.6444/0.5759; M2 = 0.6527/0.5837; M2-shuffled = 0.6469/0.5790.
- External random mask: PCA = 0.9026/0.8909; M1 = 0.7478/0.7088; M2 = 0.7462/0.7072; M2-shuffled = 0.7354/0.6959.
- Conclusion: unified evaluation confirms PCA is the strongest current baseline. M2 only slightly improves internal reconstruction over M1/shuffled and is slightly below M1 on external metrics. Promoter-pairing benefit remains weak/not established.

### Validation-Based Checkpointing Implemented
- Date: 2026-06-01
- Prior risk: M1/M2 selected best checkpoints on test module-block r.
- Decision: future M1/M2 training must use `split_train.txt` for training, `split_val.txt` for checkpoint selection, and `split_test.txt` only for final reporting.
- Implementation: `src.training.Trainer.run()` now accepts `(train_dl, val_dl, test_dl)`, saves best checkpoint by validation `mod_r`, reloads it, and writes final test metrics to `results/<run>_summary.json`.
- `cli.py --run-name` allows smoke tests without overwriting canonical `m1_best.pt`, `m2_best.pt`, or `m2-shuffled_best.pt`.
- `scripts/evaluate_unified.py` now uses `torch.load(..., weights_only=True)` when supported, removing the PyTorch FutureWarning path for current trusted checkpoints.
- Next required result: run `M2-val-smoke` for 5 epochs, then retrain canonical M1/M2/M2-shuffled under the validation protocol if the smoke test succeeds.

### Validation-Protocol M2 Smoke Test Completed
- Date: 2026-06-01
- Command: `CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 5 --run-name M2-val-smoke`
- Outputs: `results/m2-val-smoke_log.json`, `results/m2-val-smoke_best.pt`, `results/m2-val-smoke_summary.json`.
- Protocol check: `Selection split: val` printed; log entries include `selection_split: "val"`; summary includes `final_test`.
- Best validation checkpoint: epoch 5, validation random-mask r=0.5048, validation module-block r=0.3038, validation tissue macro-F1=0.3333, external r=0.3359.
- Final test from validation-best checkpoint: random-mask r=0.5336, module-block r=0.3857, tissue macro-F1=0.1724, external r=0.3321.
- Resource finding: on NVIDIA A100-SXM4-40GB, user-observed process allocation was about 32.4 GB VRAM. Future M2 canonical runs should be scheduled on >=40 GB GPUs or with reduced batch/model settings if using smaller cards.
- Decision: validation-based training path is runnable; proceed to promoter embedding gene-order audit before long canonical reruns.

### Promoter Embedding Gene-Order Audit Completed
- Date: 2026-06-01
- Command: `python scripts/audit_promoter_embedding_order.py`
- Output: `data/promoter_emb_genes.txt`.
- Result: audit passed; 5000 HVG genes were recorded and `data/promoter_emb.npy` shape is `(5000, 256)`.
- Remaining requirement: add loader-side alignment checking before canonical M2/M2-shuffled reruns.

### Promoter Embedding Loader Alignment Check Implemented
- Date: 2026-06-01
- Training and unified evaluation now check the current HVG order against `data/promoter_emb_genes.txt` before loading M2 promoter embeddings.
- If the order differs, the code raises a clear error with the first mismatched row.
- Verification passed with `data/promoter_emb.npy` shape `(5000, 256)`.
- Decision: canonical M1/M2/M2-shuffled validation-protocol reruns can start.

### Validation-Protocol Canonical M1/M2 Reruns Completed
- Date: 2026-06-01
- Commands: `CUDA_VISIBLE_DEVICES=3 python cli.py train m1 --epochs 80`; `CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80`.
- M1 validation-best checkpoint: epoch 70, val mod_r=0.6761; final test random-mask r=0.7266, module-block r=0.5661, tissue macro-F1=0.3596, external r=0.7071.
- M2 validation-best checkpoint: epoch 80, val mod_r=0.6658; final test random-mask r=0.7323, module-block r=0.5642, tissue macro-F1=0.3334, external r=0.7061.
- Early conclusion before shuffled rerun/unified eval: M2 does not clearly improve over M1 under validation-selected final test metrics; it is higher on random-mask r but lower on module-block r, tissue macro-F1, and external r.
- Next required result: validation-protocol M2-shuffled 80-epoch rerun, then unified evaluation regeneration.

### Validation-Protocol Canonical M2-shuffled Rerun Completed
- Date: 2026-06-02
- Command: `CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80 --shuffle-promoters`.
- Best validation checkpoint: epoch 80, val random-mask r=0.8032, val module-block r=0.6644, val tissue macro-F1=0.5094, external r=0.7023.
- Final test metrics: random-mask r=0.7308, module-block r=0.5651, tissue macro-F1=0.3512, external r=0.7047.
- Compared with validation-protocol true M2 final test metrics, M2-shuffled is slightly lower on random-mask r but slightly higher on module-block r and tissue macro-F1.
- Current decision: promoter-gene pairing benefit is not established; rerun unified evaluation before final Gate A interpretation.

### Unified Evaluation After Validation-Protocol Reruns Completed
- Date: 2026-06-02
- Command: `CUDA_VISIBLE_DEVICES=1 python scripts/evaluate_unified.py`.
- Outputs: `results/unified_eval.json`, `docs/2026-05-31-unified-eval.md`.
- PCA remains far stronger: random-mask mean per-sample Pearson 0.9056, module-mask 0.7487, external 0.8909.
- Deep models after validation-selected reruns: M1 random/module/external mean per-sample Pearson = 0.7267/0.5738/0.7045; M2 = 0.7325/0.5727/0.7035; M2-shuffled = 0.7310/0.5679/0.7019.
- True M2 only marginally exceeds M2-shuffled on random-mask and module-mask metrics, and M2-shuffled has higher tissue F1 in unified random-mask evaluation.
- M2 does not clearly exceed M1: M2 is slightly higher on random-mask r, but slightly lower on module-mask and external mean per-sample Pearson.
- Gate A decision: current M2 promoter branch is No-Go for promoter-gene pairing or promoter-modality success claims. Further work should be ablation/task redesign, not cross-species expansion or strong biological claims.

### M2 No-Gene-Embedding Ablation Implemented
- Date: 2026-06-02
- Purpose: test whether promoter embeddings have independent value when the trainable gene identity lookup is removed.
- Implementation: `SoyFormerM2(use_gene_embed=False)` replaces gene identity embeddings with zeros while retaining promoter embeddings, expression encoding, and the same fusion width.
- CLI: `--no-gene-embed` writes noncanonical run names `M2-nogene` or `M2-nogene-shuffled`, avoiding overwrite of canonical M2 results.
- Evaluation: `scripts/evaluate_unified.py --models m2-nogene m2-nogene-shuffled` can evaluate the ablation checkpoints after training.
- Verification: syntax and minimal forward-pass checks passed.

### M2 No-Gene-Embedding Ablation Result
- Date: 2026-06-02
- Command: `CUDA_VISIBLE_DEVICES=1 python cli.py train m2 --epochs 80 --no-gene-embed`.
- Best validation checkpoint: epoch 10, val random-mask r=0.0068, val module-block r=0.0069, val tissue macro-F1=0.3786, external r=-0.0033.
- Final test metrics: random-mask r=0.0117, module-block r=0.0137, tissue macro-F1=0.2457, external r=-0.0067.
- Interpretation: without trainable gene identity embeddings, promoter embeddings alone do not support expression reconstruction in the current architecture; reconstruction correlations collapse to near zero.
- Next required control: run `M2-nogene-shuffled` to complete the paired no-gene ablation.

### M2 No-Gene-Embedding Shuffled Control Result
- Date: 2026-06-02
- Command: `CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80 --no-gene-embed --shuffle-promoters`.
- Best validation checkpoint: epoch 40, val random-mask r=0.0040, val module-block r=0.0099, val tissue macro-F1=0.3114, external r=-0.0037.
- Final test metrics: random-mask r=0.0009, module-block r=-0.0044, tissue macro-F1=0.2888, external r=-0.0055.
- Paired interpretation: both true and shuffled no-gene M2 collapse to near-zero reconstruction. Current random-CNN promoter embeddings are not independently useful for expression reconstruction in this architecture.
- Decision: this strengthens the No-Go decision for the current promoter branch. Any future promoter work should use a pretrained/trainable DNA encoder and a task/objective designed to test promoter information directly.

