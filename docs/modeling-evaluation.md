# Modeling and Evaluation

Model selection, training protocol, metrics, and the rolling-origin evaluation methodology used in karAQI.

---

## Models Evaluated

Six models were evaluated during development. Two were selected for production (CI/CD).

### Production Models

| Model | Type | Why Selected |
|---|---|---|
| **Ridge Regression** | Linear (L2-regularized) | Fast, interpretable, stable across all horizons |
| **XGBoost** | Gradient-boosted trees | Best RMSE on hourly points (1–24h), captures non-linear interactions |

### Development-Only Models

| Model | Type | Why Excluded from CI |
|---|---|---|
| Random Forest | Bagged trees | Excluded from CI for deployment simplicity |
| LSTM | Deep learning (RNN) | 1+ hour training on CPU, worst RMSE (24.21) |
| SARIMA | Statistical (seasonal ARIMA) | Negative R², performs worse than persistence baseline |
| Persistence | Naive baseline | Used as release gate benchmark, not a trained model |

---

## Training Protocol

### Daily Training (`src/train.py`)

- **Targets:** `target_1d`, `target_2d`, `target_3d` (next-day AQI for 1/2/3 days ahead)
- **Features:** 43 columns (pollutants, weather, lags, rolling stats, cyclical time)
- **Split:** Chronological holdout — train on 2022-08-12 to 2024-12-31, test on 2025-01-01 to present
- **Models:** Ridge, XGBoost, Random Forest, SARIMA, LSTM, Persistence
- **Evaluation:** RMSE, MAE, R² per horizon

### Hourly Training (`src/train_hourly.py`)

- **Targets:** 30 outputs — 24 hourly points (`aqi_plus_01h` through `aqi_plus_24h`), four 6-hour block means, two 12-hour block means
- **Features:** 63 columns (same as daily plus hourly-specific lags and cyclical features)
- **Models:** Ridge, XGBoost (only — RF and LSTM excluded from CI)
- **Evaluation:** Rolling-origin cross-validation with 3 expanding folds

---

## Evaluation Protocol

### Rolling-Origin Cross-Validation (Hourly)

The hourly model uses expanding-window evaluation:

1. **Fold 1:** Train on first 80% of data, test on next 168 hours (7 days)
2. **Fold 2:** Train on first 85%, test on next 168 hours
3. **Fold 3:** Train on first 90%, test on next 168 hours

Each test window is 168 hours (7 days) with a 72-hour embargo (gap between train and test to prevent data leakage from autocorrelation).

### Output Groups

Predictions are evaluated on three groups reflecting the dashboard's display:

| Group | Outputs | Horizon |
|---|---|---|
| Hourly points | 24 individual hourly AQI values | 1–24h |
| Six-hour means | 4 block averages | 25–48h |
| Twelve-hour means | 2 block averages | 49–72h |

### Release Gate

The pipeline only deploys a model if it beats persistence on average across all groups:

```
Average RMSE(best_model) < Average RMSE(persistence)
Average MAE(best_model) < Average MAE(persistence)
```

If the gate fails, the pipeline crashes and no model is deployed. No fallbacks.

---

## Metrics

### Hourly Model — Rolling-Origin Results

| Output Group | RMSE | MAE | R² | Category Accuracy | High-AQI Recall |
|---|---:|---:|---:|---:|---:|
| Hourly points (1–24h) | 11.16 | 8.59 | 0.326 | 90.0% | 69.3% |
| Six-hour means (25–48h) | 18.52 | 13.95 | -0.026 | 77.7% | 55.3% |
| Twelve-hour means (49–72h) | 19.11 | 15.33 | -0.110 | 66.8% | 48.4% |
| Persistence baseline (1–24h) | 12.57 | 9.73 | 0.142 | 87.2% | 62.8% |
| Persistence baseline (25–48h) | 22.95 | 17.91 | -0.556 | 69.7% | 56.0% |
| Persistence baseline (49–72h) | 27.50 | 22.95 | -1.269 | 56.3% | 51.6% |

### Daily Model Comparison

| Model | +1 Day RMSE | +1 Day R² | +2 Day RMSE | +2 Day R² | +3 Day RMSE | +3 Day R² |
|---|---:|---:|---:|---:|---:|---:|
| XGBoost | 15.78 | 0.513 | 19.40 | 0.263 | 20.72 | 0.168 |
| Random Forest | 16.28 | 0.482 | 19.48 | 0.257 | 20.71 | 0.169 |
| Ridge | 16.66 | 0.457 | 19.98 | 0.219 | 20.66 | 0.173 |
| Persistence | 18.55 | 0.326 | 22.95 | -0.018 | 25.38 | -0.259 |

### Metric Definitions

- **RMSE** — Root mean squared error in AQI units. Lower is better. RMSE of 11 means typical error ±11 AQI points.
- **R²** — Variance explained relative to predicting the mean. Negative means worse than guessing the average.
- **Category accuracy** — Percentage of predictions in the correct US EPA AQI band.
- **High-AQI recall** — Percentage of truly polluted hours the model correctly flags (important for health alerts).

---

## Champion Comparison

The training pipeline compares new model metrics against the current champion:

1. Train all models
2. Evaluate using rolling-origin protocol
3. Find best model = lowest average RMSE across all groups
4. Load current champion from MLflow registry
5. Compare: if new model is better → promote. If worse → keep old champion, log comparison
6. Push updated model files and eval JSON to karAQI-data

---

## Why XGBoost Wins Over Ridge in Hourly

XGBoost dominates on hourly points (1–24h) with RMSE 8.89 vs Ridge's 10.88 — an 18% improvement. This matters because hourly points represent 24 of 30 outputs. Ridge wins slightly on longer ranges (25–72h), but XGBoost's advantage on the primary group outweighs it in the average.

XGBoost captures non-linear interactions (temperature × humidity × time-of-day) that Ridge's linear model cannot. For short-term predictions, these interactions matter. For 49–72h ahead, the signal degrades and linear models are more stable.
