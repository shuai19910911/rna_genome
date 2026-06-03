# Task Plan: SoyFormer-U 主执行计划

**Created**: 2026-05-29  
**Status**: active  
**Scope**: soybean-only Gate 1/2 收敛, 再进入跨物种 Gate 3/4  
**Operating rule**: 后续每次任务结束必须同步更新本计划目录下的 `progress.md`; 若产生新结论、指标或决策, 同步更新 `task_plan.md` 或 `findings.md`。

---

## 1. 当前项目定位

SoyFormer-U 的长期目标是跨物种 promoter-expression 表征学习。当前仓库的实际阶段不是完整跨物种模型, 而是 soybean-only 可行性与强基线阶段:

1. 已完成 SoyAtlas + rnadata 的大豆统一表达矩阵。
2. 已冻结第一版 M0 强基线。
3. 已完成 M1 expression-only 初跑。
4. M2 promoter-expression 已完成正式 80 epoch 训练和 shuffled-promoter 对照; 当前未建立强 promoter-pairing 证据。

因此接下来不应直接扩展大模型叙事, 应先把 soybean-only 的实验闭环做扎实: 数据一致性、强基线、M1/M2 对照、negative control、可复现报告。

---

## 2. 已完成事实

### 2.1 数据管线

- [x] rnadata 260 samples 使用 Ensembl cDNA + Salmon 重新定量。
- [x] 汇总 gene-level TPM 到 `05.matrix/gene_tpm_ensembl.tsv`。
- [x] 构建 SoyAtlas + rnadata 统一矩阵。
- [x] 基因 ID 归一化策略: lowercase + `_` -> `.`。
- [x] 统一矩阵: 47,046 shared genes x 5,741 samples。
- [x] 样本来源: SoyAtlas 5,481; rnadata external 260。
- [x] BioProject-grouped split 已生成: `split_train.txt`, `split_val.txt`, `split_test.txt`。
- [x] 已提取 5000 HVG 的 2kb promoter: `data/promoters.tsv`。
- [x] 已预计算 promoter embedding: `data/promoter_emb.npy`。

### 2.2 M0 强基线

结果文件: `results/baseline_results.json`, `results/harder_baselines.json`

| Method | Internal random-mask r | External r | Notes |
|---|---:|---:|---|
| PCA-200 | 0.9181 | 0.9029 | 当前最强表达插补基线 |
| PCA-100 | 0.9111 | - | 接近 PCA-200 |
| AE-256 | 0.8905 | - | 弱于 PCA |
| AE-512-256 | 0.8904 | 0.8771 | 加深无收益 |
| kNN-20 | 0.8025 | 0.8208 | 明显弱于 PCA |
| GlobalMean | -0.0097 | -0.0064 | 无效 |

Harder mask / tissue classification:

| Task | Best baseline | Metric |
|---|---|---:|
| random 15% mask | PCA-200 | r=0.9181 |
| module block mask, 2 modules | PCA-200 | r=0.8157 |
| module block mask, 5 modules | PCA-200 | r=0.7531 |
| tissue variable 1000 genes | PCA-200 | r=0.8485 |
| tissue classification | PCA-200 + LogisticRegression | macro-F1=0.6049 |

### 2.3 M1 expression-only

结果文件: `results/m1_log.json`, `results/m1_best.pt`

Best observed at epoch 80:

| Metric | M1 |
|---|---:|
| random-mask r | 0.7314 |
| module-block r | 0.5986 |
| tissue macro-F1 | 0.3613 |
| external random-mask r | 0.7057 |

Decision: M1 当前未达到 PCA-200, 也未达到 tissue classifier baseline。M1 不能作为“优于经典方法”的结果写入主叙事, 只能作为深度模型 ablation 起点。

### 2.4 M2 promoter-expression

结果文件: `results/m2_log.json`, `results/m2_best.pt`

Best observed at epoch 80:

| Metric | M2 | M1 delta |
|---|---:|---:|
| random-mask r | 0.7348 | +0.0034 |
| module-block r | 0.6036 | +0.0049 |
| tissue macro-F1 | 0.3645 | +0.0032 |
| external random-mask r | 0.7070 | +0.0013 |

Decision: M2 目前仅略高于 M1, 仍远低于 PCA-200 random-mask baseline。M2-shuffled 在 module-block r 上几乎追平真实 M2, 因此当前不能声称 promoter-gene pairing 带来稳定收益。

Important caveat:

当前 `promoter_emb.npy` 来自项目内随机初始化 CNN, 不是 AgroNT/HyenaDNA 等预训练 DNA encoder。除非后续替换为预训练或训练好的 DNA encoder, M2 的 promoter 增益不能强解释为生物学 promoter prior。

---

## 3. 总体路线

### Gate A: soybean-only 实验闭环

目标: 在当前大豆数据上完成可复现 baseline/M1/M2/negative-control 对照, 明确是否值得继续投入 promoter-expression 架构。

Exit criteria:

- 所有关键结果有 JSON 日志和命令记录。
- M2 与 M1、PCA-200、shuffled promoter 至少完成 1 seed 对照。
- 如果 M2 没有超过 M1 或 shuffled promoter, 不做 promoter superiority claim。
- 如果 M2 超过 M1 但低于 PCA, 叙事限定为 architecture ablation, 不声称 SOTA。

### Gate B: M2 合理化与严谨对照

目标: 解决当前 M2 的解释性问题。

Exit criteria:

- 明确 promoter embedding 来源。
- 至少有 shuffled promoter negative control。
- 优先补一个 trainable promoter encoder 或预训练 DNA encoder 路线评估。
- 若无法接入预训练 DNA encoder, 文档中明确降级为 sequence-derived random CNN feature ablation。

### Gate C: 任务重构

目标: 避免只在 random mask 插补上和 PCA 硬拼, 转向更适合深度/多模态模型的任务。

候选任务:

- module-block masked reconstruction
- tissue-variable gene reconstruction
- tissue classification / linear probe
- cross-dataset external validation
- promoter-gene pairing negative control

Exit criteria:

- 每个主任务都有 PCA/AE/M1/M2/shuffled M2 对照。
- 指标统计口径统一, 写入 `results/summary_*.json`。

### Gate D: 跨物种准备

目标: 只有在 soybean-only 对照结论清楚后, 再启动跨物种数据工程。

Start criteria:

- Gate A-C 已完成。
- 明确主张: transfer / interpretability / promoter modality, 而不是单纯 imputation SOTA。

Exit criteria:

- 至少 3 物种数据源和 gene/orthogroup 映射路线冻结。
- 完成跨物种最小数据 report。

---

## 4. 下一步任务队列

### P0: 计划和执行规则

- [x] 新建 2026-05-29 主执行计划。
- [x] 将 `.planning/.active_plan` 指向本计划。
- [x] 在 `CLAUDE.md` 写入“每次任务结束更新计划文档”的规则。

### P1: M2 重新训练可运行性验证

- [x] 使用 `python cli.py train m2 --epochs 5` 做 smoke test。
- [x] 若 smoke test 失败, 修复数据维度、GPU/CPU、模型接口问题。
- [x] 成功后运行完整 M2 训练, 产出 `results/m2_log.json` 和 `results/m2_best.pt`。
- [x] 运行 `python cli.py train m2 --shuffle-promoters`, 产出 shuffled promoter 对照。

Acceptance:

- 至少完成 5 epoch smoke test 且日志正常。
- 完整训练至少保存一个 M2 JSON 日志。

### P2: 统一评估脚本

- [x] 新建或整理统一 evaluator, 汇总 PCA/M1/M2/M2-shuffled。
- [x] 实现 `scripts/evaluate_unified.py` 统一 mask/sample/metric 评估脚本。
- [x] 运行 full unified evaluation, 输出 `results/unified_eval.json` 和 `docs/2026-05-31-unified-eval.md`。
- [x] 输出 `results/soybean_gate_a_summary.json`。
- [x] 输出可读表格到 `docs/2026-05-31-soybean-gate-a-results.md`。

Acceptance:

- 指标至少包含 random-mask r、module-block r、external r、tissue macro-F1。
- 所有结果可追溯到输入 JSON 或日志。

### P3: M1/M2 质量修复

- [x] 检查 M1/M2 训练目标与 PCA baseline 是否口径一致。
- [x] 检查 tissue label 构造是否存在 BioProject/tissue leakage 或类别缺失问题。
- [x] 加入 validation split 或明确当前 test-only 评估风险。
- [x] 实现 validation-based checkpointing: train 训练, val 选 checkpoint, test 仅最终报告。
- [x] 运行 validation-protocol smoke test: `CUDA_VISIBLE_DEVICES=3 python cli.py train m2 --epochs 5 --run-name M2-val-smoke`。
- [x] 使用 validation protocol 重新训练 M1/M2/M2-shuffled 80 epoch。
- [ ] 评估模型容量、mask 策略、loss 权重是否合理。
- [x] 实现 M2 no-gene-embedding ablation。
- [x] 运行 M2-nogene 与 M2-nogene-shuffled 80 epoch 对照。

Acceptance:

- 写入 `findings.md`: 当前模型未超过 PCA 的最可能原因。
- 如果修改代码, 每次修改后记录命令、结果和变更摘要。

### P4: promoter modality 合理化

- [x] 评估当前随机 CNN promoter embedding 的有效性和局限。
- [ ] 决策是否接入预训练 DNA encoder, 或改为 trainable lightweight encoder。
- [ ] 若使用预训练, 记录模型来源、版本、输入窗口和 embedding 维度。
- [x] 若暂不使用预训练, 在计划和报告中降级 claims。
- [x] 保存并校验 promoter embedding 的 HVG gene order。
- [x] 在 M2 训练和统一评估中加入 promoter embedding gene-order alignment check。

Acceptance:

- `findings.md` 中有明确 promoter embedding decision。
- M2 结果必须和 shuffled promoter 对照同表报告。

### P5: 跨物种 Gate D 准备

- [ ] 更新跨物种数据源清单。
- [ ] 冻结第一轮物种: soybean + maize + Arabidopsis, rice 作为条件候选。
- [ ] 设计 orthogroup 映射和 shared vocabulary 文件格式。
- [ ] 起草 `docs/M0_cross_species_data_feasibility_report.md`。

Acceptance:

- 不在 Gate A-C 完成前投入大规模下载或重处理。

---

## 5. Go / No-Go 标准

### M2 Go

任一成立即可进入更系统 M2:

- M2 在 module-block r 上相对 M1 提升 >= 5%。
- M2 在 tissue macro-F1 上相对 M1 提升 >= 10%。
- M2 明显超过 shuffled promoter, 且差异在至少 3 seeds 稳定。

### M2 No-Go

**Current decision after 2026-06-02 unified evaluation**: Gate A is No-Go for the current M2 promoter branch. PCA remains much stronger, and true M2 does not clearly beat M1 or M2-shuffled under validation-selected unified evaluation.

任一成立则暂停 promoter 主张:

- M2 <= M1。
- M2 与 shuffled promoter 无差异。
- M2 仅在 random-mask r 上小幅提升, 但仍远低于 PCA-200。
- 统一评估下 PCA 明显强于 M1/M2, 且 M2 未稳定超过 M1/shuffled。

### Cross-species Go

- soybean-only 的主张已经明确且不依赖“超过 PCA random mask”。
- 至少 3 物种有可用表达矩阵和可比 metadata。
- orthogroup 映射路线可执行。

---

## 6. 文档更新协议

每次执行任务结束后必须做以下更新:

1. `progress.md`: 添加日期、任务、执行命令、产物路径、结果摘要、失败原因。
2. `findings.md`: 只有产生新事实、新指标、新 bug 根因、新技术决策时更新。
3. `task_plan.md`: 任务状态变化时勾选 checkbox; Gate 标准变化或路线调整时更新相关章节。
4. Final response: 简短说明本次改了哪些计划文档, 并明确下一项建议执行任务。

5. GPU 命令资源说明: 提供任何 GPU 命令时, 必须同时写清楚所需计算资源: GPU 数量和建议卡号、预估显存、CPU 核数、内存、预计运行时间、需要检查的输出文件。

---

## 7. 当前推荐下一步

进入 P3/P4 交界阶段:

validation-protocol M2 smoke test 已通过。下一步先补 promoter embedding gene-order 审计, 再启动 validation protocol 下的 canonical M1/M2/M2-shuffled 80 epoch 重训。

建议任务:

保存并检查 `data/promoter_emb.npy` 对应的 HVG gene order, 目标产物为 `data/promoter_emb_genes.txt` 和加载时的顺序校验。完成后再运行:

```bash
CUDA_VISIBLE_DEVICES=0 python cli.py train m1 --epochs 80
```

资源: GPU 1 张, 建议 A100 40GB 或同级空闲卡; M2 smoke test 实测约 32.4GB VRAM, M1 预计更低但仍建议 >=24GB; CPU 4-8 cores; RAM 32-64 GB; 输出检查 `results/m1_log.json`, `results/m1_best.pt`, `results/m1_summary.json`。
