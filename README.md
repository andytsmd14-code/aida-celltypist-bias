# aida-celltypist-bias

Does the ethnic composition of a single-cell reference atlas affect CellTypist annotation accuracy, and does this effect generalise across Asian ethnic groups?

## Background

Yang et al. (2026) documented significant demographic skews in major single-cell atlases — European ancestry is overrepresented across HCA, HTAN, and PsychAD relative to reference populations ([*Cell Genomics*](https://doi.org/10.1016/j.xgen.2026.101300); see also [my reproduction and reanalysis](https://github.com/andytsmd14-code/single-cell-omics-reproduction)). This raises a downstream question their paper did not address: does this demographic imbalance actually affect cell type annotation accuracy?

I test this directly using the **AIDA dataset** (Kock et al., *Cell* 2025), the first large-scale Asian immune cell atlas with over 1.26M PBMCs from six distinct ethnic groups. Reference-based tools like CellTypist train a classifier on a labeled reference and apply it to a query — so a reference mismatched to the query population could hurt accuracy. The key question: even when both reference and query are broadly Asian, does the specific ethnic subgroup composition still matter?

I systematically vary the same-ethnicity donor proportion in an all-Asian reference panel across **six ethnic cohorts** and measure the downstream effect on CellTypist annotation accuracy. This extends the initial SG Chinese experiment to test whether the dose-response relationship is a general property of ethnically matched references or a cohort-specific finding.

---

## Experiment

**Per-cohort design (identical across all six cohorts):**

- **Query**: 20 same-ethnicity donors, fixed (`seed=42`)
- **Reference**: 60 donors per run; same-ethnicity proportion swept from 0% to 100% in steps of `1/n_donors`
- **Repeats**: multiple independent runs per proportion point (see table below)
- **Tool**: CellTypist trained on 2,000 highly variable genes
- **Metric**: Macro-F1 and per-cell-type F1/precision/recall

**Current status:**

| Cohort | Runs | Proportions covered | F1 at 0% | F1 at max | ΔF1 |
|---|---|---|---|---|---|
| SG Chinese | 36 | 0–100% ✓ | 0.811 | 0.822 | +0.011 |
| Japanese | 18 | 0–100% ✓ | 0.799 | 0.819 | +0.020 |
| Korean | 16 | 0–100% ✓ | 0.808 | 0.819 | +0.012 |
| Thai | 16 | 0–60%  | 0.782 | 0.806† | +0.025† |
| Singaporean Malay | 8 | 0–60% | 0.801 | 0.822† | +0.021† |
| Singaporean Indian | 8 | 0–80%  | 0.797 | 0.809† | +0.012† |

† measured at the highest proportion run so far, not necessarily at 100%.

> **Note on the Indian cohort:** The AIDA Indian donors available for reference construction are limited, yielding only two feasible proportion points (0% and 10%). The resulting F1 values are anomalously low (~0.09–0.17) and the data are excluded from all cross-cohort analyses until the sampling issue is resolved.

---

## Results

### A consistent positive dose-response across cohorts

Across the three complete cohorts and two of the partial ones, macro-F1 increases monotonically as the same-ethnicity proportion in the reference rises. The improvement is modest but reproducible: **+1.1 pp to +2.5 pp** from 0% to max same-ethnicity proportion. Thai shows the largest gain so far; Japanese the most proportions available beyond SG Chinese.

### Donor-weighted ΔF1 by cell type

Pooling all six cohorts (weighted by number of reference donors: SG Chinese = 10, Japanese = 10, Korean = 10, Singaporean Indian = 8, Thai = 6, Singaporean Malay = 6), the cell types most consistently affected across ethnicities are:

| Cell Type | Weighted Mean ΔF1 |
|---|---|
| natural killer cell | +0.085 |
| T cell | +0.060 |
| double negative T regulatory cell | +0.054 |
| hematopoietic stem cell | +0.052 |
| innate lymphoid cell | +0.042 |
| CD4-positive, alpha-beta T cell | +0.037 |
| B cell | +0.021 |
| platelet | −0.005 |

NK cells, rare T cell subsets, and hematopoietic stem cells show the largest and most consistent benefit from ethnically matched references. The B cell signal is small and positive in aggregate, though SG Chinese is an exception (see below).

### Notable per-cohort findings

**The B cell paradox (SG Chinese):** B cell F1 paradoxically *decreases* (Δ = −0.050) as more SG Chinese donors enter the reference. SG Chinese donors contribute only 4.8% B cells versus 11.6% in other-ethnicity donors (2.4×). As the same-ethnicity proportion rises, the absolute number of B cell training examples falls, the classifier's recall weakens, and F1 drops even though the reference is ethnically closer to the query. This effect does not appear in other cohorts (Korean B cell: +0.090), where the same-ethnicity B cell representation is not comparably depleted. Detailed misclassification analysis in `aida_bcell_analysis.ipynb`.

**NK cells — ethnicity-specific expression profiles matter:** NK cell F1 improves in all cohorts (+0.074 SG Chinese, +0.041 Japanese, +0.147 Korean, +0.121 Singaporean Malay). Level-4 annotations reveal that query NK cells cluster predominantly as `flagged_NK_low_exp`, a profile shared by same-ethnicity reference donors but not by other-ethnicity donors (which skew toward `NK_unknown`, a transcriptomically distinct subtype with lower NKG7/GNLY/KLRD1 expression). Detailed analysis in `aida_nk_analysis.ipynb`.

**Rare cell types — interpret with caution:** Double negative T regulatory cells (4 query cells) and innate lymphoid cells (16 cells) show large ΔF1 in multiple cohorts (+0.054 weighted mean), but cell counts are too small for the signal to be robust. F1 for these types is highly sensitive to individual misclassifications. `aida_rare_cells_analysis.ipynb` provides per-run diagnostics.

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

The long training cell in each ethnicity notebook is marked with a ⚠️ warning — skip it. Pre-computed results in `outputs/<ethnicity>/` let you run all analysis cells immediately.

---

## Notebooks

### Dose-response experiments (one per cohort)

| Notebook | Cohort | Status |
|---|---|---|
| `aida_sg_chinese_full_curve.ipynb` | SG Chinese | Complete (0–100%) |
| `aida_japanese_full_curve.ipynb` | Japanese | Complete (0–100%) |
| `aida_korean_full_curve.ipynb` | Korean | Complete (0–100%) |
| `aida_thai_full_curve.ipynb` | Thai | (0–60%) |
| `aida_singaporean_malay_full_curve.ipynb` | Singaporean Malay | (0–60%) |
| `aida_singaporean_indian_full_curve.ipynb` | Singaporean Indian | (0–80%) |

### Cross-cohort summary

| Notebook | Description |
|---|---|
| `aida_summary_analysis.ipynb` | Loads all results, comparative dose-response curve, ΔF1 heatmap, donor-weighted cell-type ranking |

### Cell-type deep dives

| Notebook | Description |
|---|---|
| `aida_bcell_analysis.ipynb` | Why B cell F1 decreases with more SG Chinese reference (training-example depletion) |
| `aida_nk_analysis.ipynb` | NK cell annotation quality and ethnicity-specific expression subtypes |
| `aida_rare_cells_analysis.ipynb` | ILC, dnT, and pre-conv DC: rare cell type F1 analysis and per-run diagnostics |
