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
| pre-conventional dendritic cell | −0.035 (n.s.) |

Rare cell types show the largest F1 changes in either direction, reflecting high variance from small sample sizes rather than a consistent benefit from matched reference representation. Pre-conv DC Δ F1 = −0.035 is not supported by a statistically significant Kendall τ (τ = 0.127, p = 0.648).

### The B cell paradox

B cell F1 paradoxically decreases with more SG_Chinese reference.  SG_Chinese donors contribute only 4.8% B cells to the reference pool, versus 11.6% in other-ethnicity donors (2.4×). As SG_Chinese proportion rises, the absolute number of “B cell” training examples falls, classifier recall weakens, and F1 drops — even though the reference is ethnically closer to the query.
**This is a reminder that matched ethnicity in the reference does not guarantee matched training-example availability for each cell type.**

### NK cells, dnT cells, and ILCs

NK cells follow the opposite pattern (+0.074). Level-4 annotations, used here as a diagnostic tool, reveal the transcriptomic contrast: query NK cells are ~60% `flagged_NK_low_exp` and 0% `NK_unknown`; SG_Chinese reference NK cells are similarly ~68% `flagged_NK_low_exp`. Other ethnicities' NK cells are 44% `NK_unknown` — a distinct profile with lower canonical NK marker expression (NKG7, GNLY, KLRD1) and elevated MALAT1/NEAT1 per AIDA STAR Methods. When other-ethnicity donors dominate the reference, the training class skews toward the NK_unknown profile, which does not match the query; more SG_Chinese reference shifts it toward the flagged_NK_low_exp profile shared by the query, and F1 improves.

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
| `aida_bcell_analysis.ipynb` | Why B cell F1 decreases with more SG_Chinese reference |
| `aida_nk_analysis.ipynb` | NK cell annotation quality and F1 trend analysis |
| `aida_rare_cells_analysis.ipynb` | ILC, dnT, and pre-conv DC: rare cell type F1 analysis |
