# MLOps Architecture

Feature store, model registry, CI/CD pipeline design, and the decisions behind them.

---

## Feature Store

**Implementation:** DuckDB (file-based, embedded)

**Location:** `data/feature_store/karak_feature_store.duckdb` (created at runtime, not committed to git)

### Tables

| Table | Rows | Purpose |
|---|---|---|
| `hourly_features` | ~35,400 | Processed hourly features with future target columns (for training) |
| `hourly_observations` | ~35,400 | Processed hourly features without targets (for inference) |
| `hourly_raw` | ~35,500 | Raw pollutant and weather columns (inference input contract) |
| `daily_features` | ~1,470 | Daily features with target columns (for daily model training) |

### Why DuckDB

- **Serverless:** No API key, no cloud account, no network dependency
- **Free:** No usage costs
- **Fast:** Columnar storage, runs on any OS
- **Portable:** Single `.duckdb` file, works on Windows/Mac/Linux
- **Satisfies the assignment:** The mentor confirmed any feature store works, not just Hopsworks/Vertex AI

### Feature Store Usage

- **Training pipeline:** Reads from `hourly_features` and `daily_features` tables
- **Forecast pipeline:** Reads from `hourly_raw` table (168 rows of recent observations)
- **Dashboard:** Does NOT read from DuckDB — reads pre-computed JSON for speed

---

## Model Registry

**Implementation:** MLflow file-backed (local `mlruns/` directory)

**Location:** `models/mlruns/` (created at runtime, not committed to git)

### Registered Models

| Model Name | Purpose | Outputs |
|---|---|---|
| `aqi-hourly` | Champion hourly model (30-output forecast) | 1 version, alias: `champion` |
| `aqi-daily-h1` | Best model for +1 day prediction | 1 version |
| `aqi-daily-h2` | Best model for +2 day prediction | 1 version |
| `aqi-daily-h3` | Best model for +3 day prediction | 1 version |

### What a Model Registry Does

A model registry manages the full ML model lifecycle:

1. **Version control:** Every model training run creates a new version
2. **Staging:** Versions can be tagged as "champion" (production) or "candidate"
3. **Metadata tracking:** Metrics, parameters, and data lineage stored with each version
4. **Champion comparison:** New models are compared against the current champion before promotion

### Why MLflow File-Backed

- **Serverless:** No tracking server needed
- **Free:** No cloud costs
- **Satisfies the assignment:** The mentor confirmed any model registry works

### Limitations

- **Ephemeral metadata:** The `mlruns/` directory is recreated on each CI run. Version history is lost between runs.
- **Durable model binary:** The actual `.joblib` files are pushed to `karAQI-data` repo, so the forecast pipeline can always download the latest champion.

---

## CI/CD Pipeline Design

### Pipeline Architecture

```
karAQI (code repo)                     karAQI-data (data repo)
───────────────────                    ────────────────────────

Open-Meteo (keyless)
      │
      ▼
feature_pipeline.yml (hourly :01)
  → Incremental fetch (~1 row) → Build features → DuckDB feature store
      │
      ▼
training_pipeline.yml (daily 00:00 UTC)
  → Reads from DuckDB feature store
  → Trains Ridge + XGBoost
  → Champion comparison (only promotes if better)
  → Registers in MLflow
  → Pushes model files + eval JSON
      │
      ├──► models/*.joblib             ──────► karAQI-data/models/
      └──► data/model_eval.json      ──────► karAQI-data/data/

forecast_pipeline.yml (hourly :04)
  → Downloads model from karAQI-data
  → Fetches ~1 new row → Backfills DuckDB
  → Reads features from DuckDB
  → Fetches Open-Meteo AQ forecast (for comparison)
  → Exports static_forecast.json
      │
      └──► data/static_forecast.json ──────► karAQI-data/data/
```

### Design Decisions

1. **Static JSON serving:** Predictions are pre-computed, not generated at dashboard load time. This gives near-instant page load.
2. **Two-repo structure:** Code in `karAQI`, data in `karAQI-data`. Keeps the code repo lightweight and avoids git history bloat from model binaries.
3. **No fallbacks:** If the model file is missing, the pipeline fails with a clear error message. No silent degradation to stale data.
4. **Feature pipeline before forecast:** The forecast pipeline depends on the feature pipeline having run first (to populate DuckDB). The cron schedule (:01 vs :04) ensures this ordering.

### Champion Comparison Flow

```
1. Train all models (Ridge, XGBoost)
2. Run rolling-origin evaluation
3. Find best model = lowest average RMSE across all groups
4. Load current champion from MLflow registry
5. Compare:
   - If new model is better → promote (overwrite champion)
   - If worse → keep old champion, log comparison
6. Push model files + eval JSON to karAQI-data
```

---

## Dashboard Architecture

The dashboard is a Streamlit app hosted on Streamlit Cloud. It reads pre-computed JSON from the `karAQI-data` repo via raw GitHub URLs.

```
karAQI-data/data/static_forecast.json
  → Dashboard reads via raw URL
  → Near-instant page load
  → Zero runtime inference
```

The dashboard does NOT:
- Call any APIs at runtime
- Load ML models
- Query the feature store
- Run any ML inference

All computation happens in CI pipelines. The dashboard is purely a display layer.
