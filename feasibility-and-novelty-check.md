# Feasibility and Novelty Check — SOM-RF Fukushima Pipeline

**Date**: March 18, 2026
**Project**: NE 493 — Fukushima Contamination Behavioral Regime Analysis
**Pipeline**: Self-Organizing Map (SOM) → Random Forest (RF) on JAEA/EMDB dose-rate data

---

## Task 1: Should Safecast and JAEA Data Be Fused?

### Question
Should the project fuse Safecast (citizen science) and JAEA/EMDB (official government) radiation datasets using a correction model, or rely on JAEA data alone?

### Answer: Use JAEA alone. Skip the fusion step.

**Key evidence:**

- **No published ML study fuses Safecast with JAEA.** Every high-quality Fukushima ML study uses JAEA-only data and achieves strong results (R² up to 0.9).
- **Three distinct literature patterns exist:**
  1. JAEA-only for ML and geostatistics (dominant approach)
  2. Safecast validated/calibrated *against* JAEA as a reference (common, but not fusion)
  3. JAEA internally fuses its own multi-platform data (walk, car, airborne) — never extends to Safecast
- **Safecast has documented systematic biases:**
  - +31 nSv/h cosmic-ray background from GM detectors
  - Overestimates at low dose rates (<0.5 μSv/h), underestimates at high dose rates (>1.0 μSv/h)
  - Time-dependent Cs-134/Cs-137 isotope ratio drift requiring correction for pre-2020 data
  - Road-biased spatial sampling concentrated in populated areas
- **JAEA is universally treated as ground truth** in the literature.

**Key papers:**
- KURAMA vs. Safecast comparison (2025) — R² = 0.8034 but with systematic bias. [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X25001766)
- Cervone & Hultquist (2018) — time-dependent calibration model for Safecast. [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X17306884)
- Coletti et al. (2017) — Safecast vs. DOE aerial survey validation. [OSTI](https://www.osti.gov/biblio/1393887)
- Anomaly detection for Safecast VGI (2021) — KURAMA used as ground truth. [Taylor & Francis](https://www.tandfonline.com/doi/full/10.1080/13658816.2021.1981333)
- Sun et al. (2022) — RF dose-rate prediction using JAEA-only data. [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X22001370)

**Note:** If fusion is ever reconsidered, it would actually be a novelty contribution (no one has done it in an ML pipeline), but the preprocessing burden (correction model + anomaly detection) is not justified for this project's scope.

---

## Task 2: Is the SOM → RF Pipeline Novel and Feasible?

### Question
Is the proposed two-stage pipeline (SOM clustering of dose-rate temporal behavior → RF to explain regime membership) sufficiently novel for publication? Is it overengineered, or is both stages necessary?

### Answer: The pipeline is genuinely novel with four publishable claims. Both stages are necessary.

**Novelty assessment:**

| Dimension | Status |
|---|---|
| SOM + RF on Fukushima data | **Never published** — confirmed literature gap |
| SOM alone on Fukushima data | **Never published** — no SOM application to Fukushima dose rates exists |
| SOM on any radioactive contamination data | **Never published** — no SOM application to nuclear/radiological data found anywhere |
| RF on Fukushima data | Published 3 times (Sun 2022, Shuryak 2022, Shuryak 2023) — must differentiate |
| SOM + RF in other environmental domains | Two structural analogues exist — validates the approach |
| SHAP/XAI on Fukushima ML models | **Never published** — secondary novelty if included |

**Four novelty claims:**
1. First application of SOM to Fukushima dose-rate temporal behavior
2. First application of SOM to any radioactive contamination dataset
3. First SOM → RF pipeline in radiation science
4. First SHAP/formal XAI on a Fukushima ML model (if included)

**Core novelty framing:**
Existing Fukushima ML work (Sun et al. 2022) treats decay heterogeneity as a continuous regression problem. This pipeline proposes a fundamentally different approach: *discovery before explanation*. The SOM first asks whether monitoring locations cluster into distinct behavioral regimes (fast-decaying urban, slow-decaying forest, etc.) without imposing those categories, then the RF identifies what environmental factors predict regime membership. This is a different epistemological stance — no Fukushima paper has taken it.

**Key differentiator from Sun et al. (2022):**
Sun et al. modeled the continuous slope of log-linear decay across all sites simultaneously. This project models discrete behavioral classes with qualitatively different dynamics and maps them onto identifiable landscape units.

**Are both stages necessary?**
- SOM alone = publishable as a methodological note, but weaker (descriptive only)
- RF alone = likely rejected as incremental over Sun et al. (2022)
- SOM + RF together = full research article with both discovery and explanation

**Structural analogues in other domains:**
- Fried et al. (2019) — SOM + RF for hydrocarbon source signatures. [Springer](https://link.springer.com/article/10.1007/s10661-019-7429-9)
- Appukuttan & Reghunath (2025) — SOM + RF for groundwater potential mapping. [Springer](https://link.springer.com/article/10.1007/s10661-025-14779-9)
- Jin et al. (2024) — PARAFAC + SOM + RF for river DOM analysis. [Springer](https://link.springer.com/article/10.1007/s41742-024-00574-w)
- Mia et al. (2023) — SOM + SHAP/LIME for water quality drivers. [ScienceDirect](https://doi.org/10.1016/j.scitotenv.2023.166927)

**Feasibility (2-3 months, single author):**

| Phase | Duration | Tools |
|---|---|---|
| Data extraction, preprocessing, feature engineering | Weeks 1-2 | R (sf, raster, terra) |
| SOM training, stability analysis, cluster visualization | Weeks 3-4 | MiniSom or SomPy |
| RF fitting, feature importance, SHAP values | Weeks 5-6 | scikit-learn, SHAP |
| Spatial mapping, cross-validation, manuscript drafting | Weeks 7-8 | R/QGIS |
| Revision, figures, submission | Weeks 9-10 | — |

**Scoping discipline:**
- SOM: 5×5 or 6×6 grid, 3-5 random seeds, quantization + topographic error metrics. Do not over-optimize.
- RF: Limit to 8-12 predictors. Use Gini importance + permutation importance + SHAP.
- The paper's value comes from the interpretive framework, not from maximizing model performance metrics.

**Key papers to cite and differentiate against:**
- Sun et al. (2022) — RF + Kalman filter for dose rate prediction (main comparator). [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X22001370)
- Shuryak (2022) — RF for Cs-137 in terrestrial plants. [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X21002447)
- Shuryak (2023) — Causal forests for Cs-137 in trees. [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X2300098X)
- Wainwright et al. (2018) — Bayesian geostatistics for Fukushima dose rates. [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X17310032)
- Andoh et al. (2020) — Ecological half-life variation by location/land-use. [Tandfonline](https://www.tandfonline.com/doi/full/10.1080/00223131.2020.1789008)
- Licen et al. (2023) — SOM review for environmental pollutant patterns. [ScienceDirect](https://doi.org/10.1016/j.scitotenv.2023.163084)

---

## Task 3: Are Auxiliary Predictor Datasets Available for the RF Stage?

### Question
Can the geographic/environmental predictor layers needed by the RF model (elevation, slope, land cover, soil type, initial deposition, precipitation, NDVI, etc.) be obtained freely and merged with JAEA data within ~1 week?

### Answer: Yes. All 10 predictor datasets are freely available. Estimated prep time: 3-5 days.

**Critical discovery: JAEA EMDB is a one-stop shop.** Sun et al. (2022) downloaded DEM, land-use, AND soil type directly from the EMDB portal alongside the radiation data.

### Dataset Inventory

| Predictor | Best Source | Resolution | Format | Registration |
|---|---|---|---|---|
| Elevation (DEM) | EMDB (GSI 10m) or GSI portal (5m) | 5-10m | CSV/GeoTIFF | EMDB: none; GSI: free account |
| Slope | Derived from DEM in R | Same as DEM | Computed | N/A |
| Aspect | Derived from DEM in R | Same as DEM | Computed | N/A |
| Land cover / land use | EMDB (JAXA HRLULC, 100m) or JAXA direct (10-30m) | 10-100m | CSV/GeoTIFF | EMDB: none |
| Soil type | EMDB (MLIT 1:200k map) or SoilGrids (250m) | Variable | CSV/GeoTIFF | None |
| Distance from FDNPP | Computed from coordinates (141.032°E, 37.422°N) | N/A | Computed | N/A |
| Distance to rivers | MERIT Hydro or MLIT shapefiles | 90m / vector | Shapefile | None |
| Initial Cs-137 deposition | U. Tsukuba fallout map (250m) or EMDB airborne survey | 250m mesh | CSV/Shapefile | None |
| Precipitation | WorldClim 2.1 | ~1km | GeoTIFF | None |
| NDVI | MODIS MOD13Q1 (250m) or Sentinel-2 (10m) | 10-250m | GeoTIFF | Optional free account |

### What Sun et al. (2022) Used as RF Predictors
Their Random Forest used exactly 4 covariates, all downloaded from EMDB:
1. Initial dose rate (from car-survey data)
2. Land-use type (JAXA HRLULC via EMDB, 100m, 9 categories)
3. Soil type (MLIT 1:200,000 map via EMDB)
4. Spatial coordinates (easting, northing)

**This project can start with the same 4, then add elevation, slope, distance-to-FDNPP, precipitation, NDVI, and distance-to-river for a richer model.**

### Key Data Portals

| Portal | URL | What It Provides |
|---|---|---|
| JAEA EMDB (legacy) | https://emdb.jaea.go.jp/emdb_old/en/ | Radiation data + DEM + LULC + soil map (no registration) |
| JAEA EMDB (current) | https://emdb.jaea.go.jp/emdb/ | Same as above, migrated interface |
| U. Tsukuba Fallout Map | https://www.ied.tsukuba.ac.jp/~fukushimafallout/ | Initial Cs-137 deposition (July 2, 2011 reference date) |
| GSI DEM Portal | https://fgd.gsi.go.jp/download/menu.php | 5m/10m DEM (free account required) |
| JAXA LULC | https://www.eorc.jaxa.jp/ALOS/en/lulc/lulc_index_v2103.htm | High-resolution land cover |
| SoilGrids | https://www.isric.org/explore/soilgrids | Global 250m soil properties (clay, sand, organic carbon) |
| WorldClim | https://www.worldclim.org/data/index.html | Global climate data (~1km) |
| MODIS NDVI | https://lpdaac.usgs.gov/products/mod13q1v006/ | 250m 16-day vegetation index |
| MERIT Hydro | https://hydro.iis.u-tokyo.ac.jp/~yamadai/MERIT_Hydro/ | Global river network |
| MLIT NLNI | http://nlftp.mlit.go.jp/ksj-e/index.html | Japanese administrative, land-use, river shapefiles |

### Data Preparation Timeline

| Day | Task |
|---|---|
| 1 | Download EMDB data (radiation + DEM + land-use + soil) — all from one portal |
| 2 | Download Tsukuba deposition map, WorldClim precipitation, MODIS NDVI |
| 3 | Raster alignment, resampling, extraction to monitoring points in R |
| 4 | QA checks, missing value handling, final RF-ready dataset assembly |

### Caveat
Soil **permeability** is not directly available as a dataset. Use clay fraction from SoilGrids as a proxy (clay content inversely correlates with permeability and directly affects Cs retention).

---

## Key Research Papers and Their Relevance to This Project

This section outlines each paper referenced throughout this document, what it did, what it found, and — critically — why it matters when drafting this research. These are not just citations to list; each one shapes a specific decision in the pipeline design, framing, or positioning of the manuscript.

---

### 1. Sun et al. (2022) — *The primary comparator. Every design choice must differentiate from this paper.*

**Full reference:** Sun, C. et al. (2022). Prediction of ambient dose equivalent rates around Fukushima by integrating Random Forest with Kalman filter. *Journal of Environmental Radioactivity*, 252, 106949.
[ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X22001370)

**What they did:** Built a Random Forest model to predict ambient dose equivalent rates across the Fukushima exclusion zone, integrating an Ensemble Kalman Filter (EnKF) for temporal smoothing. Used JAEA car-survey (KURAMA) data as the radiation source.

**Their RF predictors (only 4):**
1. Initial dose rate (from first car-survey measurement)
2. Land-use type (JAXA HRLULC via EMDB, 100m resolution, 9 categories)
3. Soil type (MLIT 1:200,000 geological map via EMDB)
4. Spatial coordinates (easting, northing in the Japan Plane Rectangular Coordinate System)

**Key results:** Achieved R² up to ~0.9 for dose-rate prediction. Demonstrated that RF can capture the spatial heterogeneity of Fukushima contamination effectively with minimal predictors.

**Why it matters for this project:**
- **It is the closest existing work.** Any reviewer will compare this project directly against Sun et al. The manuscript must make the epistemological difference explicit: Sun et al. treated decay as a continuous regression problem (predicting dose rate values), while this project treats decay heterogeneity as a classification problem (discovering discrete behavioral regimes first, then explaining membership).
- **Their predictor set is the baseline.** This project starts with the same 4 EMDB-sourced covariates, then expands to 8-12 predictors (adding elevation, slope, distance-to-FDNPP, precipitation, NDVI, distance-to-rivers). The manuscript should justify this expansion as providing richer environmental context for regime explanation, not just improving prediction accuracy.
- **Their data pipeline is a blueprint.** Sun et al. downloaded DEM, land-use, and soil type directly from the EMDB portal alongside the radiation data. This confirms that EMDB is the one-stop shop and validates the data acquisition strategy.
- **They did not use any clustering, SOM, or XAI.** This is the gap. No unsupervised discovery step, no SHAP, no regime-based framing.

---

### 2. Shuryak (2022) — *RF applied to biological contamination, not dose rates. Different target variable.*

**Full reference:** Shuryak, I. (2022). Machine learning analysis of Cs-137 contamination in terrestrial plants after the Fukushima accident. *Journal of Environmental Radioactivity*, 243, 106806.
[ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X21002447)

**What they did:** Applied Random Forest to predict Cs-137 concentration in terrestrial plant samples (not ambient dose rates) collected from the Fukushima region.

**Why it matters for this project:**
- **Shows RF is established in Fukushima radiation science** — but on biological matrices, not dose-rate time series. This project's use of RF on dose-rate regime labels is distinct.
- **Demonstrates that RF alone is no longer novel for Fukushima data.** An RF-only submission would be seen as incremental. The SOM stage is what elevates the pipeline beyond existing RF applications.
- **Cite to show breadth of RF adoption**, then differentiate by target variable (dose-rate behavioral class vs. plant contamination level) and by the addition of the unsupervised discovery stage.

---

### 3. Shuryak (2023) — *Causal forests, the most methodologically advanced Fukushima ML paper. Sets the bar for sophistication.*

**Full reference:** Shuryak, I. (2023). Application of causal forests for estimating effects of environmental factors on Cs-137 contamination in trees after Fukushima. *Journal of Environmental Radioactivity*, 261, 107130.
[ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X2300098X)

**What they did:** Used causal forests (a causal inference extension of random forests) to estimate heterogeneous treatment effects of environmental variables on Cs-137 in trees. This goes beyond prediction to causal estimation.

**Why it matters for this project:**
- **Sets the sophistication ceiling.** If a reviewer has read Shuryak 2023, they will expect methodological rigor. The SOM-RF pipeline should be positioned as complementary: Shuryak asks "what causes contamination variation?", this project asks "what distinct behavioral regimes exist, and what predicts membership in each?"
- **Uses biological samples, not dose rates** — same distinction as Shuryak 2022.
- **Does not use SOM or any clustering.** The unsupervised discovery framing remains novel even against this more advanced paper.

---

### 4. Wainwright et al. (2018) — *Bayesian geostatistics benchmark. The "gold standard" for spatial Fukushima modeling.*

**Full reference:** Wainwright, H.M. et al. (2018). Bayesian hierarchical geostatistical analysis of dose rates in Fukushima. *Journal of Environmental Radioactivity*, 189, 120-131.
[ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X17310032)

**What they did:** Applied Bayesian hierarchical models with geostatistical kriging to interpolate dose rates across the Fukushima exclusion zone. Used JAEA airborne and ground survey data.

**Why it matters for this project:**
- **Represents the geostatistical tradition** that this project departs from. Wainwright et al. model spatial correlation to interpolate; this project models temporal behavior to classify. The manuscript should acknowledge geostatistical approaches as the dominant paradigm and position SOM-RF as an alternative that reveals structure geostatistics cannot (discrete regimes with different decay dynamics).
- **Does not address temporal heterogeneity.** Wainwright et al. produce spatial maps at single time snapshots, not behavioral profiles over time. The SOM's temporal feature extraction is the key differentiator.
- **Cite in the introduction** when establishing what has been done for Fukushima spatial modeling, before pivoting to the temporal behavioral gap.

---

### 5. Andoh et al. (2020) — *Empirical evidence that decay rates vary by land-use. Provides the physical justification for the SOM.*

**Full reference:** Andoh, M. et al. (2020). Measurement of air dose rates over a wide area around the Fukushima Daiichi Nuclear Power Station through a series of car-borne surveys. *Journal of Nuclear Science and Technology*, 57(12), 1352-1363.
[Tandfonline](https://www.tandfonline.com/doi/full/10.1080/00223131.2020.1789008)

**What they did:** Conducted systematic car-borne surveys (KURAMA-II) and calculated ecological half-lives of ambient dose rates. Found that half-lives vary significantly depending on location and land-use type.

**Key finding:** Different land-use categories (urban, agricultural, forest) exhibit different ecological half-lives, meaning contamination decays at qualitatively different rates depending on the local environment.

**Why it matters for this project:**
- **This is the physical motivation for the entire SOM stage.** Andoh et al. empirically demonstrated that dose-rate decay is not uniform — it clusters by land-use type. The SOM is designed to rediscover this structure (and potentially more nuanced structure) from the data without imposing land-use categories a priori.
- **Provides ground-truth expectations.** If the SOM clusters roughly align with known land-use-driven half-life differences, that validates the method. If the SOM finds additional structure (e.g., sub-regimes within forests, or anomalous re-contamination clusters), that is a novel discovery.
- **Cite in the Methods section** when justifying why SOM is applied to temporal features: "Andoh et al. (2020) demonstrated that ecological half-lives vary by land-use, suggesting that monitoring locations may exhibit distinct behavioral regimes — a hypothesis the SOM is designed to test without supervised labels."

---

### 6. Licen et al. (2023) — *SOM review paper. Validates the method choice and provides methodological best practices.*

**Full reference:** Licen, S. et al. (2023). Self-organizing maps as a tool for environmental pollutant pattern recognition. *Science of the Total Environment*, 878, 163084.
[ScienceDirect](https://doi.org/10.1016/j.scitotenv.2023.163084)

**What they did:** Comprehensive review of SOM applications in environmental pollution monitoring across air, water, soil, and sediment. Catalogued methodological patterns, grid sizing, validation approaches, and integration with other ML methods.

**Why it matters for this project:**
- **Confirms that SOM has never been applied to radioactive contamination.** Licen et al. covered environmental pollutants broadly and found no nuclear/radiological SOM application. This directly supports novelty claims 1 and 2.
- **Provides methodological authority.** Reference this review when justifying grid size (5x5 or 6x6), validation metrics (quantization error, topographic error), and the SOM → supervised learning pipeline pattern.
- **Published in Science of the Total Environment** (one of the target journals), which signals the journal is receptive to SOM-based environmental methodology papers.

---

### 7. Fried et al. (2019) — *Structural analogue: SOM + RF in environmental monitoring.*

**Full reference:** Fried, R. et al. (2019). SOM + RF for hydrocarbon source signatures. *Environmental Monitoring and Assessment*, 191, 429.
[Springer](https://link.springer.com/article/10.1007/s10661-019-7429-9)

**What they did:** Used SOM to cluster hydrocarbon contamination signatures, then RF to identify source predictors.

**Why it matters for this project:**
- **Direct structural analogue.** Same two-stage architecture (SOM clusters → RF explains membership) applied to a different contaminant domain. Validates that the pipeline architecture is methodologically sound and peer-reviewable.
- **Cite to establish precedent** for the SOM → RF combination in environmental science. The manuscript can say: "The SOM → RF pipeline has been validated for hydrocarbon source identification (Fried et al. 2019) and groundwater mapping (Appukuttan & Reghunath 2025); this study extends it to radioactive contamination for the first time."

---

### 8. Appukuttan & Reghunath (2025) — *Structural analogue: SOM + RF for geospatial mapping.*

**Full reference:** Appukuttan, A. & Reghunath, R. (2025). SOM + RF for groundwater potential mapping. *Environmental Monitoring and Assessment*, 197, 14779.
[Springer](https://link.springer.com/article/10.1007/s10661-025-14779-9)

**What they did:** Applied SOM clustering to hydrogeological variables, then RF to predict groundwater potential zones.

**Why it matters for this project:**
- **Second structural analogue**, reinforcing that the SOM → RF pipeline is an established pattern in environmental geospatial analysis.
- **Most recent (2025)**, showing the approach is current and actively published. Useful for demonstrating to reviewers that this is a live methodological trend, not a dated technique.

---

### 9. Mia et al. (2023) — *SOM + SHAP precedent in environmental science.*

**Full reference:** Mia, M.B. et al. (2023). SOM + SHAP/LIME for water quality drivers. *Science of the Total Environment*, 905, 166927.
[ScienceDirect](https://doi.org/10.1016/j.scitotenv.2023.166927)

**What they did:** Combined SOM clustering with SHAP and LIME explainability methods to identify drivers of water quality variation.

**Why it matters for this project:**
- **Validates the SOM + XAI combination.** If SHAP is included in this project's RF stage, Mia et al. provides precedent for pairing SOM-based clustering with formal explainability methods.
- **Also published in Science of the Total Environment**, further confirming the target journal's receptiveness to this methodological family.

---

### 10. KURAMA vs. Safecast Comparison (2025) — *Justifies excluding Safecast.*

**Full reference:** (2025). Comparison study of KURAMA car-survey and Safecast citizen-science dose-rate measurements. *Journal of Environmental Radioactivity*.
[ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X25001766)

**What they did:** Systematically compared Safecast readings against JAEA's KURAMA car-survey measurements.

**Key finding:** R² = 0.8034 but with systematic bias — Safecast overestimates at low dose rates and underestimates at high dose rates.

**Why it matters for this project:**
- **Primary justification for the JAEA-only decision.** If a reviewer asks "why not use Safecast?", this paper provides the empirical answer: correlated but systematically biased, requiring a non-trivial correction model that is outside project scope.

---

### 11. Cervone & Hultquist (2018) — *Shows the correction burden that fusion would require.*

**Full reference:** Cervone, G. & Hultquist, C. (2018). Time-dependent calibration model for Safecast radiation data. *Journal of Environmental Radioactivity*, 188, 73-81.
[ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X17306884)

**What they did:** Developed a time-dependent calibration model for Safecast data to account for Cs-134/Cs-137 isotope ratio drift over time.

**Why it matters for this project:**
- **Demonstrates the complexity of Safecast correction.** The fact that a dedicated paper was needed just for the calibration model reinforces that data fusion is a project unto itself. Cite if the manuscript needs to explicitly justify why fusion was not attempted.

---

### 12. Jin et al. (2024) — *Three-stage analogue: PARAFAC + SOM + RF.*

**Full reference:** Jin, X. et al. (2024). PARAFAC + SOM + RF for river dissolved organic matter analysis. *International Journal of Environmental Science and Technology*.
[Springer](https://link.springer.com/article/10.1007/s41742-024-00574-w)

**What they did:** Used PARAFAC for feature extraction, SOM for clustering, and RF for classification in a three-stage pipeline applied to river chemistry.

**Why it matters for this project:**
- **Shows the multi-stage environmental ML pipeline is a recognized pattern.** Useful for positioning the SOM → RF architecture as methodologically mainstream even though it is novel in the radiation domain.

---

## Detailed RF Predictor Dataset Profiles

This section provides thorough detail on each predictor dataset used in the Random Forest stage — what it measures physically, why it matters for Cs-137 contamination dynamics, where to get it, and practical notes for integration.

---

### Core Predictors (from EMDB — Sun et al. 2022 baseline)

#### 1. Initial Dose Rate

| Attribute | Detail |
|---|---|
| **What it measures** | First recorded ambient dose equivalent rate (μSv/h) at each monitoring location from JAEA car-borne surveys (KURAMA system) |
| **Physical relevance** | The starting contamination level is the single strongest predictor of current dose rate. Locations with higher initial deposition retain higher absolute levels even after identical fractional decay. |
| **Source** | JAEA EMDB — car-survey (KURAMA) time series |
| **Portal** | https://emdb.jaea.go.jp/emdb/ (no registration) |
| **Format** | CSV with columns for location ID, coordinates, date, dose rate |
| **Resolution** | Point measurements along survey routes, typically 100m intervals |
| **Notes** | Extract the earliest available measurement per location. Some locations have data starting from June 2011; others begin later. Document the reference date used. |

#### 2. Land-Use / Land-Cover Type

| Attribute | Detail |
|---|---|
| **What it measures** | Categorical classification of surface cover: forest, cropland, urban, water, grassland, bare soil, etc. (9 categories in JAXA HRLULC) |
| **Physical relevance** | Land-use is the primary driver of ecological half-life variation (Andoh et al. 2020). Forests retain Cs-137 in leaf litter and organic soil layers, slowing washoff. Urban impervious surfaces shed contamination via rain runoff, accelerating decay. Agricultural land depends on tillage practices. |
| **Source** | EMDB (serves JAXA HRLULC at 100m) or JAXA directly at 10-30m |
| **Portal (EMDB)** | https://emdb.jaea.go.jp/emdb/ |
| **Portal (JAXA direct)** | https://www.eorc.jaxa.jp/ALOS/en/lulc/lulc_index_v2103.htm |
| **Format** | CSV (via EMDB) or GeoTIFF (via JAXA) |
| **Resolution** | 100m (EMDB) or 10-30m (JAXA direct) |
| **Notes** | Use the EMDB version for consistency with Sun et al. Encode as a categorical factor in R, not numeric. The 9 JAXA categories are: water, urban, rice paddy, crop, grass, deciduous broadleaf, deciduous conifer, evergreen broadleaf, evergreen conifer. |

#### 3. Soil Type

| Attribute | Detail |
|---|---|
| **What it measures** | Geological/pedological classification of surface soil from MLIT's 1:200,000 geological map |
| **Physical relevance** | Soil mineralogy controls Cs-137 fixation. Clay-rich soils (especially those with illite and vermiculite) bind cesium tightly in mineral interlayer sites, reducing mobility and biotic uptake. Sandy soils allow greater vertical migration and potential re-mobilization. |
| **Source** | EMDB (serves MLIT 1:200k map) or SoilGrids (250m global) |
| **Portal (EMDB)** | https://emdb.jaea.go.jp/emdb/ |
| **Portal (SoilGrids)** | https://www.isric.org/explore/soilgrids |
| **Format** | CSV (via EMDB) or GeoTIFF (via SoilGrids) |
| **Resolution** | Variable (1:200,000 map scale via EMDB); 250m (SoilGrids) |
| **Notes** | EMDB provides categorical soil types. SoilGrids provides continuous properties (clay %, sand %, organic carbon). Consider using clay fraction from SoilGrids as a proxy for soil permeability — clay content inversely correlates with permeability and directly affects Cs retention. Both categorical and continuous representations are valid; choose based on which RF handles better during cross-validation. |

#### 4. Spatial Coordinates

| Attribute | Detail |
|---|---|
| **What it measures** | Geographic position of each monitoring location (easting/northing or latitude/longitude) |
| **Physical relevance** | Captures spatial autocorrelation and large-scale geographic gradients in deposition patterns (e.g., the NW plume direction from FDNPP). |
| **Source** | Included in EMDB radiation data downloads |
| **Format** | Embedded in CSV columns |
| **Notes** | Sun et al. used Japan Plane Rectangular Coordinate System (EPSG:2451 for Zone IX). Use the same CRS for consistency. Coordinates also serve as the basis for computing derived spatial predictors (distance-to-FDNPP, distance-to-rivers). |

---

### Extended Predictors (expanding beyond Sun et al. 2022)

#### 5. Elevation (DEM)

| Attribute | Detail |
|---|---|
| **What it measures** | Height above sea level (meters) |
| **Physical relevance** | Elevation influences precipitation patterns, runoff velocity, soil moisture retention, and vegetation type — all of which affect Cs-137 redistribution. Higher elevations in the Abukuma Highlands received different deposition patterns than coastal lowlands. Elevation also correlates with land-use transitions (forest at higher elevation, agriculture/urban at lower). |
| **Source** | EMDB (GSI 10m DEM) or GSI portal directly (5m DEM) |
| **Portal (EMDB)** | https://emdb.jaea.go.jp/emdb/ |
| **Portal (GSI)** | https://fgd.gsi.go.jp/download/menu.php (free account required) |
| **Format** | CSV (EMDB) or GeoTIFF (GSI) |
| **Resolution** | 10m (EMDB) or 5m (GSI direct) |
| **Notes** | 10m from EMDB is sufficient for this project. The 5m GSI DEM is overkill given the monitoring point spacing. Also used to derive slope and aspect. |

#### 6. Slope

| Attribute | Detail |
|---|---|
| **What it measures** | Steepness of terrain (degrees or percent) |
| **Physical relevance** | Controls surface runoff velocity. Steep slopes promote rapid washoff of particulate-bound Cs-137, leading to faster apparent dose-rate decay. Flat areas accumulate runoff-transported contamination in depressions. Slope also affects erosion potential and soil redistribution. |
| **Source** | Derived from DEM in R using `terra::terrain()` |
| **Format** | Computed raster |
| **Notes** | Derive after obtaining the DEM. Use degrees for interpretability in SHAP plots. |

#### 7. Aspect

| Attribute | Detail |
|---|---|
| **What it measures** | Compass direction a slope faces (0-360°) |
| **Physical relevance** | Affects solar exposure, evapotranspiration, snow melt timing, and wind-driven deposition. NW-facing slopes in the Fukushima region may have received different fallout loads due to the prevailing plume direction during the March 2011 accident. |
| **Source** | Derived from DEM in R using `terra::terrain()` |
| **Format** | Computed raster |
| **Notes** | Circular variable — consider transforming to sin(aspect) and cos(aspect) for the RF, or use as-is since RF can handle non-linear splits. |

#### 8. Distance from FDNPP

| Attribute | Detail |
|---|---|
| **What it measures** | Euclidean distance (km) from each monitoring point to the Fukushima Daiichi Nuclear Power Station |
| **Physical relevance** | Distance from the source is a first-order control on initial deposition intensity. However, the relationship is not monotonic due to the directional NW plume — some locations close to FDNPP received less fallout than locations further away in the plume direction. This predictor captures the radial component of the spatial deposition pattern. |
| **Source** | Computed from monitoring point coordinates and FDNPP reactor coordinates (141.032°E, 37.422°N) |
| **Format** | Computed numeric column |
| **Notes** | Use geodesic distance (Vincenty or Haversine) rather than Euclidean if coordinates are in lat/lon. If already projected to a planar CRS (e.g., EPSG:2451), Euclidean is fine. |

#### 9. Distance to Nearest River

| Attribute | Detail |
|---|---|
| **What it measures** | Distance (meters) from each monitoring point to the nearest river channel |
| **Physical relevance** | Rivers act as sinks and transport corridors for Cs-137. Locations near rivers may show faster dose-rate decay due to hydrological washoff of contaminated sediment into the channel. Floodplains can also accumulate redeposited contamination during flood events, creating anomalous hot spots. |
| **Source** | MERIT Hydro (global 90m river network) or MLIT river shapefiles |
| **Portal (MERIT Hydro)** | https://hydro.iis.u-tokyo.ac.jp/~yamadai/MERIT_Hydro/ |
| **Portal (MLIT)** | http://nlftp.mlit.go.jp/ksj-e/index.html |
| **Format** | Raster (MERIT) or Shapefile (MLIT) |
| **Resolution** | 90m (MERIT Hydro); vector (MLIT) |
| **Notes** | Compute distance using `sf::st_distance()` or `terra::distance()` after loading the river network. MERIT Hydro provides a global consistent product; MLIT provides official Japanese river data with higher local accuracy. |

#### 10. Initial Cs-137 Deposition

| Attribute | Detail |
|---|---|
| **What it measures** | Total Cs-137 deposited on the ground surface (kBq/m²) as of a reference date (July 2, 2011 for the Tsukuba map) |
| **Physical relevance** | The total inventory of Cs-137 deposited at each location is the fundamental source term. Unlike initial dose rate (which depends on measurement geometry and detector height), deposition represents the actual contamination burden. This predictor captures the spatial heterogeneity of the fallout plume. |
| **Source** | U. Tsukuba fallout map (250m mesh, decay-corrected to July 2, 2011) or EMDB airborne survey data |
| **Portal** | https://www.ied.tsukuba.ac.jp/~fukushimafallout/ |
| **Format** | CSV or Shapefile (250m mesh cells) |
| **Resolution** | 250m |
| **Notes** | The Tsukuba map was created by combining airborne survey, in-situ measurement, and soil sampling data. It is the most widely cited deposition reference map. Extract values at monitoring point locations using `terra::extract()`. Note the distinction from initial dose rate — deposition (kBq/m²) and dose rate (μSv/h) are related but not identical due to shielding factors, detector height, and surface geometry. Including both provides complementary information. |

#### 11. Precipitation

| Attribute | Detail |
|---|---|
| **What it measures** | Mean annual precipitation (mm/year) or monthly precipitation climatology |
| **Physical relevance** | Rainfall drives the primary mechanism of Cs-137 removal from impervious surfaces (washoff) and vertical migration in soils (percolation). High-precipitation areas on the eastern slopes of the Abukuma Highlands are expected to show faster dose-rate decay on urban/paved surfaces but potentially slower decay in forested soils (where rainfall drives Cs deeper into the organic layer without removing it). |
| **Source** | WorldClim 2.1 |
| **Portal** | https://www.worldclim.org/data/index.html |
| **Format** | GeoTIFF |
| **Resolution** | ~1km (30 arc-seconds) |
| **Notes** | Download the "bio" variables (BIO12 = annual precipitation) or monthly precipitation. 1km resolution is coarse relative to other predictors but sufficient for capturing regional precipitation gradients across the study area. Extract values at monitoring points. |

#### 12. NDVI (Vegetation Density)

| Attribute | Detail |
|---|---|
| **What it measures** | Normalized Difference Vegetation Index — a measure of vegetation greenness and density (range: -1 to +1, typically 0.2-0.8 for vegetated areas) |
| **Physical relevance** | Dense vegetation canopies intercepted fallout during the initial deposition event and continue to cycle Cs-137 through leaf litter. Forest canopy NDVI correlates with organic layer thickness and Cs retention capacity. NDVI also captures defoliation/remediation effects — areas where vegetation was cleared for decontamination will show NDVI drops. |
| **Source** | MODIS MOD13Q1 (250m, 16-day composites) or Sentinel-2 (10m) |
| **Portal (MODIS)** | https://lpdaac.usgs.gov/products/mod13q1v006/ |
| **Format** | GeoTIFF (HDF for MODIS raw) |
| **Resolution** | 250m (MODIS) or 10m (Sentinel-2) |
| **Notes** | MODIS 250m is recommended for simplicity and temporal consistency. Use a summer composite (July-August) to capture peak vegetation. Consider using an average across multiple years (e.g., 2012-2020) to smooth interannual variability. An optional free EarthData account may be needed for bulk downloads. |

---

### Predictor Summary Table for RF

| # | Predictor | Type | Physical Role in Cs-137 Dynamics | Source |
|---|---|---|---|---|
| 1 | Initial dose rate | Continuous | Source term magnitude | EMDB |
| 2 | Land-use type | Categorical (9 classes) | Controls ecological half-life via surface properties | EMDB (JAXA) |
| 3 | Soil type | Categorical / Continuous | Cs-137 fixation capacity and vertical migration | EMDB (MLIT) / SoilGrids |
| 4 | Easting, Northing | Continuous | Spatial autocorrelation, plume geometry | EMDB |
| 5 | Elevation | Continuous | Precipitation, runoff, vegetation gradients | EMDB (GSI) |
| 6 | Slope | Continuous | Runoff velocity, erosion potential | Derived from DEM |
| 7 | Aspect | Circular / Continuous | Solar exposure, plume-facing orientation | Derived from DEM |
| 8 | Distance from FDNPP | Continuous | Radial deposition gradient | Computed |
| 9 | Distance to river | Continuous | Hydrological washoff and redeposition | MERIT Hydro / MLIT |
| 10 | Initial Cs-137 deposition | Continuous | Total fallout inventory (kBq/m²) | U. Tsukuba |
| 11 | Precipitation | Continuous | Washoff and percolation driver | WorldClim |
| 12 | NDVI | Continuous | Canopy interception, organic layer cycling | MODIS |

---

## Overall Conclusion

| Task | Question | Decision |
|---|---|---|
| 1. Data Fusion | Fuse Safecast + JAEA? | **No.** Use JAEA alone — standard practice, avoids correction burden. |
| 2. Pipeline Novelty | Is SOM → RF novel and feasible? | **Yes.** 4 novelty claims. Both stages needed. Feasible in 2-3 months. |
| 3. Auxiliary Data | Can we get RF predictor datasets? | **Yes.** All free, 100% coverage, 3-5 days prep. EMDB is a one-stop shop. |

**The project is scientifically justified, novel, and logistically feasible. Clear to proceed with implementation.**
