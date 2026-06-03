# Gate 1 Findings

## Research Findings

### Baseline Leaderboard (2026-05-28)
- **PCA-200 是最强基线**: Internal r=0.918, External r=0.903
- AE 深度增加无益: AE-256 (r=0.891) ≈ AE-512-256 (r=0.890)
- kNN 显著弱于 PCA: r=0.80-0.81
- 跨数据集泛化 gap 小: PCA 从 0.918→0.903 仅降 1.5%
- 表达插补天花板已被 PCA 触及 — 继续追求更高 r 的边际收益低

### 基因 ID 对齐
- SoyAtlas (Phytozome): glyma.XXG 命名, 52,837 genes
- rnadata (Ensembl re-quant): GLYMA_XXG 命名, 55,897 genes
- 归一化 (lowercase + _→. ) 后 overlap **89.0%** (47,046 genes)
- Re-quantification 映射率 88.9% (mean), min 65.2%

### BioProject × Tissue 混淆
- 87.2% BioProjects 是单组织 → 按 BP split 会直接引入组织偏差
- 采用组织分层 + BP-grouped split: Train 272 / Val 62 / Test 69 BPs
- 低频组织 (radicle/epicotyl/petiole, 各1 BP) 从主评估排除

### 跨物种数据可用性
- Soybean: 5,481 samples ✅
- Maize: EBI ~2,283 assays + Gramene → 估计 ~2,500-3,500
- Arabidopsis: EBI 615 + Figshare TPM matrix → 估计 ~2,000-3,000
- Rice: EBI ~250 + RED → 估计 ~800-1,500 ⚠
- Sorghum/Tomato: <800 samples ⚠
- **推荐起步: 3 物种** (soybean + maize + Arabidopsis)

## Technical Decisions
| Decision | Rationale |
|----------|-----------|
| Baseline 目标改为 PCA-200 | 实测 PCA 是最强基线, 不是 AE |
| Phase 3 仅做快速 M1 验证 | 避免在 expression-only 方向过度投入 |
| 叙事重构 | Imputation 天花板低, 转向 transfer + interpretability |
| 3 物种起步 | Rice/Tomato/Sorghum 数据不足 |

## Issues Encountered
| Issue | Resolution |
|-------|------------|
| RDA 文件无法直接读取 (无 R/pyreadr) | 改用 parquet 文件读取 SoyAtlas |
| Salmon 索引 --decoys 参数错误导致 0 hits | 重建索引去除 --decoys |
| SoyAtlas ↔ rnadata 基因 ID 0% overlap | 发现 _ vs . 差异, 归一化修复 → 89% |
| SLURM 脚本中 mamba activate 不生效 | 改用 Python 完整路径 |
| AE sklearn 维度错误 | 转置 fix: ae.fit(X.T, X.T) |
| numpy/pyarrow 未安装 | mamba install |
