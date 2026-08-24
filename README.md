# karAQI — Karak Air Quality Index Predictor

A **72-hour US EPA AQI forecast** for Karak, Pakistan, built as an end-to-end machine learning pipeline with automated CI/CD, a feature store, model registry, and a live Streamlit dashboard. The system fetches hourly pollutant and weather data from Open-Meteo (free, keyless), trains multiple ML models daily, and serves pre-computed predictions to every dashboard visitor with near-instant page load.

## Project Name Explanation

1. **Karaqi** stands for **Karak Air Quality Index**.
2. **Karaqi** refers to the people of Karak, serving as a direct reference to the region and its residents.

> **Data disclaimer:** Karak has no local ground-air-quality monitor. All features and the AQI target are derived from Open-Meteo CAMS modeled/reanalysis data, not station measurements. Results describe model-to-model agreement, not ground-truth station accuracy.

---

> **⚠️ IMPORTANT: This project uses TWO separate GitHub repositories.**
>
> | Repository | What it contains | Link |
> |---|---|---|
> | **karAQI** (this repo) | Source code, CI/CD pipelines, tests, documentation | [faisaliqbalkhattak/karAQI](https://github.com/faisaliqbalkhattak/karAQI) |
> | **karAQI-data** | Trained model files, forecast JSON, evaluation metrics, weather images | [faisaliqbalkhattak/karAQI-data](https://github.com/faisaliqbalkhattak/karAQI-data) |
>
> The code repo contains **zero** model files, **zero** data files, and **zero** generated artifacts. All ML outputs live in the data repo. The CI pipelines bridge the two: training pushes model files to karAQI-data, the forecast pipeline reads them back for inference.
>
> This two-repo architecture keeps the code lightweight, avoids git history bloat from binary model files, and ensures the dashboard can fetch pre-computed predictions via raw GitHub URLs with near-instant page load.

---

## Why Static JSON Serving (Not Runtime Inference)

Most ML dashboards load the model at page load and run inference for every visitor. This is slow — especially on Streamlit Cloud where cold starts are common.

**karAQI uses a different approach:** CI pipelines pre-compute all predictions and store them as JSON files in the `karAQI-data` repo. The dashboard fetches these pre-computed JSONs via raw GitHub URLs. No model is loaded. No inference runs. Every visitor gets the same near-instant page load.

| Approach | Page Load | Model Loading | API Calls |
|---|---|---|---|
| Runtime inference | 5–15 seconds | Every page load | Every page load |
| **Static JSON (karAQI)** | **< 1 second** | **None** | **None (cached 5 min)** |

The tradeoff: predictions are at most 1 hour old (refreshed hourly by CI). For a weather/AQI dashboard, this is acceptable — conditions don't change faster than that.

---

## Papers Reviewed

The model selection was informed by the following research papers:

1. **Comparative Analysis of Forecasting Models for Air Quality Index Prediction** (2024) — Compares LSTM, ARIMA/SARIMA, Facebook Prophet, curve fitting for AQI prediction
2. **Machine Learning Models for Daily AQI Prediction: An In-depth Analysis** (2024) — Evaluates Extra Trees, Random Forest, LightGBM, KNN for daily AQI
3. **Forecasting the Effect of Parameters on AQI Values with ML: Multiple Linear Regression** (2024) — Studies MLR, LSTM, CNN for AQI with metaheuristic optimization
4. **A Temporal Deep Learning Framework for AQI Prediction using Time-Series Data** (2024) — LSTM-based framework for AQI time-series prediction
5. **Mapping Socioeconomic Air Quality Disparities In Rwanda Using Sentinel-5P TROPOMI Data** (2024) — Satellite-derived AQI mapping for regions without ground monitors
6. **Real-Time AQI Estimation: A Smart and Lightweight AI-Based System Using Gas Sensors** (2024) — DL and ML models for real-time AQI from sensor data

---

## Live Dashboard

**[kaqindex.streamlit.app](https://karaqi.streamlit.app/)**

---

## Architecture

The system uses a **static JSON serving architecture** — predictions are pre-computed by CI pipelines and stored in a separate data repository **[karAQI-data](https://github.com/faisaliqbalkhattak/karAQI-data)**. The dashboard fetches these JSON files via GitHub URLs, giving every visitor a near-instant page load with zero runtime inference.

```
karAQI (code repo)                      karAQI-data (data repo)
───────────────────                     ────────────────────────
training_pipeline (daily)      ──────►   models/*.joblib
  trains Ridge + XGBoost                  models/*_models.json
  registers in MLflow (model registry)    data/model_eval.json
  pushes model files + eval      ──────►

forecast_pipeline (hourly)      ◄──────  models/*.joblib (downloads for inference)
  loads champion model from karAQI-data   data/static_forecast.json (pushes result)
  reads features from DuckDB
  (feature store)

feature_pipeline (hourly)               data/ (raw CSVs, feature store)
  populates DuckDB at data/feature_store/

                                       ◄── Dashboard reads via raw URLs
```

### CI/CD Pipeline Schedule

All pipelines run on GitHub Actions. See [docs/cicd-pipelines.md](docs/cicd-pipelines.md) for the full architecture and troubleshooting history.

| Workflow | Schedule (UTC) | What it does | Output |
|---|---|---|---|
| `feature_pipeline.yml` | Hourly at `:01` | Incremental data fetch (~1 row), build features, backfill DuckDB, run tests | Feature store (DuckDB) |
| `forecast_pipeline.yml` | Hourly at `:04` | Fetch ~1 new row, backfill DuckDB, load champion model from karAQI-data, run inference, fetch Open-Meteo AQ forecast (with retry), export JSON | `static_forecast.json` → karAQI-data |
| `training_pipeline.yml` | Daily at `00:00` | Incremental fetch, train Ridge + XGBoost, rolling-origin evaluation, champion comparison, register in MLflow (model registry), export eval JSON | `model_eval.json` + model files → karAQI-data |

> **Incremental fetching:** The feature pipeline runs hourly and fetches only the new data since the last pull (typically ~1 row). Open-Meteo reanalysis data is immutable — historical values never change — so incremental fetching is safe and avoids re-downloading 4 years of data on every run. See [docs/data-sources.md](docs/data-sources.md).

> **Champion comparison:** The training pipeline compares new model metrics against the current champion in MLflow. Only promotes if the new model is better on average RMSE across all output groups. If worse, the old champion is kept and the comparison is logged.

> **GitHub Actions timing caveat:** Cron triggers are best-effort, not precise. Workflows are frequently delayed 5–30 minutes (sometimes longer) due to platform load — see [github/community#156282](https://github.com/orgs/community/discussions/156282). If the dashboard shows a previous hour's AQI, the CI pipeline was delayed. The data is still correct — it reflects the most recent successful run. See [docs/cicd-pipelines.md](docs/cicd-pipelines.md).

---

## Model Performance (Live Dashboard Metrics)

The training pipeline trains **2 models** for the hourly 30-output forecast: Ridge and XGBoost. The champion is selected by lowest average RMSE across all three output groups (hourly points, six-hour means, twelve-hour means).

30 outputs per forecast origin: 24 hourly points (`t+1h` through `t+24h`), four six-hour block means (`t+25h` through `t+48h`), and two twelve-hour block means (`t+49h` through `t+72h`).

The forecast payload also includes an **Open-Meteo AQ reference forecast** (72h of hourly US AQI from `air-quality-api.open-meteo.com`) for transparent comparison on the dashboard. The fetch uses 3 retries with escalating timeouts (15s/30s/45s) because the Open-Meteo air-quality endpoint is occasionally slow under GitHub Actions shared infrastructure.

### Hourly Models — Rolling-Origin Evaluation

Protocol: 3 expanding folds, 168-hour test windows, 72-hour embargo.

| Output Group | RMSE | MAE | R² | Category Accuracy | High-AQI Recall |
|---|---:|---:|---:|---:|---:|
| **Hourly points (1–24h)** | **11.16** | **8.59** | **0.326** | **90.0%** | **69.3%** |
| **Six-hour means (25–48h)** | **18.52** | **13.95** | -0.026 | **77.7%** | **55.3%** |
| **Twelve-hour means (49–72h)** | **19.11** | **15.33** | -0.110 | **66.8%** | **48.4%** |
| Persistence baseline (1–24h) | 12.57 | 9.73 | 0.142 | 87.2% | 62.8% |
| Persistence baseline (25–48h) | 22.95 | 17.91 | -0.556 | 69.7% | 56.0% |
| Persistence baseline (49–72h) | 27.50 | 22.95 | -1.269 | 56.3% | 51.6% |

**The selected model beats both persistence and seasonal persistence on RMSE and MAE for all three output groups.** This is the release gate. The champion comparison ensures only models that improve on the current best are promoted — see [docs/cicd-pipelines.md](docs/cicd-pipelines.md).

### Daily Model Comparison (chronological holdout, +1/+2/+3 days)

| Model | +1 Day RMSE | +1 Day R² | +2 Day RMSE | +2 Day R² | +3 Day RMSE | +3 Day R² |
|---|---:|---:|---:|---:|---:|---:|
| XGBoost | 15.78 | 0.513 | 19.40 | 0.263 | 20.72 | 0.168 |
| Random Forest | 16.28 | 0.482 | 19.48 | 0.257 | 20.71 | 0.169 |
| Ridge | 16.66 | 0.457 | 19.98 | 0.219 | 20.66 | 0.173 |
| SARIMA | 26.40 | -0.363 | 23.79 | -0.108 | 22.74 | -0.002 |
| LSTM | 24.86 | -0.208 | 26.99 | -0.426 | 25.08 | -0.219 |
| Persistence | 18.55 | 0.326 | 22.95 | -0.018 | 25.38 | -0.259 |

**Best daily model by horizon:** XGBoost for +1/+2 days, Ridge for +3 days.

### Interpretation Guide

- **RMSE** — Root mean squared error in AQI units. Lower is better. An RMSE of 11 means the typical prediction error is ±11 AQI points.
- **R²** — Proportion of variance explained relative to the mean-prediction baseline. R² = 0 means "no better than guessing the average." Negative R² means the model is worse than the average.
- **Category accuracy** — Percentage of predictions in the correct EPA AQI band (Good/Moderate/Unhealthy for Sensitive Groups/Unhealthy/Very Unhealthy/Hazardous).
- **High-AQI recall** — Percentage of truly polluted hours the model correctly flags. Important for health alerts.

---

## Dashboard Tabs

### "My model" tab
- **Primary AQI:** Current hour value computed from observed Open-Meteo data using US EPA breakpoint formula
- **Secondary AQI:** Open-Meteo live current-hour AQI
- **Tertiary:** The model's next-hour prediction
- **Chart:** 30-output forecast (24 hourly points + block means) compared against Open-Meteo 72h AQ forecast

### "Open-Meteo" tab
- **Primary AQI:** Live AQI from Open-Meteo's forecast API
- **Secondary AQI:** The current-hour computed AQI

### Other sections
- Hourly forecast timeline (next 24h, hour by hour)
- Extended forecast (6h and 12h block means for 24–72h)
- Model comparison with Open-Meteo reference
- SHAP explanations (LinearExplainer for feature importance)
- Model evaluation metrics (MLflow registry, rolling-origin results)
- Weather insights (26-year trends and AQI seasonality)

---

## Data Sources

| Source | Purpose | API Key | Rate Limit |
|---|---|---|---|
| Open-Meteo Air Quality API | Primary pollutant data + AQI forecast reference | None | Fair use |
| Open-Meteo Weather Archive API | Weather features (temperature, humidity, wind, etc.) | None | Fair use |
| Open-Meteo Forecast API | Live weather verification | None | Fair use |

All timestamps normalized to `Asia/Karachi` (UTC+5). Historical data spans 2000–present for weather trends, 2022-08-05–present for AQI training.

**Resilience:** The Open-Meteo air-quality forecast endpoint (`air-quality-api.open-meteo.com`) is occasionally slow under GitHub Actions shared infrastructure. The forecast pipeline retries up to 3 times with escalating timeouts (15s → 30s → 45s) and exponential backoff (2s, 4s) before giving up. If all attempts fail, the forecast is exported without the Open-Meteo comparison line (the dashboard shows `ref_forecast: []`). See [docs/data-sources.md](docs/data-sources.md) for the full contract.

---

## AQI Target Definition

`src/aqi.py` calculates US EPA AQI using the official breakpoint method, including the May 2024 PM₂.₅ breakpoint update. The target is:

- **Daily:** `aqi_us_epa` — complete calendar-day 24-hour PM averages, maximum valid 8-hour ozone/CO windows, maximum valid 1-hour SO₂/NO₂ windows, EPA unit conversions/truncation, breakpoint interpolation, and category assignment.
- **Hourly:** `aqi_hourly_rolling` — a rolling-hour estimate from EPA pollutant sub-indices; not the official once-per-day AQI report.

Both are derived from Open-Meteo modeled concentrations, not ground-station measurements.

---

## Environment Setup

Python 3.11 is the only supported version. Run all commands from the project root.

```bash
python -m venv .venv
# Windows:
.venv\Scripts\python -m pip install --upgrade pip
.venv\Scripts\python -m pip install -r requirements.txt
# Linux/macOS:
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
```

### Requirements Files

| File | Used by | Packages |
|---|---|---|
| `requirements.txt` | Full dev environment | All |
| `requirements-feature.txt` | Feature pipeline CI | 9 packages |
| `requirements-forecast.txt` | Forecast pipeline CI | 8 packages |
| `requirements-training.txt` | Training pipeline CI | 16 packages |
| `app/requirements.txt` | Streamlit Cloud | 11 packages (lean) |

---

## Workflow

```bash
# 1. Fetch data
python -m src.build_features --fetch

# 2. Backfill feature store
python -m src.feature_store backfill-hourly --replace
python -m src.feature_store backfill-daily --replace

# 3. Train modelspython -m src.train --store # daily models
python -m src.train_hourly --store # hourly models (Ridge, XGBoost)
# 4. Register champion (with comparison against current best)
python -m src.model_registry register-hourly

# 5. Generate forecast JSON
python -m src.export_forecast --source live

# 6. Generate evaluation JSON
python -m src.export_eval

# 7. Run tests
pytest -q

# 8. Run dashboard locally
streamlit run app/dashboard.py
```

---

## Feature Store and Model Registry

| Component | Implementation | Where It Lives |
|---|---|---|
| Feature store | DuckDB (embedded, file-based) | Created at runtime at `data/feature_store/karak_feature_store.duckdb` (not in git) |
| Model registry | MLflow file-backed | Created at runtime at `models/mlruns/` (not in git) |
| Durable model storage | karAQI-data GitHub repo | `models/*.joblib` + `models/*_models.json` |

### What a Model Registry Is

A model registry manages the full ML model lifecycle: version control, staging (dev → staging → production), metadata tracking (metrics, parameters, data lineage), and access control. Industry examples include MLflow Model Registry (server-backed), Vertex AI Model Registry, and Hopsworks.

### What We Use

Our MLflow instance is **file-backed** — it stores model versions and metadata in local directories, not on a persistent server. The trained model binary (`.joblib`) is pushed to the `karAQI-data` GitHub repo so the forecast pipeline can download it. This functions as a lightweight model registry: it tracks which model version is the champion, logs metrics, and supports version-based loading.

The feature store (DuckDB) is created and populated by the feature pipeline at runtime. It is not committed to git — each CI runner creates its own instance. The training and forecast pipelines read from it; the dashboard does not.

Registered models: `aqi-hourly` (champion, hourly 30-output), `aqi-daily-h1`, `aqi-daily-h2`, `aqi-daily-h3`.

---

## Project Structure

```
karAQI/
├── app/                          # Dashboard + API
│   ├── dashboard.py              # Streamlit dashboard (main app)
│   ├── api.py                    # FastAPI REST endpoints
│   ├── live_data.py              # Live data fetching from Open-Meteo
│   ├── explain.py                # SHAP explanations
│   └── requirements.txt          # Lean dashboard-only deps
├── src/                          # Core ML pipeline
│   ├── config.py                 # Central configuration
│   ├── aqi.py                    # US EPA AQI calculation
│   ├── ingest.py                 # Data ingestion from Open-Meteo
│   ├── build_features.py         # Feature engineering
│   ├── feature_store.py          # DuckDB feature store (runtime)
│   ├── train.py                  # Daily model training
│   ├── train_hourly.py           # Hourly model training
│   ├── inference_hourly.py       # Hourly inference (30-output contract)
│   ├── model_registry.py         # MLflow model registry (runtime)
│   ├── export_forecast.py        # Export forecast JSON to karAQI-data
│   └── export_eval.py            # Export evaluation JSON
├── notebooks/                    # EDA and analysis notebooks
│   ├── 01_raw_data_check.ipynb   # Data quality checks
│   ├── 02_feature_eda.ipynb      # Feature EDA
│   ├── 03_live_open_meteo_check.ipynb  # Live API verification
│   └── 04_karak_weather_trends.ipynb   # 26-year weather trends
├── tests/                        # Automated test suite (38+ tests)
├── docs/                         # Detailed documentation
│   ├── evolution.md              # Full project journey narrative
│   ├── modeling-evaluation.md    # Model selection, metrics, evaluation protocol
│   ├── data-sources.md           # API contracts, data quality rules
│   ├── mlops-architecture.md     # Feature store, registry, CI/CD decisions
│   ├── cicd-pipelines.md         # CI pipeline architecture + troubleshooting
│   └── modeling-readiness.md     # Pre-deployment checklist
├── .github/workflows/            # CI/CD pipelines
│   ├── feature_pipeline.yml      # Hourly data fetch + feature build
│   ├── forecast_pipeline.yml     # Hourly inference + JSON export
│   └── training_pipeline.yml     # Daily model training + registration
├── requirements.txt              # Full dev dependencies
├── requirements-feature.txt      # Feature pipeline CI deps
├── requirements-forecast.txt     # Forecast pipeline CI deps
├── requirements-training.txt     # Training pipeline CI deps
└── README.md                     # This file
```

**Runtime directories** (created by pipelines, not in git):
- `data/` — Raw CSVs, processed features, DuckDB feature store
- `models/` — Trained model artifacts, MLflow registry

---

## Deployment

1. Push to GitHub (`faisaliqbalkhattak/karAQI`)
2. [Streamlit Cloud](https://share.streamlit.io) → New app → `karAQI` → `app/dashboard.py`
3. Ensure `karAQI-data` repo exists with the same owner
4. Set `DATA_REPO_TOKEN` secret in karAQI repo settings (fine-grained PAT with Contents: Read/Write on karAQI-data only)

---

## Documentation

| Document | What it covers |
|---|---|
| [docs/evolution.md](docs/evolution.md) | Full project journey — from research papers to deployment |
| [docs/modeling-evaluation.md](docs/modeling-evaluation.md) | Model selection methodology, metrics, rolling-origin protocol |
| [docs/data-sources.md](docs/data-sources.md) | API contracts, data quality rules, why Open-Meteo was chosen |
| [docs/mlops-architecture.md](docs/mlops-architecture.md) | Feature store (DuckDB), model registry (MLflow), CI/CD design decisions |
| [docs/cicd-pipelines.md](docs/cicd-pipelines.md) | GitHub Actions pipeline architecture and troubleshooting |
| [docs/modeling-readiness.md](docs/modeling-readiness.md) | Pre-deployment checklist and validation record |
