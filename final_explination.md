# scFoundation Single-Cell Inference Guide

## 1. Purpose

This document records how to run and understand inference with the pretrained
scFoundation single-cell foundation model.

The immediate goal is to understand and evaluate three types of representation:

1. Cell embedding
2. Gene embedding
3. Contextual gene representation / batched gene embedding

The inference procedure should follow the preprocessing and model-loading
procedure implemented by the official scFoundation repository.

Official repository:

https://github.com/biomap-research/scFoundation

---

# 2. High-Level View

scFoundation takes gene-expression profiles and transforms them into learned
representations.

For single-cell RNA-seq:

                    Gene expression
                          │
                          ▼
                  Cells × genes matrix
                          │
                          ▼
                 Gene vocabulary alignment
                          │
                          ▼
                  Expression preprocessing
                          │
                          ▼
                  Resolution information
                          │
                          ▼
                  scFoundation encoder
                          │
              ┌───────────┴────────────┐
              │                        │
              ▼                        ▼
       Cell representation      Gene representations
              │                        │
              ▼                        ▼
         Cell embedding         Gene-level analysis


The pretrained model has learned patterns from a very large collection of
single-cell transcriptomes.

Instead of using the original expression vector directly for every downstream
task, we can extract learned representations from scFoundation.

---

# 3. scFoundation Gene Vocabulary

scFoundation uses a predefined vocabulary of:

    19,264 genes

The repository provides:

    OS_scRNA_gene_index.19264.tsv

This file defines the genes and their required order.

Our dataset may contain:

    15,000 genes
    20,000 genes
    30,000 genes

but the model expects its own 19,264-gene vocabulary.

Therefore the first major preprocessing step is gene alignment.

Example:

Our data:

| Gene  | Expression |
|-------|-----------:|
| TP53  | 20 |
| EGFR  | 0 |
| GAPDH | 60 |
| CD3D  | 15 |
| NKG7  | 5 |

Suppose scFoundation expects:

    TP53
    EGFR
    KRAS
    GAPDH
    CD3D
    NKG7
    ...

Then the aligned representation becomes:

| scFoundation gene | Expression |
|-------------------|-----------:|
| TP53 | 20 |
| EGFR | 0 |
| KRAS | 0 |
| GAPDH | 60 |
| CD3D | 15 |
| NKG7 | 5 |
| ... | ... |

A gene missing from our dataset is filled with zero.

The genes are then placed into exactly the order expected by scFoundation.

Final expression matrix:

    N cells × 19,264 genes

---

# 4. What Should the Input Look Like?

For single-cell inference, the expression matrix should have:

    rows    = cells
    columns = genes
    values  = expression values

Example:

| Cell | TP53 | EGFR | GAPDH | CD3D | CD79A | NKG7 |
|------|-----:|-----:|------:|-----:|------:|-----:|
| Cell_001 | 2 | 0 | 45 | 18 | 0 | 3 |
| Cell_002 | 1 | 0 | 51 | 22 | 0 | 2 |
| Cell_003 | 0 | 3 | 38 | 0 | 15 | 0 |

For the CSV used directly by `get_embedding.py`, the cell identifier does not
need to be an expression column.

Example:

    TP53,EGFR,GAPDH,CD3D,CD79A,NKG7,...
    2,0,45,18,0,3,...
    1,0,51,22,0,2,...
    0,3,38,0,15,0,...

The meaning of the expression values must be established before inference.

The critical question is:

    Are these RAW counts?

or:

    Are these already normalized + log1p values?

This determines `--pre_normalized`.

---

# 5. `pre_normalized`

This is one of the most important inference parameters.

## Raw counts

If the matrix contains raw count-like expression values:

    0
    2
    17
    53
    120

use:

    --pre_normalized F

scFoundation will perform its expected normalization and log transformation.

## Already normalized + log1p

If the matrix has already undergone normalization and log1p:

    0
    1.43
    3.12
    4.87

use:

    --pre_normalized T

The model should not normalize and log-transform the data a second time.

## GEARS-specific representation

`--pre_normalized A` is a special mode in which normalized + log1p expression
is supplied together with total-count information.

This is used by the GEARS workflow.

---

# 6. Preprocessing for Raw Single-Cell Counts

When:

    --input_type singlecell
    --pre_normalized F

the inference pipeline performs the following important operations.

| Step | Operation | Purpose |
|------|-----------|---------|
| 1 | Read cells × genes matrix | Load expression |
| 2 | Match gene symbols | Map genes to model vocabulary |
| 3 | Convert to 19,264 genes | Create expected model feature space |
| 4 | Fill missing genes with zero | Maintain fixed vocabulary |
| 5 | Reorder genes | Match model gene order |
| 6 | Calculate original total count | Preserve read-depth information |
| 7 | Normalize each cell to 10,000 | Correct library-size differences |
| 8 | Apply `log1p` | Compress expression magnitude |
| 9 | Construct resolution information | Represent source/target read depth |
| 10 | Identify positive expression entries | Construct model sequence |
| 11 | Create expression/gene representations | Encode expression and gene identity |
| 12 | Run Transformer encoder | Generate contextual representations |
| 13 | Extract requested embedding | Cell or gene representation |

---

# 7. Example of Expression Normalization

Suppose one cell contains:

| Gene | Raw count |
|------|----------:|
| TP53 | 20 |
| EGFR | 0 |
| GAPDH | 60 |
| CD3D | 15 |
| NKG7 | 5 |

Total expression:

    20 + 0 + 60 + 15 + 5 = 100

For raw counts, scFoundation performs normalization equivalent to:

    normalized_expression =
        expression / total_expression × 10,000

For TP53:

    20 / 100 × 10,000
    = 2,000

For GAPDH:

    60 / 100 × 10,000
    = 6,000

The normalized values become:

| Gene | Raw | Normalized |
|------|----:|-----------:|
| TP53 | 20 | 2000 |
| EGFR | 0 | 0 |
| GAPDH | 60 | 6000 |
| CD3D | 15 | 1500 |
| NKG7 | 5 | 500 |

---

# 8. log1p Transformation

The normalized expression is transformed using:

    log(1 + x)

For example:

| Gene | Normalized | Approx. log1p |
|------|-----------:|---------------:|
| TP53 | 2000 | 7.60 |
| EGFR | 0 | 0 |
| GAPDH | 6000 | 8.70 |
| CD3D | 1500 | 7.31 |
| NKG7 | 500 | 6.22 |

The expression therefore changes roughly from:

    20, 0, 60, 15, 5

to a normalized/log-transformed representation such as:

    7.60, 0, 8.70, 7.31, 6.22

No PCA-style z-score scaling should be added unless a specific downstream
workflow explicitly requires it.

---

# 9. Resolution / Read-Depth Information

scFoundation explicitly models sequencing-resolution information.

Before normalization, the original total count of the cell is retained.

For example:

    Cell A total count = 5,000
    Cell B total count = 50,000

After library-size normalization, both cells are placed on a common expression
scale.

The original sequencing-depth information would otherwise be lost.

scFoundation therefore constructs additional resolution information.

For our test we used:

    --tgthighres a5

With the `a` mode, the target resolution is derived relative to the source
resolution.

Conceptually:

    19,264 expression features
              +
        source resolution
              +
        target resolution
              =
          19,266

This explains the model configuration observed during inference:

    gene_num = 19266
    seq_len  = 19266

---

# 10. Model Architecture Observed From Our Checkpoint

When `models.ckpt` was loaded, the configuration reported:

    encoder hidden dimension = 768
    encoder depth            = 12
    attention heads          = 12
    dimension per head       = 64

The model converts expression information into contextual representations.

Conceptually:

    Expression magnitude
            +
        Gene identity
            +
     Resolution information
            │
            ▼
       Transformer
            │
            ▼
    Contextual representation

A gene is therefore represented in the context of the expression state of the
cell rather than solely by its gene name.

---

# 11. Embeddings We Want to Study

Our current evaluation focuses on:

1. Cell embedding
2. Gene embedding
3. Contextual/batched gene embedding

These should be evaluated separately because they answer different biological
questions.

---

# 12. Cell Embedding

## Command option

    --output_type cell

Example:

    python get_embedding.py \
        --task_name example \
        --input_type singlecell \
        --output_type cell \
        --pool_type all \
        --tgthighres a5 \
        --data_path DATA.csv \
        --save_path ./outputs/ \
        --pre_normalized F \
        --version ce

## Meaning

A complete cell is represented by one vector.

Input:

    Cell 1 → 19,264 gene-expression values
    Cell 2 → 19,264 gene-expression values
    Cell 3 → 19,264 gene-expression values

Output with `pool_type=all`:

    Cell 1 → 3,072-dimensional vector
    Cell 2 → 3,072-dimensional vector
    Cell 3 → 3,072-dimensional vector

Shape:

    N × 3072

## Why 3,072?

The encoder hidden dimension is:

    768

With `pool_type=all`, the inference code combines four 768-dimensional
representations.

Conceptually:

    resolution representation
            +
    resolution representation
            +
       max gene pooling
            +
       mean gene pooling

Therefore:

    768 × 4 = 3072

## What can we test?

Cell embeddings are suitable for:

- cell-type classification;
- clustering;
- cell similarity;
- nearest-neighbor retrieval;
- PCA/UMAP visualization of embeddings;
- comparison of biological states;
- downstream predictive models.

Example:

    scRNA-seq
        ↓
    scFoundation
        ↓
    N × 3072
        ↓
    PCA / UMAP
        ↓
    color by known cell type

If the representation contains meaningful cell biology, cells of related types
should show structured relationships.

---

# 13. Gene Embedding

## Command option

    --output_type gene

Example:

    python get_embedding.py \
        --task_name example_gene \
        --input_type singlecell \
        --output_type gene \
        --tgthighres a5 \
        --data_path DATA.csv \
        --save_path ./outputs/ \
        --pre_normalized F

## Meaning

Instead of compressing the entire cell into one vector, scFoundation retains
gene-level representations.

Conceptually:

    Cell 1
        TP53  → vector
        EGFR  → vector
        KRAS  → vector
        CD3D  → vector
        ...
    
    Cell 2
        TP53  → vector
        EGFR  → vector
        KRAS  → vector
        CD3D  → vector
        ...

The same gene can therefore have a different representation in different
cells.

For example:

    TP53 in Cell A
        ↓
    [gene representation A]

    TP53 in Cell B
        ↓
    [gene representation B]

The difference can encode the surrounding transcriptional context.

## Potential uses

Gene-level representations can be investigated for:

- gene similarity;
- pathway structure;
- gene modules;
- cell-state-specific gene relationships;
- perturbation biology;
- target-context analysis.

---

# 14. Contextual Gene Embedding

There is an important terminology detail.

The official CLI does not provide:

    --output_type contextual_gene

Instead it exposes:

    gene
    gene_batch

The gene representations generated by the Transformer are already contextual
because they are produced after processing expression information from a cell.

Therefore we should distinguish:

    Gene representation from individual-cell processing

and:

    Batched gene representation

rather than assuming the repository defines a completely separate embedding
called "contextual gene embedding".

---

# 15. `gene_batch`

## Command option

    --output_type gene_batch

Example:

    python get_embedding.py \
        --task_name example_gene_batch \
        --input_type singlecell \
        --output_type gene_batch \
        --tgthighres a5 \
        --data_path DATA.csv \
        --save_path ./outputs/ \
        --pre_normalized F

## Difference from `gene`

`gene` processes gene representations on an individual-cell basis.

`gene_batch` processes supplied cells together as a batch before retaining the
gene-level output.

The official repository uses this mode in its GEARS integration.

The documentation recommends keeping the number of cells small for this mode,
with no more than approximately five cells in the batch.

Therefore:

    gene
        → individual-cell gene representation

    gene_batch
        → batched gene-level representation
        → relevant to GEARS / perturbation workflow

We should not automatically treat `gene_batch` as a completely separate
biological embedding type without examining the downstream workflow.

---

# 16. Three Representations Side by Side

| Representation | CLI | Approximate output | Biological unit | Useful for |
|----------------|-----|-------------------|-----------------|------------|
| Cell embedding | `cell` | N × 3072 with `pool_type=all` | Entire cell | cell type, clustering, similarity |
| Gene embedding | `gene` | N × 19264 × hidden representation | Gene within a cell | gene relationships, pathways |
| Batched/contextual gene representation | `gene_batch` | gene-level representation for small batches | Genes across batched cells | GEARS/perturbation workflows |

---

# 17. Important Inference Parameters

| Parameter | Important values | Meaning |
|-----------|-----------------|---------|
| `--input_type` | `singlecell`, `bulk` | Type of expression data |
| `--output_type` | `cell`, `gene`, `gene_batch` | Representation to extract |
| `--pool_type` | `all`, `max` | Cell-representation pooling |
| `--tgthighres` | `t...`, `f...`, `a...` | Target resolution specification |
| `--pre_normalized` | `F`, `T`, `A` | State of input preprocessing |
| `--version` | `ce`, task-specific alternatives | Model/inference mode |
| `--data_path` | path | Expression input |
| `--save_path` | path | Embedding output |
| `--model_path` | path | Optional model location |
| `--ckpt_name` | checkpoint name | Model checkpoint selection |

---

# 18. Our Successful Smoke Test

Environment:

    Python 3.10
    PyTorch 2.6.0+cu124
    NVIDIA RTX A6000
    ~45 GB VRAM

Checkpoint:

    models/models.ckpt

Checkpoint validation:

    ZIP valid: True
    entries: 797
    corrupt entry: None

Synthetic input:

    5 cells × 19,264 genes

Command:

    --input_type singlecell
    --output_type cell
    --pool_type all
    --tgthighres a5
    --pre_normalized F
    --version ce

Output:

    shape = (5, 3072)
    dtype = float32

Statistics:

    min  = -3.7171419
    max  = 4.3378673
    mean = 0.16202562
    std  = 1.099519

    NaN = 0
    Inf = 0

This confirms that:

    checkpoint loading      works
    CUDA inference          works
    19,264-gene conversion  works
    preprocessing           executes
    Transformer inference   works
    cell embedding output   works

The synthetic data should not be used to evaluate biological performance.

---

# 19. Correct Validation Strategy

The next evaluation should use real single-cell data with known biological
labels.

Before running the model, determine:

    1. What organism is represented?
    2. Are gene identifiers gene symbols, Ensembl IDs, or something else?
    3. Does the matrix contain raw counts?
    4. Has normalize_total already been applied?
    5. Has log1p already been applied?
    6. Are cells rows and genes columns?
    7. How many genes overlap the 19,264-gene vocabulary?
    8. What cell-type/state labels are available?

Only after answering these questions should `pre_normalized` and other
inference parameters be selected.

---

# 20. Proposed Single-Cell Evaluation

Use one real annotated dataset and generate all relevant representations from
the same cells.

                            Real scRNA-seq
                                  │
                        inspect preprocessing
                                  │
                    map to 19,264-gene vocabulary
                                  │
                             scFoundation
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
                ▼                 ▼                 ▼
              CELL              GENE           GENE_BATCH
                │                 │                 │
             N×