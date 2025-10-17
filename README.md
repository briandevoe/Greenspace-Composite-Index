# 🌿 Developing a Longitudinal Census-Tract Greenness Indicator for Health & Environment Risk Index

**Author(s):** Brian DeVoe  

---

## 1. Purpose and Motivation

Existing opportunity and environmental indices often rely on **cross-sectional** or **city-level** measures of greenness. They lack a consistent, **longitudinal, census-tract–scale** measure, limiting our ability to understand how changing vegetation, trees, park access, and other **greenspace** measures affect health and opportunity over time.

Further integration with environmental indicators such as air quality measures, heat exposure, industrial pollution sites, noise pollution, etc. will then culminate in a robust environmental index that will be apart of a Health & Environment risk index. 

---

## 2. Data Sources

| **Indicator** | **Dataset / Source** | **Notes** |
|---------------|----------------------|------------|
| NDVI & EVI | Landsat Surface Reflectance – USGS | Annual/seasonal composites (30 m); cloud-masked and harmonized via Earth Engine |
| Tree Canopy Cover (TCC) | USDA Forest Service Tree Canopy Cover | 30 m gridded canopy fraction (1985–2023) |
| Land Cover Classes | USGS National Land Cover Database (NLCD) | Used to compute % vegetated area (forest, grass, shrub, wetland) |
| Park Proximity / Access | PAD-US-AR (Browning et al., 2022), ParkServe, CDC-derived park data | Percent of tract (and 0.5 mi buffer) considered park area |
| Geographies | Census Tracts, Block Groups, Counties — U.S. Census Bureau TIGER/Line | Consistent 2020 boundaries across years |

**Potential additions:** Sentinel-2 indices (10 m)

---

## 3. Data Harmonization and Aggregation Process

### Goal
Create a uniform, yearly *greenness stack* — all indicators aligned to the same spatial grid — and compute tract-level summaries via zonal statistics.

#### TODO: 
A lot of time needs to be spent on the efficiency of aggregating these data, especially satellite raster data which can be massive in size.

### Why Raster Harmonization?
All major inputs (NDVI, EVI, TCC, Land Cover) are raster-based.  
Aligning them to a single (30mx30m ideally) grid ensures every pixel represents the same footprint across datasets and years.  
This prevents spatial mismatch and allows reproducible tract, block-group, or county aggregation.

### Process Overview
1. **Define a master grid**
   - CRS: EPSG 5070 (Albers Equal Area) -- (TODO: research the best map projection)
   - Resolution: 30 m  
   - Alignment: Match base NDVI mosaic (TODO: maybe define a master grid that isn't based on one of the indicators)
2. **Resample each input raster**
   - Continuous layers (NDVI, EVI, TCC): Bilinear resampling  
   - Categorical layers (land cover): Nearest neighbor  
   - Park polygons: Rasterized to fractional % coverage per 30 m pixel (TODO: maybe figure out a better way to aggregate or define park access? The park indicator is difficult...)
3. **Stack and harmonize**
   - Build annual raster stacks or VRTs with identical extent, resolution, and nodata masks  
   - Optionally store as Cloud-Optimized GeoTIFFs
4. **Zonal aggregation**
   - Overlay Census Tract polygons  
   - Compute mean NDVI, mean EVI, mean TCC, % vegetated area, % park area (TODO: add thresholds in addition to mean/median, i.e. we want a dataset like.... geoid | mean NDVI | NDVI > 0.50 | NDVI > 0.65 | ....)
   - Output one record per tract per year, including coverage diagnostics

**Outcome:**  
A tract-year panel dataset (~70 k tracts × 20 years ≈ 1.4 M records) providing consistent greenness metrics across time, ready for index construction and validation.

---

## 4. Challenges in Harmonization

- **Temporal mismatch:** Different update frequencies (Landsat yearly, NLCD every 2–3 years).  
- **Boundary drift:** Changing Census tract definitions (2010 → 2020) require harmonized boundary crosswalks.  
- **Resolution conflicts:** Reprojecting 10 m–250 m sources to a common 30 m grid.  
- **Missing or cloud-masked pixels:** Need valid-pixel thresholds or interpolation.  
- **Rural extremes:** Extremely high greenness may dominate scales — apply top-coding?  
- **Processing scale:** National mosaics require efficient parallel/tiled processing (e.g., VRT + windowed reads).

- **Desert (or other) geographical extremes and differences:** Is it fair to let the environment index values collapse in areas like the high desert in CA and then to categorize tracts within that geographical area as "low"? Should we apply weights to different geographical regions to more appropriately represent relative opportunity access to greenspace? (like how SEDA compares test scores between states?) -- This is probably the most difficult problem 

- use Köppen climate zone ??? Ask BU department

---

## 5. Methods and Weighting Strategy

### Index Construction
- **Equal-weight baseline:** Each standardized indicator contributes equally.  
- **Expert-weight model:** Emphasize tree canopy and park proximity.  
- **PCA-weighted model:** Use unsupervised PCA loadings (PC1) to derive empirical weights and reduce collinearity.

### Validation
Use tract-level health and well-being data:
- CDC PLACES (obesity, mental health, physical activity)  
- NCHS life expectancy  
- EPA AQS (air quality, asthma)  
- SEDA test scores (education outcomes)  

Evaluate via correlations, regression R², predictive validity, and spatial residual diagnostics.  
Select the index version balancing interpretability and predictive performance.

---

## 6. Expected Deliverables

- Annual tract-level CSV + GPKG files (2000–present) — *for this class, one year of data*  
- Technical documentation describing harmonization and weighting  
- Validation tables/figures linking greenness to health outcomes  
- Reproducible scripts (Python + Earth Engine) for future updates
