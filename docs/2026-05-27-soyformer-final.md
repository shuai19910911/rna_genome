# SoyFormer-U: 跨物种启动子-表达多模态表征学习 — 最终执行方案

**创建日期**: 2026-05-27
**状态**: 方案冻结, 进入 Gate 0
**定位**: 跨物种植物 bulk RNA-seq 表征模型, 以 promoter-expression 多模态为核心, 预注册 leave-one-species-out benchmark
**投稿目标**: Genome Biology (primary) / Nature Communications (stretch)

---

## 0. 方案演进摘要

| 版本 | 日期 | 核心变化 |
|---|---|---|
| v1 (初版) | 05-26 | 6 大创新, 单物种大豆, "foundation model" |
| v2 (修订版) | 05-26 | 收敛为 2 核心 + Gate 路线, 降级 FM 叙事 |
| v3 (跨物种) | 05-26 | 4 物种 + orthogroup 统一词汇 |
| v4 (NC80) | 05-27 | 6 物种 + 多模态 (promoter) + GW-OT + 因果 + 6 支柱 |
| **v5 (最终)** | **05-27** | **收敛版: 4-6 物种 + promoter-expression + LOSO benchmark + 1 生物 case study; GW-OT 降为分析, 因果降级, 去掉 80% 叙事** |

---

## 1. 核心方案 (30 秒版)

> **A cross-species promoter-expression representation model for plant transcriptomes, evaluated under preregistered leave-one-species-out benchmarks with strong classical and deep baselines.**

**一句话**: 用 orthogroup 统一基因词汇, 联合 promoter DNA 序列和 bulk expression 做跨物种表征学习, 在最严格的 leave-one-species-out 评估下证明泛化能力。

**核心贡献** (3 个):
1. **Orthogroup-aware cross-species expression benchmark**: 首个跨多个植物科的预注册 leave-one-species-out 转录组评估框架
2. **Promoter-expression multi-modal pretraining**: 证明启动子序列信息能稳定提升跨物种表达表征的泛化能力
3. **Plant biology discovery**: 通过 C3/C4 光合相关模块跨物种比较, 发现保守与分化的表达程序

**不做的事情** (明确排除):
- 端到端 GW-OT 训练 loss
- NOTEARS/DAG-GNN 因果主张
- Species GRL 作为默认训练目标
- "80% NC 中稿率" 或 "不可拒绝" 叙事
- Web portal 在初稿前开发

---

## 2. 数据环境

### 2.1 物种矩阵

| # | 物种 | 科 | 目标样本 | 基因组 | 优先级 |
|---|---|---|---|---|---|
| 1 | **Glycine max** (大豆) | Fabaceae | **5,481** ✓ | GCF_000004515.6 ✓ | 必选 |
| 2 | **Arabidopsis thaliana** (拟南芥) | Brassicaceae | **~3,000-4,000** | TAIR10 | 必选 |
| 3 | **Oryza sativa** (水稻) | Poaceae | **~1,000-2,000** | IRGSP-1.0 | 必选 |
| 4 | **Zea mays** (玉米) | Poaceae | **~1,000-2,000** | B73_v5 | 推荐 |
| 5 | **Solanum lycopersicum** (番茄) | Solanaceae | **~800-1,500** | SL4.0 | 推荐 |
| 6 | **Sorghum bicolor** (高粱) | Poaceae | **~500-1,000** | BTx623_v3 | 可选 |

**最小可行配置 (若数据获取困难)**: soybean + Arabidopsis + rice (3 物种, 双子叶 2 科 + 单子叶 1 科)

### 2.2 基因桥接: Orthogroup 统一词汇

```
OrthoFinder on 6 species proteomes
  → ~18,000-25,000 total OGs
  → ~5,000-8,000 core OGs (all-6-shared) → 主跨物种 benchmark
  → ~10,000-12,000 extended OGs (≥4-shared) → 预训练增强 (使用 missing mask)
  → Species-specific genes → fine-tuning/case study only
```

### 2.3 启动子序列

每个基因提取 2 kb upstream promoter (bedtools getfasta), 跨物种统一。备选 1 kb/500 bp 通过 sensitivity analysis 确定。

### 2.4 数据目录 (只读)

| 数据集 | 路径 |
|---|---|
| SoyAtlas v2 | `/home/user/zhangzhishuai/myhermes/rna_only/` |
| rnadata (external validation) | `/home/user/zhangzhishuai/myhermes/rna_seq_model/rnadata/` |
| Gmax genome + annotation | `/home/user/zhangzhishuai/myhermes/rna_seq_model/rnadata/00.ref/` |

---

## 3. 模型架构: SoyFormer-M2

```
┌──────────────────────────────────────────────────────────┐
│                      INPUT                                │
│  Per gene (in ~8K core OGs):                              │
│    - Expression: log1p(TPM) vector across samples         │
│    - Promoter: 2 kb DNA sequence                          │
│    - OG identity: shared lookup embedding                 │
│    - Species identity: learnable species embedding        │
└────────────┬─────────────────────────────────────────────┘
             │
┌────────────▼─────────────────────────────────────────────┐
│  MODALITY ENCODERS                                         │
│                                                           │
│  Expression: REE(log1p(TPM)) → d_expr (per-species μ,σ)   │
│  Promoter:   Frozen DNA encoder (AgroNT/HyenaDNA)         │
│              + trainable projection → d_seq               │
│                                                           │
│  Joint gene embedding:                                    │
│    g_i = OG_lookup[og_id]                                 │
│        + REE(log1p(tpm_i))                                │
│        + Proj(DNA_encoder(promoter_i))                    │
│        + Species_embed[s]                                 │
└────────────┬─────────────────────────────────────────────┘
             │
┌────────────▼─────────────────────────────────────────────┐
│  PERFORMER ENCODER × 8-12 layers                          │
│  - Co-expression graph diffusion kernel (train-set only,  │
│    post tissue/BioProject regression)                     │
│  - Species LoRA adapters (r=8, lightweight)               │
│  - FAVOR+ linear attention                                │
│  - NO MoE, NO GW-OT loss, NO GRL (in M1-M3)              │
└────────────┬─────────────────────────────────────────────┘
             │
┌────────────▼─────────────────────────────────────────────┐
│  TRAINING OBJECTIVES                                       │
│                                                           │
│  L = L_MEM(Huber, masked expression reconstruction)       │
│    + λ1 × L_contrastive(seq↔expr, per-gene cross-modal)   │
│    + λ2 × L_pathway(sparse Plant Reactome consistency)     │
│                                                           │
│  NO: L_GW, L_GRL, L_causal (excluded from M1-M3)          │
└────────────┬─────────────────────────────────────────────┘
             │
┌────────────▼─────────────────────────────────────────────┐
│  READOUT HEADS                                             │
│  - Expression reconstruction (per OG)                     │
│  - Cross-modal retrieval (promoter ↔ expression)           │
│  - Tissue/organ classifier (linear probe)                 │
└──────────────────────────────────────────────────────────┘
```

### 3.1 模型版本渐进路线

| 版本 | 目标 | 模块 | Go/No-Go 条件 |
|---|---|---|---|
| **M0** | 数据基线 | PCA, NMF, WGCNA, AE, FFNN, Ridge | 判断任务难度, 冻结 baseline leaderboard |
| **M1** | Expression-only | OG embedding + MEM (Huber), Performer encoder | 必须在内部 LOSO 上 > AE/FFNN |
| **M2** | Promoter+expression | M1 + frozen DNA encoder + cross-modal contrastive | **必须在 ≥2 LOSO tasks 上 > M1 by >5-10% relative** |
| **M3** | Pathway prior | M2 + sparse pathway/module regularization (Plant Reactome) | 生物解释和少量性能增益 |
| **M4** | OT analysis (post-hoc) | M3 embeddings → GW-OT species distance → conserved modules | 仅分析, 不加入训练 loss |

---

## 4. 评估协议 (预注册)

### 4.1 主评估: Leave-One-Species-Out (LOSO)

```
Protocol:
  For each species s in {soybean, Arabidopsis, rice, maize, tomato, [sorghum]}:
    1. Train M1/M2 on all species except s
    2. Zero-shot evaluate on species s
    3. Report metrics ± CI (3-5 seeds)

  主表: 4 tasks × N_species → compact summary
```

### 4.2 主任务 (4 个)

| # | 任务 | 指标 | 基线 |
|---|---|---|---|
| 1 | Expression imputation (masked OG recon.) | Pearson r, NRMSE | tissue mean, kNN, PCA, AE, FFNN |
| 2 | Tissue/organ classification (linear probe) | macro-F1, MCC | LightGBM, Ridge, FFNN, AE |
| 3 | Cross-modal retrieval (promoter↔expression) | Recall@K, MRR | promoter-only model, expression-only model |
| 4 | Pathway/module activity prediction | Pearson r, Spearman ρ | WGCNA eigengene, mean expression |

### 4.3 外部验证

rnadata 260 soybean samples: 作为 soybean-specific external validation (非 LOSO)。

### 4.4 Negative Controls (必须)

| Control | 目的 |
|---|---|
| Shuffled promoter-gene pairing | 验证 promoter modality 提供 gene-specific 信息 |
| Random orthogroup labels | 验证跨物种 transfer 依赖真实 orthology |
| Tissue label permutation | 验证组织分类不是 batch/species shortcut |
| Species-balanced matched tissues only | 验证结果不受 tissue composition 偏差驱动 |

### 4.5 Statistical Reporting

- ≥3 seeds (主结果 5 seeds)
- Bootstrap 95% CI
- Holm-Bonferroni correction across tasks
- Effect size (Cohen's d / Cliff's delta)

---

## 5. 生物学 Case Study: C3/C4 光合模块跨物种比较

**主生物故事**: 用 rice (C3) vs maize+sorghum (C4) 的光合相关 OG modules 比较:

1. 哪些 OG modules 在 C3 和 C4 物种间保守? (表达模式保持)
2. 哪些 OG modules 显示分化? (表达模式在 C4 中重编程)
3. 这些分化 modules 是否富集已知的 C4 pathway 基因?
4. Promoter motif 分析: 分化 modules 的启动子是否富集不同的 cis-regulatory motifs?
5. 与已知 C4 文献 (Kranz anatomy, PEPC, PPDK, NADP-ME 等) 对比验证

**证据链**: 模型嵌入聚类 + promoter motif enrichment + public database + literature cross-reference

---

## 6. 强基线清单

必须实现并冻结 leaderboard 的基线:

| 类别 | 方法 |
|---|---|
| Mean | Global mean, tissue mean, species mean |
| Nearest neighbors | kNN imputation (k=5,10,20) |
| Matrix factorization | PCA (20-200d), NMF, SoftImpute |
| Linear | Ridge, ElasticNet, PLS |
| Co-expression | WGCNA module eigengenes + linear model |
| Shallow NN | FFNN (Yang-style, 1-3 layers), MLP |
| Deep generative | Denoising AE (2-5 layers), VAE |
| Deep expression | Geneformer-style (retrain on soybean), BulkFormer-style (adapt to soybean) |
| Promoter-only | Promoter encoder → predict tissue-averaged expression |
| Ortholog baseline | Direct 1-to-1 ortholog correlation |

---

## 7. Gate 执行路线

### Gate 0: 数据可行性扫描 (2-3 周)

**输入**: 6 物种候选列表
**输出**: `M0_data_feasibility_report.md`

逐物种列出:
- Accession/source, 真实可下载样本数
- Metadata 完整率 (tissue, condition, developmental stage)
- 是否需要从 FASTQ 统一重处理 vs 可用已处理表达矩阵
- 组织标签映射到统一 ontology 的覆盖率
- License 和使用限制

**Go 条件**: ≥4 物种有 >800 可用样本和可比 tissue metadata
**No-Go**: 降级为 3 物种, 调整预期

### Gate 1: Soybean M1 验证 + Baseline Freeze (2-3 周)

**在 soybean-only 上先验证基础架构**:

1. 从 SoyAtlas 提取 gene_TPM, 建立 gene→OG 映射
2. 提取 2 kb promoters
3. 实现 M0 baselines + M1 (expression-only)
4. 在 soybean internal split 上验证 M1 > AE/FFNN

**Go 条件**: M1 soybean internal imputation Pearson r > AE by ≥5%
**No-Go**: 重新评估 MEM 设计, 考虑更简单的架构

### Gate 2: SoyFormer-M2 Promoter-Expression (3-4 周)

**加入 promoter modality, 在 soybean 上验证**:

1. 集成 AgroNT 或备选 DNA encoder
2. 实现 cross-modal contrastive objective
3. 在 soybean internal + rnadata external 上对比 M1 vs M2

**Go 条件**: M2 > M1 on ≥2 metrics by >5-10% relative improvement
**No-Go**: 保持 expression-only (M1), 不强调多模态, 投 GB/NAR

### Gate 3: 多物种扩展 (4-6 周)

**扩展到 4-6 物种**:

1. 统一 OG 矩阵构建
2. 跨物种 promoter 提取
3. M2 多物种联合训练
4. LOSO × N_species 评估
5. 所有 negative controls

**Go 条件**: LOSO 主结果在 ≥3/4 任务上 > 强基线
**No-Go**: 降为 soybean-only + external validation 论文

### Gate 4: 生物学 Case Study + 消融 (4-6 周)

**C3/C4 光合模块分析 + 消融实验**:

1. Rice vs Maize+Sorghum C3/C4 OG module divergence
2. Promoter motif enrichment in divergent modules
3. 关键消融: -promoter, -pathway, -graph, encoder depth
4. Sensitivity analyses: promoter length, OG vocabulary, normalization

### Gate 5: 论文写作 + 开源 (4-6 周)

1. Manuscript draft (GB/NC format)
2. Code release (GitHub)
3. Pre-trained weights (HuggingFace/Zenodo)
4. Pre-registration (OSF)

---

## 8. 时间线

| Gate | 内容 | 时间 | 累计 |
|---|---:|---:|
| Gate 0 | 数据可行性扫描 | 2-3 周 | 3 周 |
| Gate 1 | Soybean M1 + Baseline | 2-3 周 | 6 周 |
| Gate 2 | SoyFormer-M2 Promoter | 3-4 周 | 10 周 |
| Gate 3 | 多物种扩展 + LOSO | 4-6 周 | 16 周 |
| Gate 4 | Case Study + 消融 | 4-6 周 | 22 周 |
| Gate 5 | 论文 + 开源 | 4-6 周 | **28 周** |

**总周期: 24-32 周** (取决于数据获取难度和模型表现)

---

## 9. 投稿策略

| 执行结果 | NC 概率 | GB 概率 | 推荐行动 |
|---|---|---|---|
| M2 LOSO 显著超强基线 + 强 case study | 30-40% | 60-75% | 先投 NC, 被拒转 GB |
| M2 LOSO 中等提升 + case study | 15-25% | 50-60% | 直接投 GB |
| M1 only, 提升有限 | 5-10% | 30-45% | 投 GB/NAR 作为 benchmark/resource paper |

**Primary target: Genome Biology。Stretch target: Nature Communications (仅在 M2 + case study 都强阳性时)。**

---

## 10. 风险矩阵

| 风险 | 概率 | 缓解 |
|---|---|---|
| <4 物种通过 Gate 0 | 中 | 降为 3 物种, 调整预期 |
| Promoter modality 增益 <5% | 中 | Gate 2 No-Go → expression-only, 投 GB |
| M1/M2 未超 AE/WGCNA | 中 | Gate 1 No-Go → 重新评估架构; 或转 benchmark paper |
| LOSO 结果受 tissue composition 偏差驱动 | 中 | Negative controls + species-balanced matched tissue subset |
| AgroNT 不可用 | 低 | 备选 HyenaDNA/Nucleotide Transformer/轻量 CNN |
| GPU 资源不足 | 低 | Gradient checkpointing + bf16 + gene subset 先跑通 |

---

## 11. 参考文献 (精选)

1. Theodoris et al. *Nature* 618:616-624 (2023). Geneformer.
2. Kang et al. *bioRxiv* (2025). BulkFormer.
3. Qiu et al. *ACL* (2025). GRNFormer.
4. GREmLN. *bioRxiv* (2025). Graph-aware transcriptomics FM.
5. Wu et al. *Genome Biology* 26:334 (2025). scFM benchmark.
6. Ahlmann-Eltze et al. *Nature Methods* 22:1657-1661 (2025). Perturbation benchmark.
7. Almeida-Silva et al. *The Plant Journal* (2023). SoyAtlas v2.
8. Yang Y. *bioRxiv* (2024). SoyAtlas FFNN baseline.
9. Fradkin et al. *Nature Methods* (2026). Orthrus.
10. Species-OT: Gromov-Wasserstein for cross-species transcriptome. *bioRxiv* (2025).
