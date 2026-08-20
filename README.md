# Zomato Data Analysis

End-to-end analysis of a global Zomato restaurant dataset: data cleaning, exploratory data analysis (EDA), and a roadmap toward feature engineering, statistical validation, and predictive modeling.

**Current scope (decision 2026-08-16): India-only analysis.** The full dataset mixes 12 currencies without conversion, so cross-country cost comparisons are invalid. See `../plan.md` for the full roadmap.

## Project status

| Phase | Status | Notebook |
|---|---|---|
| Data loading & EDA | ✅ Complete | `EDA.ipynb` (40 cells) |
| Data cleaning (India scope) | ✅ Complete | `Data_cleaning.ipynb` (26 cells) |
| Outlier handling (India) | ✅ Done — **outliers kept** (premium segment) | `Data_cleaning.ipynb` |
| Feature engineering | ⬜ Planned | — |
| Deep EDA | ⬜ Planned | — |
| Statistical validation | ⬜ Planned | — |
| Predictive modeling | ⬜ Planned | — |
| Reporting & final dataset | ⬜ Planned | — |

## Dataset

- **`Zomatodataset/zomato.csv`** — 9,551 restaurants × 21 columns (name, city, coordinates, cuisines, average cost, currency, ratings, votes, booking/delivery flags, etc.)
- **`Country-Code.xlsx`** — maps country codes to names (15 countries: India, US, UK, UAE, Indonesia, Brazil, and more)
- **`file1–5.json`** — optional: ~2,358 Delhi restaurants (photos/URLs) for a separate city-focused analysis

> ⚠️ The CSV is **latin-1 encoded**, not UTF-8 — always read it with `encoding="latin-1"`.

## What has been done

### EDA (`EDA.ipynb` — 40 cells)

- **Data loading:** Read CSV with latin-1 encoding; `info()`, `describe()`, missing-value heatmap
- **Missing values:** Only 9 missing in `Cuisines` — filled with mode
- **Country merge:** Merged `Country-Code.xlsx` to get country names (15 countries)
- **Visualizations:**
  - Country distribution pie chart (India dominates at 90.6%)
  - Top cities: New Delhi is the most popular
  - Rating distribution by rating color/text
  - 0-rating countries (many unrated restaurants, India has the most)
  - Currency ↔ country mapping
  - Online delivery availability (only India & UAE)
- **Observations** recorded as markdown cells

### Data cleaning & India scope (`Data_cleaning.ipynb` — 26 cells)

- **Missing values:** Filled `Cuisines` with mode (9 rows)
- **Duplicates:** Dropped 2,105 duplicate restaurant names (kept first occurrence)
- **Cost normalization:** `Average Cost for two` divided by country mean (kept as relative units; ×624 ≈ INR)
- **India filter:** Created `df_india` with **6,592 rows**
- **Zero-cost rows:** Dropped 9 rows with cost = 0 (impossible values) → **6,583 rows**
- **Outlier analysis (India):**
  - IQR method: 691 outliers (10.48%)
  - Profiled outliers: New Delhi 391, Gurgaon 149, Noida 39; avg rating 3.62 vs 2.46
  - **Decision: KEEP outliers** — they are genuine premium/luxury segment
  - Alternatives considered (3×IQR, log+IQR) but rejected

## Key findings so far

1. **The data mixes 12 currencies without conversion** (INR, USD, IDR, GBP, …). Apparent "outliers" across countries were a currency artifact — e.g. every cost above 10,000 is Indonesian Rupiah (800,000 IDR ≈ $50 USD). This drove the India-only decision.

2. **The 691 India outliers are the premium segment, not errors** — higher average rating (3.62 vs 2.46). Removing them would bias the analysis, so they are kept. For future modeling, use robust methods (median/MAD, Huber, tree-based models).

3. **`df_india['Average Cost for two']` is normalized** (relative to country mean, mean = 1.0); to report real prices, multiply by ≈624 INR.

4. **18 rows originally had cost = 0** (9 in India) — the India ones were dropped as data errors.

5. **Rating tiers:** 4.5–4.9 Excellent, 4.0–4.4 Very Good, 3.5–3.9 Good, 2.5–3.4 Average, 1.8–2.4 Poor. Many restaurants rated 0 (India has the most unrated).

6. **Online delivery** is only available in India and UAE. New Delhi is the most popular city.

## Repository structure

```
├── README.md                    # This file
├── EDA.ipynb                    # Exploratory analysis (40 cells)
├── Data_cleaning.ipynb          # Cleaning, normalization, India filter, outlier decision (26 cells)
└── Zomatodataset/               # Raw data (CSV, XLSX, JSON)
    ├── zomato.csv               # Main dataset (9,551 rows × 21 cols)
    ├── Country-Code.xlsx        # Country code → name mapping
    ├── file1–5.json             # Delhi restaurant photos/URLs (optional)
    └── Untitled.ipynb           # Empty scratch notebook
```

Tracking files live one level up (repo root): `plan.md` (roadmap), `progress.md` (task log + challenges), `ai_context.md` (LLM resume context).

## Important gotchas

- **CSV encoding:** Always use `encoding="latin-1"` — not UTF-8
- **Console unicode:** Windows cp1252 crashes on currency symbols (₹, etc.) — prefix with `PYTHONIOENCODING=utf-8`
- **Notebook dependency:** `Data_cleaning.ipynb` imports `eda.final_df` via `importnb` — **EDA.ipynb must run first**
- **Pandas boolean masks:** Need explicit parentheses: `df[(a > ub) | (a < lb)]`

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

1. **Feature engineering** on `df_india`: cuisine counts + primary cuisine, Yes/No → 0/1 flags, geo features
2. **Deep EDA**: distributions (rating/votes/cost/price range), rating vs cost/votes/flags, cuisine & city analysis, geospatial scatter
3. **Statistical validation**: correlation heatmap, t-test/ANOVA (booking → rating?), chi-square (delivery vs price range)
4. **Predictive modeling**: predict `Aggregate rating` (regression + classification); features: cost, price range, cuisine count, city, flags, votes; compare Linear/Logistic, Random Forest, XGBoost; RMSE/accuracy + feature importance
5. **Insights & final report**: business takeaways, summary charts, save cleaned India dataset
6. **Optional:** Delhi-focused analysis with `file1–5.json` (~2,358 Delhi restaurants)
