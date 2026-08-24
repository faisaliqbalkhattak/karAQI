# Modeling Readiness

Pre-deployment checklist and validation record for the karAQI pipeline.

---

## Checklist

### Data Pipeline

- [x] Open-Meteo API integration working (air quality + weather)
- [x] Incremental fetching working (only new rows appended)
- [x] Feature engineering producing correct columns
- [x] DuckDB feature store populated with hourly and daily tables
- [x] Data quality checks passing (no NaN targets, correct date ranges)

### Model Training

- [x] Daily models training correctly (Ridge, XGBoost, RF, SARIMA, LSTM)
- [x] Hourly models training correctly (Ridge, XGBoost)
- [x] Rolling-origin evaluation producing valid metrics
- [x] Release gate passing (best model beats persistence)
- [x] Champion comparison working (only promotes if better)

### Model Registry

- [x] MLflow registration working for hourly and daily models
- [x] Champion alias correctly assigned
- [x] Model files pushed to karAQI-data
- [x] Eval JSON pushed to karAQI-data

### Forecast Pipeline

- [x] Model downloaded from karAQI-data
- [x] DuckDB populated on forecast runner
- [x] Inference producing 30 outputs
- [x] Open-Meteo AQ reference fetched (with retry)
- [x] static_forecast.json pushed to karAQI-data

### Dashboard

- [x] Streamlit app loading forecast JSON via raw URL
- [x] 30-output bar chart displaying correctly
- [x] Open-Meteo comparison line showing
- [x] SHAP explanations working
- [x] Weather insights rendering (trend images from karAQI-data)

### Tests

- [x] 38+ tests passing (pytest)
- [x] Feature pipeline tests passing
- [x] Training pipeline tests passing
- [x] Export forecast tests passing (with proper skips for missing models)

---

## Validation Record

### Latest Training Run (2026-08-24)

**Hourly models:**
- Ridge: hourly_points RMSE 10.88, MAE 8.16
- XGBoost: hourly_points RMSE 8.89, MAE 6.42
- Champion: XGBoost (lowest average RMSE across all groups)
- Release gate: PASSED

**Daily models:**
- XGBoost: +1d RMSE 15.78, +2d RMSE 19.40, +3d RMSE 20.72
- Ridge: +1d RMSE 16.66, +2d RMSE 19.98, +3d RMSE 20.66
- Best by horizon: XGBoost for +1/+2d, Ridge for +3d

### Pipeline Health

| Pipeline | Last Status | Duration |
|---|---|---|
| Feature | ✅ Passing | ~1 min |
| Training | ✅ Passing | ~8 min |
| Forecast | ✅ Passing | ~45 sec |

### Known Issues

1. GitHub Actions cron triggers are delayed 5–30 minutes (platform limitation, not a bug)
2. Open-Meteo AQ forecast endpoint occasionally times out (handled with 3 retries)
3. MLflow metadata is ephemeral (recreated each CI run) — model binaries are durable in karAQI-data

---

## Deployment Steps

1. Push to GitHub (`faisaliqbalkhattak/karAQI`)
2. Streamlit Cloud → New app → `karAQI` → `app/dashboard.py`
3. Ensure `karAQI-data` repo exists with same owner
4. Set `DATA_REPO_TOKEN` secret in karAQI repo settings (fine-grained PAT with Contents: Read/Write on karAQI-data only)
