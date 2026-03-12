# Air Quality Index (AQI) Analysis — India

A data analysis project examining air quality patterns across 
Indian cities using pollutant concentration data from 2015–2020.

---

## Dataset

Five relational tables sourced from CPCB (Central Pollution Control Board):

| File | Granularity | Records |
|---|---|---|
| `city_day.csv` | City × Day | ~26K |
| `city_hour.csv` | City × Hour | ~100K |
| `station_day.csv` | Station × Day | ~108K |
| `station_hour.csv` | Station × Hour | ~2.3M |
| `stations.csv` | Station metadata | ~400 |

**Pollutants tracked:** PM2.5, PM10, NO, NO2, NOx, NH3, CO, 
SO2, O3, Benzene, Toluene, Xylene

---

## Data Cleaning

- Dropped `Xylene` — missing rate of 61–79% across all tables
- Dropped rows where almost all values were NaN
- Imputed pollutant nulls using **city/station-wise median** 
  to preserve local pollution baselines and resist skew from 
  extreme events
- Applied global median as fallback where group-wise median 
  was unavailable (e.g. cities where entire column was missing)
- Removed rows with null `AQI` — target variable cannot be imputed
- Parsed `Date`/`Datetime` to `datetime64`

---

## EDA Conclusions

### `city_day.csv`
- AQI distribution is right-skewed — extreme pollution events 
  in north Indian cities pull the mean above median
- Delhi and Lucknow have the most inactive monitoring stations
- Kochi and Ernakulam are the only cities with unknown (NaN) 
  station status
- PM2.5 shows strongest correlation with AQI (0.84) among 
  all pollutants
- O3 shows near-zero correlation with AQI (-0.14)
- `AQI_Bucket` is a deterministic function of `AQI` — 
  not an independent feature

### `stations.csv`
- Monitoring infrastructure is concentrated in larger 
  metropolitan areas — smaller cities are under-monitored
- 98.3% of station readings are from Active stations — 
  network is healthy
- Only Delhi and Lucknow have Inactive stations (1.3% total)
- Inactive and unknown status stations concentrated in 
  just 4 cities — Delhi, Lucknow, Kochi, Ernakulam

### `station_hour.csv` (merged with stations.csv)
- Merged on StationId to add City and State information
- 2.3M rows after merge
- Diurnal pollution cycle confirmed at station level

---

## Feature Engineering

| Feature | Derived From | Rationale |
|---|---|---|
| `Month` | Date/Datetime | Captures seasonal variation |
| `Year` | Date/Datetime | Captures year-over-year trends |
| `Season` | Month | Winter/Summer/Monsoon grouping |
| `Hour` | Datetime (hourly tables) | Encodes diurnal pollution cycle |
| `City_encoded` | City | Target encoding — mean AQI per city |
| `State_encoded` | State | Target encoding — mean AQI per state |
| `Date` | Datetime | Extracted date component |
| `Time` | Datetime | Extracted time component |

**Season Mapping:**
```
Winter  → Dec, Jan, Feb
Summer  → Mar, Apr, May
Monsoon → Jun, Jul, Aug, Sep
```

---

## Feature Selection & Encoding

### Encoding
| Column | Strategy | Reason |
|---|---|---|
| `Season` | One Hot Encoding | No natural order between seasons |
| `City` | Target Encoding | High cardinality — 26 cities |
| `State` | Target Encoding | High cardinality — multiple states |
| `AQI_Bucket` | Dropped | Data leakage — direct derivative of target |

### Feature Selection Strategy

Three methods applied:

**1. Filter — Correlation Analysis**
- Dropped `O3` → correlation of -0.14 with AQI (near zero)
- Dropped `NOx` → highly correlated with NO (0.78), redundant
- Dropped `PM10` → highly correlated with PM2.5 (0.90), redundant

**2. Filter — Variance Threshold**
- Applied with threshold = 0.0
- No columns dropped — all features had meaningful variance
- Confirms dataset is rich and varied

**3. Embedded — Random Forest Feature Importance**
- Dropped `Season_*` columns → near zero importance
- Dropped `NH3`, `NO`, `CO` → near zero importance
- PM2.5 emerged as dominant predictor (0.92 importance score)

### Final Selected Features
| Feature | Importance |
|---------|-----------|
| PM2.5 | 0.92 — primary AQI driver |
| NO2 | Moderate |
| SO2 | Moderate |
| Toluene | Moderate |
| Benzene | Moderate |

### Key Finding
> PM2.5 alone accounts for 92% of AQI prediction importance. 
> Location (City/State) encoding dominates when included — 
> indicating where you are matters as much as what you breathe.

---
