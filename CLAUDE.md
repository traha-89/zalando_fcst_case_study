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
- Day-of-week encoding: **1 = Monday, 7 = Sunday** (verified against pandas `dt.dayofweek`)

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
The lag distribution must be segmented by `day_of_week_order`. Fri/Sat/Sun have distinctly different profiles from weekdays:
- **Friday:** lag 2 = 0%, spike at lag 3 (22%) — Friday orders skip Saturday processing, arrive Monday
- **Saturday:** lag 1 = 3%, lag 2 = 37% — most Saturday orders arrive Monday
- **Sunday:** lag 1 = 15%, lag 2 = 35% — reduced Sunday warehouse operations
- **Mon–Thu:** broadly similar (lag 0: ~43–50%, lag 1: ~22–25%, lag 2: ~21–33%)

### Lag cap at 4 days
Item-weighted percentiles (correct view for warehouse planning):
- Lag 0–2: **91.7%** of item volume
- Lag 0–3: **98.2%**
- Lag 0–4: **~99.9%**

Cap the lag distribution at **lag ≤ 4 days**. The row-weighted 99th percentile of 16 days significantly overstates the tail — driven by many rows with tiny trickle amounts at long lags.

### May spillover must be accounted for
Late May orders (particularly from ~May 27 onwards) contribute to early June warehouse receipts. This is computable directly from input_1 since it contains actual May order data with WH receive dates extending to June 11.

### Calendar-week segmentation not pursued
The CW chart shows stable order/receiving alignment across all ~22 calendar weeks. Segmenting the lag distribution by CW would produce thin, unreliable buckets. Not pursued.

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

## Notebook Structure (analysis.ipynb)
1. Load Data
2. Data Quality Checks
   - input_1: shape, missing values, date range, negatives, duplicates, share sums, outliers (IQR + magnitude gap), imputation
   - input_2: shape, missing values, date range, negatives, outliers, day-of-week distribution, cross-dataset encoding + coverage checks
   - Quality summaries for both inputs
3. Data Exploration
   - 3.1 Orders by Month (Trendline) ✓
   - 3.2 Orders vs. Receivings by Calendar Week ✓
   - 3.3 Orders vs. Receivings by Day of Week ✓
   - 3.4 Outliers in Historical Receiving Data ✓
   - 3.5 Lag Distribution Analysis ✓
   - 3.6 Forecasted Items by Day of Week ✓
   - 3.7 Merged into 3.2 — June forecast CWs (CW23–26) extended onto the CW chart with shaded region + vertical dividing line to distinguish input_1 vs input_2 ✓
   - 3.8 May Spillover Quantification *(pending)*

---

## Presentation Theme
Zalando brand colours throughout:
- **Orange** `#FF6900` — primary / actuals
- **Dark grey** `#1A1A1A` — secondary / forecast / lines
- **Mid grey** `#666666` — tertiary / annotations
- White background, `whitegrid` seaborn theme
