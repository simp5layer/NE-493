---
title: |
  \vspace{-2cm}
  **NE 493 — Special Topics in Radiation Protection**\
  \Large{Research Project Proposal: AI in Radiation Protection}\
  \large{Self-Organizing Maps & Random Forest in R}
author: |
  Department of Nuclear Engineering\
  King Abdulaziz University
date: "February 2026"
geometry: margin=1in
fontsize: 11pt
header-includes:
  - \usepackage{booktabs}
  - \usepackage{longtable}
  - \usepackage{array}
  - \usepackage{float}
  - \usepackage{graphicx}
  - \usepackage{fancyhdr}
  - \usepackage{xcolor}
  - \definecolor{kaugreen}{RGB}{0,100,60}
  - \pagestyle{fancy}
  - \fancyhead[L]{\small NE 493 — Project Proposal}
  - \fancyhead[R]{\small King Abdulaziz University}
  - \fancyfoot[C]{\thepage}
  - \renewcommand{\headrulewidth}{0.4pt}
  - \usepackage{hyperref}
  - \hypersetup{colorlinks=true, linkcolor=kaugreen, urlcolor=blue, citecolor=kaugreen}
---

\newpage
\tableofcontents
\newpage

# Introduction

This proposal presents three candidate research topics for the NE 493 course project. Each topic applies **Self-Organizing Maps (SOM)** and **Random Forest (RF)** in R to a radiation protection dataset, with the goal of producing a journal-quality scientific paper within an 8-week timeline (Weeks 5--12).

The three topics are:

| # | Topic | Dataset Source |
|---|-------|---------------|
| 1 | Fukushima Contamination Mapping | Safecast + JAEA |
| 2 | Environmental Radiation Monitoring | EPA RadNet |
| 3 | Quantitative Analysis of Gamma Radiation Detection | Mendeley (Univ. Stuttgart) |

Each topic is evaluated on **publishability**, **feasibility**, **novelty**, and **data accessibility**. The course instructor is requested to review and select which topic will be pursued for the remainder of the semester.

## Project Constraints

- **Methods**: Self-Organizing Maps (SOM) and Random Forest (RF) in R
- **R Packages**: `kohonen`, `randomForest`/`ranger`, `caret`, `ggplot2`, `Metrics`
- **Timeline**: ~8 weeks (Weeks 5--12 of the semester)
- **Deliverable**: Journal-quality scientific paper (70% of course grade)

\newpage

# Topic 1: Fukushima Contamination Mapping

## 1.1 Overview

This topic applies SOM and RF to map radioactive contamination in the Fukushima region following the 2011 Daiichi nuclear disaster, using citizen-science (Safecast) and official government (JAEA) monitoring data.

**Novelty Statement**: *"This study proposes a hybrid SOM-RF methodology for contamination zone delineation and dose rate prediction in the Fukushima exclusion zone using citizen-science (Safecast) and official (JAEA) monitoring data. SOM provides unsupervised, data-driven zone identification that complements official evacuation boundaries, while RF enables predictive mapping of contamination in areas with sparse measurements."*

## 1.2 Dataset Accessibility

### Primary Source 1: Safecast

| Field | Detail |
|-------|--------|
| **URL** | \url{https://safecast.org/data/} and \url{https://api.safecast.org} |
| **Registration** | None required |
| **License** | Creative Commons CC0 (public domain) |
| **Format** | CSV (bulk download) or JSON (API) |
| **Size** | 150+ million data points total; Fukushima subset: 5--20 million points |
| **Columns** | `captured_at`, `latitude`, `longitude`, `value` (CPM), `unit`, `device_id`, `location_name` |

Safecast is a citizen-science radiation monitoring project. Volunteers drive vehicles equipped with bGeigie Nano radiation detectors, recording GPS-tagged gamma dose rate measurements. CPM values are convertible to $\mu$Sv/h by dividing by ~334 (for Cs-137 energy).

### Primary Source 2: JAEA (Japan Atomic Energy Agency)

| Field | Detail |
|-------|--------|
| **URL** | \url{https://emdb.jaea.go.jp/emdb/en/} |
| **Registration** | Free account required (available in English) |
| **Format** | CSV download via web interface |
| **Content** | Airborne survey dose rates, soil deposition (Cs-134, Cs-137 in Bq/m$^2$), fixed monitoring posts, car-borne surveys |
| **Size** | ~10,000--50,000 points per survey campaign |

### Supplementary Data

- **Land use maps**: Japan National Land Numerical Information (\url{https://nlftp.mlit.go.jp/ksj-e/index.html})
- **Elevation**: SRTM digital elevation model from USGS
- **Distance from plant**: Calculated from coordinates

### Limitations

- Safecast citizen-science data has variable quality (indoor vs. outdoor, no standardized calibration)
- Spatial coverage is uneven (dense along roads, sparse in mountains)
- The sheer volume requires downsampling or spatial aggregation
- JAEA web interface is complex; some data in Japanese only

## 1.3 Preprocessing Pipeline

```
Step 1:  Download Safecast bulk CSV. Filter to Fukushima region:
         latitude 36.8-37.8, longitude 139.8-141.0
Step 2:  Download JAEA soil deposition data (2-3 survey campaigns)
Step 3:  Load in R using data.table::fread() for speed
Step 4:  Quality filtering:
         - Remove CPM = 0 or > 100,000 (unrealistic)
         - Remove poor GPS accuracy points
Step 5:  Spatial aggregation: 1km x 1km grid cells
         Compute mean, median, max CPM per cell
Step 6:  Convert CPM to dose rate (uSv/h)
Step 7:  Engineer spatial features per grid cell:
         - Distance from Fukushima Daiichi (km)
         - Elevation (m), bearing from plant
         - Land use type, measurement density
Step 8:  Merge Safecast grid with JAEA deposition data
Step 9:  Normalize features for SOM
```

## 1.4 ML Structure & Pipeline

### SOM Implementation

- **Input**: ~2,000--5,000 grid cells $\times$ 10--15 features
- **Grid**: 12$\times$12 to 15$\times$15 hexagonal
- **Reveals**: Contamination zones matching the real NW plume pattern, distance-dose relationships, land use effects on Cs-137 retention, temporal decay patterns
- **Key figure**: SOM clusters re-projected onto geographic coordinates, compared with official evacuation zones

### Random Forest Implementation

- **Primary task**: Regression --- predicting log$_{10}$(mean dose rate) per grid cell
- **Predictors**: distance\_km, elevation, bearing, land\_use, year, n\_measurements
- **Cross-validation**: Spatial block CV using `blockCV` R package (prevents spatial autocorrelation leakage)
- **Feature importance**: Distance from plant expected to dominate (>40%)

### SOM-RF Integration

1. **Phase 1 (SOM)**: Unsupervised discovery of contamination zones
2. **Phase 2 (RF)**: Supervised prediction for spatial interpolation
3. **Integration**: SOM cluster membership as additional RF predictor; or separate RF models per SOM cluster (local expert models)
4. **Validation**: Compare SOM zones with official evacuation boundaries

## 1.5 Expected Results

| Metric | Expected Value |
|--------|---------------|
| RF R$^2$ for log(dose\_rate) | 0.85--0.95 |
| Relative error | 15--30% (realistic for environmental data) |
| SOM clusters | 4--6 (matching plume, distance bands, land use) |
| SOM vs. official zone agreement | 70--85% |
| SOM+RF improvement over RF alone | +2--5% R$^2$ |
| Top RF feature | Distance from plant |

### Expected Figures (12)

1. Study area map with plant location and analysis extent
2. Safecast measurement density map
3. Observed dose rate contamination map (NW plume visible)
4. Workflow diagram
5. SOM U-matrix and training progress
6. SOM component planes (2$\times$3 grid)
7. **SOM cluster map vs. official evacuation zones (hero figure)**
8. RF predicted vs. actual scatter plot (log-scale)
9. RF-predicted contamination map (full study area)
10. Spatial residual map
11. RF variable importance plot
12. Temporal comparison maps (optional)

## 1.6 Effort Estimate

| Week | Task | Hours |
|------|------|-------|
| 1 | Literature review, download Safecast, set up JAEA account | 12--15 |
| 2 | Filter Safecast to Fukushima region, quality control, gridding | 15--18 |
| 3 | Download/integrate JAEA data, feature engineering, merge | 12--15 |
| 4 | EDA, contamination maps, SOM implementation | 12--14 |
| 5 | RF implementation, tuning, evaluation | 12--14 |
| 6 | Publication-quality figures, begin writing | 14--16 |
| 7 | Write full paper draft | 14--16 |
| 8 | Revise, finalize, presentation | 10--12 |
| **Total** | | **101--120** |

**Overall difficulty: Moderate to Hard**

## 1.7 Target Journals

| Journal | IF | Fit | Review Time |
|---------|----|-----|-------------|
| **J. Environmental Radioactivity** (Elsevier) | ~2.6 | Core Fukushima mapping journal | 6--10 weeks |
| **Science of the Total Environment** (Elsevier) | ~8.2 | Ambitious but achievable with strong execution | 8--14 weeks |
| **Environmental Pollution** (Elsevier) | ~7.6 | Radioactive contamination within scope | 6--12 weeks |

## 1.8 Strengths & Weaknesses

### Strengths

1. **Highest novelty** of all topics --- SOM-RF on combined Safecast + JAEA data is genuinely new
2. **Highest journal IF potential** --- Science of the Total Environment (IF ~8.2) is a realistic target
3. **Compelling visual output** --- Fukushima contamination maps are inherently striking
4. **Strong real-world significance** --- post-nuclear-accident assessment is critical to radiation protection
5. **Citizen science angle** adds methodological contribution

### Weaknesses

1. **Highest preprocessing burden** --- 150M+ Safecast points need filtering and gridding
2. **Spatial statistics complexity** --- must address spatial autocorrelation
3. **Data quality concerns** with citizen-science measurements
4. **R spatial packages learning curve** (`sf`, `terra`)
5. **Risk of scope creep** with such a rich dataset

### Mitigations

- Filter immediately to 80km radius bounding box around plant
- Use `blockCV` package for spatial cross-validation
- Dedicate a paper subsection to data quality assessment
- Use `sf` (Simple Features) which integrates with `ggplot2`
- Strictly define scope in Week 1; avoid temporal analysis unless ahead of schedule

## 1.9 Recommendation Score

| Criterion | Score (/10) | Notes |
|-----------|-------------|-------|
| Publishability | 9 | High novelty + high IF journals |
| Feasibility | 5 | Hardest preprocessing, spatial complexity |
| Novelty | 9 | Very few papers combine Safecast + JAEA + SOM + RF |
| Data Access | 7 | Safecast open but huge; JAEA requires navigation |
| **Overall** | **7.5** | Highest ceiling but also highest risk |

\newpage

# Topic 2: Environmental Radiation Monitoring (EPA RadNet)

## 2.1 Overview

This topic applies SOM and RF to the US Environmental Protection Agency's RadNet monitoring network for automated classification of environmental radiation levels and anomaly detection across ~130 stations nationwide.

**Novelty Statement**: *"This study develops a hybrid SOM-RF framework for automated classification of environmental radiation levels using EPA RadNet monitoring data. SOM provides unsupervised pattern discovery across the US monitoring network, while RF delivers supervised classification of radiation anomalies. The integration of SOM-derived cluster features into the RF model yields improved anomaly detection compared to either method alone."*

## 2.2 Dataset Accessibility

### Primary Source: EPA RadNet

| Field | Detail |
|-------|--------|
| **URL** | \url{https://www.epa.gov/radnet/radnet-csv-file-downloads} |
| **Registration** | None required --- fully open US government data |
| **Format** | CSV files, directly downloadable |
| **Size** | ~1.1 million rows/year (gamma); ~50,000--100,000 rows (air filters) |

**RadNet measures:**

- **Gamma gross count rate** --- hourly CPM readings from ~130 stations (primary dataset)
- **Air filter samples** --- gross beta, I-131, Cs-137, Be-7, K-40
- **Precipitation samples** --- tritium, gamma-emitting radionuclides
- **Drinking water samples** --- gross alpha/beta, tritium, Ra-226/228, Sr-90, uranium
- **Milk samples** --- I-131, Cs-137, Sr-89/90

**CSV columns**: `[Location, Date/Time, Gamma Gross Count Rate (CPM), Cosmic/Terrestrial components]`

**Recommended subset**: 3--5 years (2011--2015 captures Fukushima fallout in US) from ~50 stations $\rightarrow$ ~200,000--500,000 rows (gamma) or ~10,000--20,000 rows (air filters).

### Supplementary Data (Optional)

- **NOAA weather data**: Temperature, precipitation, wind --- explains radon washout and temperature inversions (\url{https://www.ncei.noaa.gov/})
- **Station metadata**: Latitude, longitude, elevation, urban/rural classification

### Limitations

- Some stations have data gaps; older data (pre-2005) has more missing values
- Real-time gamma data can be noisy
- Multiple files (one per state/year) must be downloaded and combined

## 2.3 Preprocessing Pipeline

```
Step 1:  Download gamma CSV files by state/year from EPA
Step 2:  Download air filter CSVs for same stations/period
Step 3:  Load with read.csv(), combine with dplyr::bind_rows()
Step 4:  Parse date/time with lubridate
Step 5:  Handle missing values:
         - Flag CPM = 0 or negative as NA
         - Below-MDC air filter values -> MDC/2 (standard practice)
Step 6:  Aggregate hourly -> daily or weekly means
         (reduces size by 24x-168x and noise)
Step 7:  Engineer features:
         - Temporal: month, season, day_of_year, year
         - Geographic: latitude, longitude, elevation, region
         - Statistical: rolling_mean_7day, rolling_sd_7day,
           daily_max, daily_min, daily_range
Step 8:  Merge gamma with air filter data by station + date
Step 9:  Optionally merge with NOAA weather data
Step 10: Normalize for SOM input
```

## 2.4 ML Structure & Pipeline

### Station-Month Aggregation

Each row = one station in one month. This yields up to **7,800 rows** (130 stations $\times$ 12 months $\times$ 5 years).

**Predictors:**

- Geographic: latitude, longitude, elevation
- Temporal: month, season
- Statistical: mean\_CPM, median\_CPM, sd\_CPM, max\_CPM, min\_CPM, range\_CPM, skewness, kurtosis
- Weather (optional): mean\_temp, total\_precipitation, mean\_wind\_speed
- Air filter: mean\_gross\_beta, mean\_Be7, mean\_K40

**Target variable (classification):**

- **Normal**: CPM within mean $\pm$ 1 SD of station baseline
- **Elevated**: CPM > mean + 2 SD
- **Anomalous**: CPM > mean + 3 SD

### SOM Implementation

- **Grid**: 15$\times$15 to 20$\times$20 hexagonal (or 8$\times$8 for station-only analysis)
- **Reveals**: Geographic clustering of radiation environments, seasonal patterns, anomaly detection (Fukushima fallout at West Coast stations in March--April 2011)
- **Key visualization**: SOM cluster assignments on a US geographic map

```r
library(kohonen)
som_grid  <- somgrid(xdim = 12, ydim = 12, topo = "hexagonal")
som_model <- som(som_matrix, grid = som_grid, rlen = 1500,
                 alpha = c(0.05, 0.01))
```

### Random Forest Implementation

- **Task**: Classification (Normal / Elevated / Anomalous)
- **Class imbalance handling**: `classwt` parameter or SMOTE oversampling
- **Cross-validation**: 10-fold CV with `caret::trainControl()`; also `timeslice` CV for temporal integrity

```r
library(caret)
ctrl <- trainControl(method = "cv", number = 10,
                     classProbs = TRUE,
                     summaryFunction = multiClassSummary)
rf_tune <- train(rad_class ~ latitude + longitude + elevation +
                   Month + mean_temp + precipitation,
                 data = train, method = "rf",
                 trControl = ctrl, ntree = 1000)
```

### SOM-RF Integration (Three-Model Comparison)

| Model | Description |
|-------|-------------|
| RF-only (baseline) | Standard RF on raw features |
| SOM+RF (hybrid) | RF with SOM cluster label as additional predictor |
| SOM-only | Labels assigned by SOM cluster membership alone |

The hybrid model is expected to outperform both individual approaches. The SOM cluster encodes non-linear feature combinations into a single categorical variable.

## 2.5 Expected Results

| Metric | Expected Value |
|--------|---------------|
| RF accuracy (Normal vs. Elevated) | 85--92% |
| F1-score for "Elevated" class | 0.75--0.88 |
| SOM clusters | 4--6 radiation environment types |
| Top RF features | Elevation, latitude, month |
| SOM+RF improvement over RF | +3--7% F1 |
| Fukushima signal (if 2011 included) | Distinct SOM cluster/outlier |

### Expected Figures (10)

1. US map of RadNet stations color-coded by mean gamma CPM
2. Time series from 5 representative stations (seasonality + March 2011 Fukushima spike)
3. SOM-RF workflow diagram
4. SOM U-matrix with station clustering
5. SOM component planes (2$\times$3 grid)
6. **US geographic map with SOM cluster assignments (hero figure)**
7. RF confusion matrix (heatmap)
8. RF feature importance plot
9. ROC curves per class
10. Comparison bar chart: RF vs. SOM-RF hybrid accuracy/F1

## 2.6 Effort Estimate

| Week | Task | Hours |
|------|------|-------|
| 1 | Literature review, download RadNet CSVs, initial inspection | 10--12 |
| 2 | Data cleaning, date parsing, hourly$\rightarrow$daily aggregation, features | 14--16 |
| 3 | Finalize preprocessing, EDA: time series, geographic patterns | 10--12 |
| 4 | SOM implementation, train on station-level features, visualize | 10--12 |
| 5 | RF classification, tuning, evaluation | 12--14 |
| 6 | Generate figures/tables, begin writing Intro + Methods | 12--15 |
| 7 | Write Results, Discussion, Conclusion | 12--15 |
| 8 | Revise, format, finalize paper + presentation | 8--10 |
| **Total** | | **89--106** |

**Overall difficulty: Moderate**

## 2.7 Target Journals

| Journal | IF | Fit | Review Time |
|---------|----|-----|-------------|
| **J. Environmental Radioactivity** (Elsevier) | ~2.6 | Top journal for environmental radiation + ML | 6--10 weeks |
| **Environmental Monitoring and Assessment** (Springer) | ~3.0 | Broad environmental monitoring scope | 8--12 weeks |
| **Radiation Protection Dosimetry** (OUP) | ~0.9 | Higher acceptance rate; good backup | 4--8 weeks |

## 2.8 Strengths & Weaknesses

### Strengths

1. **Real-world data** --- actual environmental measurements, not computed/synthetic
2. **High visual impact** --- SOM clusters on US geographic map
3. **Practical relevance** --- automated radiation anomaly detection is an operational need
4. **Direct CSV access** --- no web scraping or manual data generation
5. **Good novelty** --- SOM + RF on EPA RadNet has very few published precedents
6. **Interdisciplinary appeal** --- radiation protection + environmental science + data science

### Weaknesses

1. **Non-trivial preprocessing** --- missing values, sensor malfunctions, timezone issues
2. **Large dataset** --- millions of rows if not carefully subset (R memory issues possible)
3. **Class imbalance** --- anomalous readings are rare
4. **Feature engineering requires domain knowledge** --- radon washout, cosmic ray variation

### Mitigations

- Start with 10 stations, 2 years to build pipeline; then scale
- Aggregate hourly $\rightarrow$ daily immediately (24$\times$ reduction)
- Use binary classification (Normal vs. Elevated) + SMOTE + `classwt`
- Follow established features from the environmental radiation literature

## 2.9 Recommendation Score

| Criterion | Score (/10) | Notes |
|-----------|-------------|-------|
| Publishability | 8 | Real data + novel application |
| Feasibility | 7 | Moderate preprocessing, manageable with guidance |
| Novelty | 8 | SOM-RF on RadNet is rare in literature |
| Data Access | 9 | Direct CSV, no registration, multi-file assembly |
| **Overall** | **8.0** | Best balance of novelty and feasibility |

\newpage

# Topic 3: Quantitative Analysis of Gamma Radiation Detection

## 3.1 Overview

This topic applies SOM and RF to laboratory gamma radiation sensor measurements from a novel photodiode-based detection system developed at the University of Stuttgart. The dataset contains measurement results from an advanced multi-sensor gamma/NIR detection system designed for applications such as sentinel lymph node detection in nuclear medicine.

**Novelty Statement**: *"This study presents a novel dual-method machine learning framework combining Self-Organizing Maps and Random Forest for quantitative analysis of gamma radiation sensor data. SOM-derived topological features enhance RF prediction accuracy for radiation source characterization, while RF proximity matrices provide superior input for SOM visualization. To our knowledge, this is the first application of the SOM-RF hybrid framework to photodiode-based gamma radiation detection data."*

## 3.2 Dataset Accessibility

### Primary Source: Mendeley Data

| Field | Detail |
|-------|--------|
| **Title** | Gamma radiation source detector: Measurement results |
| **URL** | \url{https://data.mendeley.com/datasets/nbwjw5bnr2/1} |
| **DOI** | 10.17632/nbwjw5bnr2.1 |
| **Creator** | Thilo Mayer, Universitat Stuttgart |
| **Published** | October 22, 2024 (Version 1) |
| **License** | CC BY 4.0 (fully open, free with attribution) |
| **Registration** | Free Mendeley account required |
| **Categories** | Biomedical Signal Processing, Gamma Source, Biomedical Sensor |

### Sensor System Description

The detection system uses 4--6 **PIN photodiodes (Osram BPW34)** with transistor-based pre-amplifiers and two-stage operational amplifiers connected to an Arduino microcontroller.

| Specification | Value |
|---------------|-------|
| Detection efficiency | ~0.37--1.2% |
| Saturation threshold | Up to 1,479 gamma-quants/second |
| Sensor head diameter | 12 mm outer, 25 mm length |
| Localization accuracy | 3.64 $\pm$ 1.66 mm (2$\sigma$) at up to 6 cm |
| Validation method | NIR radiation as gamma substitute |

### Expected Data Content

Based on the sensor publications (Mayer et al. 2025, Sorysz et al. 2023):

- Photon count readings from multiple photodiodes (4--6 channels)
- Angular position / geometric configuration variables
- Distance measurements (source-to-sensor)
- NIR validation readings
- Signal amplitude and noise level measurements

### Associated Publications

1. Mayer et al. (2025) --- "Combined gamma radiation source detector and ultrasound imaging system: Proof of concept," *Computers in Biology and Medicine*, 193, 110378
2. Sorysz et al. (2023) --- "Novel and inexpensive gamma radiation sensor: initial concept and design," *Int. J. Computer Assisted Radiology and Surgery*
3. Behling et al. (2021) --- Miniature low-cost gamma-radiation sensor

### Limitations

- Dataset size unknown until download --- may be small (laboratory bench-top experiment)
- NIR proxy data rather than direct gamma measurements in some experiments
- Limited column documentation on the Mendeley page
- Single sensor system limits generalizability claims

## 3.3 Preprocessing Pipeline

```
Step 1:  Create Mendeley account, download dataset
Step 2:  Load in R: read.csv() or readxl::read_xlsx()
Step 3:  Inspect structure: str(), summary(), check for NAs
Step 4:  Feature engineering:
         - Signal-to-noise ratio (SNR)
         - Inter-sensor count ratios
         - Total count across all channels
Step 5:  Outlier detection (IQR method)
Step 6:  Normalization:
         - Min-max scaling to [0,1] for SOM
         - Z-score for RF consistency
Step 7:  80/20 stratified train/test split
```

## 3.4 ML Structure & Pipeline

### Feature Categories

| Category | Variables | Purpose |
|----------|-----------|---------|
| Raw sensor | count\_ch1, ..., count\_ch4 | Primary input signals |
| Geometric | distance, angle\_theta, angle\_phi | Source positioning |
| Derived | SNR, inter-sensor ratios, total\_count | Engineered features |
| Environmental | temperature, ambient\_noise (if available) | Confounders |

### SOM Implementation

- **Input**: N samples $\times$ 8--15 features
- **Grid**: Determined by heuristic: dim $\approx \sqrt{5 \cdot \sqrt{N}}$
- **Training**: 1000 iterations, learning rate decay 0.05 $\rightarrow$ 0.01

```r
library(kohonen)
som_grid  <- somgrid(xdim = grid_dim, ydim = grid_dim,
                     topo = "hexagonal")
som_model <- som(som_input, grid = som_grid, rlen = 1000,
                 alpha = c(0.05, 0.01), keep.data = TRUE)
```

**What SOM reveals:**

- Natural grouping structure in multi-sensor radiation measurements
- Topological relationships between measurement conditions (distance, angle, intensity)
- Anomalous readings mapped to peripheral SOM nodes
- Component planes showing individual sensor contribution patterns

### Random Forest Implementation

**Dual formulation:**

- **Regression**: Predict source distance or activity from sensor readings
- **Classification**: Classify measurement conditions (source present/absent, position bin)

```r
library(randomForest)
rf_model <- randomForest(distance ~ ., data = train_data,
                         ntree = 500, mtry = floor(sqrt(p)),
                         importance = TRUE)
```

**Hyperparameter tuning**: `mtry` (2 to p), `ntree` (100--1000), `nodesize` (1--10) via 10-fold CV with `caret`.

### SOM-RF Integration (Three Strategies)

**Strategy A --- SOM as Feature Enrichment:**

- Extract SOM cluster assignments, BMU coordinates, and quantization error as new RF features
- Retrain RF with augmented feature set

**Strategy B --- RF Proximity Matrix to SOM:**

- Extract RF proximity matrix (sample similarity)
- Train SOM on proximity-based distance space
- Produces discriminative visualization (Plonski \& Zaremba, 2014)

**Strategy C --- Dual-Pipeline Comparison:**

| Pipeline | Description |
|----------|-------------|
| Pipeline 1 | Raw $\rightarrow$ SOM clustering $\rightarrow$ RF with cluster labels |
| Pipeline 2 | Raw $\rightarrow$ RF $\rightarrow$ proximity matrix $\rightarrow$ SOM visualization |
| Pipeline 3 | Raw $\rightarrow$ RF alone (baseline) |

Comparative analysis demonstrates value of the hybrid approach.

## 3.5 Expected Results

| Metric | RF Baseline | SOM+RF Hybrid |
|--------|-------------|---------------|
| R$^2$ (regression) | 0.88--0.93 | 0.91--0.96 |
| RMSE (distance, mm) | 2.5--4.0 | 1.8--3.2 |
| MAE (distance, mm) | 1.8--3.0 | 1.3--2.5 |
| Classification accuracy | 85--92% | 88--95% |
| AUC-ROC | 0.90--0.95 | 0.93--0.97 |

### Expected Figures (12)

1. Sensor system schematic diagram
2. Correlation heatmap of all features
3. Distribution plots of sensor readings
4. SOM training convergence curve
5. SOM U-matrix showing cluster boundaries
6. SOM component planes (4-panel, one per sensor)
7. SOM cluster map with class overlay
8. RF variable importance (bar chart)
9. RF predicted vs. actual scatter plot
10. ROC curves (if classification)
11. **Comparison bar chart: RF vs. SOM+RF vs. SOM-proximity-RF (novelty figure)**
12. RF proximity-based SOM vs. Euclidean SOM (side-by-side)

### Expected Tables (5)

1. Dataset summary statistics
2. SOM grid parameter comparison (quantization error, topographic error)
3. RF hyperparameter tuning results
4. Comparative model performance (RF-only vs. SOM+RF)
5. Feature importance ranking

## 3.6 Effort Estimate

| Week | Task | Hours |
|------|------|-------|
| 1 | Download dataset, EDA, install R packages, inspect data structure | 10--12 |
| 2 | Literature review (10--15 papers), draft Introduction + Related Work | 12--15 |
| 3 | Data preprocessing, feature engineering, normalization, split | 8--10 |
| 4 | SOM implementation: grid search, training, U-matrix, component planes | 12--15 |
| 5 | RF implementation: tuning, feature importance, cross-validation | 12--15 |
| 6 | Hybrid SOM-RF pipeline, comparative analysis | 10--12 |
| 7 | Write Methodology, Results, Discussion; prepare all figures | 15--18 |
| 8 | Revise, format for journal, Abstract + Conclusion | 10--12 |
| **Total** | | **89--109** |

**Overall difficulty: Moderate (6.0/10)**

## 3.7 Target Journals

| Journal | IF | Fit | Review Time |
|---------|----|-----|-------------|
| **Radiation Detection Technology and Methods** (Springer) | ~4.2 | Directly covers radiation detection + data processing | 4--8 weeks |
| **Radiation Measurements** (Elsevier) | ~2.3 | Ionizing radiation detection and instrumentation | 6--12 weeks |
| **Sensors** (MDPI) | ~3.4 | Sensor technology + ML applications; fast review | 2--4 weeks |

## 3.8 Strengths & Weaknesses

### Strengths

1. **Open, licensed dataset** --- CC BY 4.0, zero legal barriers
2. **Clear novelty angle** --- SOM-RF hybrid for gamma sensor data is unpublished
3. **Clean lab data** --- controlled experimental conditions, minimal noise
4. **Manageable scope** --- bounded lab dataset, no big-data infrastructure
5. **Medical/clinical relevance** --- sentinel lymph node detection is high-impact
6. **Strong NE 493 alignment** --- radiation detection + quantitative analysis
7. **Multiple publishable outputs** --- SOM alone, RF alone, and hybrid each contribute

### Weaknesses

1. **Dataset size uncertainty** --- may be small (<500 rows); needs Week 1 verification
2. **NIR proxy** --- some measurements use NIR instead of direct gamma
3. **Limited column documentation** --- must reverse-engineer from publications
4. **Single sensor system** --- limits generalizability claims
5. **RF is not cutting-edge** --- but positions well as "interpretable ML"

### Mitigations

- Download and verify dataset in first 2--3 days of Week 1
- If too small: bootstrap resampling or k-fold CV with repeated runs
- Contact dataset creator (Thilo Mayer, Univ. Stuttgart) for documentation
- Frame as "sensor-agnostic ML framework" applicable to other detector systems
- Cite NIR-gamma equivalence validation from Mayer et al. (2025)

## 3.9 Recommendation Score

| Criterion | Score (/10) | Notes |
|-----------|-------------|-------|
| Publishability | 7 | Clear novelty; suitable journals exist |
| Feasibility | 7.5 | Clean lab data, well-supported R packages |
| Novelty | 6.5 | Application novelty moderate; methodological novelty limited |
| Data Access | 8 | CC BY 4.0 open download |
| **Overall** | **7.0** | Solid project with clear path to publication |

\newpage

# Comparative Summary

## Side-by-Side Comparison

| Criterion | Topic 1: Fukushima | Topic 2: RadNet | Topic 3: Gamma Detection |
|-----------|--------------------|-----------------|--------------------------|
| **Data access** | Moderate | Easy | Easy |
| **Preprocessing** | High | Moderate | Low--Medium |
| **Coding complexity** | Moderate--High | Moderate | Moderate |
| **Paper writing** | Moderate--High | Moderate | Moderate |
| **Expected accuracy** | R$^2$ 0.85--0.95 | Acc 85--92% | R$^2$ 0.88--0.96 |
| **Novelty** | High | Moderate--High | Moderate |
| **Journal IF potential** | 2.6--8.2 | 2.6--3.0 | 2.3--4.2 |
| **Risk of failure** | Moderate | Low--Moderate | Low |
| **Visual impact** | Very High | High | Moderate |
| **Timeline hours** | 101--120 | 89--106 | 89--109 |
| **Overall score** | **7.5/10** | **8.0/10** | **7.0/10** |

## Dataset Links Summary

| Topic | Primary Dataset | URL |
|-------|----------------|-----|
| **Fukushima** | Safecast Open Data | \url{https://safecast.org/data/} |
| | JAEA Environmental DB | \url{https://emdb.jaea.go.jp/emdb/en/} |
| **RadNet** | EPA RadNet CSV Downloads | \url{https://www.epa.gov/radnet/radnet-csv-file-downloads} |
| **Gamma Detection** | Mendeley Data (Mayer) | \url{https://data.mendeley.com/datasets/nbwjw5bnr2/1} |

## Recommendation

All three topics are viable for producing a journal-quality paper within the 8-week timeline using SOM and RF in R. Each offers a distinct risk-reward profile:

\begin{itemize}
\item \textbf{Topic 1 (Fukushima)} --- \textit{Highest ceiling, highest risk.} Best for demonstrating ambition and targeting high-impact journals (IF > 5). Requires strong R spatial computing skills and careful scope management. The visual output (contamination maps) is the most compelling of all three topics.

\item \textbf{Topic 2 (RadNet)} --- \textit{Best risk-reward balance.} Offers real-world data, strong novelty, reasonable preprocessing, and direct CSV access. The SOM clusters on a US geographic map make for an outstanding hero figure. This is the safest path to a strong, publishable paper.

\item \textbf{Topic 3 (Gamma Detection)} --- \textit{Most focused and manageable.} Clean laboratory data with the lowest preprocessing burden. Strong alignment with nuclear engineering and medical physics applications. The main risk is dataset size, which must be verified early.
\end{itemize}

The instructor is respectfully requested to review these three proposals and indicate which topic should be pursued for the remainder of the semester.

\vspace{1cm}

---

*Proposal prepared for NE 493 --- Special Topics in Radiation Protection*\
*King Abdulaziz University, Department of Nuclear Engineering*\
*February 2026*
