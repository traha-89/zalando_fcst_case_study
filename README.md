# Zalando — Warehouse Inbound Forecast

Logistics network planning case study for a large online retailer (Zalando), focused on predicting **daily warehouse receipt volumes for June 2022** from a sales order forecast.

---

## Purpose

The goal is to answer: *given a June 2022 order forecast, on which days will items physically arrive at the warehouse?*

Orders are not received on the same day they are placed — they spread across the following 0–4 days depending on the day of the week (weekend warehouse closures, carrier schedules). The model learns this lag pattern from 5 months of historical order-to-receipt data and applies it to the June forecast. The output supports warehouse staffing and inbound capacity planning.

---

## Folder Structure

```
zalando_fcst_case_study/
│
├── data_input/                  # Raw input files
│   ├── input_1_histrocial.csv   # Jan–May 2022 order & receipt history (1,763 rows)
│   ├── input_2_sales_fc.csv     # June 2022 daily order forecast (30 rows)
│   └── *.pdf                    # Original case study brief
│
├── data_output/                 # Model output
│   └── 202402_Data - Case Study - Analyst Network Planning_vShared - Expected_output.csv
│
├── fcst_report/                 # Presentation assets
│   ├── forecast_report.md       # Full written report with methodology & results
│   ├── images/                  # Charts exported from the notebook
│   └── *.pptx                   # Slide deck (Zalando brand theme)
│
├── analysis.ipynb               # Main analysis notebook (run this)
├── CLAUDE.md                    # Project context & decisions for Claude Code
├── MODELING_PLAN.md             # Detailed modelling plan with code patterns
└── requirements.txt             # Python dependencies
```

---

## How to Run

```bash
# 1. Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Launch the notebook
jupyter lab analysis.ipynb
```

Run cells top to bottom. The notebook is self-contained — it reads from `data_input/`, performs all analysis, and writes the forecast to `data_output/`.

---

## Using Claude Code

This repo is set up for use with [Claude Code](https://claude.ai/code). The `CLAUDE.md` file provides Claude with full project context — data schema, key decisions, validation results, and modelling approach — so it can assist meaningfully without re-reading every file.

```bash
# Start Claude Code in the project root
claude
```

Claude can help with: extending the analysis, adjusting the lag model, explaining methodology, or generating the presentation from `fcst_report/forecast_report.md`.

---

## Final Output

`data_output/202402_Data - Case Study - Analyst Network Planning_vShared - Expected_output.csv` — 30 rows covering Jun 1–30, 2022:

| Column | Description |
|--------|-------------|
| `date_wh_receive` | Date items arrive at the warehouse |
| `items` | Predicted daily receipt volume (integer) |

**Total forecast: 2,435,582 items** across June 2022.

Key patterns in the output:
- Sundays consistently low (23k–46k) — reduced warehouse operations
- Mondays consistently peak (76k–101k) — absorb Fri/Sat/Sun spillover
- Jun 1–4 elevated — 75,284 items of known May spillover added
- Jun 27–30 spike — driven by a high-order day on Jun 28 (183,785 orders)

The model was validated out-of-sample on May 2022: MAE of 7,862 items/day (11.6% MAPE), near-zero bias, monthly total within 0.5%.
