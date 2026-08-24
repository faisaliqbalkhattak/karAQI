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

## Cross-Source Comparison (Why Open-Meteo Won)

Before committing to Open-Meteo as the sole provider, a cross-source validation was performed against OpenWeather and WAQI/AQICN:

### Timezone was the dominant source of disagreement

When the two sources were first compared, PM2.5 correlation was only ~0.4. This seemed suspiciously low for two sources drawing from similar atmospheric models.

**Root cause:** Open-Meteo returns timestamps in Asia/Karachi time, while OpenWeather returns UTC. Without normalizing both to the same timezone, the two datasets were shifted by 5 hours. This misaligned the diurnal AQI cycle — morning peaks in one source lined up with afternoon values in the other.

**After fixing:** Explicit timezone normalization to `Asia/Karachi` (UTC+5) at ingestion time raised correlation from ~0.4 to:
- Hourly PM2.5: r = 0.68
- Daily PM2.5: r = 0.77

### Live comparison at 6:00 PM PKT (2026-07-29)

| Variable | Open-Meteo (18:00 PKT) | OpenWeather (19:00 PKT) |
|---|---|---|
| PM2.5 | 27.9 µg/m³ | 50.61 µg/m³ |
| PM10 | 42.7 µg/m³ | 105.61 µg/m³ |
| Ozone | 135.0 µg/m³ | 113.32 µg/m³ |
| NO₂ | 3.6 µg/m³ | 2.71 µg/m³ |
| SO₂ | 4.4 µg/m³ | 2.23 µg/m³ |
| CO | 278.0 µg/m³ | 310.07 µg/m³ |

The two sources differ substantially on PM10 and PM2.5 — consistent with model-to-model disagreement over a rural/topographically complex area like Karak.

### Cross-source pollutant correlations (after timezone fix)

| Pollutant | Hourly r | Daily r |
|---|---|---|
| PM2.5 | 0.596 | 0.592 |
| PM10 | 0.523 | 0.522 |
| Ozone | 0.480 | 0.479 |

The remaining ~0.3–0.4 unexplained variance is expected model-to-model noise because Karak has no ground station and the APIs use different atmospheric models / grid cells.

### WAQI/AQICN — stale data

WAQI/Peshawar station reported AQI 25, but the timestamp was **2025-03-04** — more than 4 months old. WAQI cannot be relied on as a live validation source.

### IQAir — rate-limited

IQAir was originally used as the dashboard comparison source. It failed because IQAir aggressively rate-limits anonymous scraping. GitHub Actions runners (cloud IPs) were blocked after just a few requests, returning 429 errors.

**Decision:** Replaced IQAir with Open-Meteo's AQ forecast endpoint. It provides 96 hours of hourly US AQI for Karak, free and keyless.

### Why Open-Meteo ultimately won

1. **Free and keyless** — no API key needed, no rate-limiting issues
2. **Timezone-correct** — returns Asia/Karachi timestamps directly
3. **Full historical archive** — 2000–present for weather, 2022–present for AQI
4. **Built-in AQI calculation** — US EPA breakpoints included
5. **Forecast endpoint** — 7-day AQI forecast for transparent dashboard comparison
6. **Cross-source correlation is acceptable** — r = 0.68 (hourly) and 0.77 (daily) after timezone normalization, which is expected for model-to-model agreement over a rural area

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
