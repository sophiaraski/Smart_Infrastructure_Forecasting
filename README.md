# Siemens Smart Infrastructure — Monthly Sales Forecasting

A machine learning project developed as part of an academic case study on **Siemens Smart Infrastructure**, focused on forecasting monthly product-level sales using historical transactional data and macroeconomic market indicators.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Repository Structure](#repository-structure)
- [Datasets](#datasets)
- [Methodology](#methodology)
- [Models](#models)
- [Results](#results)
- [Requirements](#requirements)
- [How to Run](#how-to-run)
- [Authors](#authors)
---

## Project Overview

This project builds a sales forecasting pipeline for **14 Siemens product groups** (GCK categories), combining daily transactional sales records with a rich set of global macroeconomic indicators — production indices, shipment indices, commodity prices, and producer prices across 8+ countries and regions.

The pipeline includes end-to-end steps: data cleaning and preparation, exploratory analysis, feature engineering with lagged variables, feature selection, model selection, and future-period prediction.

**Forecast horizon:** Monthly sales from May 2022 to February 2023.

---

## Repository Structure

```
.
├── Notebook_Part1_final.ipynb   # Data preparation, exploration, and feature engineering
├── Notebook_Part2.ipynb         # Modelling and final predictions
├── Case2_Sales data.csv         # Daily sales records by product group (input)
├── Case2_Market data.xlsx       # Monthly macroeconomic indicators (input)
├── Case2_Test set Template.csv  # Test set template for final submission
├── df_*_selected.pickle         # Serialized feature-selected datasets per product group
└── README.md
```

---

## Datasets

### Sales Data (`Case2_Sales data.csv`)
- **Granularity:** Daily
- **Period:** October 2018 – April 2022
- **Content:** Sales in EUR per product group (GCK categories), pivoted into wide format
- **Preprocessing applied:**
  - Negative sales set to zero (returns/adjustments)
  - Missing dates filled with zero sales
  - German national holidays identified; confirmed zero-sale dates removed
  - Daily data aggregated to monthly totals for modeling

### Market Data (`Case2_Market data.xlsx`)
- **Granularity:** Monthly
- **Content:** Production and shipment indices (Machinery & Electricals), producer prices, electrical equipment indices, and commodity/world prices for China, France, Germany, Italy, Japan, Switzerland, UK, US, and Europe
- **Preprocessing applied:**
  - Column renaming for clarity
  - USD-denominated price columns converted to EUR using the USD/EUR exchange rate
  - Duplicate column (`China Shipments Index ME`) removed
  - Missing values retained intentionally to leverage CatBoost's native NaN handling

### Merged Dataset
Sales and market data are joined on a monthly date key. Three engineered boolean features are added:
- `Covid_boolean` — flags 2020–2022 to capture pandemic effects
- `Year_closure_boolean` — flags June–September to capture Siemens fiscal year-end sales pressure
- `Suez_block_boolean` — flags March 23–29, 2021 (Suez Canal blockage)

---

## Methodology

### Part 1 — Data Preparation & Feature Engineering (`Notebook_Part1_final.ipynb`)

1. **Data Loading & Cleaning** — Load CSV/XLSX files, fix decimal formats, standardize date columns, handle negative values and missing dates.
2. **Exploratory Data Analysis** — Visualize daily and monthly sales trends by year, weekday, and quarter; analyze production/shipment indices by country; compute moving averages and log-sales distributions.
3. **Feature Engineering**
   - Date decomposition (day, month, year, weekday)
   - National holiday flagging (Germany)
   - COVID, fiscal year closure, and Suez Canal boolean indicators
   - Creation of up to **12 monthly lag features** per macroeconomic variable
4. **Per-Product Feature Selection**
   - For each of the 14 product groups, the lag with the highest absolute correlation to the target is identified per macroeconomic feature
   - A `RandomForestRegressor` with `SelectFromModel` further filters to the most predictive features
   - Highly correlated feature pairs (r > 0.95) are deduplicated to reduce multicollinearity
5. **Stationarity & Autocorrelation Analysis** — ADF tests, ACF, and PACF plots are produced for each product group to inform model selection
6. **Train/Test Split Optimization** — Seven candidate split ratios (60%–90%) are evaluated using XGBoost; the split minimizing RMSE on a held-out chronological test set is selected per product

### Part 2 — Modelling & Forecasting (`Notebook_Part2.ipynb`)

1. **Model Comparison** — For each product group, three candidate approaches are compared by test-set RMSE:
   - Historical **Mean**
   - Historical **Median**
   - **CatBoost Regressor**
   - **Facebook Prophet** (for product groups with detected seasonality)
2. **Future Feature Projection** — Prophet is used to extrapolate macroeconomic features into the forecast horizon (May 2022 – February 2023)
3. **Final Predictions** — The best-performing model per product group generates forecasts; results are assembled into the submission format

---

## Models

| Product Group | Best Model | Notes |
|---|---|---|
| GCK_1 | CatBoost | No significant ACF/PACF; stationary |
| GCK_3 | CatBoost | Seasonal pattern at lag 12 |
| GCK_4 | CatBoost | Seasonal pattern at lag 6 |
| GCK_5 | CatBoost | Stationary; lag 6 in ACF |
| GCK_6 | CatBoost | Stationary; no autocorrelation |
| GCK_8 | CatBoost | Non-stationary; lag 3 autocorrelation |
| GCK_9 | CatBoost | Lag 10 partial autocorrelation |
| GCK_11 | CatBoost | Seasonal pattern at lag 12 |
| GCK_12 | CatBoost | ACF significant at lags 1–3 |
| GCK_13 | Mean / CatBoost | Stationary; no autocorrelation |
| GCK_14 | CatBoost | Stationary; lag 6 partial autocorrelation |
| GCK_16 | CatBoost | Stationary; lag 3 ACF/PACF |
| GCK_20 | CatBoost | Stationary; no autocorrelation |
| GCK_36 | CatBoost | Stationary; lag 10 partial autocorrelation |

**CatBoost** was selected as the primary model across most product groups due to its native support for missing values, strong performance on tabular data with mixed feature types, and competitive RMSE relative to Prophet and statistical baselines.

---

## Requirements

```
python >= 3.8
pandas
numpy
matplotlib
seaborn
plotly
scikit-learn
xgboost
catboost
prophet
statsmodels
pickle
```

Install all dependencies with:

```bash
pip install pandas numpy matplotlib seaborn plotly scikit-learn xgboost catboost prophet statsmodels
```

---

## How to Run

1. **Clone the repository** and place the input data files in the root directory:
   - `Case2_Sales data.csv`
   - `Case2_Market data.xlsx`
   - `Case2_Test set Template.csv`

2. **Run Part 1** to generate cleaned datasets and serialized feature sets:
   ```bash
   jupyter notebook Notebook_Part1_final.ipynb
   ```
   This will produce `df_*_selected.pickle` files for each product group.

3. **Run Part 2** to train models and generate the final forecast:
   ```bash
   jupyter notebook Notebook_Part2.ipynb
   ```
   The final output is a long-format DataFrame (`test_pred`) containing predicted monthly sales per product group for the test period.

> **Note:** Both notebooks must be run from the same working directory, as Part 2 reads the pickle files produced by Part 1.
