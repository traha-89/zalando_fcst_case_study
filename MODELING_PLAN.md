# Section 4 — Modeling Plan: June Receipt Forecast

## Goal
Predict daily warehouse receipt volumes for June 1–30, 2022 by applying a day-of-week segmented lag distribution to the June order forecast, then adding known May spillover for early June dates.

---

## Section Structure

```
4.1 Build lag distribution        → trained on Jan–Apr only (train set)
4.2 Validate on May               → apply to actual May orders + April spillover
                                    report MAE, MAPE, Bias, RMSE + actual vs forecast chart
4.3 Rebuild lag distribution      → retrain on full Jan–May history
4.4 Apply to June                 → apply to fc + May spillover → write output
```

Validation comes before the June forecast intentionally — it proves the approach works before the final output is shown.

---

## 4.1 Build Lag Distribution (Jan–Apr Train Set)

Train set: 120 unique order dates, 17–18 per day-of-week (well-balanced).

**Aggregate-shares approach** (chosen over mean-of-shares — see Key Decisions in CLAUDE.md):

```python
hist_train = hist[hist["date_order"].dt.month <= 4]
hist_train_capped = hist_train[hist_train["lag"] <= 4]

# Total items per day-of-week (denominator)
# date_order is the lowest granularity — direct groupby is equivalent to two-step
total_items_by_dow = hist_train_capped.groupby("day_of_week_order")["items"].sum()

# Aggregate share per (day_of_week, lag)
lag_dist_agg = (
    hist_train_capped
    .groupby(["day_of_week_order", "lag"])["items"]
    .sum()
    .div(total_items_by_dow, level="day_of_week_order")
)

# Normalise so each day-of-week sums to exactly 1.0; rename to "share" for apply function
lag_dist_norm = (lag_dist_agg / lag_dist_agg.groupby("day_of_week_order").sum()).rename("share")
```

---

## 4.2 Validate on May

### Prepare May inputs

**May orders** — aggregate input_1 May rows to daily totals, mirroring the structure of `fc`:

```python
may_orders = (
    hist[hist["date_order"].dt.month == 5]
    .groupby(["date_order", "day_of_week_order"])["items"]
    .sum()
    .reset_index()
    .rename(columns={"items": "Forecated_items"})
)
```

Renaming to `Forecated_items` (matching the typo in `fc`) allows the apply function to be reused unchanged.

**April spillover into May** — exact known receipts from input_1:

```python
april_spillover = (
    hist[
        (hist["date_order"].dt.month == 4) &
        (hist["date_wh_receive"].dt.month == 5)
    ]
    .groupby("date_wh_receive")["items"]
    .sum()
)
```

**Actual May receipts** — the ground truth to compare against:

```python
actual_may = (
    hist[hist["date_wh_receive"].dt.month == 5]
    .groupby("date_wh_receive")["items"]
    .sum()
)
```

### Apply lag distribution

```python
def apply_lag_model(orders_df, lag_dist_norm, spillover_series, target_month):
    fc_lag = orders_df.merge(
        lag_dist_norm.reset_index(), on="day_of_week_order"
    )
    fc_lag["receipt_date"] = fc_lag["date_order"] + pd.to_timedelta(fc_lag["lag"], unit="D")
    fc_lag["receipt_items"] = fc_lag["Forecated_items"] * fc_lag["share"]

    receipts = (
        fc_lag[fc_lag["receipt_date"].dt.month == target_month]
        .groupby("receipt_date")["receipt_items"]
        .sum()
    )
    return receipts.add(spillover_series, fill_value=0)
```

```python
predicted_may = apply_lag_model(may_orders, lag_dist_norm, april_spillover, target_month=5)
```

### Error metrics

```python
comparison = pd.DataFrame({
    "actual": actual_may,
    "predicted": predicted_may
}).dropna()

mae   = (comparison["actual"] - comparison["predicted"]).abs().mean()
mape  = ((comparison["actual"] - comparison["predicted"]).abs() / comparison["actual"]).mean() * 100
bias  = (comparison["predicted"] - comparison["actual"]).mean()
rmse  = ((comparison["actual"] - comparison["predicted"]) ** 2).mean() ** 0.5
```

| Metric | Description | Warehouse planning relevance |
|---|---|---|
| **MAE** | Mean absolute daily error (items) | Primary — operationally meaningful |
| **MAPE** | Mean absolute % error | Relative error for business audience |
| **Bias** | Mean signed error (predicted − actual) | Systematic over/under-forecast — worse than random for staffing |
| **RMSE** | Root mean squared error | Penalises peak-day misses — capacity risk |

### Validation chart

Actual vs predicted daily receipts for May as a line chart (orange = actual, dark grey = predicted). Day-of-week patterns and any systematic bias should be immediately visible.

---

## 4.3 Rebuild Lag Distribution on Full Jan–May History

Same logic as 4.1 but using the complete `hist` dataset:

```python
hist_capped = hist[hist["lag"] <= 4]
total_items_by_dow_full = hist_capped.groupby("day_of_week_order")["items"].sum()

lag_dist_full = (
    hist_capped
    .groupby(["day_of_week_order", "lag"])["items"]
    .sum()
    .div(total_items_by_dow_full, level="day_of_week_order")
)
lag_dist_full_norm = (lag_dist_full / lag_dist_full.groupby("day_of_week_order").sum()).rename("share")
```

More data → more reliable estimates, especially for weekend days (fewer unique dates).

---

## 4.4 Apply to June Forecast

May spillover (already computed in section 3.8):

```python
june_receipts = apply_lag_model(fc, lag_dist_full_norm, spillover_daily, target_month=6)
```

Where `spillover_daily` is the daily May spillover series from section 3.8:
- Jun 1: 51,835 | Jun 2: 20,650 | Jun 3: 2,626 | Jun 4: 173

### Final output

```python
output = (
    june_receipts
    .reindex(pd.date_range("2022-06-01", "2022-06-30"))
    .fillna(0)
    .round()
    .astype(int)
    .reset_index()
)
output.columns = ["date_wh_receive", "items"]
output.to_csv("data_output/expected_output.csv", index=False)
```

### Sanity check

Total output items should be slightly less than the June forecast total (2,433,610) — the difference represents late-June orders arriving in July (Jun 27–30 lag 1–4 contributions fall outside the output window). This is correct behaviour, not an error.

---

## Trend Preservation

| Trend | Preserved? | How |
|---|---|---|
| Monthly growth (+8.6% Jun vs May) | ✓ | Directly from input_2 total volume |
| Day-of-week order pattern | ✓ | Directly from input_2 daily values |
| Day-of-week lag/processing pattern | ✓ | Core of the segmented lag distribution |
| Weekly receipt peaks (Monday heavy) | ✓ Emergent | Convolution of DOW order volume × lag shares |
| Jun 28 spike (183k orders) | ✓ | Propagates naturally to Jun 28–30 receipts |
| Month-end taper | Correct by design | Late June orders (lag 1–4) arrive in July — receipt output will decline in last 4 days even if orders are high. Not a model error — communicate explicitly. |

---

## Key Caveats

1. **Lag distribution accuracy vs forecast accuracy are independent.** May validation tests only the lag model. If input_2 is wrong, the receipt forecast will be wrong by a similar proportion — outside the model's control.
2. **Month-end taper:** Jun 27–30 lag contributions fall in July and are excluded. The output will show declining receipts at month-end even though orders are not declining.
3. **No ML:** Dataset too small (~151 unique order dates) and the operational mechanism is clear. Statistical segmentation is more appropriate and interpretable.
