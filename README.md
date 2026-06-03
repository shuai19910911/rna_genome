# rna_genome

这个仓库用于同步和查看 RNA / 基因组建模项目的阶段性文档，不用于保存原始数据、大矩阵、日志或模型权重。

## 1. 项目一：SoyFormer-U

SoyFormer-U 当前定位为“大豆内部可行性验证”项目，目标是先把 soybean-only 的实验闭环做扎实，再决定是否进入跨物种建模。

当前不是最终跨物种模型阶段，而是以下几个问题的验证阶段：

1. 数据是否可靠。
2. PCA / AE / kNN 等强基线是否已经充分建立。
3. M1 expression-only 深度模型是否有价值。
4. M2 promoter-expression 多模态模型是否真的利用了 promoter 信息。
5. shuffled-promoter negative control 是否能排除伪增益。

### 当前数据状态

- 统一表达矩阵：47,046 个 shared genes × 5,741 个 samples。
- 样本来源：SoyAtlas 5,481 个样本 + rnadata external 260 个样本。
- 切分方式：BioProject-grouped train / val / test。
- 当前最强经典基线：PCA-200。
- PCA-200 random-mask r = 0.9181，external r = 0.9029。

### 当前模型状态

- M1 expression-only 模型已完成训练和评估。
- M2 promoter-expression 模型已完成训练和 shuffled-promoter 对照。
- M2 目前略高于 M1，但与 shuffled-promoter 对照非常接近。
- 因此目前不能声称 promoter-gene pairing 已经带来稳定收益。
- 当前 promoter embedding 来自项目内轻量 CNN，不是 AgroNT / HyenaDNA 等预训练 DNA foundation model。

### 当前结论

SoyFormer-U 目前应谨慎表述为：

> 在当前 soybean-only 数据上，M2 多模态结构可以运行并略高于 M1，但 promoter 模态贡献仍未被强 negative control 充分支持。

不应表述为：

> promoter 信息显著提升表达预测，或模型超过经典 PCA 强基线。

## 2. 项目二：SoyGT-Former BiB

SoyGT-Former 是面向 Briefings in Bioinformatics 的泛基因组表达预测项目。

数据规模：

- 26 个大豆 accession。
- 9 个 tissue。
- 58,427 个 gene。

核心任务：

- 使用 PAV / CNV / promoter / variant / accession / tissue 信息预测表达量。
- 重点验证跨组织、跨材料、跨 accession+tissue 条件下的泛化能力。

### 当前已验证结果

XGBoost baseline：

- LOA Pearson = 0.786。
- LOT mean Pearson = 0.782。

SoyGT-Former：

- LOT 9-fold 已完成。
- LOT 必须使用 `--target-mode raw`。
- LOT mean Pearson = 0.8261。
- LOT SD = 0.0438。
- fold Pearson 范围：0.7567 到 0.8773。

### 重要技术规则

LOT 是 leave-one-tissue-out，测试 tissue 在训练集中完全不可见，因此不能使用 residual target。

错误方式：

```text
--target-mode residual
```

正确方式：

```text
--target-mode raw
```

如果 LOT 使用 residual，会因为 held-out tissue 的 tissue_mean 不存在，导致 Pearson 接近 0。

## 3. 仓库内容说明

本仓库主要保存以下内容：

- `README.md`：项目总览。
- `MODEL_STRUCTURE.md`：模型结构解析。
- `GITHUB_SYNC.md`：GitHub 同步规则。
- `docs/`：阶段报告和实验总结。
- `.planning/`：计划、进展记录和发现。
- `CLAUDE.md`：项目执行规则。

## 4. 不上传的内容

本仓库不保存以下内容：

- 原始 RNA-seq 数据。
- FASTQ / FASTA / GTF / GFF 文件。
- 大型表达矩阵。
- Salmon / SLURM 日志。
- `.npy` / `.npz` / `.pt` / `.pth` / `.ckpt` 文件。
- 模型 checkpoint。
- VPN、token、密钥、订阅链接等隐私信息。

## 5. 后续同步规则

以后每完成一个小阶段，需要同步更新本仓库：

1. 如果阶段结论变化，更新 `README.md`。
2. 如果模型结构或训练逻辑变化，更新 `MODEL_STRUCTURE.md`。
3. 如果产生新指标、新 bug 或新决策，更新 `.planning/.../findings.md`。
4. 如果完成新任务，更新 `.planning/.../progress.md`。
5. 如果下一步计划变化，更新 `.planning/.../task_plan.md`。
6. 如果形成阶段报告，放入 `docs/`。

这个仓库的用途是方便查看和追踪研究过程，而不是作为完整数据仓库。
