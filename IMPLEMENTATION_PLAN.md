# Implementation Plan — 8-Week Timeline

> **Project:** Predicting Early Signals of Neighborhood Financial Instability
> **Unit:** ZCTA (ZIP) | **Scope:** U.S. | **Sources:** ACS + HUD + Eviction Lab + CFPB + FRED

Each week ends with a **checkpoint** — a concrete artifact you can show to
teammates or advisors. If a week slips, skip to the "Minimum Viable" version
in the risk column rather than abandoning later weeks.

---

## Week 1 — Setup & Data Collection

**Goal:** Every raw file on disk, reproducibly.

- [ ] Register for Census API key and FRED API key (free, instant)
- [ ] Register for Eviction Lab data access (signup required)
- [ ] Populate `.env` with both keys
- [ ] Run `notebooks/01_data_collection.ipynb`:
  - [ ] ACS 5-year ZCTA data for 2015–2022 (income, poverty, rent, unemployment, education, household size)
  - [ ] HUD Fair Market Rent (county, 2015–2022)
  - [ ] HUD USPS ZIP ↔ County crosswalk
  - [ ] Eviction Lab county-level file (2015 through latest available)
  - [ ] CFPB complaints (full dump, filter to 2015+, keep ZIP-level)
  - [ ] FRED: national unemployment rate, CPI, housing price index
- [ ] All raw files land in `data/raw/` with clear filenames

**Checkpoint:** `data/raw/` populated. Inventory table in the notebook showing row counts, date ranges, and coverage per source.

**Risk:** Eviction Lab signup delays → use their public preview data (2000–2018) which downloads without signup.

---

## Week 2 — Data Cleaning (Per-Source)

**Goal:** Each source is a clean pandas DataFrame with consistent column names and types.

- [ ] Standardize geography columns (`zcta`, `county_fips`, `state_fips`) as zero-padded strings
- [ ] Harmonize year columns (all as `int`)
- [ ] Handle ACS quirks: suppressed values (negative), "Not Applicable" codes
- [ ] CFPB: parse dates, filter to 2015+, drop rows with no ZIP, aggregate to (zip, year) counts by product/issue
- [ ] HUD FMR: keep 2-bedroom FMR as canonical rent benchmark
- [ ] Eviction Lab: drop duplicated county-year rows, handle suppression flags
- [ ] Outputs saved to `data/processed/` as parquet (one per source)
- [ ] Write per-source summary tables to `outputs/tables/source_coverage.csv`

**Checkpoint:** Five cleaned parquet files, each with a 1-paragraph summary in the notebook (rows, coverage, missingness).

**Risk:** CFPB file is multi-GB → read in chunks with `pd.read_csv(chunksize=...)` and aggregate on the fly. Never load the full file into RAM.

---

## Week 3 — Merging & Feature Engineering

**Goal:** One master panel dataset: one row per (ZCTA, year).

- [ ] Build ZCTA-level population totals from ACS (needed for CFPB per-capita rate)
- [ ] Use HUD ZIP→County crosswalk to broadcast county-level eviction and HUD FMR to ZCTA level
- [ ] Aggregate CFPB complaints to (ZCTA, year)
- [ ] Join ACS + HUD + Eviction + CFPB on (ZCTA, year); add FRED as year-level columns
- [ ] Feature engineering in `src/features.py`:
  - `rent_to_income_ratio` = median_rent × 12 / median_household_income
  - `unemployment_change_1yr`
  - `rent_growth_1yr`
  - `complaint_rate_per_10k` = cfpb_complaints / (population / 10,000)
  - `eviction_rate_yoy_change`
  - `poverty_rate`, `housing_cost_burden` (% of households paying >30%)
  - `pop_density` if land area from TIGER is available (stretch)
- [ ] Save master panel as `data/processed/master_panel.parquet`

**Checkpoint:** Master panel with ~33k ZCTAs × ~8 years × ~15 features. Missingness report per column.

**Risk:** Merge inflates/deflates row count unexpectedly → assert row counts match ACS ZCTA list at every join step.

---

## Week 4 — Target Construction & EDA Part 1

**Goal:** Financial Instability Score + distributions.

- [ ] Implement the composite target in `src/features.py`:
  1. For each year, z-score the 4 components (eviction rate, rent burden, poverty rate, complaint rate)
  2. Average the four z-scores into a single **Financial Instability Score**
  3. Flag `unstable = 1` if ZCTA is in the **top 25%** of the score that year
- [ ] Create the **lagged target**: for each (ZCTA, year *t*), attach the label from year *t+1*
- [ ] Start `notebooks/03_eda.ipynb`:
  - [ ] Distribution of the instability score (overall and per year)
  - [ ] Trends over time — what % of ZCTAs are unstable each year, which ones persist
  - [ ] Stable vs. unstable ZCTAs compared across key features (bar/violin plots)

**Checkpoint:** Target is a boolean column on the master panel. Score distribution figure saved to `outputs/figures/`.

**Risk:** Score is dominated by one variable → check component correlations, consider rescaling weights.

---

## Week 5 — EDA Part 2 & Feature Selection

**Goal:** Understand relationships; optional map.

- [ ] Correlation heatmap of all features vs. target
- [ ] Top-10 most correlated features bar chart
- [ ] Choropleth of a recent year's risk score (using `geopandas` + TIGER ZCTA shapefiles)
- [ ] Bivariate scatters for the 4 strongest predictors
- [ ] Feature multicollinearity check (VIF); drop redundant pairs

**Checkpoint:** EDA notebook is self-contained — someone reading just `03_eda.ipynb` understands the data story.

**Risk:** ZCTA shapefiles are large (~300MB) → simplify geometries with `geopandas.simplify()` or fall back to a table/heatmap.

---

## Week 6 — Modeling

**Goal:** Three trained models with honest, time-aware evaluation.

- [ ] Time-aware split: train = all year-pairs ending ≤ 2020; test = 2021 → 2022 (or whatever the latest pair is)
- [ ] Baseline: Logistic Regression (with standardized features, class-weighted)
- [ ] Random Forest (class-weighted, reasonable hyperparameters)
- [ ] HistGradientBoosting (handles NaN natively, strong default baseline)
- [ ] Cross-validation on the training years only
- [ ] Save fitted models to `outputs/models/`
- [ ] Build a comparison table: accuracy, precision, **recall (primary)**, F1, ROC-AUC, PR-AUC

**Checkpoint:** `outputs/tables/model_performance.csv` with all metrics side-by-side; best model identified.

**Risk:** All models do poorly → the score may be too noisy or too easy (trivially predicted by one feature). Diagnose with a dummy classifier and per-feature single-variable logistic.

---

## Week 7 — Interpretation & Final Outputs

**Goal:** Translate the best model into a story.

- [ ] Logistic regression coefficient table (sorted by magnitude)
- [ ] Random Forest / GBM feature importances (permutation importance is more honest than impurity-based)
- [ ] SHAP values on the GBM for global and local explanations (one or two real example ZIPs)
- [ ] Ranked list of **highest-risk ZIPs predicted for next year** → `outputs/tables/risk_ranking.csv`
- [ ] One map figure of predicted high-risk ZIPs
- [ ] Written findings section (1–2 pages) in `05_interpretation.ipynb` answering: *Which early signals are strongest predictors of future instability?*

**Checkpoint:** Everything a judge would want to see is in `outputs/`.

**Risk:** SHAP is slow on large models → sample 5,000 test rows instead of the full set.

---

## Week 8 — Presentation & Polish

**Goal:** 10–20 slide deck + clean repo.

- [ ] Fill in `presentation/final_slide_outline.md` and build the actual deck
- [ ] Rehearse at least 2×, time yourselves
- [ ] Emphasize the **WHY** (per rubric) — hardship prevention, not model minutiae
- [ ] README polish: make sure Quick Start works end-to-end on a fresh machine
- [ ] All notebooks re-run top-to-bottom without errors

**Checkpoint:** You could hand the repo to a stranger and they could reproduce the results.

---

## Minimum Viable Project (if things go sideways)

If Week 3 slips, degrade to:
- **Single year of data** (just 2022→2023) instead of a panel
- **Just 3 sources**: ACS + CFPB + Eviction (drop HUD and FRED)
- **Single model**: Logistic Regression only

You still have every rubric element covered — just less impressively.

---

## Global Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Eviction Lab signup takes days | Start Week 1 Day 1; meanwhile use their public preview (2000–2018) |
| CFPB file size | Chunked reading, aggregate-on-load, never hold full file in RAM |
| Merge row-count explosions | Assertion checks after every join |
| ZCTA boundaries change over years | Stick to a single vintage (e.g., 2020 ZCTAs) and document |
| Population denominators for CFPB rate | Cross-check ACS totals vs. 2020 Decennial |
| Class imbalance in target | `class_weight='balanced'` + report PR-AUC, not just ROC-AUC |
| Data leakage through t+1 features | Target built strictly from year *t+1*; features strictly from year *t* |
