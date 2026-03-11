# Air Quality Index (AQI) Analysis — India

A data analysis project examining air quality patterns across Indian cities using pollutant concentration data from 2015–2020.

---

## Dataset

Five relational tables sourced from CPCB (Central Pollution Control Board):

| File | Granularity | Records |
|---|---|---|
| `city_day.csv` | City × Day | ~26K |
| `city_hour.csv` | City × Hour | ~100K |
| `station_day.csv` | Station × Day | ~108K |
| `station_hour.csv` | Station × Hour | ~400K |
| `stations.csv` | Station metadata | ~400 |

**Pollutants tracked:** PM2.5, PM10, NO, NO2, NOx, NH3, CO, SO2, O3, Benzene, Toluene, Xylene

---

## Data Cleaning

- Dropped `Xylene` — missing rate of 61–79% across all tables
- Imputed pollutant nulls using **city/station-wise median** to preserve local pollution baselines and resist skew from extreme events
- Applied global median as fallback where group-wise median was unavailable
- Removed rows with null `AQI` — target variable cannot be imputed
- Parsed `Date`/`Datetime` to `datetime64`; extracted `Hour`, `Month`, `Year`, `Season`

---

## EDA Conclusions

### `city_day.csv`
- AQI distribution is right-skewed — extreme pollution events in north Indian cities pull the mean above median
- Top 5 most polluted cities: **Ahmedabad, Delhi, Patna, Gurugram, Lucknow**
- PM2.5 and PM10 exhibit strongest linear correlation with AQI among all pollutants
- Winter months (Dec–Feb) record significantly higher AQI due to atmospheric boundary layer compression and crop burning
- `AQI_Bucket` is a deterministic function of `AQI` — not an independent feature

### `city_hour.csv`
- AQI follows a consistent V-shaped diurnal pattern — peaking at 00:00–05:00 hrs, minima at 13:00–16:00 hrs
- Nocturnal AQI is approximately 2.5× higher than afternoon values across all cities
- Ahmedabad anomaly: nocturnal AQI ≈ 640 vs inter-city mean of ~240 — indicative of localized industrial sources
- Secondary AQI elevation observed at 13:00 hrs — consistent with midday vehicular and industrial activity
- Hour-of-day is a statistically significant predictor of AQI regardless of city

### `station_day.csv`
- Station-level data shows higher variance than city-level — city aggregation smooths out extreme readings
- NH3 and PM10 have highest missing value rates (~40–44%) across all stations
- Pollutant correlation patterns are consistent with city_day findings
- Some stations record consistently anomalous values — potential sensor calibration issues

### `station_hour.csv`
- Diurnal cycle confirmed at station level — consistent with city_hour findings
- Hourly variance is higher at individual stations than city-aggregated data
- PM2.5 spikes are more pronounced at night at station level than reflected in city averages

### `stations.csv`
- Monitoring infrastructure is concentrated in larger metropolitan areas — smaller cities are under-monitored
- Significant portion of stations have unknown operational status (`NaN`) — data collection gaps exist
- States with higher industrialization have more active monitoring stations

---

## Merging Strategy

City-level and station-level tables are kept separate due to different granularities.

Logical merges performed:
```
stations + station_day  → merged on StationId → adds City, State info
stations + station_hour → merged on StationId → adds City, State info
```

City tables (`city_day`, `city_hour`) are used independently.

---

## Feature Engineering

Applied to all tables:

| Feature | Derived From | Rationale |
|---|---|---|
| `Month` | Date/Datetime | Captures seasonal pollution variation |
| `Year` | Date/Datetime | Captures year-over-year trends |
| `Season` | Month | Winter/Summer/Monsoon/Post-Monsoon grouping |
| `Is_Winter` | Month | Binary flag for peak pollution season |
| `Hour` | Datetime (hourly tables only) | Encodes diurnal pollution cycle |
| `Time_of_Day` | Hour (hourly tables only) | Morning/Afternoon/Evening/Night bucket |

**Season Mapping:**
```
Winter       → Dec, Jan, Feb
Summer       → Mar, Apr, May
Monsoon      → Jun, Jul, Aug, Sep
Post-Monsoon → Oct, Nov
```

---

## Feature Selection & Encoding

### Encoding
| Column | Strategy | Reason |
|---|---|---|
| `Season` | Label Encoding | Ordinal — natural pollution order across seasons |
| `City` / `State` | Target Encoding | Too many categories for One-Hot |
| `AQI_Bucket` | Drop | Data leakage — direct derivative of target |

### Scaling
- All pollutants + engineered features → **StandardScaler**
- Chosen over MinMaxScaler due to presence of outliers from real pollution spike events

### Feature Selection Strategy
- Drop features with correlation |r| < 0.1 against AQI
- Apply VIF analysis — drop features with VIF > 10 to handle multicollinearity
- Drop one of `NO` / `NO2` / `NOx` — highly intercorrelated (same chemical family)
- Retain: PM2.5, PM10, NH3, CO, SO2, O3, Benzene, Toluene + engineered time features
- Target: `AQI`

---
