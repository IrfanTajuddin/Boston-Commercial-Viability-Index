# Boston Commercial Viability Index

This project builds a tract-level Commercial Viability Index (CVI) for Boston using transit access, demographic catchment, and existing commercial density. It also derives a Market Gap Score identifying where commercial demand is underserved relative to current supply.

The goal is not to produce a single definitive ranking, but to provide a location intelligence tool that supports commercial site selection and investment risk assessment.

---

## Project Purpose

Retail site selection and commercial lending decisions are often made without a systematic view of neighbourhood-level conditions. Developers and financial institutions typically rely on intuition, comparable transactions, or coarse demographic summaries.

This project asks:

- Which Boston neighbourhoods have the strongest fundamentals for commercial activity?
- Where does demand (transit access + demographics) outpace existing commercial supply?
- How do transit access, spending power, and market density interact spatially?
- Which tracts are underserved (strong demand, thin supply) and which are oversaturated?

---

## 1. Data

The analysis integrates:

- **Census tract boundaries** — 2020 TIGER/Line via `pygris`, clipped to Boston city boundary
- **ACS 5-year estimates (2022)** — median household income, total population, population aged 25-54
- **MBTA GTFS static feed** — bus, subway, and light rail stop locations (2024)
- **OpenStreetMap POIs** — retail, food & beverage, services, and office locations via `osmnx`

Final dataset: 205 census tracts x 15 variables

All data sources are publicly available and free. No API key required.

---

## 2. Project Workflow

1. Collect and save raw inputs (tract boundaries, ACS demographics, MBTA stops, OSM POIs)
2. Spatial join — assign each POI and transit stop to its containing tract (`geopandas.sjoin`)
3. Aggregate to tract level using SQL in **DuckDB** (POI counts by type, stop counts by mode)
4. Pre-process — log-transform POI and stop counts, median-impute 14 null income tracts
5. Score Pillar 1 — Transit Access (log stop count + rapid transit flag)
6. Score Pillar 2 — Demographic Catchment (income + population + age share)
7. Score Pillar 3 — POI Density (inverted-U curve, peak at 60th percentile)
8. Compute composite CVI (weighted sum, rescaled 0-100)
9. Derive Market Gap Score (demand minus supply, rescaled -100 to +100)
10. Produce choropleth maps per pillar, composite CVI, and gap score
11. Export interactive Folium map with hover tooltips

---

## 3. Data Layers

All input layers before scoring — census tract boundaries, commercial POIs (blue), bus stops (yellow), subway stops (red), and light rail stops (green).

<img src="outputs/check_all_layers.png" width="700"/>

POI density is visibly concentrated in the city centre and along commercial corridors. Transit stops follow the MBTA network, with subway and light rail clustered in the inner city and bus coverage extending into outer neighbourhoods.

---

## 4. Index Structure

The CVI is built from three pillars, each scored 0-1 and combined into a weighted composite.

| Pillar | Weight | Variables |
|---|---|---|
| Transit Access | 35% | MBTA stop count (log), rapid transit presence flag |
| Demographic Catchment | 35% | Median HH income, total population, share aged 25-54 |
| POI Density | 30% | Commercial POI count (log, inverted-U scored) |

Pillar 3 uses an inverted-U curve rather than a linear scale. Very low density signals no market has formed; very high density signals oversaturation and elevated competition risk. Moderate density scores highest.

---

## 5. Pillar Maps

### Pillar 1 — Transit Access
<img src="outputs/pillar1_transit.png" width="700"/>

Transit access concentrates along the Orange and Green Line corridors through the city centre, with the Blue Line visible through East Boston. 121 of 205 tracts have zero transit stops inside their boundary, hence the rapid transit flag (40% of pillar weight) prevents this from collapsing most of the city to zero.

### Pillar 2 — Demographic Catchment
<img src="outputs/pillar2_demographics.png" width="700"/>

More evenly distributed than transit. Pale tracts in the centre reflect low residential population in downtown and financial district areas. Stronger scores in outer residential neighbourhoods (Brighton, Dorchester, Jamaica Plain) where resident density and income are higher.

### Pillar 3 — POI Density
<img src="outputs/pillar3_poi_density.png" width="700"/>

The inverted-U scoring produces a mixed spatial pattern rather than a simple centre-periphery gradient. Tracts at both extremes of POI density score lower. Mid-range tracts with active but not saturated commercial presence score highest.

---

## 6. Composite CVI

<img src="outputs/cvi_composite.png" width="700"/>

Weighted sum of three pillars rescaled to 0-100. Highest scores cluster along transit corridors and inner-ring neighbourhoods. Lowest scores are in the waterfront, airport, and outer areas with limited transit and thin commercial activity.

**Top 5 tracts by CVI score:**

| Tract | Neighbourhood | CVI Score |
|---|---|---|
| 25025100800 | Dorchester | 100.0 |
| 25025040600 | Charlestown | 96.0 |
| 25025120301 | Jamaica Plain | 93.9 |
| 25025061203 | South Boston | 92.3 |
| 25025000701 | Brighton | 90.3 |

---

## 7. Market Gap Score

<img src="outputs/market_gap.png" width="700"/>

Demand score = average of Pillar 1 and Pillar 2 (what the tract's population and transit can support).
Supply score = POI density re-scored linearly (what already exists).
Gap = Demand minus Supply, rescaled to -100 to +100.

**Most underserved tracts (positive gap — demand exceeds supply):**

| Tract | Neighbourhood | Gap Score |
|---|---|---|
| 25025081102 | Mission Hill | +64.4 |
| 25025101002 | Mattapan | +63.7 |
| 25025092200 | Dorchester | +52.7 |
| 25025000503 | Brighton | +40.8 |
| 25025081301 | Jamaica Plain | +37.9 |

**Most oversaturated tracts (negative gap — supply exceeds demand):**

| Tract | Neighbourhood | Gap Score |
|---|---|---|
| 25025070202 | Chinatown | -100.0 |
| 25025000805 | Allston | -96.4 |
| 25025030400 | North End | -90.6 |
| 25025050600 | East Boston | -88.8 |
| 25025000804 | Allston | -88.0 |

[View interactive map](https://rawcdn.githack.com/IrfanTajuddin/Boston-Commercial-Viability-Index/main/outputs/cvi_interactive.html)

---

## Key Findings

### 1. Commercial viability concentrates along transit corridors
The highest CVI scores align with the MBTA rapid transit network, particularly the Orange Line through Dorchester and the Green Line through Brighton and Jamaica Plain. Transit access is the sharpest differentiator between high and low-scoring tracts.

### 2. The most underserved tracts combine transit access with thin commercial presence
Mission Hill and Mattapan both score above 63 on the gap score with only 1 POI each, yet have 5-7 transit stops and resident populations above 2,800. These are not commercially inactive by accident as they represent genuine unmet demand that existing supply has not caught up with.

### 3. Downtown-adjacent tracts appear oversaturated on residential metrics alone
Chinatown, Allston, and the North End score -88 to -100 on the gap score. This reflects a real limitation: supply is measured against residential demographics only. Daytime population (workers, students, visitors) is not captured, so employment-dense and nightlife-heavy areas will systematically appear more oversaturated than they are in practice.

### 4. POI density and transit access are spatially decoupled
High transit access does not reliably predict high commercial density. Several tracts with strong transit scores have very low POI counts hence these are the underserved opportunity zones the gap analysis surfaces. The two pillars provide independent information.

### 5. Demographic catchment is relatively uniform across the city
Pillar 2 shows less spatial variation than Pillars 1 or 3. Boston's residential income and population are distributed broadly enough that demographics alone is a weak differentiator, and that transit access and existing market conditions are the stronger signals for commercial viability.

---

## Limitations

- Residential demographics only with no daytime population, which understates demand in employment-dense areas
- OSM POI coverage is community-maintained and may undercount informal or newer businesses
- Transit access measured by stop count within tract boundary, not walking-time isochrones
- No commercial vacancy rates, rents, or lease availability data
- Inverted-U peak calibrated to Boston's distribution and not directly transferable to other cities without recalibration

---

## Summary

This project builds a tract-level Commercial Viability Index for Boston from publicly available transit, demographic, and POI data. It identifies where commercial conditions are strongest and through the Market Gap Score, it shows where demand fundamentals are not matched by existing supply. The clearest signals are in Mission Hill and Mattapan: strong transit access and resident populations with near-zero commercial presence.

The index is a location intelligence input, not a standalone decision tool. Vacancy data, rental rates, and site-level assessment would be needed before any investment decision.
