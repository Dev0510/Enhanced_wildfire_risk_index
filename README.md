# EWRI: Enhanced Wildfire Risk Index

A wildfire risk assessment framework that compares **traditional satellite features** against **Google AlphaEarth foundation model embeddings** to predict fire-prone areas.

**Author:** Dev Sharma | **Institution:** NYIT | **Status:** Master's Thesis

---

## What This Does

1. Extracts environmental features from satellite imagery for 4 US counties
2. Creates a "fire signature" from historically burned areas
3. Scores every location by similarity to that signature
4. Compares traditional satellite features vs. AI embeddings

**Key Finding:** Embeddings improve hazard prediction by **+19% AUC** on average.

---

## Results

| County | Fire Rate | AUC (Satellite) | AUC (Embeddings) | Improvement |
|--------|-----------|-----------------|------------------|-------------|
| Los Angeles | 2.4% | 0.87 | **0.91** | +5% |
| Napa | 2.2% | 0.68 | **0.82** | +20% |
| Suffolk | 2.3% | 0.65 | **0.77** | +19% |
| Maricopa | 1.8% | 0.69 | **0.93** | +34% |
| **Average** | | 0.72 | **0.86** | **+19%** |

---

## Data Sources

### Satellite Features (13 total)

**Static Features (4)** — Same for all years

| Feature | Source | Dataset | Resolution |
|---------|--------|---------|------------|
| Elevation | SRTM | `USGS/SRTMGL1_003` | 30m |
| Slope | SRTM derived | `ee.Terrain.slope()` | 30m |
| Aspect | SRTM derived | `ee.Terrain.aspect()` | 30m |
| Hillshade | SRTM derived | `ee.Terrain.hillshade()` | 30m |

**Dynamic Features (9)** — Collected for 2017, 2018, 2019

| Feature | Source | Dataset | Resolution |
|---------|--------|---------|------------|
| NDVI | Landsat 8 | `LANDSAT/LC08/C02/T1_L2` | 30m |
| EVI | Landsat 8 | `LANDSAT/LC08/C02/T1_L2` | 30m |
| LST | MODIS Terra | `MODIS/006/MOD11A1` | 1km |
| Soil Moisture | SMAP | `NASA/SMAP/SPL3SMP_E/005` | 9km |
| Humidity | PRISM | `OREGONSTATE/PRISM/AN81d` | 800m |
| Wind Speed | ERA5-Land | `ECMWF/ERA5_LAND/DAILY_AGGR` | 9km |
| Precipitation | PRISM | `OREGONSTATE/PRISM/AN81d` | 800m |
| Fuel Type | MODIS | `MODIS/006/MCD12Q1` (LC_Type1) | 500m |
| Land Cover | MODIS | `MODIS/006/MCD12Q1` (LC_Type5) | 500m |

### Vulnerability Data

| Feature | Source | Dataset | Resolution |
|---------|--------|---------|------------|
| Population | WorldPop | `WorldPop/GP/100m/pop` | 100m |
| Built-Up Area | GHSL | `JRC/GHSL/P2016/BUILT_LDSMT_GLOBE_V1` | 38m |
| Social Vulnerability | FEMA NRI | National Risk Index CSV | Census Tract |

### AlphaEarth Embeddings

| Feature | Source | Resolution | Dimensions |
|---------|--------|------------|------------|
| Satellite Embeddings | Google AlphaEarth | 100m | 64 per year × 8 years = 512 |

### External Data

| Data | Source | Format |
|------|--------|--------|
| Fire Perimeters | NIFC / CAL FIRE | Shapefile |
| Census Tracts | TIGER/Line | Shapefile |

---

## Study Areas

| County | State | Hexagons | Validation Fire |
|--------|-------|----------|-----------------|
| Los Angeles | CA | 171,414 | Bobcat Fire 2020 |
| Napa | CA | 19,307 | Glass Fire 2020 |
| Suffolk | NY | 58,587 | Pine Barrens 2020 |
| Maricopa | AZ | 197,573 | Bush Fire 2020 |

---

## How It Works

```
1. Convert all data to H3 hexagons (resolution 9, ~174m)
2. Label hexagons inside fire perimeter as "burned"
3. Create fire signature = average features of burned hexagons
4. Hazard score = cosine similarity to fire signature
5. Risk = Hazard × Vulnerability
```

**Vulnerability Formula:**
```
vulnerability = 0.45×population + 0.45×buildings + 0.10×SOVI
```

---

## Repository Structure

```
EWRI/
├── src/
│   ├── h3_conversion.ipynb         # Satellite + FEMA + Exposure → H3
│   ├── embeddings_conversion.ipynb # AlphaEarth TIF → H3
│   └── EWRI_risk_calculation.ipynb # Risk scores + validation
├── outputs/
│   └── {county}/{county}_EWRI_final.csv
└── README.md
```

---

## Quick Start

```bash
# 1. Install dependencies
pip install pandas geopandas numpy h3 rasterio scikit-learn tqdm shapely

# 2. Set COUNTY variable in each notebook
COUNTY = "los_angeles"  # or napa, suffolk, maricopa

# 3. Run notebooks in order
h3_conversion.ipynb → embeddings_conversion.ipynb → EWRI_risk_calculation.ipynb
```

---

## Output Format

Each county produces a CSV with these columns:

| Column | Description |
|--------|-------------|
| `h3_index` | H3 hexagon ID (resolution 9) |
| `hazard_baseline` | Satellite-based hazard score |
| `hazard_enhanced` | Embeddings-based hazard score |
| `vulnerability` | Normalized vulnerability (0-1) |
| `risk_baseline` | hazard_baseline × vulnerability |
| `risk_enhanced` | **EWRI score** (main output) |
| `risk_category` | Low / Moderate / High / Very High |
| `burned` | Ground truth (1 = burned, 0 = not) |

---

## Contact

Dev Sharma — dsharm25@nyit.edu

---

*Last updated: November 2025*
