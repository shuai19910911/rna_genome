# 模型结构解析

本文档用于说明当前两个 RNA / 基因组建模项目中的主要模型结构、输入输出、评估注意事项和已知限制。

---

## 1. SoyFormer-U

SoyFormer-U 当前包含两个主要神经网络模型：

1. M1：expression-only 模型。
2. M2：promoter + expression 多模态模型。

另有一个轻量 DNA promoter 编码器，用于生成 promoter embedding。

---

## 1.1 M1：Expression-only Performer

源码位置：

```text
src/model/m1.py
```

### 输入

```text
x: 表达矩阵 batch，形状为 (B, G)
```

其中：

- `B`：batch size。
- `G`：基因数量。
- `x`：每个样本在每个基因上的表达值。

### 主要结构

M1 是一个只使用表达矩阵的模型，不使用 promoter、DNA 序列或其他基因组特征。

结构流程：

```text
expression x
    ↓
REE 表达值编码
    ↓
gene embedding
    ↓
concat(gene embedding, expression embedding)
    ↓
linear projection
    ↓
加入 CLS token
    ↓
Performer blocks
    ↓
两个输出头：
    1. masked expression reconstruction
    2. tissue classification
```

### 关键模块

#### 1. Gene embedding

```python
nn.Embedding(n_genes, d_model)
```

作用：给每个基因一个可学习的 ID embedding。

#### 2. REE expression encoder

作用：把单个表达值映射成 `d_model` 维表示。

#### 3. Expression projection

把 gene embedding 和 expression embedding 拼接后投影回 `d_model`。

#### 4. CLS token

在所有 gene token 前加入一个 CLS token，用于 tissue classification。

#### 5. Performer blocks

用于建模基因之间的长程关系。

#### 6. MEM head

输出每个基因的重建表达值。

#### 7. Tissue head

使用 CLS token 输出 tissue 分类 logits。

### 输出

```text
1. reconstructed expression: (B, G)
2. tissue logits: (B, n_tissues)
```

### 当前解释

M1 是 expression-only 深度模型基线。它的作用是判断：

> 仅使用表达矩阵自身，深度模型能否超过 PCA / AE / kNN 等经典基线。

当前结果显示，M1 低于 PCA-200，因此不能作为强于经典方法的主结果。

---

## 1.2 M2：Promoter + Expression 多模态模型

源码位置：

```text
src/model/m2.py
```

### 输入

```text
x: 表达矩阵 batch，形状为 (B, G)
prom_emb: promoter embedding，形状通常为 (B, G, d_model) 或可广播到该形状
```

其中：

- `x`：表达矩阵。
- `prom_emb`：每个基因对应的 promoter embedding。
- `G`：基因数量。
- `d_model`：模型隐藏维度。

### 主要结构

M2 在 M1 的基础上加入 promoter embedding。

结构流程：

```text
promoter embedding
    ↓
promoter projection

expression x
    ↓
REE 表达值编码

gene id
    ↓
gene embedding（可关闭，用于 no-gene ablation）

concat(promoter, gene, expression)
    ↓
fusion projection
    ↓
加入 CLS token
    ↓
Performer blocks
    ↓
两个输出头：
    1. masked expression reconstruction
    2. tissue classification
```

### 关键模块

#### 1. Promoter projection

```python
Linear(d_model, d_model) + GELU
```

作用：把预计算的 promoter embedding 投影到模型空间。

#### 2. Gene embedding

```python
nn.Embedding(n_genes, d_model)
```

作用：给每个基因一个可学习 ID 表示。

该模块可以关闭：

```text
use_gene_embed=False
```

关闭后用于 no-gene-embedding 消融实验，检查模型是否过度依赖 gene ID。

#### 3. REE expression encoder

作用：编码表达值。

#### 4. Fusion projection

将三类信息拼接：

```text
promoter representation + gene embedding + expression embedding
```

然后投影回 `d_model`。

#### 5. Performer blocks

用于建模基因 token 之间的关系。

#### 6. 输出头

与 M1 相同：

1. masked expression reconstruction head。
2. tissue classification head。

### 输出

```text
1. reconstructed expression: (B, G)
2. tissue logits: (B, n_tissues)
```

### 当前解释

M2 的目标是检验 promoter 模态是否能提供表达矩阵之外的额外信息。

但当前结果显示：

- M2 只略高于 M1。
- M2 与 shuffled-promoter negative control 非常接近。
- M2 仍低于 PCA-200。

因此目前不能声称 promoter 信息已经显著提升表达预测。

更稳妥的表述是：

> M2 多模态结构可以运行，但当前 promoter embedding 尚未提供强到足以支撑主结论的增益。

---

## 1.3 DNAEncoder：轻量 promoter CNN 编码器

源码位置：

```text
src/model/dna_encoder.py
```

### 输入

```text
tokenized promoter sequence: (B, G, L)
```

其中：

- `B`：batch size。
- `G`：基因数量。
- `L`：promoter 序列长度。
- 每个碱基被编码为 0/1/2/3。

### 结构

```text
DNA token
    ↓
4-base embedding → 64 dim
    ↓
Conv1d 64 → 128
    ↓
Conv1d 128 → 256
    ↓
Conv1d 256 → d_model
    ↓
AdaptiveMaxPool1d(1)
    ↓
promoter embedding: (B, G, d_model)
```

### 当前作用

该模块用于生成 promoter embedding，但它不是预训练 DNA foundation model。

因此当前 M2 的 promoter embedding 更适合表述为：

> sequence-derived lightweight CNN feature

而不是：

> pretrained promoter representation

---

## 1.4 SoyFormer-U 当前主要限制

1. M1 / M2 均未超过 PCA-200 强基线。
2. M2 与 shuffled-promoter control 接近。
3. promoter embedding 来自轻量 CNN，而不是预训练 DNA 模型。
4. 当前证据不足以支持 promoter-gene pairing 的强生物学结论。
5. 后续若要强化 promoter 叙事，需要接入预训练 DNA encoder 或重新设计任务。

---

## 2. SoyGT-Former BiB

SoyGT-Former 是面向 Briefings in Bioinformatics 的泛基因组表达预测模型。

源码项目位置：

```text
myhermes/rna_seq_model/rnamodel/soygt_former
```

数据规模：

```text
26 个 soybean accessions × 9 个 tissues × 58,427 个 genes
```

---

## 2.1 模型输入

SoyGT-Former 使用多类输入：

1. promoter / k-mer 特征。
2. PAV，即 gene presence / absence variation。
3. CNV，即 copy number variation。
4. gene family。
5. variant counts。
6. accession index。
7. tissue index。
8. group label 作为辅助预测目标。

---

## 2.2 核心结构

整体结构：

```text
promoter / k-mer branch
        ↓
variant / PAV / CNV branch
        ↓
accession + tissue context branch
        ↓
3-token fusion transformer
        ↓
tissue FiLM conditioning
        ↓
expression regression head
        ↓
auxiliary heads:
    - group classification
    - masked k-mer reconstruction
    - masked PAV prediction
```

---

## 2.3 主要模块

### 1. Genome / promoter branch

作用：编码 promoter 或 k-mer 特征。

### 2. Variant branch

整合以下信息：

- PAV。
- CNV。
- gene family。
- variant counts。

### 3. Context branch

编码：

- accession。
- tissue。

注意：v2 中 group label 不再作为输入，以避免 group label 既作为输入又作为预测目标。

### 4. Fusion transformer

将 promoter token、variant token、context token 融合。

### 5. PAV-conditioned attention gate

当某个 accession 中某基因缺失时，PAV gate 会限制 variant token 的作用。

### 6. Tissue FiLM conditioning

使用 tissue embedding 对融合表示做 feature-wise modulation。

### 7. Expression head

输出表达量预测。

### 8. Auxiliary heads

包括：

- group classification。
- masked k-mer reconstruction。
- masked PAV prediction。

---

## 2.4 重要评估规则

### LOT 必须使用 raw target

LOT 是 leave-one-tissue-out。测试 tissue 在训练集中完全不可见。

因此不能使用 residual target：

```text
--target-mode residual
```

原因：residual target 依赖：

```text
tissue_mean[gene, tissue]
```

但 held-out tissue 在训练集中不存在，所以该 tissue 的均值不可估计。

LOT 正确设置：

```text
--target-mode raw
```

如果 LOT 错用 residual，Pearson 会接近 0。

### LOA / LOAxT

LOA 和 LOAxT 是否使用 residual，需要根据具体 claim 和 split 设计单独判断。

---

## 2.5 当前已验证结果

### XGBoost baseline

```text
LOA Pearson = 0.786
LOT mean Pearson = 0.782
```

### SoyGT-Former LOT 9-fold

使用设置：

```text
--target-mode raw
masking 开启
epochs = 30
```

结果：

```text
LOT mean Pearson = 0.8261
LOT SD = 0.0438
Pearson range = 0.7567 - 0.8773
RMSE mean = 1.3200
```

逐折结果：

| Fold | Pearson | RMSE | MAE |
|---|---:|---:|---:|
| A | 0.7740 | 1.4046 | 0.8705 |
| B | 0.8773 | 1.0421 | 0.7260 |
| C | 0.8550 | 1.1149 | 0.8229 |
| D | 0.8397 | 1.2992 | 0.9660 |
| E | 0.8403 | 1.3205 | 0.9723 |
| F | 0.8526 | 1.1247 | 0.8095 |
| G | 0.8594 | 1.1852 | 0.8421 |
| H | 0.7795 | 1.5253 | 1.1542 |
| I | 0.7567 | 1.8636 | 1.5424 |

### 当前解释

SoyGT-Former 在 LOT 上超过 XGBoost baseline：

```text
0.8261 vs 0.782
```

这说明模型在跨 tissue 泛化上有一定优势。

---

## 2.6 当前注意事项

1. group_acc 接近 1.0 并不代表强泛化，因为 group 与 accession 基本是确定映射。
2. group classification 不能作为主要创新点过度宣传。
3. 外部验证数据目前不可用，文章需要明确 limitation。
4. 后续仍需完成 LOA 3-fold 和 LOAxT 5-group。
5. 所有结果必须和 XGBoost / naive baseline 对照报告。

---

## 3. 当前总判断

### SoyFormer-U

当前更像是模型探索和负结果分析项目，结论要保守。

### SoyGT-Former BiB

当前 LOT 结果有效，已具备继续推进 LOA / LOAxT / baseline 对照 / 解释性分析的价值。

下一阶段重点：

1. 完成 LOA 3-fold。
2. 完成 LOAxT 5-group。
3. 汇总 SoyGT-Former vs XGBoost / naive baseline。
4. 生成论文表格和图。
5. 谨慎处理 group_acc 和外部验证限制。
