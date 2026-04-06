# bicing-ebike-depletion

**Predicting e-bike depletion at Barcelona university clusters using Bicing open data (2021–2024)**

A spatiotemporal analysis of Barcelona's Bicing bike-sharing system focused on e-bike availability around major university campuses. The project builds a 24-hour-ahead Random Forest model that predicts hourly e-bike availability and flags depletion events — enabling proactive restocking before students are stranded.

---

## Context

Barcelona's Bicing system charges €0.10 per e-bike ride regardless of duration, but only charges for mechanical bike rides longer than 30 minutes. Most student commutes are short, meaning e-bikes are the primary revenue-generating vehicle type. When e-bike availability hits zero — which happens at UB/UPC Nord on over 80% of term-time weekday afternoons — Bicing loses that revenue and students are left stranded.

This project asks: **can we predict which days will deplete, and when, 24 hours in advance?**

---

## Repository structure

```
bicing-ebike-depletion/
│
├── step_1_data_cleaning_merging.ipynb     # Load raw Bicing station data, merge with metadata
├── step_2_add_weather_data.ipynb          # Fetch hourly weather from Open-Meteo, merge with station data
├── step_3_uni_cluster_analysis.ipynb      # Define university clusters, exploratory analysis
├── final_ub_upc_model.ipynb               # Full model pipeline: features, regression, classifier, evaluation
│
├── Bicing_Final_Presentation.pptx         # Presentation slides
└── requirements.txt
```

---

## Pipeline overview

### Step 1 — Data cleaning & merging
Loads raw Bicing monthly station files and station metadata. Merges on `station_id`, drops 2020 (COVID anomaly), adds derived features (capacity, hour, day of week), and saves yearly compressed files (`bikes_20XX.csv.gz`).

**Inputs:** Raw Bicing CSVs from [Barcelona Open Data](https://opendata-ajuntament.barcelona.cat), station metadata files  
**Outputs:** `bikes_2021.csv.gz` through `bikes_2024.csv.gz`

### Step 2 — Weather enrichment
Fetches hourly historical weather for Barcelona (temperature, precipitation, wind speed, weather codes) from the [Open-Meteo Archive API](https://open-meteo.com) — no API key required. Merges with station data on exact timestamp.

**Inputs:** `bikes_20XX.csv.gz`  
**Outputs:** `bikes_20XX_with_weather.csv.gz`

### Step 3 — University cluster analysis
Defines 3 university clusters using hardcoded campus centroids and a 500m radius spatial join:

| Cluster | Centroid | Stations |
|---------|----------|----------|
| UB / UPC Nord | 41.3888°N, 2.1147°E | 6 |
| UB / UPC Sud | 41.3773°N, 2.1492°E | 14 |
| UPF Ciutadella | 41.3854°N, 2.1938°E | 10 |

Produces temporal, seasonal, and weather-based analysis across all 3 clusters. Identifies UB/UPC Nord as the only cluster with a meaningful e-bike depletion problem — driven by its high altitude (avg 85m) relative to student residential areas.

**Inputs:** `bikes_20XX_with_weather.csv.gz`  
**Outputs:** Figures, `cluster_stations.csv`

### Step 4 — Model (final_ub_upc_model.ipynb)
Full end-to-end model pipeline for UB/UPC Nord:

**Regression model**
- Target: hourly e-bike NAB (Normalised Available Bikes) for next day
- One row = one station × one hour (operating hours 06:00–22:00 only)
- Train: 2021–2023 term-time weekdays | Test: Jan–Mar 2024
- Algorithm: Random Forest Regressor (300 trees, max depth 8)
- Features: 24h lag NAB, 168h lag NAB, weather (today + yesterday), hour sin/cos, day of week sin/cos, month sin/cos, academic period flag, class shift departure flags
- Result: **12.8% MAE improvement** over naive baseline (NAB same hour yesterday)

**Depletion classifier** (derived from regressor — no separate model)
- Definition: predicted NAB < 5% for 1+ consecutive hours between 12:00–20:00
- Result: **Precision 0.95 · Recall 0.86 · F1 0.90**

**Revenue analysis**
- Estimated missed rides during zero-availability hours × €0.10 = lost revenue
- **€314 lost across 6 stations, Jan–Mar 2024** (~€942 projected annual)

---

## Key findings

- UB/UPC Nord depletes on >80% of term-time weekday afternoons — a structural problem driven by altitude and commute direction
- The top predictive feature is NAB at the same hour yesterday (importance: 0.77) — the drain pattern is highly repeatable
- Hour of day (sin-encoded) is the 2nd most important feature — the drain curve mirrors a sine wave almost exactly
- Weather has minimal effect on the drain pattern — students commuting uphill appear committed regardless of rain
- Summer (Jul–Sep) is a fundamentally different operational regime — the model should not be applied for restocking decisions then
- The model is most useful for variable depletors (stations 302, 429) where prediction adds real operational value; chronic depletors (422, 433) deplete almost every day regardless

---

## Data sources

| Source | Description | Access |
|--------|-------------|--------|
| [Barcelona Open Data — Bicing](https://opendata-ajuntament.barcelona.cat/data/en/dataset/us-etat-estacions-bicing) | Hourly station availability 2019–2024 | Free, public |
| [Open-Meteo Archive API](https://open-meteo.com/en/docs/historical-weather-api) | Hourly historical weather for Barcelona | Free, no API key |
| [Barcelona Open Data — Educational Institutions](https://opendata-ajuntament.barcelona.cat/data/en/organization/educacio) | Location of 1,650+ educational institutions | Free, public |

> **Note:** Raw Bicing data files are not included in this repository due to size (~250M rows across 4 years). Download them from Barcelona Open Data and run the pipeline from Step 1.

---

## Requirements

See `requirements.txt`. Install with:

```bash
pip install -r requirements.txt
```

Python 3.9+ recommended.

---

## Limitations

- Model trained on term-time weekdays only — not applicable in summer or on public holidays
- Monday lag features use Friday's data (weekends excluded); public holidays not explicitly handled
- Revenue estimate is a conservative lower bound — suppressed demand is not captured
- No student enrolment or Bicing subscription data available — would significantly improve demand modelling
- If restocking succeeds and depletion events disappear, the model loses its training signal and would need continuous retraining
- Scope limited to UB/UPC Nord — other clusters have lower e-bike allocation and different demand dynamics

---

## Results summary

| Metric | Value |
|--------|-------|
| Baseline MAE (NAB yesterday) | ~14.66% |
| Random Forest MAE | ~12.79% |
| Improvement over baseline | 12.8% |
| Depletion precision | 0.95 |
| Depletion recall | 0.86 |
| Depletion F1 | 0.90 |
| Estimated lost revenue (Jan–Mar 2024, 6 stations) | €314 |
| Projected annual loss (9 school months) | ~€942 |
