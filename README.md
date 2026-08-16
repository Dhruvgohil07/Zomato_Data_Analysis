# Zomato Data Analysis

End-to-end analysis of a global Zomato restaurant dataset: data cleaning, exploratory data analysis (EDA), and a roadmap toward feature engineering, statistical validation, and predictive modeling.

**Current scope (decision 2026-08-16): India-only analysis.** The full dataset mixes 12 currencies without conversion, so cross-country cost comparisons are invalid. See `../plan.md` for the full roadmap.

## Project status

| Phase | Status |
|---|---|
| Data loading & EDA | ✅ Complete (`EDA.ipynb`) |
| Data cleaning (incl. India scope) | ✅ Complete (`Data_cleaning.ipynb`) |
| Outlier handling (India) | ✅ Done — **outliers kept** (premium segment) |
| Feature engineering → Predictive modeling | ⬜ Planned |

## Dataset

- **`Zomatodataset/zomato.csv`** — 9,551 restaurants × 21 columns (name, city, coordinates, cuisines, average cost, currency, ratings, votes, booking/delivery flags, etc.)
- **`Country-Code.xlsx`** — maps country codes to names (15 countries: India, US, UK, UAE, Indonesia, Brazil, and more)
- **`file1–5.json`** — optional: ~2,358 Delhi restaurants (photos/URLs) for a separate city-focused analysis

> ⚠️ The CSV is **latin-1 encoded**, not UTF-8 — always read it with `encoding="latin-1"`.

## What has been done

### EDA (`EDA.ipynb`)
- Data loading, `info()`, `describe()`, missing-value heatmap (only 9 missing in `Cuisines`)
- Merged country names via `Country-Code.xlsx`
- Visualized country distribution (top: India, US, UK), top cities (New Delhi), and rating distribution by rating color/text
- Found many restaurants unrated (rating 0), with India having the most
- Mapped currencies ↔ countries and online-delivery availability (only India & UAE)

### Data cleaning & India scope (`Data_cleaning.ipynb`)
- Filled missing cuisines with the mode; dropped duplicate restaurants by name
- Normalized `Average Cost for two` by country mean (kept as-is — relative units; ×624 ≈ INR)
- Observation: cross-country outliers are a currency/PPP artifact → decided **country-specific (India) analysis**
- Created `df_india` (**6,592 rows**) and dropped 9 rows with cost = 0 (impossible values)
- Re-ran IQR on India → **691 outliers (10.48%)** — investigated and **kept**: they are the genuine premium/luxury segment (New Delhi 391, Gurgaon 149, Noida 39; e.g. ITC Mughal, Radisson Blu; avg rating 3.62 vs 2.46)

## Key findings so far

1. **The data mixes 12 currencies without conversion** (INR, USD, IDR, GBP, …). Apparent "outliers" across countries were a currency artifact — e.g. every cost above 10,000 is Indonesian Rupiah (800,000 IDR ≈ $50 USD). This drove the India-only decision.
2. **The 691 India outliers are the premium segment, not errors** — higher average rating (3.62 vs 2.46). Removed they would bias the analysis, so they are kept.
3. **`df_india['Average Cost for two']` is normalized** (relative to country mean, mean = 1.0); to report real prices, multiply by ≈624 INR.
4. 18 rows originally had cost = 0 (9 in India) — the India ones were dropped as data errors.

## Repository structure

```
├── README.md                    # This file
├── EDA.ipynb                    # Exploratory analysis
├── Data_cleaning.ipynb          # Cleaning, normalization, India filter, outlier decision
└── Zomatodataset/               # Raw data (CSV, XLSX, JSON)
```

Tracking files live one level up (repo root): `plan.md` (roadmap), `progress.md` (task log + challenges), `ai_context.md` (LLM resume context).

## Getting started

```bash
# 1. Explore the data (latin-1 encoding is required)
PYTHONIOENCODING=utf-8 python -c "
import pandas as pd
df = pd.read_csv('Zomatodataset/zomato.csv', encoding='latin-1')
print(df.shape)
"

# 2. Run the notebooks in order: EDA.ipynb → Data_cleaning.ipynb
#    (Data_cleaning.ipynb imports the merged dataframe from EDA.ipynb via importnb)
```

Requires Python 3 with pandas, numpy, matplotlib, seaborn, and `importnb` (Anaconda distribution covers most).

## Roadmap ahead (India-focused)

1. Feature engineering on `df_india`: cuisine counts, booking/delivery flags, geo features
2. Deep EDA: rating vs cost/votes, cuisine & city analysis, geospatial scatter
3. Statistical validation: correlation heatmap, t-tests, chi-square tests
4. Predictive modeling: predict `Aggregate rating` (regression + classification) — Linear/Logistic, Random Forest, XGBoost
5. Insights & final report; save the cleaned India dataset
6. Optional: Delhi-focused analysis with `file1–5.json`
