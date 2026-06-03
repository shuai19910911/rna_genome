# Validation-Based Training Protocol

**Date**: 2026-06-01

## Purpose

Future M1/M2 training must not select checkpoints on `split_test.txt`. The test split is reserved for final reporting and unified evaluation.

## Protocol

1. `split_train.txt`: fit HVG selection, scaler, model weights, and module-mask clustering.
2. `split_val.txt`: select the best checkpoint by validation module-block reconstruction `mod_r`.
3. `split_test.txt`: evaluate once after selecting the validation-best checkpoint, then use `scripts/evaluate_unified.py` for canonical cross-method reporting.
4. External rnadata remains a held-out external check; it must not be used as a checkpoint-selection criterion.

## Implementation

- `src.data.load_splits()` now returns train, validation, and test BioProject sets.
- `src.training.Trainer.run()` accepts `(train_dl, val_dl, test_dl)` and saves the best checkpoint by validation `mod_r`.
- Training logs include `selection_split: "val"` for validation-selected runs.
- A final test summary is written to `results/<run>_summary.json` after loading the validation-best checkpoint.
- `cli.py` has `--run-name` for smoke tests and ablations without overwriting the canonical checkpoints.

## Recommended Run Order

Run a smoke test first with a noncanonical run name. If it succeeds, rerun M1, M2, and M2-shuffled for 80 epochs under this protocol, then rerun `scripts/evaluate_unified.py`.
