# Macroeconomic Shock–Response Functions of Health System Resilience and Equity: A Global Simulation Study

**Shekoofeh Sadat Momahhed & Atefehsadat Haghighathoseini**

## Overview

This repository contains all data and code needed to reproduce the empirical results of the paper
*"Macroeconomic Shock–Response Functions of Health System Resilience and Equity: A Global Simulation Study"*.

The study quantifies how macroeconomic disturbances — such as recessions, inflationary episodes, and
labour-market shocks — propagate into health system resilience and health equity across 205 countries
from 2000 to 2022. Using a unified index-based stress-testing framework, we simulate how changes in a
composite Economic Shock Index translate into within-country deviations of a Health System Resilience
Index and a Health Equity Index, under six symmetric hypothetical shock magnitudes (±5%, ±10%, ±20%).

The analysis is structured as a sequential pipeline — from raw World Bank data through index construction,
shock simulation, primary response estimation, and robustness checks — with each stage corresponding to a
dedicated Python script. Every script can be run independently in the order described below, as long as
the input file from the preceding stage is present.


## Repository Structure

```
├── data/
│   ├── raw/
│   │   └── merged_health_economic_income_data.csv   ← Starting point (raw World Bank download)
│   └── processed/
│       ├── cleaned_normalized_data.csv              ← Output of Script 1
│       ├── cleaned_normalized_data_with_ESI.csv     ← Output of Scripts 2–3
│       ├── HSR_calculated.csv                       ← Output of Scripts 4–5
│       ├── HEI_calculated.csv                       ← Output of Scripts 6–7
│       └── All_indices.csv                          ← Combined ESI + HSR + HEI (input to Stage 2)
├── scripts/
│   ├── 01_data_cleaning.py                          ← Data preprocessing and normalisation
│   ├── 02_esi_weight_estimation.py                  ← PCA weight estimation for ESI
│   ├── 03_esi_calculation.py                        ← ESI index construction
│   ├── 04_hsr_weight_estimation.py                  ← PCA weight estimation for HSR
│   ├── 05_hsr_calculation.py                        ← HSR index construction
│   ├── 06_hei_weight_estimation.py                  ← PCA weight estimation for HEI
│   ├── 07_hei_calculation.py                        ← HEI index construction
│   ├── 08_esi_shock_simulation.py                   ← Stage 1: ESI shock propagation (Table 3)
│   ├── 09_esi_shock_visualisation.py                ← Figure 3: ESI deviation distributions
│   ├── 10_primary_response_estimation.py            ← Stage 2 primary: ΔHSR & ΔHEI (Table 4)
│   ├── 11_descriptive_statistics.py                 ← Descriptive stats by region (Tables 1–2, Figures 1–2)
│   ├── 12_direct_shock_response.py                  ← Alternative Stage 2 response estimation
│   ├── 13_robustness_elasticity_rf_xgboost.py       ← Robustness checks with CIs (Table 5)
│   ├── 14_robustness_visualisation.py               ← Figures 4–5: Shock-response curves with CI bands
│   └── Economic_Shocks_3.ipynb                      ← Original monolithic notebook (for reference)
├── outputs/
│   ├── tables/                                      ← All CSV tables produced by scripts
│   └── figures/                                     ← All PNG figures (300 dpi)
├── requirements.txt
└── README.md
```

> **Note on the notebook.** The file `Economic_Shocks_3.ipynb` is preserved in the repository as the
> original single-file analysis. The numbered scripts in `scripts/` are the split, cleaned version
> that reviewers recommended. Both are fully equivalent; the scripts are recommended for reproducibility
> and reuse.



## Data

All data used in this study are drawn from the **World Bank Open Data** platform and are freely available
with no access restrictions. The raw data file included in `data/raw/` was downloaded directly from:

- **World Bank Data Portal:** [https://data.worldbank.org/](https://data.worldbank.org/)

The merged raw file (`merged_health_economic_income_data.csv`) contains the following indicators for up
to 205 countries over 2000–2022:

| Indicator | Category | Source |
|---|---|---|
| GDP growth (annual %) | Macroeconomic | World Bank WDI |
| Unemployment, total (% of labour force, modelled ILO) | Macroeconomic | World Bank WDI |
| Inflation, consumer prices (annual %) | Macroeconomic | World Bank WDI |
| Adjusted net national income per capita (current US$) | Economic capacity | World Bank WDI |
| Out-of-pocket expenditure (% of current health expenditure) | Health system | World Bank Health Nutrition & Population |
| Hospital beds (per 1,000 people) | Health system | World Bank Health Nutrition & Population |
| Current health expenditure (% of GDP) | Health system | World Bank Health Nutrition & Population |

To update the data or extend the time period, download fresh indicator files from the World Bank portal
and re-run the pipeline from Script 1.

---

## Installation

### Requirements

Python 3.9 or later is required. Install all dependencies with:

```bash
pip install -r requirements.txt
```

The key packages are:

```
pandas>=1.5
numpy>=1.23
scikit-learn>=1.1
xgboost>=1.7
matplotlib>=3.6
seaborn>=0.12
scipy>=1.9
joblib>=1.2
tqdm>=4.64
```



## Running the Pipeline

The analysis is organised as a **sequential pipeline** with three conceptual stages. Each script must be
run in order, because each one produces the input file consumed by the next. Processed files are saved to
`data/processed/` and figures to `outputs/figures/`. You can run all scripts from the repository root.

### Quick start (all scripts in sequence)

```bash
cd scripts/
python 01_data_cleaning.py
python 02_esi_weight_estimation.py
python 03_esi_calculation.py
python 04_hsr_weight_estimation.py
python 05_hsr_calculation.py
python 06_hei_weight_estimation.py
python 07_hei_calculation.py
python 08_esi_shock_simulation.py
python 09_esi_shock_visualisation.py
python 10_primary_response_estimation.py
python 11_descriptive_statistics.py
python 12_direct_shock_response.py
python 13_robustness_elasticity_rf_xgboost.py
python 14_robustness_visualisation.py
```

> ⏱ **Expected runtime:** Scripts 1–12 and 14 run in under two minutes each on a standard laptop.
> Script 13 (robustness estimation with 1,000 bootstrap replications for Random Forest and XGBoost)
> takes approximately 3–6 minutes with parallel processing enabled; see the note in that script.



## Stage-by-Stage Narrative

The following describes what each script does, why it exists, and what it produces. Reading this section
gives you a complete understanding of how the paper's results are generated, in the order they appear
in the manuscript.



### Stage 1 — Data Preparation and Index Construction (Scripts 1–7)

The goal of Stage 1 is to take raw, heterogeneous World Bank panel data covering 205 countries and
23 years, clean and normalise it, and then construct three composite indices: the Economic Shock Index,
the Health System Resilience Index, and the Health Equity Index. Each index uses PCA-derived,
bootstrap-validated weights to ensure that the relative contribution of each component is determined
by the data rather than by arbitrary subjective choices.

---

#### Script 1 — `01_data_cleaning.py`

**What it does.**
This is the entry point of the entire pipeline. It loads the raw merged dataset
(`merged_health_economic_income_data.csv`) and implements a multi-step preprocessing pipeline:

1. Selects the nine core raw indicator variables and drops all pre-processed or normalised columns
   to avoid contaminating downstream transformations.
2. Reshapes the data into a strict country–year panel format and enforces temporal ordering within
   each country.
3. Flags and removes structurally implausible observations (e.g., negative health expenditure).
4. Excludes indicators with more than 30% missingness over the full 2000–2022 period.
5. Applies a two-step within-country imputation strategy: backward filling for gaps at the beginning
   of country time series, followed by linear interpolation for short internal gaps (maximum two
   consecutive missing years).
6. Winsorises all retained indicators at the 1st and 99th percentiles to limit the influence of
   extreme outliers without discarding genuine cross-country variation.
7. Applies global min–max normalisation to scale every indicator to [0, 1], storing the
   normalisation parameters for later use across shock scenarios.
8. Inverts indicators where higher values reflect worse outcomes (out-of-pocket expenditure,
   unemployment, inflation), so that higher normalised values consistently indicate more favourable
   conditions across all indices.

**Why it matters.**
All subsequent analysis depends on this cleaned, normalised panel. The normalisation parameters saved
here are reused in every shock simulation, ensuring that counterfactual ESI values under shocked
macroeconomic conditions are directly comparable to baseline values.

**Output:** `data/processed/cleaned_normalized_data.csv`



#### Script 2 — `02_esi_weight_estimation.py`

**What it does.**
This script estimates the PCA-derived weights used to aggregate the four Economic Shock Index
components (GDP growth, unemployment, inflation, adjusted net national income per capita). It:

1. Extracts the four normalised ESI component columns from the cleaned dataset.
2. Applies PCA to the component matrix; the first principal component is retained as it captures
   the dominant latent dimension of macroeconomic shock intensity.
3. Normalises PCA loadings to sum to one, yielding the baseline component weights.
4. Runs 1,000 bootstrap replications in which PCA is re-estimated on resampled datasets, producing
   empirical distributions of weights to confirm stability.
5. Reports 95% bootstrap confidence intervals for each weight and the average CI width.
6. Conducts deterministic sensitivity analysis by perturbing each weight by ±10%.

**Why it matters.**
Using data-driven PCA weights rather than equal or subjective weights ensures that each component
contributes proportionally to its explanatory power in the data. The bootstrap validation addresses
the common concern that composite index weights are arbitrary.

**Output:** Weight values and bootstrap diagnostics (printed and saved to `outputs/tables/`);
cleaned data carried forward to Script 3.

---

#### Script 3 — `03_esi_calculation.py`

**What it does.**
Using the validated bootstrap weights from Script 2 (GDP growth: 0.294; unemployment: 0.053;
inflation: 0.308; net national income per capita: 0.345), this script computes the Economic Shock
Index for each country–year observation in the cleaned dataset. The formula applied is:

```
ESI = 0.294 × GDP_norm + 0.053 × (1 − Unemployment_norm) + 0.308 × (1 − Inflation_norm) + 0.345 × NNI_norm
```

It reports descriptive statistics for the resulting ESI distribution, identifies the top and bottom
ten countries by average ESI in the most recent available year, and saves the full dataset with the
ESI column appended.

**Output:** `data/processed/cleaned_normalized_data_with_ESI.csv`

---

#### Scripts 4–5 — `04_hsr_weight_estimation.py` and `05_hsr_calculation.py`

**What they do.**
These two scripts mirror the logic of Scripts 2–3 but for the Health System Resilience Index. Script 4
estimates PCA weights for the three resilience components — inverted out-of-pocket expenditure (weight:
0.332), hospital bed density (0.325), and current health expenditure (0.343) — using the same
1,000-replication bootstrap approach. Script 5 applies these weights to compute the index, generates
descriptive statistics, identifies the highest- and lowest-resilience countries, and visualises the
distribution and time trend of the index.

**Why it matters.**
The Health System Resilience Index is the first primary outcome variable of the study. Constructing it
separately from the macroeconomic index and validating its weights independently confirms that the two
indices are empirically distinguishable before shock propagation is modelled.

**Outputs:**
`data/processed/HSR_calculated.csv`, `outputs/figures/HSR_distribution.png`,
`outputs/figures/HSR_by_country.png`

---

#### Scripts 6–7 — `06_hei_weight_estimation.py` and `07_hei_calculation.py`

**What they do.**
These scripts follow the same structure for the Health Equity Index, which adds a fourth component —
adjusted net national income per capita — to the three shared with the resilience index. The
PCA-derived weights are: inverted out-of-pocket expenditure (0.254), current health expenditure
(0.256), hospital bed density (0.216), and net national income per capita (0.275). Labour-market and
inflation indicators are intentionally excluded to prevent mechanical correlation with the Economic
Shock Index.

**Why it matters.**
The Health Equity Index is the second primary outcome variable. Distinguishing it from the resilience
index both conceptually and empirically — it emphasises distributional financing access rather than
aggregate structural capacity — is a core design choice of the study.

**Output:** `data/processed/HEI_calculated.csv`

---

### Stage 2 — Shock Simulation and Descriptive Analysis (Scripts 8–12)

At this point all three composite indices have been constructed and validated. Stage 2 introduces the
simulation framework. Macroeconomic shocks are applied to the GDP growth, unemployment, and inflation
components of the Economic Shock Index, and the resulting within-country deviations in all three indices
are computed and visualised. The primary response estimates for the paper's Table 4 are produced here.

---

#### Script 8 — `08_esi_shock_simulation.py`

**What it does.**
This script implements Stage 1 of the two-stage simulation framework. For each of the six shock
magnitudes (−20%, −10%, −5%, +5%, +10%, +20%), it applies proportional deviations to GDP growth,
unemployment, and inflation simultaneously, holds adjusted net national income per capita constant
(reflecting its role as a slow-moving structural variable), re-normalises all shocked variables using
the same global min–max parameters saved in Script 1, and recomputes the Economic Shock Index with
the original PCA-derived weights. Within-country deviations (ΔESI) are computed for each observation,
and summary statistics (mean, standard deviation, P10, P25, median, P90) are reported across all
3,009 country–year observations. These statistics appear in Table 3 of the manuscript.

**Output:** Summary statistics table (Table 3), saved to `outputs/tables/esi_shock_summary.csv`

---

#### Script 9 — `09_esi_shock_visualisation.py`

**What it does.**
Produces three versions of the kernel density figure showing the full cross-country distribution of
ΔESI under each shock scenario: a plain overlay of six density curves (Figure 3A), a violin plot
showing distributional shape (Figure 3B), and a filled kernel density version (Figure 3C). These
figures establish the quality and breadth of within-country input variation used for identification
in Stage 2.

**Output:** `outputs/figures/figure3_ESI_deviation_distributions.png` and variants

---

#### Script 10 — `10_primary_response_estimation.py`

**What it does.**
This is the most important script in the repository. It produces the **primary empirical results** of
the paper (Table 4). Using the combined All_indices.csv file that contains all three constructed
indices alongside their normalised components, it computes ΔHSR and ΔHEI for each shock scenario by
direct arithmetic decomposition — that is, by recomputing the resilience and equity index values under
shocked macroeconomic conditions and subtracting the baseline values. No regression model is fitted
here; the response is calculated directly from the index definitions, which is why this is the primary
specification. The script then runs 1,000-replication bootstrap resampling of country–year observations
to compute 95% confidence intervals for the mean within-country responses, and tests whether all
intervals exclude zero.

**Output:** `outputs/tables/primary_shock_response_table4.csv` (Table 4 in the manuscript)

---

#### Script 11 — `11_descriptive_statistics.py`

**What it does.**
This script produces the descriptive baseline section of the Results (Tables 1 and 2, Figures 1 and 2).
It loads All_indices.csv, merges a built-in World Bank regional classification covering all 205
economies, and generates four outputs: a regional averages table (mean and SD of ESI, HSR, and HEI
by region), a country snapshot table (top-5 and bottom-5 countries per index), a grouped regional bar
chart with ±1 SD error bars, and a three-panel time-series figure showing global average trends with
±1 SD shaded bands from 2000 to 2022, with the 2008–2009 Global Financial Crisis and the 2020
COVID-19 pandemic highlighted as reference events.

**Output:**
`outputs/tables/descriptive_table_regional.csv` (Table 1),
`outputs/tables/descriptive_table_country_snapshot.csv` (Table 2),
`outputs/figures/figure1_regional_bar_chart.png` (Figure 1),
`outputs/figures/figure2_global_timeseries.png` (Figure 2)

---

#### Script 12 — `12_direct_shock_response.py`

**What it does.**
This script provides an alternative implementation of the Stage 2 response estimation, applying the
shock–response logic through an asymmetric multiplier approach that models differential sensitivity
of out-of-pocket expenditure relative to slower-adjusting components such as hospital beds and total
health expenditure. It also computes bootstrap confidence intervals (1,000 replications) and generates
shock–response curves for both the resilience and equity indices. This script was used during analysis
to cross-validate the primary results from Script 10.

**Output:** Supplementary shock–response tables and figures

---

### Stage 3 — Robustness Checks (Scripts 13–14)

Stage 3 estimates and visualises the three alternative modelling specifications — linear elasticity,
Random Forest regression, and XGBoost — that serve as robustness checks for the primary results.
These specifications are not the primary analysis but are reported in Table 5 of the manuscript to
confirm that qualitative patterns hold across independent modelling approaches and to characterise
nonlinearities that the direct simulation cannot capture.

---

#### Script 13 — `13_robustness_elasticity_rf_xgboost.py`

**What it does.**
This is the computationally heaviest script in the pipeline. It implements the three robustness
specifications on the All_indices.csv dataset, using the Economic Shock Index as the sole predictor
and the resilience and equity indices as outcomes. The linear elasticity model fits an ordinary least-
squares regression; Random Forest uses 500 trees, maximum depth 6, and bootstrap aggregation; XGBoost
uses 500 trees, maximum depth 4, learning rate 0.05, subsampling 0.80, and column subsampling 0.80.
All three models are trained on the full sample, and counterfactual predictions under shocked Economic
Shock Index values are subtracted from baseline predictions to obtain within-country deviations.

For the 95% confidence intervals, 1,000 bootstrap replications are run in parallel across all available
CPU cores using `joblib.Parallel`. In each replication, all three models are refit on a resampled
dataset, and counterfactual predictions are recomputed. Bootstrap workers for the tree-based models
use 100 trees rather than 500 (statistically equivalent for CI estimation on a single-predictor
problem but approximately five times faster), and `n_jobs=1` is set inside each worker to avoid
thread contention with the outer parallel pool. On a four-core laptop, the full procedure completes in
approximately 3–6 minutes.

The script also computes out-of-sample R² diagnostics for all six model–outcome combinations and saves
full model diagnostics to the Supplementary Appendix folder.

**Output:**
`outputs/tables/shock_response_results.csv` (full numeric results with CI bounds),
`outputs/tables/shock_response_table5_with_CI.csv` (manuscript-ready Table 5),
`outputs/tables/model_diagnostics.csv` (R² scores)

---

#### Script 14 — `14_robustness_visualisation.py`

**What it does.**
Reads the output of Script 13 and produces the four robustness visualisation figures for the manuscript.
All figures display 95% bootstrap confidence bands as shaded regions around each model's curve:
Figure 4 shows the Health System Resilience shock–response curves for all three models; Figure 5 shows
the Health Equity Index curves; a four-panel combined figure (Panels A–D) shows both outcomes alongside
panels C and D quantifying nonlinearity as the deviation of Random Forest and XGBoost estimates from
the linear elasticity benchmark. These panels reveal that departures from linearity are concentrated at
the tails of the shock distribution (±20% scenarios) and converge to near-zero for moderate shocks
(±5% to ±10%), confirming that linear models remain adequate for routine scenario planning while
machine-learning approaches add diagnostic value specifically under extreme stress.

**Output:**
`outputs/figures/figure4_HSR_robustness_curves.png`,
`outputs/figures/figure5_HEI_robustness_curves.png`,
`outputs/figures/figure_combined_4panel.png`

---

## File Flow Summary

The diagram below shows how input files flow through the pipeline. Each arrow represents a
script's read/write relationship. All processed files are stored in `data/processed/`.

```
merged_health_economic_income_data.csv
        │
        ▼  Script 01
cleaned_normalized_data.csv
        │
        ├──▶ Script 02 ──▶ ESI weights (w_gdp=0.294, w_unemp=0.053, w_inf=0.308, w_nni=0.345)
        │         │
        │         ▼  Script 03
        │   cleaned_normalized_data_with_ESI.csv
        │         │
        │         ├──▶ Script 07 ──▶ ESI shock simulation (ΔESI table, Figure 3)
        │         └──▶ Script 08 ──▶ ESI deviation figures
        │
        ├──▶ Script 04 ──▶ HSR weights (α_oop=0.332, α_beds=0.325, α_che=0.343)
        │         │
        │         ▼  Script 05
        │   HSR_calculated.csv
        │
        └──▶ Script 06 ──▶ HEI weights (w_oop=0.254, w_che=0.256, w_beds=0.216, w_nni=0.275)
                  │
                  ▼  Script 07
            HEI_calculated.csv
                  │
                  ▼  (merge ESI + HSR + HEI)
            All_indices.csv
                  │
                  ├──▶ Script 10 ──▶ PRIMARY RESULTS Table 4 (ΔHSR, ΔHEI with 95% CI)
                  ├──▶ Script 11 ──▶ Descriptive statistics (Tables 1–2, Figures 1–2)
                  ├──▶ Script 12 ──▶ Alternative response estimation (supplementary)
                  └──▶ Script 13 ──▶ ROBUSTNESS Table 5 (elasticity / RF / XGBoost + 95% CI)
                              │
                              ▼  Script 14
                        Robustness figures (Figures 4–5 and 4-panel)
```

---

## Reproducing Specific Tables and Figures

| Manuscript output | Script to run | Output file |
|---|---|---|
| Table 1 (regional descriptive statistics) | `11_descriptive_statistics.py` | `descriptive_table_regional.csv` |
| Table 2 (country snapshot) | `11_descriptive_statistics.py` | `descriptive_table_country_snapshot.csv` |
| Table 3 (ESI deviation statistics) | `08_esi_shock_simulation.py` | `esi_shock_summary.csv` |
| **Table 4 (primary results, ΔHSR and ΔHEI)** | **`10_primary_response_estimation.py`** | **`primary_shock_response_table4.csv`** |
| Table 5 (robustness: elasticity/RF/XGBoost) | `13_robustness_elasticity_rf_xgboost.py` | `shock_response_table5_with_CI.csv` |
| Figure 1 (regional bar chart) | `11_descriptive_statistics.py` | `figure1_regional_bar_chart.png` |
| Figure 2 (global time-series) | `11_descriptive_statistics.py` | `figure2_global_timeseries.png` |
| Figure 3 (ESI deviation distributions) | `09_esi_shock_visualisation.py` | `figure3_ESI_deviation_distributions.png` |
| Figures 4–5 (robustness shock curves) | `14_robustness_visualisation.py` | `figure4_HSR_robustness_curves.png` |

---

## Adapting the Code for Your Own Research

The pipeline is designed to be modular. If you want to:

- **Change the shock magnitudes:** Edit the `shocks` list at the top of Scripts 8, 10, and 13.
- **Add a new country group:** Extend the `WB_REGIONS` dictionary in Script 11.
- **Replace a component variable:** Change the column name in the relevant weight-estimation script
  (Scripts 2, 4, or 6) and update the index formula in the corresponding calculation script.
- **Change the number of bootstrap replications:** Edit `N_BOOT` in Scripts 2, 4, 6, 10, and 13.
  Reducing to 200–500 replications cuts runtime substantially while remaining adequate for most purposes.
- **Update the data to a later year:** Download updated World Bank indicators and re-run from Script 1.
  The normalisation parameters will be automatically recomputed.
- **Apply the framework to a single country or region:** Filter the cleaned dataset by country name or
  World Bank region before running Scripts 8 onwards.

---

## Citation

If you use this code or data in your own research, please cite:

> Momahhed SS, Haghighathoseini A. Macroeconomic Shock–Response Functions of Health System Resilience and Equity: A Global Simulation Study


## Contact

For questions about the code or data, please contact:

**Shekoofeh Sadat Momahhed**  
PhD in Health Economics  
Department of Health Management and Economics, School of Public Health  
Tehran University of Medical Sciences, Tehran, Iran  
✉ Shekoufehmomahhed@gmail.com
