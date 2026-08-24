# Data Sources

API contracts, data quality rules, and the reasoning behind choosing Open-Meteo as the sole data provider.

---

## Why Open-Meteo

Open-Meteo was chosen over AQICN, OpenWeather, and IQAir for these reasons:

| Criteria | Open-Meteo | AQICN / IQAir |
|---|---|---|
| API key | Not required | Required |
| Rate limits | Fair use (generous) | Strict (429 errors observed) |
| Historical archive | 2000–present | Limited history |
| AQI calculation | US EPA included | Varies by source |
| Forecast endpoint | 7-day AQI forecast | Limited forecast |
| Cost | Free | Free tier limited |

IQAir was originally used but removed after persistent 429 rate-limiting during CI runs. Open-Meteo provides the same data without authentication.

---

## API Endpoints

### 1. Air Quality (Historical)

```
https://air-quality-api.open-meteo.com/v1/air-quality
```

**Parameters:**
- `latitude`: 33.1383653
- `longitude`: 71.1909136
- `start_date` / `end_date`: YYYY-MM-DD
- `hourly`: pm10, pm2_5, carbon_monoxide, nitrogen_dioxide, sulphur_dioxide, ozone, aerosol_optical_depth, dust, uv_index
- `timezone`: Asia/Karachi

**Rate limit:** Fair use. In practice, 3 retries with 60s timeout handle transient failures.

**Used by:** `src/ingest.py` → `fetch_open_meteo_air_quality()`

### 2. Weather Archive (Historical)

```
https://archive-api.open-meteo.com/v1/archive
```

**Parameters:**
- Same coordinates and date range
- `hourly`: temperature_2m, relative_humidity_2m, dew_point_2m, precipitation, rain, surface_pressure, cloud_cover, wind_speed_10m, wind_direction_10m, wind_gusts_10m

**Used by:** `src/ingest.py` → `fetch_open_meteo_weather()`

### 3. Weather Forecast (Live)

```
https://api.open-meteo.com/v1/forecast
```

**Used by:** `app/live_data.py` for live weather verification on the dashboard.

### 4. Air Quality Forecast (Live — Comparison)

```
https://air-quality-api.open-meteo.com/v1/air-quality?forecast=true
```

**Used by:** `src/export_forecast.py` — fetches 72h of hourly US AQI for transparent comparison on the dashboard.

**Resilience:** 3 retries with escalating timeouts (15s → 30s → 45s) and exponential backoff (2s, 4s). If all attempts fail, the forecast is exported without the comparison line.

---

## Data Ranges

| Dataset | Start Date | End Date | Frequency | Rows |
|---|---|---|---|---|
| Weather trends | 2000-01-01 | Present | Daily | ~9,700 |
| AQI training | 2022-08-05 | Present | Hourly | ~35,500 |
| Weather features | 2022-08-05 | Present | Hourly | ~35,500 |

---

## Data Quality Rules

1. **No missing values in target:** Rows where `aqi_us_epa` is NaN are dropped during feature engineering
2. **No future data leakage:** 72-hour embargo between train and test sets in rolling-origin evaluation
3. **Immutable reanalysis:** Open-Meteo reanalysis data never changes — historical values are final, so incremental fetching is safe
4. **Timezone normalization:** All timestamps normalized to `Asia/Karachi` (UTC+5)

---

## Incremental Fetching

The feature pipeline runs hourly and fetches only new data since the last pull (typically ~1 row). This avoids re-downloading 4 years of data on every run.

```
Last stored row: 2026-08-24T13:00
Fetch range: 2022-08-05 to 2026-08-24 (full range sent to API)
Result: Only new rows are appended to the feature store
```

The full date range is always sent to the Open-Meteo API (it doesn't support "since last pull"), but the feature engineering step deduplicates and only stores new rows.
