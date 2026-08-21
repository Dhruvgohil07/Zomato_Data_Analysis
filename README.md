# Zomato Data Analysis

End-to-end analysis of a global Zomato restaurant dataset: data cleaning, exploratory data analysis (EDA), feature engineering, and deep EDA — with statistical validation and predictive modeling ahead.

**Current scope (decision 2026-08-16, still valid): India-only analysis.** The full dataset mixes 12 currencies without conversion, so cross-country cost comparisons are invalid. See `../plan.md` for the full roadmap.

> A 2026-08-20 audit found 3 defects upstream of the whole analysis (a rating-zero encoding trap, a dedupe key that had deleted 2,048 valid restaurants, and a stale cost-normalization note). All three are fixed as of 2026-08-21 — see [Key findings](#key-findings-so-far) below and `../plan.md` §0 for the full writeup.

## Project status

| Phase | Status | Notebook |
|---|---|---|
| Data loading & EDA | ✅ Complete | `EDA.ipynb` (41 cells) |
| Data cleaning + Phase 2 rebuild (India scope) | ✅ Complete | `Data_cleaning.ipynb` (37 cells) |
| Outlier handling (India) | ✅ Done — **outliers kept** (premium segment, 813 rows) | `Data_cleaning.ipynb` |
| Feature engineering | ✅ Complete | `Feature_Engineering.ipynb` (21 cells) |
| Deep EDA (univariate/bivariate/multivariate) | ✅ Complete | `Deep_EDA.ipynb` (45 cells) |
| Derived metrics (Phase 3.5) | ⬜ Planned | — |
| Statistical validation | ⬜ Planned | — |
| Predictive modeling | ⬜ Planned | — |
| Reporting & final dataset | ⬜ Planned | — |

## Dataset

- **`Zomatodataset/zomato.csv`** — 9,551 restaurants × 21 columns (name, city, coordinates, cuisines, average cost, currency, ratings, votes, booking/delivery flags, etc.)
- **`Country-Code.xlsx`** — maps country codes to names (15 countries: India, US, UK, UAE, Indonesia, Brazil, and more)
- **`file1–5.json`** — optional: ~2,358 Delhi restaurants (photos/URLs) for a separate city-focused analysis
- **`Output/df_india_cleaned`** — the cleaned, India-only, feature-flagged base (8,643 rows) produced by `Data_cleaning.ipynb`
- **`Output/df_featured.csv`** — `df_india_cleaned` plus cuisine/vote features, produced by `Feature_Engineering.ipynb` and consumed by `Deep_EDA.ipynb`

> ⚠️ The raw CSV is **latin-1 encoded**, not UTF-8 — always read it with `encoding="latin-1"`.

## What has been done

### EDA (`EDA.ipynb` — 41 cells)

- **Data loading:** Read CSV with latin-1 encoding; `info()`, `describe()`, missing-value heatmap
- **Missing values:** Only 9 missing in `Cuisines` — filled with mode
- **Country merge:** Merged `Country-Code.xlsx` to get country names (15 countries)
- **Visualizations:** country distribution pie (India 90.6%), top cities, rating distribution, 0-rating countries, currency↔country mapping, online-delivery availability (India & UAE only)
- **Observations** recorded as markdown cells

### Data cleaning + Phase 2 rebuild (`Data_cleaning.ipynb` — 37 cells)

- **Missing values:** Filled `Cuisines` with mode (9 rows)
- **Dedupe:** on `Restaurant ID` (100% unique — a no-op). This *replaced* an earlier `Restaurant Name` dedupe that had wrongly deleted 2,048 valid chain-outlet restaurants (Cafe Coffee Day, Domino's, Subway, McDonald's, …) — see the audit note above.
- **India filter:** `df_india`, **8,652 rows** → drop 9 zero-cost rows → **8,643 rows**
- **Cost:** confirmed plain, rounded INR — no normalization, no conversion factor
- **New features:** `n_outlets`/`Is Chain` (2,755 chain / 5,888 independent), `Is Rated` (`Aggregate rating > 0`; 6,504 rated / 2,139 unrated), `Is NCR` (New Delhi/Gurgaon/Noida/Faridabad/Ghaziabad; 91.95%/8.05%)
- **Dead columns dropped:** `Currency`, `Country`, `Country Code`, `Switch to order menu`, `Is delivering now`
- **Outlier analysis (India):**
  - IQR method on the fixed base: **813 outliers (9.41%)**
  - Verified as genuine premium/luxury restaurants (Delhi NCR fine dining)
  - **Decision: KEEP outliers** — removing them would delete the best-rated segment

### Feature engineering (`Feature_Engineering.ipynb` — 21 cells)

- `Cuisines Count` and `Primary Cuisine` from splitting the `Cuisines` string
- `Has Table booking` / `Has Online delivery` mapped Yes/No → 1/0
- `Log_Votes = log1p(Votes)` — `Votes` is extremely right-skewed (skew 9.65); the log transform brings it to near-symmetric (skew 0.07)
- Output: `Output/df_featured.csv` (8,643 rows, 24 columns)
- No code changes were needed when the Phase 2 rebuild changed the underlying row count — every transform here is row/column-wise, so a straight re-run against the fixed base was sufficient

### Deep EDA (`Deep_EDA.ipynb` — 45 cells)

- **Univariate:** numeric distributions (histogram+KDE+boxplot) for rating/votes/log-votes/cost/cuisine-count; categorical countplots (natural order for ordinal columns, top-15+Other for `Primary Cuisine`); a full stats table (mean/median/skew/kurtosis/outlier count); an explicit rated-vs-unrated comparison
- **Bivariate** (target = `Aggregate rating`, always on `df_rated`): rating vs cost (LOWESS), vs `Log_Votes` (with a leakage caveat), vs price range, vs booking/delivery/chain flags; cost vs Locality/Primary Cuisine (top-15 bars); NCR vs rest of India
- **Multivariate:** correlation heatmap, pairplot, cost-vs-rating scatterplots hued by price range and chain status, and a geospatial scatter of NCR restaurants (color = rating, size = votes)
- Every code cell is paired with a markdown `### Observation:` cell describing what the actual output showed
- Executed end-to-end via `jupyter nbconvert --execute`, 0 errors

## Key findings so far

1. **The data mixes 12 currencies without conversion** (INR, USD, IDR, GBP, …). Apparent cross-country "outliers" were a currency artifact — e.g. every cost above 10,000 is Indonesian Rupiah (800,000 IDR ≈ $50 USD). This drove the India-only decision.
2. **`Aggregate rating == 0` means "not rated," not "rated zero."** 24.7% of India rows have no rating data (`Rating text == "Not rated"`). All rating analysis runs on `df_rated` (`Is Rated == True`, 6,504 rows) — mixing unrated rows in poisons every correlation.
3. **The correct dedupe key is `Restaurant ID`, not `Restaurant Name`.** `Restaurant ID` is 100% unique; deduping by name deletes distinct chain outlets. The honest India base is **8,643 rows**, not 6,583.
4. **Cost is plain, rounded INR** — there is no conversion factor to apply when reporting prices.
5. **The 813 India outliers (9.41%) are the premium segment, not errors** — genuine Delhi-NCR fine dining. Removing them would bias the analysis toward lower-end restaurants, so they are kept. For future modeling, use robust methods (median/MAD, Huber, tree-based models).
6. **100% of unrated restaurants are located in NCR** — every restaurant outside NCR in this dataset has been rated. Unrated restaurants are also cheaper, less often chains, and rarely offer delivery/booking — a clear cold-start profile.
7. **The "online delivery → higher rating" signal collapses on rated-only data** (mean rating 3.37 vs 3.34, essentially flat) — the larger effect seen on unfiltered data was mostly an artifact of unrated restaurants also lacking delivery.
8. **Chain status is not a rating advantage** — chains rate slightly lower than independents (3.30 vs 3.38).
9. **NCR vs rest of India is counter-intuitive**: non-NCR restaurants rate higher (3.94 vs 3.28) *and* cost more (₹895 vs ₹696) than NCR ones — but on a much smaller sample (696 vs 5,808 rated rows), flagged for a formal significance test rather than treated as concluded.
10. **`Average Cost for two` and `Price range` are redundant** (Pearson 0.83 / Spearman 0.91 on rated data) — only one should enter a linear model at a time.
11. **Rating tiers:** 4.5–4.9 Excellent, 4.0–4.4 Very Good, 3.5–3.9 Good, 2.5–3.4 Average, 1.8–2.4 Poor.
12. **Online delivery** is only available in India and UAE (in the full, multi-country dataset).

## Repository structure

```
├── README.md                    # This file
├── EDA.ipynb                    # Exploratory analysis (41 cells)
├── Data_cleaning.ipynb          # Cleaning, India filter, Phase 2 rebuild, outlier decision (37 cells)
├── Feature_Engineering.ipynb    # Cuisine/vote feature engineering (21 cells)
├── Deep_EDA.ipynb               # Univariate/bivariate/multivariate Deep EDA (45 cells)
├── Output/
│   ├── df_india_cleaned         # Phase 2 output: cleaned India base (8,643 rows)
│   └── df_featured.csv          # Feature-engineered base used by Deep_EDA.ipynb
└── Zomatodataset/                # Raw data (CSV, XLSX, JSON)
    ├── zomato.csv                # Main dataset (9,551 rows × 21 cols)
    ├── Country-Code.xlsx         # Country code → name mapping
    ├── file1–5.json              # Delhi restaurant photos/URLs (optional)
    └── Untitled.ipynb            # Empty scratch notebook
```

Tracking files live one level up (repo root): `plan.md` (roadmap), `progress.md` (task log + challenges), `ai_context.md` (LLM resume context).

## Important gotchas

- **CSV encoding:** Always use `encoding="latin-1"` — not UTF-8
- **Console unicode:** Windows cp1252 crashes on currency symbols (₹, etc.) — prefix with `PYTHONIOENCODING=utf-8`
- **Notebook dependency:** `Data_cleaning.ipynb` imports `eda.final_df` via `importnb` — **EDA.ipynb must run first**
- **Rating-zero trap:** never analyze `Aggregate rating` without filtering to `Is Rated == True` first
- **Dedupe key:** use `Restaurant ID`, never `Restaurant Name` (chain outlets share a name but are distinct restaurants)
- **Pandas boolean masks:** need explicit parentheses: `df[(a > ub) | (a < lb)]`

## Getting started

```bash
# 1. Explore the data (latin-1 encoding is required)
PYTHONIOENCODING=utf-8 python -c "
import pandas as pd
df = pd.read_csv('Zomatodataset/zomato.csv', encoding='latin-1')
print(df.shape)
"

# 2. Run the notebooks in order:
#    EDA.ipynb -> Data_cleaning.ipynb -> Feature_Engineering.ipynb -> Deep_EDA.ipynb
#    (Data_cleaning.ipynb imports the merged dataframe from EDA.ipynb via importnb;
#     Feature_Engineering.ipynb and Deep_EDA.ipynb read straight from the Output/ CSVs)
```

Requires Python 3 with pandas, numpy, matplotlib, seaborn, statsmodels, scikit-learn, and `importnb` (Anaconda distribution covers most).

## Roadmap ahead (India-focused)

1. **Phase 3.5 — Derived metrics:** credibility-weighted (Bayesian) rating, competition density from lat/long (`BallTree`), value-for-money residual, cuisine co-occurrence, locality opportunity gaps
2. **Phase 4 — Statistical validation:** Welch's t-test (booking, chain), ANOVA (price range), chi-square (delivery × price range) — with effect sizes and Holm correction; formally test the NCR-vs-rest gap found in Deep EDA
3. **Phase 5 — Predictive modeling:** predict `Aggregate rating` with two framings (descriptive vs actionable — the latter excludes `Votes` as leakage); compare Linear/Ridge, Random Forest, XGBoost; plus a separate `Is Rated` cold-start classifier
4. **Phase 6 — Insights & final report:** business takeaways, summary charts, save final datasets, limitations section
5. **Optional:** Delhi-focused analysis with `file1–5.json` (~2,358 Delhi restaurants)
