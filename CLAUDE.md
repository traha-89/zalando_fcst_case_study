# Zalando Network Planning — Project Context

## Project Overview
Logistics network planning case study for a large online retailer (Zalando), focused on optimising inbound operations within a single warehouse. The goal is to predict **daily warehouse receipt volumes for June 2022** by learning order-to-receipt lag patterns from historical data and applying them to a sales forecast.

---

## Folder Structure
```
zalando_fcst_case_study/
├── data_input/          # Input data files
├── data_output/         # Output files (expected_output.csv to be filled)
├── analysis.ipynb       # Main analysis notebook
├── .venv/               # Python 3.13 virtual environment (gitignored)
```

## Environment
- Python 3.13 (Homebrew) in `.venv`
- Key packages: `pandas`, `numpy`, `matplotlib`, `seaborn`, `scipy`

---

## Data Sources

### input_1 — Historical Data (`data_input/`)
- 1,763 rows × 6 columns
- Covers 151 unique order dates: **Jan 1 – May 31, 2022**
- WH receive dates extend to **Jun 11, 2022** (late May orders spill into June)
- Columns: `date_order`, `day_of_week_order`, `date_wh_receive`, `day_of_week_wh_receive`, `CW` (calendar week of receipt), `items`
- Day-of-week encoding: **1 = Monday, 7 = Sunday** (ISO weekday standard, 1-indexed). Note: pandas `dt.dayofweek` is 0-indexed (0 = Monday, 6 = Sunday) — differs by 1 for every day. Verified by cross-checking Jan 1 2022 (Saturday) = 6 in input data, consistent with 1-indexed encoding.

### input_2 — Sales Forecast (`data_input/`)
- 30 rows × 3 columns
- Covers all 30 days of **June 2022**
- Columns: `date_order`, `day_of_week_order`, `Forecated_items` (note: typo in source data)

### expected_output — Output Template (`data_output/`)
- 30 rows: `date_wh_receive` (Jun 1–30), `items` (to be filled)

---

## Data Cleaning

### input_1 — One anomaly detected and imputed
- **Row 1731 (May 29):** `items = 27,533,000` — identified as a data entry error via magnitude gap check (367.9x the next highest value of 74,843)
- IQR method was trialled but rejected — it flagged 326 rows (~18.5%) due to the dataset's structural right-skew (many small trickle rows at long lags pull Q1 down to 19)
- **Treatment:** imputed with the median lag=0 value for Sunday orders (32,161), preserving the realistic shape of May 29's lag distribution
- After imputation, May 2022 total orders: ~2.24M (previously inflated to ~29.7M)

### input_2 — One high value noted, retained
- **Jun 28:** `Forecated_items = 183,785` — flagged by IQR (upper bound: 142,837)
- Assessed as plausible: only 1.3x the upper bound, no magnitude error, business-provided forecast
- **Treatment:** retained as-is. May represent a planned promotion or end-of-month sales event. Will manifest as a receipt spike around Jun 29–30 in the output.

---

## Key Decisions

### No ML for lag distribution
ML was ruled out for the lag distribution estimation step. Dataset is too small (~151 unique order dates) for ML to reliably generalise. The lag relationship has a clear physical/operational mechanism that statistical methods model directly. ML would add complexity without meaningful accuracy gains and reduces interpretability for a business audience. **Use segmented statistical methods only.**

### Day-of-week segmentation confirmed
The lag distribution must be segmented by `day_of_week_order`. Fri/Sat/Sun have distinctly different profiles from weekdays.

**Jan–Apr train set** (used for May validation):

| DOW | lag 0 | lag 1 | lag 2 | lag 3 | lag 4 |
|-----|-------|-------|-------|-------|-------|
| Mon | 41.0% | 23.8% | 34.1% | 1.0% | 0.2% |
| Tue | 47.8% | 25.4% | 25.7% | 0.8% | 0.2% |
| Wed | 48.8% | 23.8% | 24.5% | 2.9% | 0.0% |
| Thu | 49.5% | 27.7% | 20.6% | 0.3% | 1.9% |
| Fri | 52.3% | 19.2% | 0.4%  | 22.6%| 5.6% |
| Sat | 45.4% | 1.8%  | 39.7% | 11.2%| 1.8% |
| Sun | 46.7% | 15.8% | 34.9% | 2.2% | 0.4% |

**Full Jan–May history** (used for June forecast):

| DOW | lag 0 | lag 1 | lag 2 | lag 3 | lag 4 |
|-----|-------|-------|-------|-------|-------|
| Mon | 41.2% | 22.4% | 33.6% | 2.5% | 0.3% |
| Tue | 46.7% | 24.7% | 26.8% | 1.6% | 0.2% |
| Wed | 48.9% | 22.8% | 25.1% | 3.2% | 0.0% |
| Thu | 49.9% | 26.1% | 21.6% | 0.3% | 2.1% |
| Fri | 52.5% | 19.2% | 0.4%  | 22.6%| 5.4% |
| Sat | 45.7% | 3.3%  | 38.3% | 10.8%| 1.9% |
| Sun | 43.3% | 15.7% | 36.9% | 3.3% | 0.7% |

Shifts (Jan–May minus Jan–Apr): largest is Sunday lag 0 (-3.4pp) and lag 2 (+2.0pp). All structural patterns preserved.

Key patterns:
- **Friday**: lag 2 collapses to 0.4%, spike at lag 3 (22.6%) — Friday orders skip Sunday processing, arrive Monday–Tuesday
- **Saturday:** near-zero lag 1 (3.3%), large lag 2 peak (38.3%) — most Saturday orders arrive Monday
- **Sunday:** split between lag 0 (43.3%) and lag 2 (36.9%), depressed lag 1 (15.7%) — reduced Sunday warehouse operations
- **Monday:** lag 2 (33.6%) exceeds lag 1 (22.4%) — absorbs Friday/weekend spillover from prior week
- **Tue–Thu:** lag 0 dominates (47–50%), lags 1 and 2 broadly balanced (~21–27%)

### Lag cap at 4 days
Item-weighted percentiles (correct view for warehouse planning):
- Lag 0–2: **91.7%** of item volume
- Lag 0–3: **98.2%**
- Lag 0–4: **~99.9%**

Cap the lag distribution at **lag ≤ 4 days**. The row-weighted 99th percentile of 16 days significantly overstates the tail — driven by many rows with tiny trickle amounts at long lags.

### May spillover must be accounted for
Late May orders contribute to early June warehouse receipts. Quantified directly from input_1:
- **Total spillover: 76,058 items (3.4% of May orders)**
- **Jun 1: 51,835 (68.2%), Jun 2: 20,650 (27.2%), Jun 3: 2,626 (3.5%), Jun 4: 173 (0.2%)**
- Jun 5–11: long-tail trickles (~774 items total) — lag > 4, excluded by model cap
- **Model boundary: Jun 4** — from Jun 5 onwards only June orders contribute
- Jun 1–4 spillover values are exact known quantities from input_1, no estimation needed

### Calendar-week segmentation not pursued
The CW chart shows stable order/receiving alignment across all ~22 calendar weeks. Segmenting the lag distribution by CW would produce thin, unreliable buckets. Not pursued.

### Aggregate-shares chosen over mean-of-shares for lag distribution
Two approaches were compared for computing the lag share per (day_of_week, lag):
- **Mean-of-shares:** average of per-date share values — each order date weighted equally regardless of volume
- **Aggregate-shares:** sum(items at lag) / sum(total items) per day-of-week — each item weighted equally

Max difference: **2.12pp on Tue lag 0** (mean: 0.63pp across all DOW/lag combos). Top-5 differences (1.51–2.12pp) all fall on lag 0–2 for Tue, Mon, Sun, Sat. Direction varies: mean-of-shares overstates lag 0 for Tue/Mon (low-volume dates with high same-day rates pull average up); understates lag 2 for Sun/Sat. Both effects push items earlier than they actually arrive.
**Decision: aggregate-shares.** We are distributing item volumes, so weighting each item equally is the correct approach for warehouse planning.

### `share` column definition
`hist["share"] = hist["items"] / hist["date_order"].map(total_per_order)`
For each row: fraction of that order date's total items that arrived at a specific lag. Shares per order date sum to exactly 1.0 (verified in data quality section). Used in mean-of-shares approach only — aggregate approach works directly from `items`.

---

## Exploration Findings (Section 3)

### 3.1 Orders by Month
- Upward trend Jan–May 2022 (with Feb trough — shortest month)
- Month-on-month growth: +12% Mar, +12% Apr, +4% May
- June forecast (2.43M) sits above the trendline — ~8.6% above May, consistent with seasonal demand acceleration
- Trendline uses `scipy.stats.linregress` (slope + intercept) for interpretability

### 3.2 Orders vs. Receivings by Calendar Week
- Orders and receivings track closely for most historical weeks — lag is short and concentrated within the same/next week
- CW3–CW4 spike: likely post-Christmas/January sales effect
- CW7–CW12 dip: aligns with February seasonal trough
- CW22 partial week: order bar artificially small — only captures May 30–31 (2 days); receivings are higher as prior weeks' orders continue to arrive
- CW23 shows both: spillover receivings from input_1 (Jun 6–11 arrivals of late-May orders) + Jun 6–12 forecast orders side by side on same x-position
- CW26 shorter bar: partial week only (Jun 27–30, 4 days) — not a demand drop; daily average aligns with rest of June
- Chart extended to include June forecast (CW23–26) as lighter orange bars, with shaded region + vertical dividing line to distinguish input_1 (actual) from input_2 (forecast). Forecast order volumes are higher than comparable historical weeks, consistent with ~8.6% June demand uplift.

### 3.3 Orders vs. Receivings by Day of Week
- Weekdays broadly aligned; receivings meet or slightly exceed orders due to spillover
- Sunday: orders high (~76k) but receivings very low (~33k) — confirms reduced Sunday warehouse operations
- Heatmap confirms day-of-week segmentation is warranted (see Key Decisions above)

### 3.4 Outliers in Historical Receiving Data
- No anomalous days in daily receiving totals (magnitude gap check: 0 flagged)
- No zero-receiving days across Jan–May 2022 — warehouse operated every calendar day
- Data is clean at both row and daily aggregate level

### 3.5 Lag Distribution Analysis
- Item-weighted view is the correct measure for warehouse planning (each item weighted equally, not each row)
- Confirmed lag cap at 4 days covering ~99.9% of item volume
- Visualisation: Zalando-themed bar chart (orange `#FF6900`) + cumulative line (dark `#1A1A1A`)

### 3.6 Forecasted Items by Day of Week
- June forecast day-of-week shape is consistent with historical pattern (Mon–Thu highest, Fri drops, Sat lowest, Sun recovers)
- June forecast is uniformly ~20–30% higher on weekdays — reflects overall higher June demand, not a structural shift in day-of-week pattern
- Sunday slightly below history (~73k forecast vs ~76k historical) — within normal variation
- Saturday closely aligned (~51–54k) — most consistent day between history and forecast
- **Modelling implication:** day-of-week segmented lag distribution is well-calibrated for the June forecast

---

## Section 4 — Modelling Approach

### Method
Day-of-week segmented lag distribution applied to the June order forecast. The model is a distribution mechanism — it smears each day's forecast orders across receipt dates using the historical lag profile. Level (total June demand) comes entirely from input_2; the model adds the operational timing pattern on top.

### Structure (see MODELING_PLAN.md for full detail + code)
- **4.1** Build lag distribution on Jan–Apr (train set)
- **4.2** Validate on May — apply to actual May orders + April spillover; compare to actual May receipts
- **4.3** Rebuild lag distribution on full Jan–May history
- **4.4** Apply to June forecast + May spillover → write `data_output/expected_output.csv`

### Apply function pattern
```python
# Merge orders with lag distribution on day_of_week_order
# receipt_date = date_order + lag days
# receipt_items = Forecated_items * normalised_share
# Filter to target month, group by receipt_date, add spillover
```

### Validation results (4.2 — May, out-of-sample)
| Metric | Value | Interpretation |
|--------|-------|----------------|
| MAE    | 7,862 items/day | ~11% of average daily receipts (72,200) — timing error |
| MAPE   | 11.6% | Acceptable for a statistical lag model |
| Bias   | +343 items/day | Negligible — no systematic over/under-forecast |
| RMSE   | 10,720 items | RMSE/MAE = 1.36 — no extreme peak-day misses |

Monthly total: predicted 2,249,715 vs actual 2,239,069 (0.5% over). Model is well-calibrated.
April spillover into May: 75,050 items across 15 receipt dates.

### May spillover for Jun 1–4 (exact values from input_1, no lag cap)
Jun 1: 51,835 | Jun 2: 20,650 | Jun 3: 2,626 | Jun 4: 173 | Total: 76,058 items
Note: lag cap applies to the model only — actual historical spillover is used as-is for Jun 1–4.

### June forecast output (4.4)
- **Total receipts: 2,435,582 items** vs June order forecast of 2,433,610 (+1,972 net)
- Receipts exceed the order total because May spillover Jun 1–4 (75,284 items) is larger than the month-end taper (~73,312 items from Jun 27–30 lag 1–4 falling in July)
- Jun 27–30 spike correctly reflects the Jun 28 high-order day (183,785 items) propagating through lags
- Sundays consistently low; Mondays consistently peak — day-of-week pattern preserved throughout

### Key caveats
- Validation tests lag model accuracy only — input_2 forecast accuracy is a separate, uncontrollable source of error
- Month-end taper: late June order contributions (lag 1–4) fall in July. Net receipt total may be slightly above or below the June order total depending on whether spillover outweighs taper — not an error

---

## Notebook Structure (analysis.ipynb)
1. Load Data
2. Data Quality Checks
   - input_1: shape, missing values, date range, negatives, duplicates, share sums, outliers (IQR + magnitude gap), imputation
   - input_2: shape, missing values, date range, negatives, outliers, day-of-week distribution, cross-dataset encoding + coverage checks
   - Quality summaries for both inputs
3. Data Exploration
   - 3.1 Orders by Month (Trendline) ✓
   - 3.2 Orders vs. Receivings by Calendar Week ✓ (extended to include Jun forecast CW23–26)
   - 3.3 Orders vs. Receivings by Day of Week ✓
   - 3.4 Outliers in Historical Receiving Data ✓
   - 3.5 Lag Distribution Analysis ✓
   - 3.6 Forecasted Items by Day of Week ✓
   - 3.7 Removed — June forecast CWs folded into 3.2
   - 3.8 May Spillover Quantification ✓
4. Forecast Generation
   - 4.1 Build lag distribution (Jan–Apr train set) ✓
     - Train set: 120 unique order dates, 17–18 per day-of-week ✓
     - Aggregate-shares approach selected over mean-of-shares ✓
     - Normalised lag distribution (7×5 table) ✓ (exact values in Key Decisions above)
   - 4.2 Validate on May — metrics + actual vs predicted chart ✓
     - MAE: 7,862 | MAPE: 11.6% | Bias: +343 | RMSE: 10,720 (see Validation results above)
   - 4.3 Rebuild lag distribution (full Jan–May) ✓ (exact values in Key Decisions above)
   - 4.4 Apply to June + write expected_output.csv ✓ (total receipts: 2,435,582 items)

---

## Presentation Theme
Zalando brand colours throughout:
- **Orange** `#FF6900` — primary / actuals
- **Dark grey** `#1A1A1A` — secondary / forecast / lines
- **Mid grey** `#666666` — tertiary / annotations
- White background, `whitegrid` seaborn theme
