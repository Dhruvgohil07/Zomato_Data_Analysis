# Zomato Data Analysis

End-to-end analysis of a global Zomato restaurant dataset: data cleaning, exploratory data analysis (EDA), feature engineering, deep EDA and statistical validation — with derived metrics and predictive modeling ahead.

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
| Statistical validation (Phase 4) | ✅ Complete | `Statistical_Validation.ipynb` (42 cells) |
| Derived metrics (Phase 3.5) | ✅ Complete | `Derived_Metrics.ipynb` (80 cells) |
| Predictive modeling (Phase 5) | ✅ Complete | `Modeling.ipynb` (58 cells) |
| Reporting & final dataset | ⬜ Planned | — |

## Dataset

- **`Zomatodataset/zomato.csv`** — 9,551 restaurants × 21 columns (name, city, coordinates, cuisines, average cost, currency, ratings, votes, booking/delivery flags, etc.)
- **`Country-Code.xlsx`** — maps country codes to names (15 countries: India, US, UK, UAE, Indonesia, Brazil, and more)
- **`file1–5.json`** — optional: ~2,358 Delhi restaurants (photos/URLs) for a separate city-focused analysis
- **`Output/df_india_cleaned`** — the cleaned, India-only, feature-flagged base (8,643 rows) produced by `Data_cleaning.ipynb`
- **`Output/df_featured.csv`** — `df_india_cleaned` plus cuisine/vote features, produced by `Feature_Engineering.ipynb` and consumed by `Deep_EDA.ipynb` and `Statistical_Validation.ipynb`
- **`Output/phase4_test_results.csv`** — one row per Phase 4 test: statistic, raw and Holm-corrected p-values, effect size with 95% CI, verdict and caveat
- **Phase 3.5 derived-metric outputs** (produced by `Derived_Metrics.ipynb`): `credibility_weighted_ratings.csv`, `value_for_money_residuals.csv`, `competition_density.csv`, `cuisine_pair_premium.csv`, `locality_cuisine_crosstab.csv`, `locality_pricerange_crosstab.csv`, `locality_opportunity_flags.csv`
- **Phase 5 modeling outputs** (produced by `Modeling.ipynb`): `rating_model_comparison.csv`, `rating_model_predictions.csv`, `rating_feature_importance.csv`, `coldstart_risk_scores.csv`, `coldstart_feature_importance.csv`

> ⚠️ The raw CSV is **latin-1 encoded**, not UTF-8 — always read it with `encoding="latin-1"`. The `Output/` CSVs are written by pandas and are **UTF-8**, so read those with the default encoding. Some accented names are already corrupted in the raw file (e.g. "Cafí©" for "Café"); this only affects how about 70 names display.

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

### Statistical validation (`Statistical_Validation.ipynb` — 42 cells)

- **Assumption checks:** QQ plots and a D'Agostino-Pearson test (rating is mildly non-normal and discrete, but every group has n ≥ 377, so tests on means are safe); Levene's test (variances differ slightly in 4 of 5 groupings, so Welch versions are used throughout)
- **6 tests on `df_rated`:** Welch's t-tests (table booking, chain, NCR vs rest), Welch's ANOVA + Tukey HSD (price range), Mann-Whitney U (online delivery), chi-square (delivery × price range)
- Every test reports a p-value, an effect size (Cohen's d, η², rank-biserial r or Cramér's V) and a 95% bootstrap CI, with Holm correction across the family; a significant but negligible effect counts as a null result
- **Confound checks** after the booking and NCR tests: within each price range, within NCR only, and within vote bands
- Output: `Output/phase4_test_results.csv` (verdict + caveat per test) and an effect-size chart
- Executed end-to-end via `jupyter nbconvert --execute`, 0 errors

### Derived metrics (`Derived_Metrics.ipynb` — 80 cells)

- **Credibility-weighted rating:** Bayesian shrinkage `Cred_Rating = (v·R + m·C)/(v+m)` (m = median vote count = 49, C = global mean = 3.35); drops 4 low-vote 4.9s out of the raw top 15 and promotes 6 high-vote 4.8–4.9s; robust in direction across m = Q1–Q3 (8–11/15 overlap)
- **Value-for-money residual:** regressed `Cred_Rating` on `Log_Cost`, fit separately within NCR and outside NCR; best value is `Naturals Ice Cream` (₹150, residual +1.61), worst is `Pind Balluchi` (₹1,000, -1.45); 3 of the 15 worst share one mall (The Great India Place, Sector 38)
- **Competition density:** `sklearn.neighbors.BallTree` (haversine) counts competitors within 500m/1km/2km for all NCR restaurants with valid GPS (7,550 of 7,947, 95%); weak, **non-monotonic** density-rating link (ρ=0.18) that reverses at price ranges 3–4 (denser rates higher there; no isolated restaurants exist at range 4 at all)
- **Cuisine co-occurrence:** full `Cuisines` list (not just `Primary Cuisine`) split into pairs; fusion-with-North-Indian pairs carry the largest rating premiums (+0.30 to +0.55, n≥30); Healthy Food/Salad/Fast Food/American combinations underperform on both rating and cost
- **Locality opportunity gaps:** thin-supply/high-demand cells flagged per-locality (not globally) across Locality×Cuisine and Locality×Price-range cross-tabs, restricted to the 102 localities with ≥30 restaurants; 167 flagged cells, 98.8% in NCR; "Ice Cream" independently flagged in two different top localities (Chandni Chowk, Connaught Place)
- Every code cell paired with a markdown `### Observation:` cell grounded in its actual output; executed end-to-end via `jupyter nbconvert --execute`, 0 errors

### Modeling (`Modeling.ipynb` — 58 cells)

- **5A rating prediction, two framings:** descriptive (votes allowed) and actionable (`Log_Votes` excluded as leakage), both on `df_rated`; target/frequency encoding (`sklearn.preprocessing.TargetEncoder`, leakage-safe cross-fitting) for `Primary Cuisine`/`Locality`; Linear/Ridge → RandomForest → XGBoost, always reported against a predict-the-mean baseline
- Best descriptive model (XGBoost): Test R² = 0.542 — dominated by `Log_Votes` (permutation importance 0.60–0.62)
- Best actionable model (XGBoost): **Test R² = 0.405** — lands exactly in the plan's predicted 0.3–0.4 range; `Locality` is the dominant lever (~3× the next feature), `Is NCR` is second and negatively signed (direction-consistent with Phase 4's NCR gap)
- NCR-vs-non-NCR breakout: R² = 0.324 (NCR) vs 0.136 (non-NCR) — the small, differently-collected non-NCR sample is harder to predict, consistent with Phase 4's sample-curation caveat
- **5B cold-start classifier:** binary `Is Rated` on all 8,643 rows; Logistic/RandomForest/XGBoost all reach Test AUC ≈ 0.90 against a 0.5/75.3% baseline; NCR-only AUC (the real result, since 100% of unrated restaurants are in NCR) is ≈ 0.89; non-NCR AUC is undefined by design (only one class present there — reported as a confirmed data artifact, not a modeling result)
- `Locality` is the top permutation-importance feature in both 5A (actionable) and 5B
- Saves 5 `Output/` CSVs including out-of-fold rating predictions and per-restaurant cold-start risk scores (all 8,643 rows)
- No SHAP (not installed, not added as a dependency) — permutation importance used throughout instead
- Every code cell paired with a markdown `### Observation:` cell grounded in its actual output; executed end-to-end via `jupyter nbconvert --execute`, 0 errors

## Key findings so far

1. **The data mixes 12 currencies without conversion** (INR, USD, IDR, GBP, …). Apparent cross-country "outliers" were a currency artifact — e.g. every cost above 10,000 is Indonesian Rupiah (800,000 IDR ≈ $50 USD). This drove the India-only decision.
2. **`Aggregate rating == 0` means "not rated," not "rated zero."** 24.7% of India rows have no rating data (`Rating text == "Not rated"`). All rating analysis runs on `df_rated` (`Is Rated == True`, 6,504 rows) — mixing unrated rows in poisons every correlation.
3. **The correct dedupe key is `Restaurant ID`, not `Restaurant Name`.** `Restaurant ID` is 100% unique; deduping by name deletes distinct chain outlets. The honest India base is **8,643 rows**, not 6,583.
4. **Cost is plain, rounded INR** — there is no conversion factor to apply when reporting prices.
5. **The 813 India outliers (9.41%) are the premium segment, not errors** — genuine Delhi-NCR fine dining. Removing them would bias the analysis toward lower-end restaurants, so they are kept. For future modeling, use robust methods (median/MAD, Huber, tree-based models).
6. **100% of unrated restaurants are located in NCR** — every restaurant outside NCR in this dataset has been rated. Unrated restaurants are also cheaper, less often chains, and rarely offer delivery/booking — a clear cold-start profile.
7. **Online delivery doesn't affect rating.** On rated restaurants the effect is negligible (3.37 vs 3.34; rank-biserial r = 0.076). The delivery–rating correlation falls from 0.296 to 0.031 once unrated rows are removed — 89% of the apparent link was an artifact of counting "not rated" as 0.
8. **Chain status doesn't matter for rating** — chains rate 0.07 lower than independents (3.30 vs 3.38), a negligible effect (Cohen's d = -0.14).
9. **Table booking is not a rating lever.** Overall, booking restaurants rate higher (d = 0.48), but booking is concentrated in the expensive tiers; within price ranges 3–4 booking restaurants rate *lower*, and with region also held fixed the gap is small and changes sign. A Simpson's paradox: booking is a proxy for price tier and region.
10. **Price range is the strongest clean driver tested** — it explains about 14% of rating variance (η² = 0.144). Ratings split into a budget tier (ranges 1–2, ~3.2–3.3) and a premium tier (ranges 3–4, ~3.7); ranges 3 and 4 don't differ.
11. **Restaurants outside NCR rate much higher (3.94 vs 3.28, d = 1.44), but the samples differ.** The gap holds in every price range, and about 42% of it is explained by vote mix (outside-NCR restaurants have a median of 192 votes vs 40, and none are unrated). The outside-NCR rows look like a curated, popular slice, so this can't be read as "restaurants outside Delhi NCR are better".
12. **Online delivery depends on price range as an inverted U** — 25% → 50% → 37% → 11% of restaurants deliver across price ranges 1–4 (Cramér's V = 0.27).
13. **`Average Cost for two` and `Price range` are redundant** (Pearson 0.83 / Spearman 0.91 on rated data) — only one should enter a linear model at a time.
14. **Rating tiers:** 4.5–4.9 Excellent, 4.0–4.4 Very Good, 3.5–3.9 Good, 2.5–3.4 Average, 1.8–2.4 Poor.
15. **Online delivery** is only available in India and UAE (in the full, multi-country dataset).
16. **Competition density has a non-monotonic, price-dependent relationship with rating.** Isolated restaurants (0 competitors within 1km) rate highest overall, but that reverses at price ranges 3–4, where denser areas rate higher — and no isolated restaurants exist at price range 4 at all.
17. **Cuisine fusion with North Indian carries a rating premium.** Pairing North Indian with Mediterranean, European, Asian, American, Mexican, Thai or Seafood adds +0.30 to +0.55 to `Cred_Rating` versus serving either cuisine alone (n≥30 per pair); Healthy Food/Salad/Fast Food/American combinations do the opposite.
18. **Locality opportunity flags are almost entirely an NCR phenomenon** (165 of 167, 98.8%) — not because non-NCR has no gaps, but because non-NCR localities rarely reach the ≥30-restaurant threshold needed to flag reliably.
19. **The actionable rating model tops out at R² ≈ 0.40.** With `Log_Votes` excluded as leakage, `Locality` becomes the dominant predictor (~3× the next feature), `Is NCR` is second (negatively signed, consistent with finding 11), and cost/price-range and operational flags (booking, delivery, chain) add comparatively little — a modest R² is the honest finding, not a modeling shortfall.
20. **The cold-start classifier reaches AUC ≈ 0.89–0.90.** `Locality` and `Has Online delivery` are its strongest drivers; because 100% of unrated restaurants are in NCR, the outside-NCR subset is a trivial, single-class case (AUC undefined there) and NCR-only performance (≈0.89) is the number that reflects real predictive skill.

## Repository structure

```
├── README.md                    # This file
├── EDA.ipynb                    # Exploratory analysis (41 cells)
├── Data_cleaning.ipynb          # Cleaning, India filter, Phase 2 rebuild, outlier decision (37 cells)
├── Feature_Engineering.ipynb    # Cuisine/vote feature engineering (21 cells)
├── Deep_EDA.ipynb               # Univariate/bivariate/multivariate Deep EDA (45 cells)
├── Statistical_Validation.ipynb # Phase 4: hypothesis tests, effect sizes, Holm correction (42 cells)
├── Derived_Metrics.ipynb        # Phase 3.5: 5 derived metrics for Phase 5/6 (80 cells)
├── Modeling.ipynb                # Phase 5: rating prediction (5A) + cold-start classifier (5B) (58 cells)
├── Output/
│   ├── df_india_cleaned         # Phase 2 output: cleaned India base (8,643 rows)
│   ├── df_featured.csv          # Feature-engineered base used by Deep_EDA / Statistical_Validation
│   ├── phase4_test_results.csv  # Phase 4 test results: effect sizes, CIs, verdicts, caveats
│   ├── credibility_weighted_ratings.csv   # Phase 3.5: Bayesian-shrunk Cred_Rating per restaurant
│   ├── value_for_money_residuals.csv      # Phase 3.5: rating-vs-cost residual, ranked
│   ├── competition_density.csv            # Phase 3.5: BallTree competitor counts (500m/1km/2km)
│   ├── cuisine_pair_premium.csv           # Phase 3.5: rating/cost premium per cuisine pair (n≥30)
│   ├── locality_cuisine_crosstab.csv      # Phase 3.5: Locality × Primary Cuisine supply/demand
│   ├── locality_pricerange_crosstab.csv   # Phase 3.5: Locality × Price range supply/demand
│   ├── locality_opportunity_flags.csv     # Phase 3.5: flagged thin-supply/high-demand cells
│   ├── rating_model_comparison.csv        # Phase 5: 5A model comparison (RMSE/MAE/R² by framing/model/region)
│   ├── rating_model_predictions.csv       # Phase 5: 5A out-of-fold predicted rating per restaurant
│   ├── rating_feature_importance.csv      # Phase 5: 5A permutation importance by framing/model
│   ├── coldstart_risk_scores.csv          # Phase 5: 5B out-of-fold cold-start risk score, all 8,643 rows
│   └── coldstart_feature_importance.csv   # Phase 5: 5B permutation importance
└── Zomatodataset/                # Raw data (CSV, XLSX, JSON)
    ├── zomato.csv                # Main dataset (9,551 rows × 21 cols)
    ├── Country-Code.xlsx         # Country code → name mapping
    ├── file1–5.json              # Delhi restaurant photos/URLs (optional)
    └── Untitled.ipynb            # Empty scratch notebook
```

Tracking files live one level up (repo root): `plan.md` (roadmap), `progress.md` (task log + challenges), `ai_context.md` (LLM resume context).

## Important gotchas

- **CSV encoding:** the raw `zomato.csv` needs `encoding="latin-1"`; the `Output/` CSVs are UTF-8 (read with the default)
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
#    -> Statistical_Validation.ipynb -> Derived_Metrics.ipynb -> Modeling.ipynb
#    (Data_cleaning.ipynb imports the merged dataframe from EDA.ipynb via importnb;
#     the later notebooks read straight from the Output/ CSVs)
```

Requires Python 3 with pandas, numpy, matplotlib, seaborn, scipy, statsmodels, scikit-learn, xgboost, and `importnb` (Anaconda distribution covers most; install `xgboost` separately if missing).

## Roadmap ahead (India-focused)

1. **Phase 6 — Insights & final report:** business takeaways tied to the Phase 3.5 derived metrics and Phase 5 model outputs (cold-start risk scores, value-for-money/model residuals), summary charts, save final datasets, limitations section (91% NCR concentration, 24.7% unrated, votes/rating simultaneity, no time dimension, no true currency conversion, ~0.40 R² ceiling on the actionable rating model)
2. **Optional:** Delhi-focused analysis with `file1–5.json` (~2,358 Delhi restaurants)
