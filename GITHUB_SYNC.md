# GitHub Sync Policy

Remote repository:

```text
git@github.com:shuai19910911/rna_genome.git
```

## What To Sync

After each small milestone, update and push:

1. `README.md` if the project-level summary changed.
2. `MODEL_STRUCTURE.md` if model architecture, inputs, outputs, or evaluation rules changed.
3. `.planning/2026-05-29-master-execution-plan/progress.md` after each task.
4. `.planning/2026-05-29-master-execution-plan/findings.md` when new metrics, bugs, limitations, or decisions appear.
5. `.planning/2026-05-29-master-execution-plan/task_plan.md` when phase status or next-step priorities change.
6. `docs/*.md` milestone reports.

## What Not To Sync

Do not commit:

- raw data directories
- FASTQ/FASTA/GTF files
- large `.tsv`, `.npy`, `.npz`, `.pt`, `.pth`, `.ckpt` outputs
- `logs/`
- `results/`
- temporary SLURM files
- private tokens, VPN config, subscription URLs, or credentials

## Standard Commit Pattern

```bash
git add README.md MODEL_STRUCTURE.md GITHUB_SYNC.md CLAUDE.md docs .planning && git commit -m "docs: update project milestone" && git push
```

Use a more specific commit message when possible, for example:

```bash
git commit -m "docs: add LOT results and next LOA plan"
```
