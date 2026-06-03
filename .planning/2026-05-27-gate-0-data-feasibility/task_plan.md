# Task Plan: Gate 0 — SoyFormer-U 数据可行性扫描

## Goal
产出 `docs/M0_data_feasibility_report.md`, 确认数据足以支撑跨物种方案, 满足 Go 条件。

## Current Phase
Phase 2

## Phases

### Phase 1: SoyAtlas gene annotation 版本确认
- [x] 读取 se_atlas_gene.rda 结构 (通过 parquet 文件分析)
- [x] 确认基因 ID 体系: Glyma.XXGYYYYYY (Phytozome 命名)
- [x] 确认组织分布: 19 tissues, 494 BioProjects
- **Status: complete**
**Deliverable**: findings.md 记录 annotation info ✓

### Phase 2: rnadata gene ID 映射
- [x] 从 rnadata GFF 读取 gene ID: gene-LOC* + agat-gene-N 双体系
- [x] 确认两套命名体系差异: SoyAtlas=Phytozome Glyma.XXG, rnadata=NCBI LOC+AGAT
- [ ] 获取 Phytozome Wm82 annotation 并构建坐标映射
- [ ] 评估映射率
- **Status: in_progress**
**Deliverable**: mapping rate table

### Phase 3: BioProject × tissue confounding 量化
- [ ] 从 SoyAtlas colData 提取 BioProject 和 body_part
- [ ] 制作 BioProject × tissue 列联表
- [ ] 计算 Cramer's V
- **Status: pending**
**Deliverable**: confounding metrics in findings.md

### Phase 4: 冻结 train/val/test split 方案
- [ ] 按 BioProject 分组划分
- [ ] 确保每个主要 tissue 在 test 中有 ≥3 BioProjects
- **Status: pending**
**Deliverable**: split scheme documented

### Phase 5: 跨物种数据源初步调研
- [ ] Arabidopsis: EBI Expression Atlas 可用性
- [ ] Rice: Gramene / RED 可用性
- [ ] Maize: Gramene / MaizeGDB 可用性
- [ ] Tomato: SGN / EBI 可用性
- [ ] Sorghum: Gramene 可用性
- **Status: pending**
**Deliverable**: per-species availability summary

## Go/No-Go Criteria

| Condition | Threshold |
|---|---|
| gene ID mapping rate | one-to-one ≥ 70% |
| Each main tissue in test | ≥ 3 BioProjects |
| Cross-species: usable species | ≥ 4 with ≥ 800 samples |

## Decisions Made
| Decision | Rationale |
|----------|-----------|

## Errors Encountered
| Error | Resolution |
|-------|------------|
