# aida-celltypist-bias

Does the ethnic composition of a single-cell reference atlas affect CellTypist annotation accuracy for underrepresented populations?

## Background

Yang et al. (2026) documented significant demographic skews in major single-cell atlases — European ancestry is overrepresented across HCA, HTAN, and PsychAD relative to reference populations ([*Cell Genomics*](https://doi.org/10.1016/j.xgen.2026.101300); see also [my reproduction and reanalysis](https://github.com/andytsmd14-code/single-cell-omics-reproduction)). This raises a downstream question their paper did not address: does this demographic imbalance actually affect cell type annotation accuracy?

I test this directly using the **AIDA dataset** (Kock et al., *Cell* 2025), the first large-scale Asian immune cell atlas with over 1.26M PBMCs from six distinct ethnic groups. Reference-based tools like CellTypist train a classifier on a labeled reference and apply it to a query — so a reference mismatched to the query population could hurt accuracy. But does this hold even *within* a broadly matched population? Even when both reference and query are Asian, does the specific ethnic subgroup composition still matter? I systematically vary the proportion of Singaporean Chinese donors in an all-Asian reference panel and measure the downstream effect on annotation accuracy.

---

## Experiment

- **Query**: 20 Singaporean Chinese donors, fixed (`seed=42`)
- **Reference**: 60 donors per run; SG_Chinese proportion swept from 0% to 100% in 10% steps
- **Repeats**: 36 independent runs per proportion (396 total)
- **Tool**: CellTypist trained on 2,000 highly variable genes
- **Metric**: Macro-F1, per-cell-type F1/precision/recall

---

## Results

### Overall macro-F1

| SG_Chinese % | 0% | 10% | 20% | 30% | 40% | 50% | 60% | 70% | 80% | 90% | 100% |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Mean Macro-F1 | 0.811 | 0.813 | 0.818 | 0.818 | 0.821 | 0.816 | 0.819 | 0.814 | 0.815 | 0.821 | 0.822 |

A modest but consistent improvement (+1.1 pp from 0% to 100%) across the majority of replicates.

### Per-cell-type highlights (Δ F1 = 100% minus 0% SG_Chinese)

| Cell Type | Δ F1 |
|---|---|
| double negative T regulatory cell | +0.140 |
| innate lymphoid cell | +0.132 |
| natural killer cell | +0.074 |
| CD4-positive, alpha-beta T cell | +0.041 |
| B cell | −0.050 |
| pre-conventional dendritic cell | −0.035 |

Rare cell types show the largest F1 changes in either direction, reflecting high variance from small sample sizes rather than a consistent benefit from matched reference representation.

### The B cell paradox

B cell F1 paradoxically *decreases* with more SG_Chinese reference. CellTypist trains a one-vs-rest logistic regression on Level-1 `cell_type` labels — it never sees Level-4 annotations. The F1 change is therefore driven entirely by differences in gene expression among cells sharing the same Level-1 label across ethnic groups.

Inspecting Level-4 annotations reveals the mechanism: all 75 query B cells and all 8,048 reference B cells across ethnic groups are QC-flagged — but the *type* of artifact differs. SG_Chinese B cells are dominated by `flagged_platelet_sum` (35%) — cells with heavy platelet gene contamination. Non-SG_Chinese B cells are dominated by `B_unknown` (58%) — ambiguous cells that retain partial B cell gene expression. When more SG_Chinese donors enter the reference, the "B cell" class becomes increasingly defined by platelet gene signatures. This platelet signal overlaps with other contaminated cell types in the query, causing the classifier's B cell boundary to become less discriminative. The non-SG_Chinese reference (B_unknown-heavy) incidentally produces a more useful training signal for identifying even the artifact-ridden SG_Chinese query B cells.

**This is a reminder that benchmarking annotation tools against atlas-derived labels requires inspecting annotation quality at all hierarchy levels** — and that matched ethnicity in the reference is not always sufficient if the ground truth labels themselves are dominated by non-representative artifacts.

### NK cells, dnT cells, and ILCs

NK cells follow the opposite pattern (+0.074) for the same mechanistic reason. SG_Chinese NK cells in both reference and query are exclusively `flagged_NK_low_exp` and `flagged_platelet_sum` — the Level-4 composition is ethnicity-consistent. Other ethnicities' NK cells are dominated by `NK_unknown` (44%), a distinct gene expression profile. With more SG_Chinese donors in the reference, the NK class gene expression distribution better matches the query NK cells, and the logistic regression learns a boundary that captures them more accurately.

**Double negative T regulatory cells** (4 cells) and **innate lymphoid cells** (16 cells) show large F1 changes (+0.140 and +0.132 respectively), but these should be interpreted with caution: with so few query cells, F1 is highly sensitive to which specific cells happen to be correctly or incorrectly classified in each run. The large Δ F1 reflects high variance from small sample sizes rather than a robust signal of population-level benefit.

---

## Data

`AIDA.h5ad` is not included due to file size. Download from [CellxGene](https://cellxgene.cziscience.com/) and place in the project root.

> Kock et al. "Asian Immune Diversity Atlas reveals distinct immune cell distributions across ethnic groups." *Cell* (2025).

---

## Setup

```bash
pip install -r requirements.txt
jupyter notebook
```

The long training cell in `aida_sg_chinese_full_curve.ipynb` is marked with a ⚠️ warning — skip it. Pre-computed results in `output/` let you run all analysis cells immediately.

---

## Notebooks

| Notebook | Description |
|---|---|
| `aida_sg_chinese_full_curve.ipynb` | Main experiment: macro-F1 dose-response curve |
| `aida_bcell.ipynb` | Why B cell F1 decreases with more SG_Chinese reference |
| `aida_nk_other_significant.ipynb` | NK, dnT, and ILC annotation quality analysis |
