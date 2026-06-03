# Progress Log: 2026-05-29 Master Execution Plan

## Session: 2026-05-29

### Task
重新梳理 SoyFormer-U 项目计划, 建立后续执行和计划更新协议。

### Actions Taken
- 阅读项目说明、历史计划、当前代码结构、已有结果和 M2 失败日志。
- 新建主执行计划目录: `.planning/2026-05-29-master-execution-plan/`。
- 将当前阶段重定为 soybean-only Gate A/B/C, 跨物种工作延后到 Gate D。
- 明确后续每次任务结束必须更新计划文档。

### Current State
- M0 baseline complete.
- M1 expression-only complete but below PCA-200.
- M2 has no valid result yet; old online promoter CNN path failed with CUDA OOM.
- Current code should use precomputed `data/promoter_emb.npy`; next task is M2 smoke test.

### Plan Documents Updated
- `.planning/2026-05-29-master-execution-plan/task_plan.md`
- `.planning/2026-05-29-master-execution-plan/progress.md`
- `.planning/2026-05-29-master-execution-plan/findings.md`
- `.planning/.active_plan`
- `CLAUDE.md`

### Next Task
Run M2 smoke test:

```bash
python cli.py train m2 --epochs 5
```

If it succeeds, run full M2 and shuffled-promoter control.

## Session: 2026-05-30

### Task
Fix M2 smoke test CLI failure reported by user.

### Command That Failed
```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m2 --gpu 0 --epochs 5
```

### Failure
`AttributeError: 'Namespace' object has no attribute 'n_heads'`

### Root Cause
`cli.py` passed `args.n_heads`, `args.d_ff`, and `args.dropout` into `SoyFormerM2`, but argparse did not define those CLI fields.

### Actions Taken
- Added CLI arguments `--n_heads`, `--d_ff`, and `--dropout`, using defaults from `src.config.DEFAULTS`.
- Updated GPU handling so the preferred invocation is now shell-level `CUDA_VISIBLE_DEVICES=... python cli.py ...`; `--gpu` remains optional legacy compatibility.
- Updated CLI docstring examples to match the project GPU command convention.

### Verification
```bash
python3 -m py_compile cli.py
```

Result: passed.

### Next Command
```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m2 --epochs 5
```

## Session: 2026-05-30

### Task
Review completed M2 smoke test results.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m2 --epochs 5
```

### Outputs
- `results/m2_log.json`
- `results/m2_best.pt`

### Smoke Test Metrics
Best checkpoint: epoch 5

| Metric | Value |
|---|---:|
| loss | 0.3229 |
| random-mask r | 0.5302 |
| module-block r | 0.3424 |
| tissue macro-F1 | 0.1830 |
| external r | 0.3329 |

### Interpretation
The precomputed-promoter M2 path now runs end-to-end and saves outputs. These 5-epoch values are only a smoke-test result and should not be compared as final performance against M1 or PCA. Next step is full M2 training followed by shuffled-promoter negative control.

### Next Command
```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m2 --epochs 80
```

## Session: 2026-05-31

### Task
Review completed full M2 training results.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=1,2 python cli.py train m2 --epochs 80
```

### Outputs
- `results/m2_log.json`
- `results/m2_best.pt`

### Full M2 Metrics
Best checkpoint: epoch 80

| Metric | M2 | M1 epoch 80 | Delta |
|---|---:|---:|---:|
| random-mask r | 0.7348 | 0.7314 | +0.0034 |
| module-block r | 0.6036 | 0.5986 | +0.0049 |
| tissue macro-F1 | 0.3645 | 0.3613 | +0.0032 |
| external r | 0.7070 | 0.7057 | +0.0013 |

### Interpretation
Full M2 training completed successfully. M2 is slightly above M1 on all tracked metrics, but the absolute gains are small: +0.0034 random-mask r, +0.0049 module-block r, +0.0032 tissue macro-F1, +0.0013 external r. M2 remains far below PCA-200 on random-mask reconstruction. The next required step is shuffled-promoter negative control before interpreting any promoter benefit.

### Next Command
```bash
CUDA_VISIBLE_DEVICES=1,2 python cli.py train m2 --epochs 80 --shuffle-promoters
```

## Session: 2026-05-31

### Task
Record GPU command resource-reporting rule.

### User Requirement
Every GPU command must include required compute resources.

### Actions Taken
- Updated `CLAUDE.md` active execution rules.
- Updated active `task_plan.md` documentation protocol.

### Rule
When providing GPU commands, include GPU count and suggested GPU IDs, expected VRAM, CPU cores, RAM, expected runtime, and output files to check.

## Session: 2026-05-31

### Task
Review completed M2 shuffled-promoter negative control.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=1 python cli.py train m2 --epochs 80 --shuffle-promoters
```

### Resources Actually Used
- GPU: 1 GPU, physical GPU 1
- CPU/RAM: default dataloader and matrix loading path; no resource issue reported
- Outputs: `results/m2-shuffled_log.json`, `results/m2-shuffled_best.pt`

### Shuffled-Control Metrics
Best checkpoint by module-block r: epoch 60

| Metric | M2-shuffled | M2 | Delta M2 - shuffled |
|---|---:|---:|---:|
| random-mask r | 0.7264 | 0.7348 | +0.0084 |
| module-block r | 0.6030 | 0.6036 | +0.0006 |
| tissue macro-F1 | 0.3359 | 0.3645 | +0.0286 |
| external r | 0.6970 | 0.7070 | +0.0100 |

### Summary Artifacts Created
- `results/soybean_gate_a_summary.json`
- `docs/2026-05-31-soybean-gate-a-results.md`

### Interpretation
M2-shuffled is essentially tied with true M2 on module-block r, the current model-selection metric. This weakens the promoter-pairing claim. True M2 is higher on random-mask r, external r, and tissue F1, but the run is single-seed and the promoter embeddings come from a random local CNN. Do not claim robust promoter modality benefit yet.

### Next Step
Run diagnostics and/or additional seeds before any promoter claim. Recommended immediate next task is P2/P3: unified evaluator plus metric-consistency and leakage diagnostics.

### Reproducibility Follow-up
- Added `scripts/summarize_gate_a.py` to regenerate `results/soybean_gate_a_summary.json` and `docs/2026-05-31-soybean-gate-a-results.md` from existing JSON logs.
- Verified syntax with `python3 -m py_compile scripts/summarize_gate_a.py`.
- Verified execution with `python3 scripts/summarize_gate_a.py`.

## Session: 2026-05-31

### Task
User reran Gate A summary script on login node.

### Command Completed
```bash
python scripts/summarize_gate_a.py
```

### Resources Used
- GPU: none
- CPU: login-node CPU execution
- RAM: <8 GB expected
- Runtime: <5 min

### Outputs Regenerated
- `results/soybean_gate_a_summary.json`
- `docs/2026-05-31-soybean-gate-a-results.md`

### Result
The script completed successfully and rewrote both expected summary artifacts.

### Next Step
Proceed to P3 diagnostics: inspect metric consistency, leakage risk, and why M2-shuffled nearly matches true M2 on module-block r.

## Session: 2026-05-31

### Task
Run P3 Gate A diagnostics for metric consistency, metadata confounding, and promoter evidence.

### Command Completed
```bash
python3 scripts/diagnose_gate_a.py
```

### Resources Used
- GPU: none
- CPU: 1 process; parquet metadata scan was slower than expected
- RAM: <16 GB expected
- Runtime: longer than initial estimate because metadata scan read parquet sample metadata

### Outputs
- `results/gate_a_diagnostics.json`
- `docs/2026-05-31-gate-a-diagnostics.md`

### Key Findings
- PCA and M1/M2 imputation metrics are not directly comparable: PCA uses global entry-wise Pearson; M1/M2 use mean per-sample Pearson.
- M1/M2 currently select checkpoints using test module-block r; `split_val.txt` exists but is unused by `Trainer.run()`.
- BioProject/tissue confounding is strong: 87.2% of BioProjects with tissue metadata are single-tissue; median majority-tissue purity is 1.000.
- M2 and M2-shuffled are effectively tied on module-block r: delta +0.0006.
- `promoter_emb.npy` comes from a randomly initialized local CNN and should not be interpreted as a pretrained promoter representation.

### Next Step
Implement a unified evaluator that applies identical masks and sample sets to PCA/M1/M2/M2-shuffled, reports both global-entry and per-sample Pearson, and uses validation split for checkpoint selection in future training.

## Session: 2026-05-31

### Task
Write short next execution plan after Gate A diagnostics.

### Output
- `docs/2026-05-31-next-execution-plan.md`

### Summary
The next step is not another training run. First implement `scripts/evaluate_unified.py` to evaluate PCA/M1/M2/M2-shuffled under identical masks, sample sets, and metric aggregation rules. The planned command after implementation is `CUDA_VISIBLE_DEVICES=1 python scripts/evaluate_unified.py` with 1 GPU, <16 GB expected VRAM, 4-8 CPU cores, 32-64 GB RAM, and 10-30 min expected runtime.

## Session: 2026-05-31

### Task
Implement unified evaluator for Gate A.

### Files Added
- `scripts/evaluate_unified.py`

### Files Generated During Smoke Test
- `data/sample_maps_cache.json`
- `results/unified_eval_smoke.json`
- `docs/2026-05-31-unified-eval-smoke.md`

### Verification
```bash
python3 -m py_compile scripts/evaluate_unified.py
/home/user/zhangzhishuai/.local/share/mamba/envs/rna_only/bin/python scripts/evaluate_unified.py --device cpu --models pca --max-samples 8 --n-random-masks 1 --n-module-masks 1 --pca-components 20 --output-json results/unified_eval_smoke.json --output-md docs/2026-05-31-unified-eval-smoke.md
```

Result: passed. The smoke test wrote the expected JSON and Markdown files.

### Implementation Notes
- The evaluator reports both global entry-wise Pearson and mean per-sample Pearson.
- PCA/M1/M2/M2-shuffled use identical masks and the same valid tissue-labelled test subset.
- A sample-map cache was added at `data/sample_maps_cache.json`; the first run may be slow while generating it, later runs read the cache.

### Next Command For User
```bash
CUDA_VISIBLE_DEVICES=1 /home/user/zhangzhishuai/.local/share/mamba/envs/rna_only/bin/python scripts/evaluate_unified.py
```

Resources: 1 GPU, suggested GPU 1 or any free card; expected VRAM <16 GB, >=24 GB preferred; CPU 4-8 cores; RAM 32-64 GB; expected runtime 10-30 min now that sample-map cache exists; check `results/unified_eval.json` and `docs/2026-05-31-unified-eval.md`.



## Session: 2026-06-01

### Task
Review user-completed full unified Gate A evaluation.

### Command Completed By User
```bash
CUDA_VISIBLE_DEVICES=0 python scripts/evaluate_unified.py
```

### Resources Used
- GPU: 1 GPU, physical GPU 0
- CPU: 4-8 cores recommended
- RAM: 32-64 GB recommended
- Outputs: `results/unified_eval.json`, `docs/2026-05-31-unified-eval.md`

### Key Results
Unified evaluation uses identical masks, sample subsets, and reports both global entry-wise Pearson and mean per-sample Pearson.

Random mask test set:

| Method | Global Pearson | Mean per-sample Pearson | Tissue F1 |
|---|---:|---:|---:|
| PCA | 0.9185 | 0.9056 | - |
| M1 | 0.7730 | 0.7316 | 0.3609 |
| M2 | 0.7754 | 0.7349 | 0.3588 |
| M2-shuffled | 0.7674 | 0.7257 | 0.3524 |

Module mask test set:

| Method | Global Pearson | Mean per-sample Pearson |
|---|---:|---:|
| PCA | 0.7609 | 0.7487 |
| M1 | 0.6444 | 0.5759 |
| M2 | 0.6527 | 0.5837 |
| M2-shuffled | 0.6469 | 0.5790 |

External random mask:

| Method | Global Pearson | Mean per-sample Pearson |
|---|---:|---:|
| PCA | 0.9026 | 0.8909 |
| M1 | 0.7478 | 0.7088 |
| M2 | 0.7462 | 0.7072 |
| M2-shuffled | 0.7354 | 0.6959 |

### Interpretation
- P2 full unified evaluation is complete.
- PCA remains much stronger than M1/M2 under the unified protocol.
- M2 is only slightly above M1 and M2-shuffled on internal reconstruction metrics.
- M2 is slightly below M1 on external unified random-mask metrics.
- Do not continue current promoter-modality claim without fixing training protocol and rerunning validation-selected models.

### Next Step
Implement validation-based checkpointing so `split_train.txt` trains, `split_val.txt` selects checkpoints, and `split_test.txt` is final-report only.

## Session: 2026-06-01

### Task
Implement validation-based checkpointing and clean up unified evaluator warning.

### Files Updated
- `src/config.py`: added `SPLIT_VAL`.
- `src/data.py`: `load_splits()` now returns train/val/test; HVG/scaling and tissue labels support validation split.
- `src/training.py`: `Trainer.run()` now selects checkpoints on validation `mod_r` when given `(train_dl, val_dl, test_dl)`, then reports final test metrics from the validation-best checkpoint.
- `cli.py`: training path now uses train/val/test and supports `--run-name` for smoke tests and ablations.
- `scripts/train_m1.py`, `scripts/train_m2.py`: updated to the validation protocol.
- `scripts/evaluate_unified.py`: updated for the new split API and uses `torch.load(..., weights_only=True)` when available.
- `docs/2026-06-01-validation-protocol.md`: documented the protocol.

### Verification
```bash
/home/user/zhangzhishuai/.local/share/mamba/envs/rna_only/bin/python -m py_compile cli.py src/config.py src/data.py src/training.py scripts/evaluate_unified.py scripts/train_m1.py scripts/train_m2.py scripts/summarize_gate_a.py scripts/diagnose_gate_a.py
/home/user/zhangzhishuai/.local/share/mamba/envs/rna_only/bin/python cli.py train m2 --help
/home/user/zhangzhishuai/.local/share/mamba/envs/rna_only/bin/python -c "from src.data import load_splits; splits=load_splits(); print(len(splits), [len(x) for x in splits])"
```

Results:
- Python compile passed in the `rna_only` environment.
- CLI help shows `--run-name`.
- `load_splits()` returns 3 split sets with BioProject counts `[272, 62, 69]` for train/val/test.

### Next Command For User
```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m2 --epochs 5 --run-name M2-val-smoke
```

Resources: 1 GPU, suggested GPU 0 or any free single card; expected VRAM 10-16 GB, >=24 GB preferred; CPU 4-8 cores; RAM 32-64 GB; expected runtime 5-15 min; check `results/m2-val-smoke_log.json`, `results/m2-val-smoke_best.pt`, and `results/m2-val-smoke_summary.json`.

## Session: 2026-06-01

### Task
Review completed validation-protocol M2 smoke test.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 5 --run-name M2-val-smoke
```

### Resources Actually Used
- GPU: 1 GPU, physical GPU 3, NVIDIA A100-SXM4-40GB.
- VRAM: user-observed process allocation about 32,378 MB; total GPU allocation 35,396 / 40,960 MB with another user process present.
- CPU/RAM: default dataloader and matrix loading path; no resource issue reported.
- Runtime: completed successfully within smoke-test scale.

### Outputs
- `results/m2-val-smoke_log.json`
- `results/m2-val-smoke_best.pt`
- `results/m2-val-smoke_summary.json`

### Validation-Selection Metrics
Best checkpoint by validation module-block r: epoch 5

| Metric | Value |
|---|---:|
| validation random-mask r | 0.5048 |
| validation module-block r | 0.3038 |
| validation tissue macro-F1 | 0.3333 |
| external r during validation eval | 0.3359 |

### Final Test Metrics From Best Validation Checkpoint

| Metric | Value |
|---|---:|
| test random-mask r | 0.5336 |
| test module-block r | 0.3857 |
| test tissue macro-F1 | 0.1724 |
| external r | 0.3321 |

### Result
The validation protocol smoke test succeeded. Console output printed `Selection split: val`, all log entries include `selection_split: "val"`, and the summary JSON contains `final_test`. The noncanonical run name avoided overwriting canonical `results/m2_best.pt`.

### Next Step
Before launching 80-epoch canonical reruns, add or verify a saved promoter embedding gene-order artifact (`data/promoter_emb_genes.txt`) and loader alignment check. Then rerun M1, M2, and M2-shuffled under validation-based checkpointing.

## Session: 2026-06-01

### Task
Create reusable promoter embedding gene-order audit script.

### Files Added
- `scripts/audit_promoter_embedding_order.py`

### Purpose
The script recomputes the current train-split HVG order, checks that `data/promoter_emb.npy` has the same number of rows, and writes `data/promoter_emb_genes.txt`.

### Verification
```bash
python3 -m py_compile scripts/audit_promoter_embedding_order.py
```

Result: passed.

### Next Command
```bash
python scripts/audit_promoter_embedding_order.py
```

Resources: GPU none; CPU 1-4 cores; RAM 16-32 GB recommended because the unified expression matrix is loaded; expected runtime a few minutes; output to check `data/promoter_emb_genes.txt`.

## Session: 2026-06-01

### Task
Run promoter embedding gene-order audit.

### Command Completed
```bash
python scripts/audit_promoter_embedding_order.py
```

### Resources Used
- GPU: none.
- CPU/RAM: login-node Python execution; loaded unified matrix and metadata.

### Output
- `data/promoter_emb_genes.txt`

### Result
Audit passed. The script wrote 5000 HVG genes and confirmed `data/promoter_emb.npy` shape is `(5000, 256)`.

### Next Step
Add loader-side alignment checking so M2 refuses to run if the current HVG order differs from `data/promoter_emb_genes.txt`. After that, start canonical validation-protocol 80-epoch reruns.

## Session: 2026-06-01

### Task
Add loader-side promoter embedding gene-order alignment checks.

### Files Updated
- `src/config.py`: added `PROMOTER_EMB_GENES` path.
- `src/data.py`: `load_promoter_embeddings()` now optionally validates current HVG order against `data/promoter_emb_genes.txt` and checks row count.
- `cli.py`: M2 training now passes current `hvg_genes` into `load_promoter_embeddings()`.
- `scripts/evaluate_unified.py`: M2/M2-shuffled evaluation now validates promoter embedding gene order before inference.

### Verification
- `python3 -m py_compile src/config.py src/data.py cli.py scripts/evaluate_unified.py scripts/audit_promoter_embedding_order.py`
- rna_only Python loader check returned `(5000, 256)`.

Result: syntax check passed; loader alignment check passed.

### Next Step
Start canonical validation-protocol reruns, beginning with M1 80 epochs.

## Session: 2026-06-01

### Task
Review validation-protocol canonical M1 and M2 80-epoch reruns.

### Commands Completed
```bash
CUDA_VISIBLE_DEVICES=3 python cli.py train m1 --epochs 80
CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80
```

### Resources Used
- GPU: 1 GPU, physical GPU 3, used sequentially for M1 then M2.
- M2 should continue to be scheduled on A100 40GB-class GPUs based on prior smoke-test VRAM observation.

### Outputs
- `results/m1_log.json`, `results/m1_best.pt`, `results/m1_summary.json`
- `results/m2_log.json`, `results/m2_best.pt`, `results/m2_summary.json`

### M1 Result
Best checkpoint by validation module-block r: epoch 70.

| Metric | Validation | Final test |
|---|---:|---:|
| random-mask r | 0.8016 | 0.7266 |
| module-block r | 0.6761 | 0.5661 |
| tissue macro-F1 | 0.5196 | 0.3596 |
| external r | 0.7024 | 0.7071 |

### M2 Result
Best checkpoint by validation module-block r: epoch 80.

| Metric | Validation | Final test |
|---|---:|---:|
| random-mask r | 0.8052 | 0.7323 |
| module-block r | 0.6658 | 0.5642 |
| tissue macro-F1 | 0.5197 | 0.3334 |
| external r | 0.7041 | 0.7061 |

### Interpretation
Both canonical runs completed under validation-based checkpointing and wrote final test summaries. On final test metrics, M2 improves random-mask r over M1 by about +0.0057, but is slightly below M1 on module-block r, tissue macro-F1, and external r. Do not interpret promoter benefit until M2-shuffled is rerun and unified evaluation is regenerated.

### Next Step
Run validation-protocol canonical M2-shuffled 80 epochs.

## Session: 2026-06-02

### Task
Review validation-protocol canonical M2-shuffled 80-epoch rerun.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80 --shuffle-promoters
```

### Resources Used
- GPU: 1 GPU, physical GPU 3, used for M2-shuffled.
- M2-family runs should continue to be scheduled on A100 40GB-class GPUs based on prior smoke-test VRAM observation.

### Outputs
- `results/m2-shuffled_log.json`
- `results/m2-shuffled_best.pt`
- `results/m2-shuffled_summary.json`

### Result
Best checkpoint by validation module-block r: epoch 80.

| Metric | Validation | Final test |
|---|---:|---:|
| random-mask r | 0.8032 | 0.7308 |
| module-block r | 0.6644 | 0.5651 |
| tissue macro-F1 | 0.5094 | 0.3512 |
| external r | 0.7023 | 0.7047 |

### Interpretation
The validation-protocol shuffled control completed successfully. Compared with validation-protocol true M2 final test metrics, M2-shuffled is slightly lower on random-mask r but slightly higher on module-block r and tissue macro-F1. Promoter-gene pairing benefit is therefore not supported by these validation-selected final test summaries.

### Next Step
Rerun `scripts/evaluate_unified.py` so PCA, M1, M2, and M2-shuffled are compared on identical masks and sample subsets using the new validation-selected checkpoints.

## Session: 2026-06-02

### Task
Review unified evaluation after validation-protocol canonical reruns.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=1 python scripts/evaluate_unified.py
```

### Outputs
- `results/unified_eval.json`
- `docs/2026-05-31-unified-eval.md`

### Key Results
Random mask test set, mean per-sample Pearson:

| Method | Mean per-sample Pearson | Global Pearson | Tissue F1 |
|---|---:|---:|---:|
| PCA | 0.9056 | 0.9185 | - |
| M1 | 0.7267 | 0.7690 | 0.3313 |
| M2 | 0.7325 | 0.7735 | 0.3378 |
| M2-shuffled | 0.7310 | 0.7723 | 0.3445 |

Module mask test set, mean per-sample Pearson:

| Method | Mean per-sample Pearson | Global Pearson |
|---|---:|---:|
| PCA | 0.7487 | 0.7609 |
| M1 | 0.5738 | 0.6475 |
| M2 | 0.5727 | 0.6460 |
| M2-shuffled | 0.5679 | 0.6426 |

External random mask, mean per-sample Pearson:

| Method | Mean per-sample Pearson | Global Pearson |
|---|---:|---:|
| PCA | 0.8909 | 0.9026 |
| M1 | 0.7045 | 0.7434 |
| M2 | 0.7035 | 0.7436 |
| M2-shuffled | 0.7019 | 0.7419 |

### Interpretation
PCA remains the strongest method by a large margin. M2 is only marginally higher than M1 and M2-shuffled on random-mask reconstruction. On module-mask reconstruction, M1 is slightly above M2, and M2-shuffled remains close. External metrics are nearly tied for M1/M2/M2-shuffled, with M1 slightly above M2 in mean per-sample Pearson. Current Gate A does not support a robust promoter-gene pairing claim.

### Next Step
Close Gate A as a No-Go for the current promoter branch, then decide whether to run targeted ablations or move to a different task/model design before cross-species expansion.

## Session: 2026-06-02

### Task
Implement M2 no-gene-embedding ablation.

### Files Updated
- `src/model/m2.py`: added `use_gene_embed`; when disabled, trainable gene identity embeddings are replaced by zero tensors.
- `cli.py`: added `--no-gene-embed`; default output names are `M2-nogene` and `M2-nogene-shuffled`.
- `scripts/train_m2.py`: added matching `--no-gene-embed` support for the legacy training script.
- `scripts/evaluate_unified.py`: added support for `m2-nogene` and `m2-nogene-shuffled` checkpoints.

### Verification
```bash
python3 -m py_compile src/model/m2.py cli.py scripts/train_m2.py scripts/evaluate_unified.py
```
Result: passed.

Minimal forward-pass check with `use_gene_embed=False` returned output shapes `torch.Size([2, 10])` and `torch.Size([2, 3])`.

### Next Commands
```bash
CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80 --no-gene-embed
CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80 --no-gene-embed --shuffle-promoters
```

Resources: 1 A100 40GB-class GPU recommended for each M2-family run; run sequentially unless using distinct GPUs; CPU 4-8 cores; RAM 32-64 GB; outputs are `results/m2-nogene_*` and `results/m2-nogene-shuffled_*`.

## Session: 2026-06-02

### Task
Review M2 no-gene-embedding ablation result.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=1 python cli.py train m2 --epochs 80 --no-gene-embed
```

### Outputs
- `results/m2-nogene_log.json`
- `results/m2-nogene_best.pt`
- `results/m2-nogene_summary.json`

### Result
Best checkpoint by validation module-block r: epoch 10.

| Metric | Validation | Final test |
|---|---:|---:|
| random-mask r | 0.0068 | 0.0117 |
| module-block r | 0.0069 | 0.0137 |
| tissue macro-F1 | 0.3786 | 0.2457 |
| external r | -0.0033 | -0.0067 |

### Interpretation
Removing trainable gene identity embeddings collapses reconstruction performance to near zero correlation, despite retaining promoter embeddings and expression scalar encoding. This strongly suggests current M2 reconstruction performance depends on trainable gene identity embeddings and/or expression-context learning, not on promoter embeddings alone. The paired shuffled no-gene run is still needed for a complete control.

### Next Step
Run `M2-nogene-shuffled` with validation protocol.

## Session: 2026-06-02

### Task
Review M2 no-gene-embedding shuffled control result.

### Command Completed
```bash
CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 80 --no-gene-embed --shuffle-promoters
```

### Outputs
- `results/m2-nogene-shuffled_log.json`
- `results/m2-nogene-shuffled_best.pt`
- `results/m2-nogene-shuffled_summary.json`

### Result
Best checkpoint by validation module-block r: epoch 40.

| Metric | Validation | Final test |
|---|---:|---:|
| random-mask r | 0.0040 | 0.0009 |
| module-block r | 0.0099 | -0.0044 |
| tissue macro-F1 | 0.3114 | 0.2888 |
| external r | -0.0037 | -0.0055 |

### Interpretation
The shuffled no-gene control also collapses to near-zero reconstruction correlation. Together with true M2-nogene, this indicates the current randomly initialized CNN promoter embeddings do not provide usable independent expression-reconstruction signal when trainable gene identity embeddings are removed.

### Next Step
Run a unified evaluation including `m2-nogene` and `m2-nogene-shuffled` to produce a single comparable table for the ablation.

