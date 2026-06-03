# rna_genome

This repository tracks the lightweight documentation state for the RNA/genome modeling projects under active development.

## Projects

### SoyFormer-U

SoyFormer-U is a soybean-only feasibility pipeline for promoter-expression representation learning. The current goal is not yet cross-species generalization; it is to close the soybean Gate A/B/C loop with strong baselines, M1/M2 comparisons, negative controls, and reproducible reports.

Current data state:

- Unified matrix: 47,046 shared genes x 5,741 samples.
- Sample sources: SoyAtlas 5,481 samples + rnadata external 260 samples.
- Split strategy: BioProject-grouped train/val/test split.
- Current strongest classical baseline: PCA-200, random-mask r = 0.9181, external r = 0.9029.
- Deep models so far remain below PCA-200 on random-mask reconstruction.

Current modeling state:

- M1 expression-only model completed.
- M2 promoter-expression model completed with shuffled-promoter negative control.
- M2 is slightly above M1 but close to shuffled-promoter control, so promoter superiority is not yet supported.
- Current promoter embeddings are from a lightweight local CNN, not a pretrained DNA foundation model.

### SoyGT-Former BiB

SoyGT-Former is the Briefings in Bioinformatics-oriented pan-genome expression prediction pipeline using 26 soybean accessions x 9 tissues x 58,427 genes.

Current validated results:

- XGBoost baseline: LOA Pearson = 0.786, LOT mean Pearson = 0.782.
- SoyGT-Former LOT 9-fold with raw target: mean Pearson = 0.8261, SD = 0.0438.
- LOT must use `--target-mode raw`; residual target is invalid for held-out tissues.

## Documentation In This Repository

- `docs/` contains readable project reports and milestone summaries.
- `.planning/` contains task plans, progress logs, and findings.
- `MODEL_STRUCTURE.md` explains the current model architectures and known limitations.
- `GITHUB_SYNC.md` records how this repository should be updated after each small milestone.

## Repository Policy

This repository is for documentation and lightweight source snapshots only. It should not contain raw data matrices, FASTQ/FASTA files, large model checkpoints, SLURM logs, or generated result directories.
