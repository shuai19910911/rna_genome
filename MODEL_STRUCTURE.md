# Model Structure Notes

## SoyFormer-U

SoyFormer-U currently contains two main neural models plus a lightweight DNA encoder.

### M1: Expression-only Performer

Source: `src/model/m1.py`

Input:

- `x`: expression matrix batch with shape `(B, G)`.

Main components:

1. Gene embedding: `nn.Embedding(n_genes, d_model)`.
2. REE expression encoder: maps scalar expression values into `d_model` features.
3. Expression projection: concatenates gene embedding and REE output, then projects back to `d_model`.
4. CLS token: prepended to gene tokens for tissue classification.
5. Performer blocks: sequence model over gene tokens.
6. MEM head: predicts masked expression values per gene.
7. Tissue head: predicts tissue class from CLS token.

Output:

- reconstructed expression vector `(B, G)`.
- tissue logits `(B, n_tissues)`.

Interpretation:

M1 is an expression-only reconstruction and tissue-classification baseline. It does not use promoter or sequence features.

### M2: Promoter + expression multimodal model

Source: `src/model/m2.py`

Inputs:

- `x`: expression matrix batch `(B, G)`.
- `prom_emb`: precomputed promoter embeddings `(B or 1, G, d_model)` depending on data loader path.

Main components:

1. Promoter projection: `Linear(d_model, d_model) + GELU`.
2. Optional gene embedding: `nn.Embedding(n_genes, d_model)`; disabled in no-gene ablation.
3. REE expression encoder.
4. Fusion projection: concatenates promoter, gene, and expression representations, then projects to `d_model`.
5. CLS token.
6. Performer blocks.
7. MEM head.
8. Tissue head.

Output:

- reconstructed expression vector `(B, G)`.
- tissue logits `(B, n_tissues)`.

Known limitation:

The current promoter embedding source is a lightweight local CNN, not a pretrained DNA foundation model. M2 has only small gains over M1 and is close to the shuffled-promoter negative control, so current evidence does not support a strong promoter-pairing claim.

### DNAEncoder

Source: `src/model/dna_encoder.py`

Input:

- tokenized DNA promoter sequences with shape `(B, G, L)`.

Architecture:

1. nucleotide embedding: 4 bases -> 64 dimensions.
2. 3-layer 1D CNN with GELU and dropout.
3. adaptive max pooling to one vector per promoter.
4. reshape to `(B, G, d_model)`.

Current role:

Used to generate promoter embeddings, but not currently a pretrained genomic language model.

## SoyGT-Former BiB

Source project: `myhermes/rna_seq_model/rnamodel/soygt_former`

Core architecture:

1. promoter/genome branch: k-mer or promoter feature encoder.
2. variant branch: PAV/CNV/gene-family/variant-count feature encoder.
3. context branch: accession and tissue embedding.
4. fusion transformer: integrates genome, variant, and context tokens.
5. tissue FiLM conditioning.
6. expression regression head.
7. group classification auxiliary head.
8. optional masked k-mer and masked PAV self-supervised heads.

Important evaluation rule:

- LOT (leave-one-tissue-out) must use `--target-mode raw` because held-out tissue means are unavailable in training.
- LOA and LOAxT can be evaluated separately under residual/raw logic depending on the claim being tested.

Current validated LOT result:

- 9-fold LOT mean Pearson = 0.8261.
- Fold range = 0.7567 to 0.8773.
- This is above XGBoost LOT mean Pearson = 0.782.
