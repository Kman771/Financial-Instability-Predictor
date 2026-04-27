# Predicting Early Signals of Neighborhood Financial Instability

**Data Dive — Spring 2026 | 8-Week Hackathon Project**

An early warning system that predicts which **Illinois ZIP codes** are at risk of financial
instability *before* a crisis occurs. Illinois is a uniquely rich setting for this analysis —
Chicago ranks among the highest-eviction cities in the U.S., while downstate rural ZIPs
face entirely different pressures around income stagnation and housing cost burden. Instead
of reacting to hardship after the fact, this project uses public housing, consumer-complaint,
and macroeconomic data to identify "tipping points" in local economies — so rental assistance,
job counseling, and other support programs can be targeted where they're most needed.

---

## Core Research Question

> Can we identify early warning signals that indicate an **Illinois** ZIP code is likely to
> experience financial instability in the following year?

---

## Project Design (Locked-in Decisions)

| Decision | Choice | Why |
|---|---|---|
| **Unit of analysis** | ZCTA (~33k U.S. ZIP Code Tabulation Areas) | Balance between granularity and data availability |
| **Scope** | Illinois | Local story; Chicago vs. downstate contrast; faster runtimes |
| **Time structure** | Year *t* features → year *t+1* label (1-year lag) | Simplest lagged setup; avoids target leakage |
| **Target** | Composite Financial Instability Score, **top 25% = unstable** | Multi-signal, robust to single-variable noise |
| **Data sources** | ACS + HUD + Eviction Lab + CFPB + FRED | Full suggested stack |
| **Eviction granularity** | County-level, broadcast to all ZCTAs in county | Cleaner than tract→ZIP crosswalk; mismatch documented |

---

## Target Variable: Financial Instability Score

The target is built from **four normalized signals**, measured per ZCTA per year:

1. **Eviction filing rate** (Eviction Lab, county → ZCTA)
2. **Rent burden** — median gross rent ÷ median household income (ACS)
3. **Poverty rate** (ACS)
4. **CFPB complaints per 10,000 residents** (CFPB, native ZIP)

Each component is **z-scored within the year** (so the score is relative, not
absolute), then averaged. ZCTAs in the **top 25% of the score in year t+1** are
labeled `unstable = 1`; everyone else is `0`.

Features from year *t* predict the label in *t+1* — a true lagged / forward-looking
setup that avoids leakage.

---

## Repository Structure

```
data-dive/
├── data/
│   ├── raw/                  # Downloaded source files (not committed)
│   └── processed/            # Cleaned & merged datasets
├── notebooks/
│   ├── 01_data_collection.ipynb
│   ├── 02_data_cleaning.ipynb
│   ├── 03_eda.ipynb
│   ├── 04_modeling.ipynb
│   └── 05_interpretation.ipynb
├── src/
│   ├── data_collection.py    # Download helpers (ACS, HUD, Eviction, CFPB, FRED)
│   ├── cleaning.py           # Per-source cleaning functions
│   ├── features.py           # Feature engineering + target construction
│   ├── modeling.py           # Time-aware splits + models
│   ├── evaluation.py         # Metrics prioritizing recall
│   └── visualization.py      # Reusable plotting helpers
├── outputs/
│   ├── figures/              # PNGs for the presentation
│   ├── tables/               # CSV summaries (risk rankings, metrics)
│   └── models/               # Pickled trained models
├── presentation/
│   └── final_slide_outline.md
├── IMPLEMENTATION_PLAN.md    # 8-week timeline
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Quick Start

```bash
# 1. Create a virtual environment
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Set API keys in a .env file in the project root
#    CENSUS_API_KEY=your_key         # https://api.census.gov/data/key_signup.html
#    FRED_API_KEY=your_key           # https://fred.stlouisfed.org/docs/api/api_key.html

# 4. Run notebooks in order
jupyter lab
```

Run notebooks 01 → 05 sequentially. Each one saves its outputs so the next notebook
picks up where the last left off.

---

## Data Sources

| Source | Native Geography | What We Pull | Access |
|---|---|---|---|
| **U.S. Census ACS 5-year** | ZCTA | Income, poverty, unemployment, rent, education, household size | Free API (key recommended) |
| **HUD Fair Market Rent** | County / metro | Rent benchmarks (rolled to ZCTA via crosswalk) | Direct CSV |
| **Eviction Lab** | Tract / county | County eviction filing rates (broadcast to ZCTAs) | Free (signup) |
| **CFPB Consumer Complaints** | ZIP (native) | Complaint counts, rolled up by ZCTA and year | Public CSV/API |
| **FRED** | National / State | **Illinois** unemployment rate (ILUR), CPI, Illinois HPI (ILSTHPI) | Free API |

All sources are public and free. **No Kaggle, no proprietary data.**

---

## Modeling Approach

Three models, ordered by interpretability:

1. **Logistic Regression** — baseline, fully explainable coefficients
2. **Random Forest** — captures nonlinearities, provides feature importance
3. **Gradient Boosting** (HistGradientBoosting) — usually best performer

**Time-aware train/test split**: train on earlier year-pairs, evaluate on the most
recent, so we never use the future to predict the past.

**Evaluation prioritizes recall** — missing a financially unstable ZIP is worse than
falsely flagging one, because a false negative leaves residents without help.

Metrics reported: accuracy, precision, recall, F1, ROC-AUC, and PR-AUC.

---

## Ethics & Limitations (Short Version)

- **Targeting, not punishing.** The ranked "high-risk" list is designed as a tool for
  directing support (rental assistance, job counseling, credit counseling) — *never*
  as a basis for redlining, differential pricing, or law-enforcement action.
- **Bias risk.** CFPB complaints reflect *who complains*, which correlates with
  education, language, and broadband access, not just underlying hardship.
- **Geographic mismatch.** Eviction data is assigned from county to ZIP, meaning all
  ZIPs in a county share the same eviction signal. This is a known limitation.
- **Time lag.** ACS 5-year data is a moving average; it smooths sudden shocks.

Full treatment lives in `notebooks/05_interpretation.ipynb`.

---

## Team

_Add your names here._

## License

Educational / academic use. Data remains under its original license from each provider.
