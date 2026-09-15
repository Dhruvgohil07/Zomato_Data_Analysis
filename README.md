# Zomato India Restaurant Analysis

An end-to-end data analysis of Zomato's global restaurant listings, narrowed to a clean **India-only** dataset of 8,643 restaurants. The project goes from raw data cleaning through exploratory analysis, statistical hypothesis testing, derived business metrics, predictive modeling, and a final business-facing report — answering concrete questions like *what actually drives a restaurant's rating*, *which restaurants are at risk of never getting rated*, and *where are the underserved cuisine/location gaps a new restaurant could exploit*.

## About the project

Zomato's public dataset lists ~9,500 restaurants across 15 countries, but mixes 12 different currencies with no conversion factor — so any cost comparison across countries is invalid out of the box. This project restricts scope to **India** (90.6% of the raw data, and the only country where cost is plain, unconverted Rupees), then treats that subset as a real analytics problem: clean it properly, verify statistical claims instead of eyeballing charts, engineer business-relevant metrics, build predictive models, and translate all of it into recommendations a restaurant owner or Zomato itself could act on.

**Why it's interesting as a project:**
- The raw data has real traps — a "rating of 0" actually means *not yet rated*, not *rated zero and terrible*; deduplicating by restaurant name silently deletes thousands of valid chain outlets (Domino's, Subway, etc. sharing a name across locations). Catching and fixing these was itself part of the analysis (see `plan.md` §0 for the full audit).
- Every statistical claim is backed by a real hypothesis test with an effect size and a multiple-comparison correction — not just "the chart looks different."
- Two predictive models are built for two different real business jobs (predicting a rating vs. predicting whether a restaurant will ever get rated at all), and their outputs are explicitly kept separate rather than conflated.

## Dataset

- **Source:** [Zomato Restaurants dataset](https://www.kaggle.com/datasets/shrutimehta/zomato-restaurants-data) (via Kaggle) — `Zomatodataset/zomato.csv`, 9,551 restaurants × 21 columns (name, city, coordinates, cuisines, average cost, currency, rating, votes, booking/delivery flags, etc.)
- **Supplementary file:** `Country-Code.xlsx` maps numeric country codes to country names
- **Working scope:** after filtering to India and dropping 9 zero-cost rows, the analysis base is **8,643 restaurants**, 91.95% of them in the Delhi NCR region (New Delhi, Gurgaon, Noida, Faridabad, Ghaziabad)
- **Encoding note:** the raw CSV is `latin-1`, not UTF-8 — everything this project writes back out to `Output/` is standard UTF-8

## Process / methodology

The analysis runs as eight sequential notebook phases, each consuming the previous phase's output:

| # | Phase | Notebook | What it does |
|---|---|---|---|
| 1 | Exploratory analysis | `EDA.ipynb` | Loads raw data, merges country names, first-pass distributions and missing-value checks across all 15 countries |
| 2 | Data cleaning | `Data_cleaning.ipynb` | Fixes the rating-zero and dedupe-key traps, filters to India, engineers baseline flags (`Is Chain`, `Is Rated`, `Is NCR`), decides how to handle outliers |
| 3 | Feature engineering | `Feature_Engineering.ipynb` | Cuisine count/primary cuisine extraction, booking/delivery flags to binary, log-transform on votes to fix extreme skew |
| 4 | Deep EDA | `Deep_EDA.ipynb` | Full univariate/bivariate/multivariate exploration against the cleaned, feature-rich base — every chart paired with a written observation |
| 5 | Statistical validation | `Statistical_Validation.ipynb` | Six formal hypothesis tests (Welch's t-test, Welch's ANOVA + Tukey HSD, Mann-Whitney U, chi-square) with effect sizes, bootstrap confidence intervals, and Holm correction for multiple comparisons |
| 6 | Derived metrics | `Derived_Metrics.ipynb` | Five business metrics built on top of the validated data: credibility-weighted ratings, value-for-money residuals, competition density, cuisine-pair premiums, locality opportunity gaps |
| 7 | Predictive modeling | `Modeling.ipynb` | A rating-prediction model (Linear/Ridge → Random Forest → XGBoost) and a separate cold-start classifier predicting whether a restaurant will ever get rated |
| 8 | Insights & reporting | `Insights_Reporting.ipynb` | Consolidates every phase into one master table, a prioritized business action list, five summary charts, and [`Report.md`](Report.md) — a code-free write-up for a non-technical reader |

Every analytical notebook runs cleanly end-to-end via `jupyter nbconvert --execute` and pairs each code cell with a markdown observation grounded in that cell's actual output — nothing is asserted without a specific number behind it.

## Results

### What drives a restaurant's rating
- **Price range is the strongest clean lever** — explains ~14% of rating variance on its own. Ratings split into a budget tier (price ranges 1–2, ~3.2–3.3 average) and a premium tier (3–4, ~3.7 average).
- **Table booking and online delivery are not rating levers.** Booking's apparent benefit is a Simpson's-paradox artifact of price tier and region; delivery's apparent link to rating was 89% an artifact of how "not yet rated" restaurants were originally counted.
- **Chain status barely matters** (chains rate 0.07 lower than independents — a negligible effect).
- **Location effects are real but sample-biased.** Restaurants outside NCR rate meaningfully higher, but that slice of the data is small, differently collected, and about 42% of the gap is explained by vote count rather than location itself.

### Predictive models
- **Rating prediction** (using only levers a restaurant owner actually controls — cost, cuisine, location, price tier, excluding vote count as leakage): best model (XGBoost) reaches **Test R² ≈ 0.405**. `Locality` is by far the dominant feature.
- **Cold-start risk** (will a restaurant ever get rated at all): reaches **Test AUC ≈ 0.89–0.90** within NCR. Notably, 100% of never-rated restaurants in this dataset are located in NCR.

### Derived business metrics
- **Credibility-weighted ratings** (Bayesian shrinkage toward the global mean, weighted by vote count) demote several low-vote 4.9-star restaurants and promote high-vote 4.8s — a more trustworthy leaderboard than raw stars.
- **Cuisine fusion premiums:** pairing North Indian food with Mediterranean, Asian, American, Mexican, Thai, or Seafood cuisines adds +0.30 to +0.55 rating points versus serving either alone (56 of 83 tested pairings show a genuine premium).
- **Competition density** has a non-monotonic relationship with rating — isolated restaurants rate highest overall, but that flips at the two most expensive price tiers, where denser commercial hubs rate higher.
- **167 locality-level demand-supply gaps** were flagged (98.8% in NCR), and cross-referenced with cold-start risk to produce a prioritized **430-restaurant action list**.

A full, code-free write-up of every finding and its business recommendation is in **[`Report.md`](Report.md)**.

## Repository structure

```
├── README.md                    # This file
├── Report.md                    # Code-free business report (derived from Insights_Reporting.ipynb)
├── EDA.ipynb                    # Phase 1: exploratory analysis
├── Data_cleaning.ipynb          # Phase 2: cleaning, India filter, trap fixes, outlier decision
├── Feature_Engineering.ipynb    # Phase 3: cuisine/vote feature engineering
├── Deep_EDA.ipynb               # Phase 4: univariate/bivariate/multivariate deep EDA
├── Statistical_Validation.ipynb # Phase 5: hypothesis tests, effect sizes, Holm correction
├── Derived_Metrics.ipynb        # Phase 6: 5 derived business metrics
├── Modeling.ipynb               # Phase 7: rating prediction + cold-start classifier
├── Insights_Reporting.ipynb     # Phase 8: consolidated dataset, action list, charts, takeaways
├── Output/
│   ├── df_india_cleaned                  # Cleaned India base (8,643 rows)
│   ├── df_featured.csv                   # Feature-engineered base
│   ├── phase4_test_results.csv           # Hypothesis test results: effect sizes, CIs, verdicts
│   ├── credibility_weighted_ratings.csv  # Bayesian-shrunk credibility rating per restaurant
│   ├── value_for_money_residuals.csv     # Rating-vs-cost residual, ranked
│   ├── competition_density.csv           # Competitor counts within 500m/1km/2km
│   ├── cuisine_pair_premium.csv          # Rating/cost premium per cuisine pair
│   ├── locality_cuisine_crosstab.csv     # Locality × cuisine supply/demand
│   ├── locality_pricerange_crosstab.csv  # Locality × price-range supply/demand
│   ├── locality_opportunity_flags.csv    # Flagged thin-supply/high-demand cells
│   ├── rating_model_*.csv                # Rating model comparison, predictions, feature importance
│   ├── coldstart_*.csv                   # Cold-start model risk scores, feature importance
│   ├── restaurant_master_insights.csv    # Every restaurant-level metric, joined (8,643 × 42)
│   ├── business_action_list.csv          # Prioritized 653-row action list
│   └── charts/                           # 5 summary chart PNGs used in Report.md
└── Zomatodataset/                        # Raw data (CSV, XLSX, JSON)
    ├── zomato.csv                        # Main dataset (9,551 rows × 21 cols)
    ├── Country-Code.xlsx                 # Country code → name mapping
    └── file1–5.json                      # Delhi restaurant photos/URLs (optional, unused extension)
```

Detailed roadmap, decision log, and task history live one level up in the parent folder: `plan.md`, `progress.md`, `ai_context.md`.

## Getting started

**Requirements:** Python 3 with `pandas`, `numpy`, `matplotlib`, `seaborn`, `scipy`, `statsmodels`, `scikit-learn`, `xgboost`, and `importnb`.

```bash
# Explore the raw data (requires latin-1 encoding)
python -c "
import pandas as pd
df = pd.read_csv('Zomatodataset/zomato.csv', encoding='latin-1')
print(df.shape)
"
```

Run the notebooks in order — each later notebook depends on the previous one's output:

```
EDA.ipynb → Data_cleaning.ipynb → Feature_Engineering.ipynb → Deep_EDA.ipynb
→ Statistical_Validation.ipynb → Derived_Metrics.ipynb → Modeling.ipynb
→ Insights_Reporting.ipynb
```

`Data_cleaning.ipynb` imports the merged dataframe directly from `EDA.ipynb` via `importnb`, so `EDA.ipynb` must run first; every notebook after that reads straight from the `Output/` CSVs.

## Key gotchas for anyone extending this project

- **CSV encoding:** raw `zomato.csv` needs `encoding="latin-1"`; everything under `Output/` is UTF-8
- **Rating-zero trap:** never analyze `Aggregate rating` without filtering to `Is Rated == True` first — a 0 means "not rated," not "rated zero"
- **Dedupe key:** use `Restaurant ID`, never `Restaurant Name` — chain outlets legitimately share a name across locations
- **Windows console:** currency symbols (₹) can crash `cp1252` terminals — prefix commands with `PYTHONIOENCODING=utf-8`
- **Pandas boolean masks** need explicit parentheses: `df[(a > ub) | (a < lb)]`

## Limitations

- **Geographic concentration:** 91.95% of restaurants are in NCR; findings about "outside NCR" describe a small, differently-collected sample, not a fair national comparison
- **No currency conversion:** the original 15-country dataset was never reconciled across currencies, so nothing here generalizes beyond India
- **Single snapshot:** no time dimension — none of these findings can distinguish a trend from a one-time level
- **Rating model ceiling:** the best controllable-lever model explains ~40% of rating variance — a modest but honest result, not a modeling shortfall (votes alone would explain ~54%, but using them would be leakage for an actionable model)

See [`Report.md`](Report.md) for the full limitations discussion and every business recommendation with its caveats.
