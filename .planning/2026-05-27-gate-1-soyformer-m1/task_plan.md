# Task Plan: Gate 1 — SoyFormer-M1 最小深度模型 + 强基线

## Goal
在 soybean 数据上验证 masked expression denoising 方法, 冻结 baseline leaderboard, 决策下一步方向。

## Current Phase
Phase 2 完成, Phase 3 待决策

## Phases

### Phase 1: 数据预处理 — 构建统一表达矩阵
- [x] 从 SoyAtlas parquet 加载全部 52,837 genes × 5,481 samples TPM 矩阵
- [x] 从 rnadata matrix 加载 55,897 genes × 260 samples TPM 矩阵
- [x] 归一化基因 ID (lowercase + _ → .)
- [x] 取交集基因 (47,046 shared), 构建联合矩阵
- [x] log1p 变换
- **Status: complete**
**Deliverable**: data/unified_gene_tpm.npz (47,046 × 5,741)

### Phase 2: 强基线实现
- [x] PCA (50/100/200) imputation
- [x] kNN (5/10/20) imputation
- [x] Autoencoder (256, 512-256)
- [x] GlobalMean baseline
- [x] Internal test + external rnadata validation
- **Status: complete**
**Deliverable**: results/baseline_results.json

### Phase 3: SoyFormer-M1 快速验证 (待决策)
- [ ] 最小 Performer encoder + MEM
- [ ] 5000 HVG, 1 GPU, 50-100 epochs
- [ ] 目标: 插补 Pearson r >= PCA-200 (0.918)
- [ ] 若匹配 → 继续扩展; 若不匹配 → 跳过 expression-only, 直接做多模态
- **Status: pending**

## Baseline Leaderboard (HVG=5000, 21min)

| Rank | Method | Internal Test r | External rnadata r |
|---|---|---|---|
| 1 | **PCA-200** | **0.9181 ± 0.0002** | 0.9029 ± 0.0007 |
| 2 | PCA-100 | 0.9111 | — |
| 3 | AE-256 | 0.8905 | — |
| 4 | AE-512-256 | 0.8904 | 0.8771 |
| 5 | PCA-50 | 0.8896 | — |
| 6 | kNN-20 | 0.8025 | 0.8208 |
| 7 | GlobalMean | −0.01 | −0.01 |

## Go/No-Go Criteria (修订)

| Condition | Threshold | 当前状态 |
|---|---|---|
| M1 > **PCA-200** (最强基线) | >1% relative | ⏳ 待 M1 训练 |
| M1 cross-dataset gap | ≤ PCA gap (1.5%) | ⏳ |
| M1 training stable | 3 seeds, CI OK | ⏳ |

## Key Findings
1. PCA-200 是真正的天花板基线 (r=0.918), 不是 AE
2. AE 层数增加无益 (256 vs 512-256 几乎一样)
3. 跨数据集泛化 gap 仅 ~1.5% — 表达矩阵高度结构化
4. 2025 benchmarks 警告被验证: 深度模型难以超越线性方法

## Decisions Made
| Decision | Rationale |
|----------|-----------|
| Baseline 目标从 AE 改为 PCA-200 | PCA 是最强基线 |
| Phase 3 仅做快速验证 | 全量 M1 风险高, 先验证再决定 |
| 叙事重心从"更好插补"转向跨物种 transfer + 可解释性 | Imputation 天花板已被 PCA 触及 |
