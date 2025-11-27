# EWRI: Enhanced Wildfire Risk Index
### A Multi-Modal Approach to Wildfire Risk Assessment Using Foundation Model Embeddings

**Author:** Dev Sharma  
**Project Type:** Master's Thesis Research  
**Status:** Active Development

---

## Abstract

This repository implements a novel wildfire risk assessment framework that integrates Google's AlphaEarth Foundation Model embeddings with traditional remote sensing features and socio-economic vulnerability indicators. The key innovation is using pre-trained satellite embeddings as a high-dimensional representation of landscape characteristics, which are then compared against "fire signatures" derived from historically burned areas using cosine similarity.

---

## Research Question

**Can foundation model embeddings from satellite imagery improve wildfire hazard prediction compared to traditional satellite-derived features (NDVI, LST, fuel type, terrain)?**

Secondary questions:
- How does the integration of socio-economic vulnerability (FEMA SOVI, population, building density) affect risk prediction accuracy?
- Do embeddings generalize across different fire regimes (WUI fires vs. wilderness fires)?

---

## Methodology

### 1. Data Acquisition & Preprocessing

All geospatial data is converted to Uber's H3 hexagonal grid at resolution 9 (~174m edge length) for consistent spatial analysis.

| Data Source | Variables | Resolution | Temporal Coverage |
|-------------|-----------|------------|-------------------|
| Google AlphaEarth | 64-dim embeddings/year | 100m | 2017-2024 (8 years) |
| MODIS/Landsat | NDVI, EVI, LST | 250m-1km | 2017-2019 |
| SRTM/ASTER | Elevation, Slope, Aspect, Hillshade | 30m | Static |
| LANDFIRE | Fuel Type, Land Cover | 30m | Static |
| WorldPop | Population density | 100m | 2017-2019 |
| Google Open Buildings | Building footprints | Vector | 2020 |
| FEMA NRI | Social Vulnerability Index (SOVI) | Census Tract | 2020 |
| NIFC | Fire perimeters (validation) | Vector | 2020 |

### 2. Hazard Score Calculation

The hazard score quantifies similarity between any location and historically burned areas:

```
fire_signature = mean(embeddings[burned_hexagons])
hazard_score = cosine_similarity(hexagon_embedding, fire_signature)
```

Two hazard variants are computed:
- **hazard_baseline**: Using traditional satellite features (31 variables)
- **hazard_enhanced**: Using AlphaEarth embeddings (192 features for pre-fire years)

### 3. Vulnerability Index

Vulnerability combines exposure (who/what is at risk) with social vulnerability:

```
vulnerability = 0.45 × population_norm + 0.45 × building_norm + 0.10 × sovi_norm
```

Weights are calibrated to minimize blockiness from tract-level SOVI data while preserving high-resolution population/building information.

### 4. Risk Index (EWRI)

```
EWRI = hazard_enhanced × vulnerability
```

### 5. Validation

- **Metric**: AUC-ROC against binary burned/unburned labels
- **Fire capture rate**: % of actual fires in top 10/20/30% risk areas

---

## Study Areas

| County | State | Fire Event | Hexagons | Fire Type |
|--------|-------|------------|----------|-----------|
| Los Angeles | CA | Bobcat Fire 2020 | 171,414 | WUI |
| Napa | CA | Glass Fire 2020 | 19,307 | WUI |
| Suffolk | NY | Pine Barrens 2020 | 58,587 | Wildland |
| Maricopa | AZ | Bush Fire 2020 | 197,573 | Wildland |

---

## Repository Structure

```
EWRI/
├── raw/{county}/                    # Raw input data
│   ├── satellite_data/              # GeoTIFF rasters (NDVI, LST, terrain)
│   ├── embeddings_data/             # AlphaEarth 512-band TIFs
│   ├── Vulnerability/               # Population & BuiltUp TIFs
│   ├── fema_data/                   # FEMA NRI CSV + metadata
│   ├── fire_data/                   # Fire perimeter shapefiles
│   └── census_tracts/               # TIGER/Line shapefiles
│
├── processed/{county}/              # H3-indexed parquet files
│   ├── satellite_h3.parquet         # 31 satellite features
│   ├── embeddings_h3.parquet        # 64 features × 8 years = 512 (192 used for pre-fire)
│   ├── fema_h3.parquet              # SOVI scores by hexagon
│   └── exposure_h3.parquet          # Population + buildings
│
├── outputs/{county}/                # Final results
│   └── {county}_EWRI_final.csv
│
└── src/                             # Processing notebooks
    ├── h3_conversion.ipynb          # Satellite, FEMA, Exposure → H3
    ├── embeddings_conversion.ipynb  # AlphaEarth TIF → H3 (chunked)
    └── EWRI_risk_calculation.ipynb  # Risk calculation & validation
```

---

## Running the Pipeline

### Prerequisites

```bash
pip install pandas geopandas numpy h3 rasterio scikit-learn tqdm shapely
```

### Step 1: Convert Raw Data to H3

```bash
# Set COUNTY variable in each notebook, then run all cells

# Satellite, FEMA, and Exposure data
jupyter notebook src/h3_conversion.ipynb

# AlphaEarth embeddings (memory-intensive, uses chunked processing)
jupyter notebook src/embeddings_conversion.ipynb
```

### Step 2: Calculate Risk Scores

```bash
jupyter notebook src/EWRI_risk_calculation.ipynb
```

**County Options:** `los_angeles`, `napa`, `suffolk`, `maricopa`

---

## Output Schema

| Column | Type | Description |
|--------|------|-------------|
| `h3_index` | string | H3 hexagon identifier (resolution 9) |
| `hazard_baseline` | float | Cosine similarity using satellite features |
| `hazard_enhanced` | float | Cosine similarity using embeddings |
| `vulnerability` | float | Normalized vulnerability score (0-1) |
| `risk_baseline` | float | hazard_baseline × vulnerability |
| `risk_enhanced` | float | **EWRI score** (hazard_enhanced × vulnerability) |
| `risk_category` | string | Percentile-based: Low/Moderate/High/Very High |
| `burned` | int | Ground truth label (1=burned, 0=unburned) |

---

## Preliminary Results

| County | Hazard AUC (Satellite) | Hazard AUC (Embeddings) | Improvement |
|--------|------------------------|-------------------------|-------------|
| Napa | 0.68 | 0.82 | +20% |
| Maricopa | 0.69 | 0.93 | +35% |

### Key Finding

**Embeddings consistently outperform traditional satellite features.** The AlphaEarth foundation model embeddings capture fire-prone environmental patterns more effectively than manually curated satellite indices (NDVI, LST, fuel type). Maricopa (wilderness fire) showed the largest improvement (+35%), suggesting embeddings excel at capturing subtle landscape characteristics in natural areas.

*Note: Risk AUC depends on correlation between vulnerability and fire occurrence. In areas where fires occur in low-population wilderness, multiplying by vulnerability reduces predictive accuracy.*

---

## Known Limitations

1. **Temporal mismatch**: Pre-2020 embeddings used to predict 2020 fires
2. **Vulnerability paradox**: High vulnerability areas may have low fire occurrence (urban cores)
3. **Fire signature generalization**: Single fire event may not represent all fire types
4. **SOVI resolution**: Census tract-level data introduces blockiness

---

## References

- **AlphaEarth**: Google Research (2024). Foundation Models for Earth Observation.
- **H3**: Uber Technologies. Hexagonal Hierarchical Spatial Index.
- **FEMA NRI**: Federal Emergency Management Agency. National Risk Index Technical Documentation.
- **WorldPop**: Tatem, A.J. (2017). WorldPop, open data for spatial demography.

---

## Contact

For questions regarding this codebase or collaboration inquiries, please contact the author.

---

*Last updated: November 2025*
