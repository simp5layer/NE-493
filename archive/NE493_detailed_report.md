# NE 493 -- Detailed Topic Analysis Report
## AI in Radiation Protection | King Abdulaziz University
## Department of Nuclear Engineering

**Student Constraint Summary:**
- Must use Self-Organizing Maps (SOM) and Random Forest (RF) in R
- Must produce a journal-quality scientific paper
- Timeline: ~8 weeks (Weeks 5-12 of the semester)
- Final paper is worth 70% of the course grade
- Tools: `kohonen`, `randomForest`/`ranger`, `caret`, `ggplot2` in R

---
---

# TOPIC A: Gamma-Ray Shielding Parameter Prediction (NIST XCOM + Phy-X/PSD)

## A.1 Dataset Accessibility

### Primary Source 1: NIST XCOM Photon Cross Sections Database
- **URL:** https://physics.nist.gov/PhysRefData/Xcom/html/xcom1.html
- **Registration:** None required. Fully open, public-domain US government data.
- **How it works:** XCOM is a web-based calculator. You select an element, compound, or mixture and an energy range, and it returns photon interaction cross-section data (coherent scattering, incoherent scattering, photoelectric absorption, pair production in nuclear field, pair production in electron field, and total attenuation with and without coherent scattering).
- **Output format:** HTML table displayed on-screen. You can copy-paste into a spreadsheet or use R's `rvest` package to scrape the tables. There is also a downloadable standalone program (XCOM version 3.1) for batch processing.
- **Data generation strategy:** Since XCOM is a calculator, you generate your own dataset by systematically querying it for different materials across a range of photon energies. A typical dataset would involve:
  - 30-80 different glass/alloy/concrete compositions (you define them by weight fractions)
  - 20-30 energy points per material (e.g., 0.015 to 15 MeV)
  - This yields 600-2400 rows, each with ~10 columns
- **Approximate size:** 800-2500 rows x 10-12 columns (small, manageable dataset)
- **Limitations:** Manual entry is tedious one material at a time through the web interface. Batch processing via the standalone program or scripted web queries is strongly recommended.

### Primary Source 2: Phy-X/PSD Online Calculator
- **URL:** https://phy-x.net/PSD
- **Registration:** Free account required (email registration). Access is immediate after registration.
- **How it works:** Phy-X/PSD is a more comprehensive online tool that calculates not only cross sections but also mass attenuation coefficients (MAC), linear attenuation coefficients (LAC), half-value layer (HVL), tenth-value layer (TVL), mean free path (MFP), effective atomic number (Zeff), effective electron density (Neff), energy absorption buildup factors (EABF), and exposure buildup factors (EBF).
- **Output format:** Results are provided in downloadable Excel (.xlsx) files. You can process multiple materials in a single session.
- **Data generation strategy:** Input 40-80 material compositions with their elemental weight fractions. Phy-X/PSD will output all shielding parameters simultaneously.
- **Approximate size:** 40-80 materials x 25 energies x 8-10 parameters = a comprehensive dataset of ~1000-2000 rows after reshaping
- **Limitations:** The online tool occasionally has server downtime. Download and save results immediately.

### Preprocessing Pipeline for R

```
Step 1: Define 50-80 material compositions (glass systems like
        TeO2-B2O3-ZnO with varying mol% fractions are very common
        in the shielding literature).
Step 2: Query XCOM for each composition across 20-30 energies.
        Use Phy-X/PSD for extended parameters (HVL, TVL, MFP, Zeff).
Step 3: Export all results to CSV files.
Step 4: In R, merge XCOM and Phy-X/PSD outputs by material ID and
        energy.
Step 5: Create a master dataframe with columns:
        [MaterialID, Composition_fractions (x1..xn), Energy,
         MAC_total, MAC_coherent, MAC_incoherent, MAC_photoelectric,
         MAC_pair, HVL, TVL, MFP, Zeff, Neff]
Step 6: Check for any missing values (there should be essentially
        none since these are computed values, not measurements).
Step 7: Normalize numerical features using min-max or z-score
        scaling for SOM input.
```

---

## A.2 Effort Estimate

### Week-by-Week Timeline

| Week | Task | Hours |
|------|------|-------|
| 1 | Literature review: read 8-10 papers on ML for shielding prediction. Define 50-70 glass/alloy compositions. | 10-12 |
| 2 | Generate data from XCOM and Phy-X/PSD. Export to CSV. Merge and clean in R. | 12-15 |
| 3 | Exploratory Data Analysis (EDA): correlation plots, histograms, scatter plots. Implement SOM. | 10-12 |
| 4 | Tune and finalize SOM. Implement Random Forest regression models. | 10-12 |
| 5 | Hyperparameter tuning for RF. Cross-validation. Generate all performance metrics. | 8-10 |
| 6 | Create publication-quality figures. Write Abstract, Introduction, Methods. | 12-15 |
| 7 | Write Results, Discussion, Conclusion. Format references. | 12-15 |
| 8 | Revise paper. Prepare presentation slides. Final submission. | 8-10 |

### Effort Breakdown
- **Data collection effort:** Moderate. XCOM/Phy-X are calculators, not download-and-go datasets. You must define compositions and query them systematically. Budget 1-2 full days.
- **Preprocessing complexity:** Low. Data is computed (not measured), so there are no missing values, no outliers, no sensor noise. The only work is merging files and reshaping.
- **Coding complexity in R:** Low-Moderate. Standard SOM and RF workflows. No exotic data wrangling.
- **Paper writing effort:** Moderate. The shielding literature has many published examples to use as templates.
- **Overall difficulty rating:** **Easy to Moderate**

---

## A.3 ML Structure & Pipeline

### Data Preparation

**Feature Selection:**
- Predictor variables (inputs): Elemental weight fractions of the material composition (e.g., wt% of Te, O, B, Zn, Bi, Pb, etc.), photon energy (MeV), density (g/cm3), and optionally molecular weight.
- Target variables (outputs for RF regression): MAC_total, HVL, TVL, MFP, Zeff, or Neff. You can build separate RF models for each target or focus on 2-3 key ones.

**Normalization:**
- Min-max scaling to [0, 1] for SOM input (SOM is distance-based, so normalization is critical).
- For RF, normalization is not strictly necessary but does not hurt.

**Train/Test Split:**
- 80/20 random split stratified by energy range.
- Alternatively, leave-one-material-out cross-validation (more rigorous: train on 49 materials, predict the 50th).

### SOM Implementation

**Input Matrix:**
- Rows = samples (each row is one material at one energy), e.g., 1500 rows
- Columns = features: [wt_fraction_1, ..., wt_fraction_n, Energy, density] plus target shielding parameters for visualization purposes
- Typically 8-15 columns depending on the number of compositional components

**Grid Size Recommendation:**
- For ~1500 samples, a 10x10 or 12x12 hexagonal grid is appropriate.
- Rule of thumb: grid nodes ~ 5 * sqrt(N), so for N=1500, about 194 nodes, meaning a ~14x14 grid is the upper bound. A 10x10 grid (100 nodes) is a safe, interpretable choice.

**What SOM Reveals:**
- Clustering of materials by their shielding similarity (materials with similar Zeff and MAC cluster together).
- Component planes: one heat map per variable. Comparing the MAC component plane with the Zeff component plane shows their spatial correlation on the SOM grid.
- Identification of which compositional features drive shielding performance (e.g., high-Bi samples cluster in a region with high MAC).
- Detection of energy-dependent regimes (low-energy photoelectric-dominated vs. high-energy pair-production-dominated regions appear as distinct SOM zones).

**Visualization:**
- SOM U-matrix (distance map showing cluster boundaries)
- Component planes for each feature and each target variable
- Cluster map with color-coded material groups
- Training progress (mean distance to closest unit vs. iteration)

**R Code Outline (kohonen package):**

```r
library(kohonen)

# Prepare scaled data matrix
som_data <- as.matrix(scale(df[, feature_columns]))

# Define SOM grid
som_grid <- somgrid(xdim = 10, ydim = 10, topo = "hexagonal")

# Train SOM
som_model <- som(som_data,
                 grid = som_grid,
                 rlen = 1000,          # training iterations
                 alpha = c(0.05, 0.01), # learning rate decay
                 keep.data = TRUE)

# Plot U-matrix
plot(som_model, type = "dist.neighbours",
     main = "U-Matrix: SOM Distance Map")

# Plot component planes
plot(som_model, type = "property",
     property = getCodes(som_model)[, "MAC_total"],
     main = "Component Plane: Total MAC")

# Cluster the SOM nodes
som_cluster <- cutree(hclust(dist(getCodes(som_model))), k = 4)
plot(som_model, type = "mapping", bgcol = rainbow(4)[som_cluster],
     main = "SOM Clusters")
```

### Random Forest Implementation

**Task Type:** Regression (predicting continuous shielding parameters).

**Target Variables (build one model per target):**
1. MAC_total (mass attenuation coefficient) -- the primary target
2. HVL (half-value layer)
3. Zeff (effective atomic number)

**Predictor Variables:**
- Elemental weight fractions (wt_1, wt_2, ..., wt_n)
- Photon energy (MeV)
- Material density (g/cm3)

**Hyperparameter Tuning Strategy:**
- `ntree`: Start with 500, test up to 1500. Typically 500-1000 is sufficient for this dataset size.
- `mtry`: Use `tuneRF()` function or `caret::train()` with method = "rf". For p predictors, try mtry = p/3 (regression default), then search p/4 to p/2.
- `nodesize`: Default of 5 for regression is usually fine. Test 3, 5, 10.
- `maxnodes`: Leave NULL (unlimited) unless overfitting is observed.

**Cross-Validation Approach:**
- OOB (Out-of-Bag) error: Built into Random Forest, provides an unbiased estimate without explicit CV.
- Additionally, perform 10-fold cross-validation using `caret::trainControl(method = "cv", number = 10)` for a more publishable evaluation.
- For extra rigor: Leave-One-Material-Out CV (group-based CV where each material is a fold).

**Feature Importance Analysis:**
- Use `importance()` function to extract %IncMSE (percent increase in MSE when variable is permuted) and IncNodePurity.
- Plot variable importance using `varImpPlot()`.
- Expect: Energy will be the most important feature, followed by the weight fraction of the heaviest element (e.g., Bi, Pb, Te).

**Performance Metrics:**
- R-squared (coefficient of determination): Expect R2 > 0.98 for MAC prediction
- RMSE (Root Mean Squared Error)
- MAE (Mean Absolute Error)
- MAPE (Mean Absolute Percentage Error)
- Scatter plot of predicted vs. actual values (should lie close to the 1:1 line)

**R Code Outline (randomForest package):**

```r
library(randomForest)
library(caret)

# Split data
set.seed(42)
train_idx <- createDataPartition(df$MAC_total, p = 0.8, list = FALSE)
train_data <- df[train_idx, ]
test_data  <- df[-train_idx, ]

# Tune mtry
tune_result <- tuneRF(train_data[, predictor_cols],
                       train_data$MAC_total,
                       stepFactor = 1.5,
                       improve = 0.01,
                       ntreeTry = 500)

best_mtry <- tune_result[which.min(tune_result[, 2]), 1]

# Train RF model
rf_model <- randomForest(MAC_total ~ .,
                          data = train_data[, c(predictor_cols, "MAC_total")],
                          ntree = 1000,
                          mtry = best_mtry,
                          importance = TRUE)

# Predictions
predictions <- predict(rf_model, test_data)

# Performance metrics
library(Metrics)
r2    <- cor(predictions, test_data$MAC_total)^2
rmse  <- rmse(test_data$MAC_total, predictions)
mae   <- mae(test_data$MAC_total, predictions)

cat("R-squared:", r2, "\nRMSE:", rmse, "\nMAE:", mae, "\n")

# Variable importance
varImpPlot(rf_model, main = "Variable Importance for MAC Prediction")

# Predicted vs Actual plot
library(ggplot2)
ggplot(data.frame(Actual = test_data$MAC_total, Predicted = predictions),
       aes(x = Actual, y = Predicted)) +
  geom_point(alpha = 0.5) +
  geom_abline(slope = 1, intercept = 0, color = "red", linewidth = 1) +
  labs(title = "RF Prediction: Actual vs Predicted MAC",
       x = "Actual MAC (cm2/g)", y = "Predicted MAC (cm2/g)") +
  theme_minimal()
```

### How SOM and RF Work Together (Novelty Angle)

This is the key to making the paper publishable. The SOM-RF integration can be framed in two ways:

**Approach 1 -- SOM for Exploratory Analysis, RF for Prediction:**
- SOM first clusters the materials into groups based on their compositional and shielding profiles. This reveals which material families exist and how they relate.
- RF then predicts shielding parameters for new, untested compositions. The SOM clusters can be used as an additional categorical feature in the RF model (i.e., cluster membership becomes a predictor), potentially improving accuracy.

**Approach 2 -- SOM-Guided Feature Engineering:**
- SOM component planes reveal correlations between input features and targets. If the SOM shows that certain compositional features are redundant (their component planes are nearly identical), you can reduce dimensionality before feeding into RF.
- This approach is more methodologically interesting for a paper.

**Novelty Statement for the Paper:**
"This study presents a hybrid SOM-RF framework for predicting gamma-ray shielding parameters of novel glass systems. SOM is employed for unsupervised clustering and feature analysis of material compositions, while RF provides high-accuracy regression predictions. The SOM-derived cluster labels are incorporated as auxiliary features in the RF model, demonstrating improved prediction accuracy compared to RF alone."

---

## A.4 Expected Results & Figures

### Expected Figures for the Paper

1. **Figure 1:** Schematic diagram of the SOM-RF workflow (data flow from XCOM/Phy-X to SOM to RF to predictions).
2. **Figure 2:** Correlation matrix (heatmap) of all input features and target variables.
3. **Figure 3:** SOM U-matrix showing cluster boundaries.
4. **Figure 4:** SOM component planes (2x3 grid) for key variables: Energy, Zeff, MAC_total, HVL, wt% of heaviest element, and density.
5. **Figure 5:** SOM cluster map with material types color-coded.
6. **Figure 6:** RF variable importance plot (bar chart, %IncMSE).
7. **Figure 7:** Predicted vs. Actual scatter plot for MAC_total (with R2, RMSE annotated).
8. **Figure 8:** Predicted vs. Actual for HVL and Zeff (multi-panel).
9. **Figure 9:** Comparison of RF performance with and without SOM cluster features (bar chart of R2 values).

### Expected Tables

1. **Table 1:** Material compositions (name, elemental weight fractions, density).
2. **Table 2:** Summary statistics of the dataset.
3. **Table 3:** RF hyperparameters used.
4. **Table 4:** Performance metrics (R2, RMSE, MAE, MAPE) for each target variable.
5. **Table 5:** Comparison with published XCOM/Phy-X values (validation).

### Expected Results
- RF R-squared for MAC_total: **0.990 - 0.999** (shielding data is very well-behaved for tree-based models)
- RF R-squared for HVL: **0.985 - 0.998**
- RF R-squared for Zeff: **0.980 - 0.995**
- SOM will clearly separate low-Z materials (light glasses) from high-Z materials (Bi/Pb-doped glasses)
- Energy will dominate as the top feature in RF importance
- Adding SOM cluster as an RF feature may improve RMSE by 2-8%

---

## A.5 Target Journals

### Journal 1: Radiation Physics and Chemistry (Elsevier)
- **Impact Factor:** ~2.8 (2024)
- **Why it fits:** This is the premier journal for computational radiation interaction studies. Papers using ML to predict shielding parameters are published here monthly. The XCOM + ML combination is a well-established topic in this journal.
- **Typical review timeline:** 4-8 weeks for first decision
- **Open access:** Optional gold OA available (~$2,900 USD). Standard subscription model is free to publish.
- **Note:** This is the single best-fit journal for this topic.

### Journal 2: Nuclear Engineering and Technology (Elsevier / Korean Nuclear Society)
- **Impact Factor:** ~2.7 (2024)
- **Why it fits:** Publishes nuclear engineering applications including shielding studies. Accepts AI/ML-based approaches. Slightly broader scope than Radiation Physics and Chemistry.
- **Typical review timeline:** 6-10 weeks
- **Open access:** Fully open access (no APC for authors).

### Journal 3: Annals of Nuclear Energy (Elsevier)
- **Impact Factor:** ~1.9 (2024)
- **Why it fits:** Covers nuclear engineering broadly, including shielding and radiation transport. ML-based prediction papers appear regularly.
- **Typical review timeline:** 6-12 weeks
- **Open access:** Optional gold OA (~$3,100 USD).

---

## A.6 Strengths & Weaknesses

### Strengths
1. **Extremely well-defined workflow.** Dozens of published papers follow the exact same pattern (material compositions from XCOM, ML prediction of shielding parameters). You have abundant templates to follow.
2. **No data access barriers.** XCOM is public domain. Phy-X/PSD requires only a free account.
3. **Clean data.** Since the data is computed (not measured), there are zero missing values, no noise, no outliers. Preprocessing is minimal.
4. **Very high expected accuracy.** RF models routinely achieve R2 > 0.99 on shielding data. Your results will look impressive.
5. **Short literature review.** The field is well-established; papers to cite are easy to find.
6. **Fastest path to completion.** This topic has the lowest risk of getting stuck at any stage.

### Weaknesses
1. **Low novelty.** This exact approach (ML + XCOM shielding) has been published dozens of times since 2020. Reviewers may question the contribution. This is the single biggest risk.
2. **Data is synthetic/computed, not experimental.** Some reviewers may argue that predicting XCOM outputs with ML is circular (you are fitting a model to the output of another model). Counter-argument: the ML model provides a fast surrogate that avoids repeated XCOM queries and can interpolate to new compositions.
3. **SOM role may feel forced.** The primary value is in RF prediction; SOM clustering of shielding data is somewhat superficial unless you build a compelling narrative around material discovery or feature engineering.

### Mitigation Strategies
- **For low novelty:** Focus on a specific material system that has not been studied (e.g., rare-earth-doped phosphate glasses, or high-entropy alloy shielding materials). The novelty shifts from the method to the material.
- **For the "synthetic data" criticism:** Include a validation section comparing RF predictions against independent experimental data from published papers. Even 5-10 experimental validation points add credibility.
- **For SOM integration:** Frame SOM as a materials informatics tool for discovering composition-property relationships, not just a clustering exercise.

---

## A.7 Recommendation Score

| Criterion | Score (out of 10) | Notes |
|-----------|-------------------|-------|
| Publishability | 7 | High if you pick a novel material system; lower if generic |
| Feasibility | 10 | Lowest risk of all three topics. Very straightforward. |
| Novelty | 4 | Many similar papers exist. Must differentiate through material choice. |
| Data Access | 10 | Public, free, unlimited, no barriers. |
| **Overall** | **7.5** | Safe choice. High floor, moderate ceiling. |

---
---

# TOPIC B: Environmental Radiation Monitoring (EPA RadNet)

## B.1 Dataset Accessibility

### Primary Source: EPA RadNet
- **URL:** https://www.epa.gov/radnet and https://www.epa.gov/radnet/radnet-csv-file-downloads
- **Registration:** None required. Fully open US government data.
- **Data format:** CSV files, directly downloadable.
- **What RadNet contains:** RadNet is the US Environmental Protection Agency's nationwide radiation monitoring network. It collects data from ~130 monitoring stations across the US, measuring:
  - **Gamma gross count rate** (near-real-time, hourly readings)
  - **Air filter samples** (gross beta activity, specific radionuclide concentrations including I-131, Cs-137, Be-7, K-40)
  - **Precipitation (rainwater) samples** (tritium, gamma-emitting radionuclides)
  - **Drinking water samples** (gross alpha, gross beta, tritium, Ra-226, Ra-228, Sr-90, uranium)
  - **Pasteurized milk samples** (I-131, Cs-137, Sr-89, Sr-90)
- **Available downloads:**
  - Gamma gross count rate data: CSV files organized by state and year. Each file has columns: [Location, Date/Time, Gamma Gross Count Rate (CPM), Cosmic/Terrestrial components]
  - Air filter data: CSV with columns: [Location, Collection Date, Analyte, Result, Units, MDC, Uncertainty]
  - Precipitation and drinking water: similar CSV format
- **Approximate size:**
  - Gamma data: Millions of rows (hourly data from ~130 stations over 20+ years). A single year of gamma data is ~1.1 million rows.
  - Air filter data: ~50,000-100,000 rows across all stations and years.
  - Recommended subset: Pick 3-5 years (e.g., 2011-2015 to capture Fukushima fallout in US) from 50 stations. This yields a manageable ~200,000-500,000 rows for gamma, or ~10,000-20,000 rows for air filters.
- **Limitations:** Some stations have data gaps. Older data (pre-2005) has more missing values. Real-time gamma data can be noisy.

### Supplementary Data (Optional but Strengthens the Paper)
- **NOAA weather data:** Temperature, precipitation, wind speed/direction from the same geographic locations. Available at https://www.ncei.noaa.gov/. Helps explain radiation variation due to radon washout, temperature inversions, etc.
- **US Census/Geographic data:** Latitude, longitude, elevation, urban/rural classification for each RadNet station. Available from EPA station metadata.

### Preprocessing Pipeline for R

```
Step 1:  Download gamma gross count rate CSV files for selected
         states and years from the EPA RadNet CSV download page.
Step 2:  Download air filter CSV files for the same stations/period.
Step 3:  In R, use read.csv() to load files. Combine multi-state
         files with rbind() or dplyr::bind_rows().
Step 4:  Parse date/time columns using lubridate package.
Step 5:  Handle missing values:
         - Gamma data: flag readings of 0 or negative as NA.
         - Air filter data: values below MDC (Minimum Detectable
           Concentration) can be replaced with MDC/2 (standard
           practice in health physics).
Step 6:  Aggregate gamma data from hourly to daily or weekly means
         (reduces dataset size and noise).
Step 7:  Engineer features:
         - Temporal: month, season, day_of_year, year
         - Geographic: latitude, longitude, elevation, region
         - Statistical: rolling_mean_7day, rolling_sd_7day,
           daily_max, daily_min, daily_range
Step 8:  Merge gamma data with air filter data by station and date.
Step 9:  Optionally merge with weather data.
Step 10: Normalize features for SOM input.
```

---

## B.2 Effort Estimate

### Week-by-Week Timeline

| Week | Task | Hours |
|------|------|-------|
| 1 | Literature review (ML in environmental radiation monitoring). Download RadNet CSV files. Initial inspection. | 10-12 |
| 2 | Data cleaning: handle missing values, parse dates, aggregate hourly to daily. Feature engineering. | 14-16 |
| 3 | Finalize preprocessing. EDA: time series plots, geographic distribution, seasonal patterns. | 10-12 |
| 4 | SOM implementation: train on station-level aggregated features. Visualize clusters. | 10-12 |
| 5 | RF implementation: classification (anomaly detection or region classification) or regression (predicting gamma rate). Tune and evaluate. | 12-14 |
| 6 | Generate all figures and tables. Begin writing Introduction and Methods. | 12-15 |
| 7 | Write Results, Discussion, Conclusion. | 12-15 |
| 8 | Revise, format, finalize paper and presentation. | 8-10 |

### Effort Breakdown
- **Data collection effort:** Low. Direct CSV download, no registration needed. However, you need to download multiple files (one per state/year) and combine them.
- **Preprocessing complexity:** Moderate-High. Real-world environmental data has missing values, sensor anomalies, timezone issues, and requires thoughtful aggregation and feature engineering.
- **Coding complexity in R:** Moderate. Time series handling, date parsing, geographic features, and the larger dataset size add complexity compared to Topic A.
- **Paper writing effort:** Moderate. Environmental monitoring is a well-studied area but the EPA RadNet + SOM/RF combination is less published than XCOM shielding, so the introduction requires more original framing.
- **Overall difficulty rating:** **Moderate**

---

## B.3 ML Structure & Pipeline

### Data Preparation

**Feature Selection (Station-Level Aggregated Dataset):**

For the most publishable approach, aggregate each station's data into a station-level profile (one row per station, or one row per station-month):

- **Predictor variables:**
  - Geographic: latitude, longitude, elevation
  - Temporal: month, season
  - Statistical summaries of gamma readings: mean_CPM, median_CPM, sd_CPM, max_CPM, min_CPM, range_CPM, skewness, kurtosis
  - Weather (if included): mean_temp, total_precipitation, mean_wind_speed
  - Air filter results: mean_gross_beta, mean_Be7, mean_K40

- **Target variable options:**
  - **For Classification:** Categorize stations as "Normal," "Elevated," or "Anomalous" based on gamma readings exceeding background thresholds (e.g., mean + 2*SD = Elevated, mean + 3*SD = Anomalous). This yields a 2- or 3-class problem.
  - **For Regression:** Predict the mean annual gamma gross count rate (CPM) from geographic and meteorological features. Useful for background estimation.

**Normalization:** Min-max scaling for SOM. Not required for RF but apply it for consistency.

**Train/Test Split:**
- 70/30 or 80/20 stratified split.
- For temporal prediction: train on 2011-2014, test on 2015 (time-based split prevents data leakage).

### SOM Implementation

**Input Matrix:**
- If using station-month profiles: ~130 stations x 12 months x 5 years = up to 7,800 rows
- Columns: 10-20 features (geographic, temporal, statistical, meteorological)

**Grid Size Recommendation:**
- For ~5,000-8,000 samples: a 15x15 to 20x20 hexagonal grid.
- For a simpler station-only analysis (~130 stations): a 8x8 or 10x10 grid.

**What SOM Reveals:**
- **Geographic clustering of radiation environments:** Stations in similar radiation backgrounds (e.g., high-altitude Rocky Mountain stations vs. coastal stations vs. Great Plains stations) will cluster together on the SOM.
- **Seasonal patterns:** Component planes for "month" and "mean_CPM" will show how radiation background varies seasonally (radon washout in spring, cosmic ray variation).
- **Anomaly detection:** Stations/periods that map to unusual SOM regions may correspond to elevated radiation events (e.g., Fukushima fallout detected at US West Coast stations in March-April 2011).
- **Component planes** reveal which features co-vary with elevated gamma readings.

**Visualization:**
- U-matrix with geographic labels
- Component planes for each feature
- Map overlay: plot SOM cluster assignments on a US map (very visually compelling)
- Time series of SOM cluster transitions for individual stations

**R Code Outline:**

```r
library(kohonen)
library(dplyr)

# Aggregate to station-month level
station_month <- radnet_data %>%
  group_by(StationID, Year, Month) %>%
  summarise(
    mean_CPM    = mean(GammaCPM, na.rm = TRUE),
    sd_CPM      = sd(GammaCPM, na.rm = TRUE),
    max_CPM     = max(GammaCPM, na.rm = TRUE),
    latitude    = first(Latitude),
    longitude   = first(Longitude),
    elevation   = first(Elevation),
    .groups = "drop"
  )

# Scale features
som_features <- c("mean_CPM", "sd_CPM", "max_CPM",
                   "latitude", "longitude", "elevation",
                   "Month")
som_matrix <- as.matrix(scale(station_month[, som_features]))

# Train SOM
som_grid  <- somgrid(xdim = 12, ydim = 12, topo = "hexagonal")
som_model <- som(som_matrix, grid = som_grid, rlen = 1500,
                 alpha = c(0.05, 0.01))

# U-matrix
plot(som_model, type = "dist.neighbours",
     main = "U-Matrix: RadNet Station Clustering")

# Component planes
for (feat in som_features) {
  plot(som_model, type = "property",
       property = getCodes(som_model)[, feat],
       main = paste("Component Plane:", feat))
}

# Hierarchical clustering of SOM nodes
som_hc <- cutree(hclust(dist(getCodes(som_model))), k = 5)
plot(som_model, type = "mapping", bgcol = viridis::viridis(5)[som_hc])
```

### Random Forest Implementation

**Task Type:** Both classification and regression are viable. Choose one primary approach:

- **Recommended: Classification** -- Classify each station-month as "Normal" vs. "Elevated" radiation. This is more interesting for radiation protection and easier to interpret.
- **Secondary option: Regression** -- Predict mean gamma CPM from geographic and temporal features.

**Classification Setup:**
- Target: radiation_category (Normal / Elevated / Anomalous), defined as:
  - Normal: CPM within mean +/- 1 SD of station historical baseline
  - Elevated: CPM > mean + 2 SD
  - Anomalous: CPM > mean + 3 SD
- Predictors: latitude, longitude, elevation, month, season, mean_temp, precipitation, wind_speed, recent_trend (slope of CPM over past 7 days)

**Hyperparameter Tuning:**
- `ntree`: 500-1500
- `mtry`: Tune via `caret::train()` with `tuneGrid = expand.grid(mtry = 2:8)`
- `nodesize`: Test 1, 3, 5 for classification
- Class imbalance handling: Use `classwt` parameter or SMOTE oversampling if the "Anomalous" class is rare (it will be)

**Cross-Validation:**
- 10-fold CV using `caret::trainControl(method = "cv", number = 10)`
- Also report OOB error from the RF model
- For time-series integrity: use `timeslice` CV in caret to avoid temporal leakage

**Feature Importance:**
- Mean Decrease in Gini (classification) or Mean Decrease in Accuracy
- Expect: elevation, latitude, and month to rank highest

**Performance Metrics (Classification):**
- Accuracy, Balanced Accuracy, Kappa
- Precision, Recall, F1-score per class
- Confusion matrix
- ROC curve and AUC (one-vs-rest for multi-class)

**R Code Outline:**

```r
library(randomForest)
library(caret)

# Define target variable
station_month$rad_class <- cut(station_month$mean_CPM,
  breaks = c(-Inf, baseline_mean + 2*baseline_sd,
             baseline_mean + 3*baseline_sd, Inf),
  labels = c("Normal", "Elevated", "Anomalous"))

# Train/test split
set.seed(42)
train_idx <- createDataPartition(station_month$rad_class,
                                  p = 0.8, list = FALSE)
train <- station_month[train_idx, ]
test  <- station_month[-train_idx, ]

# Train RF with caret for tuning
ctrl <- trainControl(method = "cv", number = 10,
                      classProbs = TRUE,
                      summaryFunction = multiClassSummary)

rf_tune <- train(rad_class ~ latitude + longitude + elevation +
                   Month + mean_temp + precipitation,
                 data = train,
                 method = "rf",
                 trControl = ctrl,
                 tuneGrid = expand.grid(mtry = 2:6),
                 ntree = 1000,
                 metric = "Accuracy")

# Evaluate on test set
pred <- predict(rf_tune, test)
confusionMatrix(pred, test$rad_class)

# Feature importance
varImpPlot(rf_tune$finalModel,
           main = "Feature Importance: RadNet Classification")
```

### How SOM and RF Work Together

**Integration Strategy -- SOM-Informed Anomaly Detection + RF Classification:**

1. **Phase 1 (SOM):** Train SOM on all station-month feature vectors. Identify which SOM nodes correspond to "normal background" and which to "anomalous" patterns. The SOM provides an unsupervised perspective: it finds clusters without being told what "elevated" means.

2. **Phase 2 (RF):** Use the SOM cluster label as an additional feature in the RF classification model. The SOM cluster encodes non-linear combinations of all features into a single categorical variable, enriching the RF feature space.

3. **Comparison Analysis:** Compare three models:
   - RF without SOM features (baseline)
   - RF with SOM cluster label as an extra feature (hybrid)
   - SOM-only classification (assign labels based on which SOM cluster each sample belongs to)

   Show that the hybrid SOM+RF model outperforms both individual approaches.

**Novelty Statement:**
"This study develops a hybrid SOM-RF framework for automated classification of environmental radiation levels using EPA RadNet monitoring data. SOM provides unsupervised pattern discovery across the US monitoring network, while RF delivers supervised classification of radiation anomalies. The integration of SOM-derived cluster features into the RF model yields improved anomaly detection compared to either method alone."

---

## B.4 Expected Results & Figures

### Expected Figures

1. **Figure 1:** Map of EPA RadNet monitoring stations across the US, color-coded by mean gamma CPM.
2. **Figure 2:** Time series of gamma readings from 5 representative stations (showing seasonal patterns and the March 2011 Fukushima signal if 2011 data is included).
3. **Figure 3:** Workflow diagram of the SOM-RF pipeline.
4. **Figure 4:** SOM U-matrix with station clustering.
5. **Figure 5:** SOM component planes (2x3 or 3x3 grid) for latitude, longitude, elevation, month, mean_CPM, sd_CPM.
6. **Figure 6:** Geographic map of the US with stations colored by their SOM cluster assignment (this is the "hero figure" of the paper -- very visually impactful).
7. **Figure 7:** RF confusion matrix (heatmap style).
8. **Figure 8:** RF feature importance plot.
9. **Figure 9:** ROC curves for each class (Normal/Elevated/Anomalous).
10. **Figure 10:** Comparison bar chart: RF vs. SOM-RF hybrid accuracy/F1.

### Expected Tables

1. **Table 1:** Summary of RadNet dataset (stations, time period, number of records).
2. **Table 2:** Feature descriptions and summary statistics.
3. **Table 3:** Class distribution (Normal/Elevated/Anomalous counts).
4. **Table 4:** RF hyperparameters and tuning results.
5. **Table 5:** Classification performance metrics (accuracy, precision, recall, F1, AUC) for all three models (SOM-only, RF-only, SOM+RF).

### Expected Results
- RF classification accuracy for Normal vs. Elevated: **85-92%**
- F1-score for the "Elevated" class: **0.75-0.88** (lower due to class imbalance)
- SOM will reveal 4-6 distinct radiation environment clusters corresponding to geographic/geologic regions
- If 2011 data is included, the Fukushima period will appear as a distinct cluster or outlier on the SOM
- Elevation and latitude will be the top RF features (cosmic ray contribution increases with altitude; natural background varies with geology)
- SOM+RF hybrid expected to improve F1 by 3-7% over RF alone

---

## B.5 Target Journals

### Journal 1: Journal of Environmental Radioactivity (Elsevier)
- **Impact Factor:** ~2.6 (2024)
- **Why it fits:** This is the top journal specifically for environmental radiation studies. RadNet data analysis and ML-based environmental monitoring papers are directly within scope.
- **Typical review timeline:** 6-10 weeks for first decision.
- **Open access:** Optional gold OA (~$3,300 USD). Standard is free to publish.

### Journal 2: Environmental Monitoring and Assessment (Springer)
- **Impact Factor:** ~3.0 (2024)
- **Why it fits:** Covers environmental monitoring broadly, including radiological monitoring. ML-based approaches to environmental data analysis are welcome.
- **Typical review timeline:** 8-12 weeks.
- **Open access:** Optional gold OA available.

### Journal 3: Radiation Protection Dosimetry (Oxford University Press)
- **Impact Factor:** ~0.9 (2024)
- **Why it fits:** Covers radiation protection including environmental monitoring. Lower IF but higher acceptance rate, making it a good backup option.
- **Typical review timeline:** 4-8 weeks.
- **Open access:** Optional gold OA.

---

## B.6 Strengths & Weaknesses

### Strengths
1. **Real-world data.** Unlike Topic A, this uses actual environmental measurements, which is more scientifically meaningful and interesting to reviewers.
2. **High visual impact.** Geographic maps with SOM clusters overlaid on the US map are compelling and publication-worthy.
3. **Practical relevance.** Automated environmental radiation anomaly detection is a real operational need. The paper has clear practical applications.
4. **Direct CSV access.** No web scraping or manual data generation required. Data is already in analysis-ready CSV format.
5. **Good novelty.** SOM + RF applied to EPA RadNet data has very few (if any) published precedents. This is a genuinely new application.
6. **Interdisciplinary appeal.** Touches radiation protection, environmental science, and data science. Broader potential readership.

### Weaknesses
1. **Preprocessing is non-trivial.** Real-world data requires careful cleaning: missing values, sensor malfunctions, timezone handling, data gaps. This is where an undergraduate could get stuck.
2. **Large dataset size.** If not carefully subset, the gamma data can be very large (millions of rows), causing R memory issues on a typical laptop. Must aggregate strategically.
3. **Class imbalance.** Anomalous readings are rare by definition. The "Anomalous" class may have very few samples, making classification challenging. Requires SMOTE or weighted classes.
4. **Feature engineering requires domain knowledge.** Choosing meaningful aggregation windows and features requires understanding of radiation physics (radon washout, cosmic ray variation, etc.).

### Mitigation Strategies
- **For preprocessing complexity:** Start with a small subset (e.g., 10 stations, 2 years) to develop the pipeline, then scale up once the code works.
- **For large data:** Aggregate hourly data to daily or weekly means immediately upon loading. This reduces data volume by 24x or 168x.
- **For class imbalance:** Use binary classification (Normal vs. Elevated) instead of three classes. Apply SMOTE from the `smotefamily` R package. Use `classwt` in randomForest.
- **For feature engineering:** Follow published approaches in the environmental radiation literature. The suggested features above are well-established.

---

## B.7 Recommendation Score

| Criterion | Score (out of 10) | Notes |
|-----------|-------------------|-------|
| Publishability | 8 | Real data + novel application = strong paper if executed well |
| Feasibility | 7 | Moderate preprocessing challenge, but manageable with guidance |
| Novelty | 8 | SOM-RF on EPA RadNet is rare in the literature |
| Data Access | 9 | Direct CSV download, no registration, but multi-file assembly required |
| **Overall** | **8.0** | Best balance of novelty and feasibility. Recommended choice. |

---
---

# TOPIC C: Fukushima Contamination Mapping (Safecast + JAEA)

## C.1 Dataset Accessibility

### Primary Source 1: Safecast
- **URL:** https://safecast.org/data/ and the bulk data API at https://api.safecast.org
- **Registration:** None required for data download. Fully open data under Creative Commons CC0 license.
- **Data format:** CSV (bulk download) or JSON (via API).
- **Bulk download:** Available at https://api.safecast.org/system/bgeigie_imports.csv (full dataset) or as filtered exports.
- **What Safecast contains:** Safecast is a citizen-science radiation monitoring project started after the 2011 Fukushima disaster. Volunteers drive vehicles equipped with bGeigie Nano radiation detectors, recording GPS-tagged gamma dose rate measurements.
  - Columns: [captured_at (timestamp), latitude, longitude, value (CPM), unit, device_id, location_name]
  - CPM (counts per minute) from pancake-type GM detectors (LND 7317), convertible to uSv/h by dividing by ~334 (for Cs-137 energy)
- **Approximate size:** This is a massive dataset: **150+ million data points** as of 2025. The Japan subset alone contains ~100 million points. A Fukushima-prefecture subset contains ~5-20 million points depending on the geographic bounding box.
- **Limitations:**
  - Citizen-science data has variable quality: some readings are from inside vehicles (shielded), some from buildings, some from open air.
  - No standardized detector calibration across all devices.
  - Spatial coverage is uneven (dense along roads, sparse in mountains/forests).
  - The sheer volume requires downsampling or spatial aggregation.

### Primary Source 2: JAEA (Japan Atomic Energy Agency) Monitoring Data
- **URL:** https://emdb.jaea.go.jp/emdb/en/ (Environmental Monitoring Database)
- **Registration:** Free account required. Available in English.
- **Data format:** CSV download after selecting parameters through a web interface.
- **What JAEA contains:** Official government monitoring data including:
  - Airborne monitoring survey results (air dose rates mapped from helicopter surveys)
  - Soil deposition maps (Cs-134, Cs-137 in Bq/m2 and Bq/kg)
  - Ambient dose rate at fixed monitoring posts
  - Car-borne survey results
- **Approximate size:** Each survey campaign produces ~10,000-50,000 data points for the Fukushima region. Multiple campaigns from 2011 to present.
- **Limitations:**
  - The web interface is somewhat complex to navigate.
  - Data is organized by survey campaign, requiring manual selection and download of each campaign.
  - Some datasets are only available in Japanese despite the English interface.

### Supplementary Data
- **Land use / land cover maps:** From Japan's National Land Numerical Information (https://nlftp.mlit.go.jp/ksj-e/index.html). Used to correlate contamination with land type (forest, urban, agricultural).
- **Elevation data:** SRTM digital elevation model, available from USGS.
- **Distance from Fukushima Daiichi:** Calculated from coordinates.

### Preprocessing Pipeline for R

```
Step 1:  Download Safecast bulk CSV. This file is very large
         (several GB). Use command-line tools (awk, grep) or R's
         data.table::fread() to filter to Fukushima region only:
         latitude 36.8-37.8, longitude 139.8-141.0.
Step 2:  Download JAEA soil deposition data for Cs-137 from the
         EMDB interface. Select 2-3 survey campaigns (e.g., 2011,
         2013, 2017) for temporal comparison.
Step 3:  In R, load the Safecast subset using data.table::fread()
         for speed.
Step 4:  Quality filtering for Safecast data:
         - Remove points with CPM = 0 or unrealistically high
           values (> 100,000 CPM)
         - Remove points with GPS accuracy > 100m (if available)
         - Remove indoor measurements (if flagged)
Step 5:  Spatial aggregation: Grid the study area into 1km x 1km
         cells. Compute mean, median, max CPM per grid cell.
         This reduces millions of points to ~1,000-5,000 grid cells.
Step 6:  Convert CPM to dose rate (uSv/h) using the standard
         conversion factor.
Step 7:  Engineer spatial features for each grid cell:
         - Distance from Fukushima Daiichi plant (km)
         - Elevation (m)
         - Bearing/direction from the plant
         - Land use type (if available)
         - Number of Safecast measurements in the cell (data density)
Step 8:  Merge Safecast grid data with JAEA deposition data
         (spatially join by nearest grid cell or same cell).
Step 9:  For temporal analysis: create separate grids for different
         time periods (e.g., 2011-2012, 2013-2014, 2015-2017) to
         study contamination decay.
Step 10: Normalize features for SOM.
```

---

## C.2 Effort Estimate

### Week-by-Week Timeline

| Week | Task | Hours |
|------|------|-------|
| 1 | Literature review on Fukushima contamination mapping and ML approaches. Download Safecast data. Set up JAEA account and begin data retrieval. | 12-15 |
| 2 | Filter Safecast data to Fukushima region. Quality control. Spatial gridding. This is the most labor-intensive data step. | 15-18 |
| 3 | Download and integrate JAEA data. Feature engineering (distance, elevation, land use). Merge datasets. | 12-15 |
| 4 | EDA: contamination maps, histograms, spatial plots. SOM implementation. | 12-14 |
| 5 | RF implementation (regression for dose rate prediction or classification for contamination zones). Tune and evaluate. | 12-14 |
| 6 | Generate publication-quality figures (maps are critical). Begin writing. | 14-16 |
| 7 | Write full paper draft. | 14-16 |
| 8 | Revise, finalize paper and presentation. | 10-12 |

### Effort Breakdown
- **Data collection effort:** High. Safecast bulk download is large and requires significant filtering. JAEA interface requires manual navigation and multiple downloads. Budget 2-3 days just for data assembly.
- **Preprocessing complexity:** High. Citizen-science data quality control, spatial gridding, coordinate system handling, merging heterogeneous data sources. This is the hardest preprocessing of all three topics.
- **Coding complexity in R:** Moderate-High. Spatial data handling requires packages like `sf`, `terra`, or `sp` in addition to the ML packages. Map plotting requires `ggplot2` + `sf` or `leaflet`.
- **Paper writing effort:** Moderate. Fukushima is a well-studied topic with abundant references, but the spatial analysis adds complexity to the methods description.
- **Overall difficulty rating:** **Moderate to Hard**

---

## C.3 ML Structure & Pipeline

### Data Preparation

**Feature Selection (Grid-Cell-Level Dataset):**
Each row represents one 1km x 1km grid cell in the Fukushima region.

- **Predictor variables:**
  - Geographic: latitude, longitude, distance_from_plant (km), bearing_from_plant (degrees), elevation (m)
  - Topographic: slope, aspect (if DEM data is used)
  - Land use: categorical (forest, urban, agriculture, water) -- one-hot encoded
  - Data quality: n_measurements (number of Safecast points in cell), measurement_sd (variability within cell)
  - Temporal: year or time_since_accident (days)

- **Target variables:**
  - **For Regression:** Mean dose rate (uSv/h) in each grid cell, or Cs-137 soil deposition (Bq/m2) from JAEA data.
  - **For Classification:** Contamination zone category based on Japanese government delineation:
    - Zone 1 (Difficult-to-Return): > 50 mSv/year
    - Zone 2 (Restricted Residence): 20-50 mSv/year
    - Zone 3 (Evacuation Order Lifted): < 20 mSv/year
    - Normal: < 1 mSv/year above background

**Normalization:** Min-max or log-transformation (radiation data is often log-normally distributed, so log-transforming the target variable before RF regression improves performance).

**Train/Test Split:**
- **Spatial split (recommended):** Divide the study area into training and test regions (e.g., train on western half, test on eastern half). This prevents spatial autocorrelation leakage and is more realistic.
- **Alternatively:** Random 80/20 split, but report spatial autocorrelation diagnostics.

### SOM Implementation

**Input Matrix:**
- Rows = grid cells (~2,000-5,000 depending on area and resolution)
- Columns = all features + dose rate: ~10-15 columns

**Grid Size Recommendation:**
- For ~3,000 grid cells: 12x12 to 15x15 hexagonal grid.
- Larger grid captures more detail but takes longer to interpret.

**What SOM Reveals:**
- **Contamination zones:** SOM will naturally cluster grid cells into zones that correspond to the real contamination plume pattern (northwest corridor from the plant, which is well-documented).
- **Distance-dose relationship:** Component planes for distance and dose rate will show strong inverse correlation.
- **Land use effects:** Forest areas retain Cs-137 longer than urban areas (well-known in Fukushima literature). SOM component planes can reveal this.
- **Temporal decay patterns:** If multiple time periods are included, the SOM can show how contamination patterns evolve.
- **Comparison with official zones:** SOM-derived clusters can be overlaid on the official evacuation zone boundaries to validate the unsupervised approach.

**Visualization:**
- U-matrix with geographic interpretation
- Component planes as a multi-panel figure
- SOM cluster map re-projected onto geographic coordinates (the key figure)
- Comparison map: SOM clusters vs. official evacuation zones

**R Code Outline:**

```r
library(kohonen)
library(sf)
library(ggplot2)
library(viridis)

# Prepare grid-cell data (already aggregated)
som_features <- c("distance_km", "elevation", "bearing",
                   "mean_dose_rate", "land_forest", "land_urban",
                   "latitude", "longitude")

som_matrix <- as.matrix(scale(grid_data[, som_features]))

# Train SOM
som_grid  <- somgrid(xdim = 12, ydim = 12, topo = "hexagonal")
som_model <- som(som_matrix, grid = som_grid, rlen = 2000,
                 alpha = c(0.05, 0.01))

# Cluster SOM nodes into contamination zones
n_clusters <- 5  # match approximate number of contamination levels
som_hc <- cutree(hclust(dist(getCodes(som_model))), k = n_clusters)

# Assign each grid cell to a cluster
grid_data$som_cluster <- som_hc[som_model$unit.classif]

# Plot on map
ggplot(grid_data) +
  geom_tile(aes(x = longitude, y = latitude,
                fill = factor(som_cluster))) +
  scale_fill_viridis_d(name = "SOM Cluster") +
  coord_equal() +
  labs(title = "SOM-Derived Contamination Zones: Fukushima Region") +
  theme_minimal()
```

### Random Forest Implementation

**Task Type:** Regression is the primary choice (predicting continuous dose rate), with classification as a secondary analysis (predicting contamination zone).

**Regression Setup:**
- Target: log10(mean_dose_rate) -- log-transform because dose rate spans several orders of magnitude
- Predictors: distance_km, elevation, bearing, land_use_type, year, n_measurements

**Classification Setup (secondary):**
- Target: contamination_zone (Zone 1 / Zone 2 / Zone 3 / Normal)
- Same predictors as regression

**Hyperparameter Tuning:**
- `ntree`: 500-2000 (larger dataset benefits from more trees)
- `mtry`: Tune via `caret`; for p=8 predictors, search mtry = 2 to 6
- `nodesize`: 5 (regression default) or 3-10
- For regression on log-transformed target: back-transform predictions for final RMSE calculation

**Cross-Validation:**
- **Spatial block CV** (strongly recommended): Divide the study area into spatial blocks and use each block as a fold. The `blockCV` R package implements this. This is methodologically superior to random CV for spatial data and is publishable in itself.
- As fallback: standard 10-fold CV with OOB error.

**Feature Importance:**
- %IncMSE from permutation importance
- Expected ranking: distance_from_plant >> elevation > bearing > land_use > year

**Performance Metrics (Regression):**
- R-squared on log-scale and original scale
- RMSE in uSv/h
- MAE in uSv/h
- Scatter plot: predicted vs. actual with 1:1 line
- Spatial map of residuals (to check for systematic spatial bias)

**R Code Outline:**

```r
library(randomForest)
library(caret)
library(Metrics)

# Log-transform target
grid_data$log_dose <- log10(grid_data$mean_dose_rate + 0.01)

# Split (random; see note about spatial CV below)
set.seed(42)
train_idx <- createDataPartition(grid_data$log_dose, p = 0.8,
                                  list = FALSE)
train <- grid_data[train_idx, ]
test  <- grid_data[-train_idx, ]

# Train RF
rf_model <- randomForest(
  log_dose ~ distance_km + elevation + bearing +
             land_forest + land_urban + year + n_measurements,
  data = train,
  ntree = 1000,
  mtry = 3,
  importance = TRUE
)

# Predict and back-transform
pred_log <- predict(rf_model, test)
pred_dose <- 10^pred_log

# Metrics on original scale
actual_dose <- test$mean_dose_rate
cat("R2:", cor(pred_dose, actual_dose)^2, "\n")
cat("RMSE:", rmse(actual_dose, pred_dose), "\n")
cat("MAE:", mae(actual_dose, pred_dose), "\n")

# Variable importance
varImpPlot(rf_model, main = "Feature Importance: Dose Rate Prediction")

# Spatial prediction map
grid_data$predicted_dose <- 10^predict(rf_model, grid_data)
ggplot(grid_data) +
  geom_tile(aes(x = longitude, y = latitude, fill = predicted_dose)) +
  scale_fill_viridis_c(trans = "log10", name = "Dose Rate\n(uSv/h)") +
  coord_equal() +
  labs(title = "RF-Predicted Contamination Map") +
  theme_minimal()
```

### How SOM and RF Work Together

**Integration Strategy -- SOM for Zone Delineation, RF for Predictive Mapping:**

1. **Phase 1 (SOM):** Unsupervised discovery of contamination zones from the multi-dimensional feature space. This produces a data-driven zoning of the Fukushima region that can be compared to the official government evacuation zones. The SOM approach is objective and reproducible.

2. **Phase 2 (RF):** Supervised prediction of dose rates for grid cells where measurements are sparse or absent (spatial interpolation). RF uses the learned relationship between geographic features and contamination to "fill in" unmeasured areas.

3. **Integration Point:** SOM cluster membership is used as an additional categorical predictor in the RF model. Alternatively, separate RF models are trained for each SOM cluster (local expert models), potentially improving accuracy in heterogeneous landscapes.

4. **Validation:** Compare SOM-derived zones with (a) official evacuation zones and (b) JAEA airborne survey maps to validate the unsupervised approach. Compare RF-predicted contamination maps with held-out Safecast/JAEA measurements.

**Novelty Statement:**
"This study proposes a hybrid SOM-RF methodology for contamination zone delineation and dose rate prediction in the Fukushima exclusion zone using citizen-science (Safecast) and official (JAEA) monitoring data. SOM provides unsupervised, data-driven zone identification that complements official evacuation boundaries, while RF enables predictive mapping of contamination in areas with sparse measurements. The combination of citizen-science and government data sources through a hybrid ML framework represents a novel approach to post-nuclear-accident environmental assessment."

---

## C.4 Expected Results & Figures

### Expected Figures

1. **Figure 1:** Study area map showing Fukushima Daiichi plant location, prefectural boundaries, and the extent of the analysis area.
2. **Figure 2:** Safecast measurement density map (number of data points per grid cell -- showing the road-biased coverage).
3. **Figure 3:** Observed dose rate map from Safecast data (the contamination plume extending northwest is immediately visible).
4. **Figure 4:** Workflow diagram.
5. **Figure 5:** SOM U-matrix and training progress.
6. **Figure 6:** SOM component planes (2x3 grid): distance, elevation, bearing, dose_rate, land_forest, land_urban.
7. **Figure 7:** SOM cluster map in geographic space, compared side-by-side with the official evacuation zone map. This is the most impactful figure.
8. **Figure 8:** RF predicted vs. actual dose rate scatter plot (log-scale axes).
9. **Figure 9:** RF-predicted contamination map covering the full study area (including areas without Safecast coverage).
10. **Figure 10:** Spatial map of RF prediction residuals.
11. **Figure 11:** RF variable importance plot.
12. **Figure 12 (optional):** Temporal comparison: dose rate maps for 2012, 2015, 2018 showing radioactive decay and decontamination effects.

### Expected Tables

1. **Table 1:** Safecast data summary (total points, Fukushima subset, quality-filtered subset, grid cell count).
2. **Table 2:** JAEA data summary (survey campaigns used, parameters).
3. **Table 3:** Feature descriptions and statistics.
4. **Table 4:** SOM cluster characteristics (mean dose rate, mean distance, predominant land use per cluster).
5. **Table 5:** Agreement between SOM clusters and official evacuation zones (confusion matrix or Cohen's kappa).
6. **Table 6:** RF regression performance (R2, RMSE, MAE) for different model configurations (RF-only, SOM+RF).
7. **Table 7:** RF feature importance rankings.

### Expected Results
- RF R-squared for log(dose_rate): **0.85-0.95** (spatial data with distance as a strong predictor performs well)
- RMSE: will depend on the scale, but relative error of 15-30% is realistic for environmental data
- SOM will identify 4-6 clusters that strongly correspond to:
  - Cluster 1: High contamination, close to plant, NW direction (the main plume)
  - Cluster 2: Moderate contamination, intermediate distance
  - Cluster 3: Low contamination, forested highlands
  - Cluster 4: Near-background, distant urban areas
  - Cluster 5: Coastal areas with distinct contamination patterns
- Distance from plant will dominate RF feature importance (>40% of importance), followed by bearing (the NW plume direction) and elevation
- SOM-derived zones will show ~70-85% agreement with official evacuation zones
- Adding SOM cluster to RF may improve R2 by 2-5%

---

## C.5 Target Journals

### Journal 1: Journal of Environmental Radioactivity (Elsevier)
- **Impact Factor:** ~2.6 (2024)
- **Why it fits:** Fukushima contamination mapping is a core topic. Dozens of Safecast-based papers have been published here. ML approaches to environmental radiation are directly in scope.
- **Typical review timeline:** 6-10 weeks.
- **Open access:** Optional gold OA (~$3,300 USD).

### Journal 2: Science of the Total Environment (Elsevier)
- **Impact Factor:** ~8.2 (2024)
- **Why it fits:** High-impact environmental science journal. Fukushima studies, especially those combining citizen science with official data and novel analytical methods, fit well. Higher IF means more competition but also more visibility.
- **Typical review timeline:** 8-14 weeks.
- **Open access:** Optional gold OA (~$4,200 USD).
- **Note:** This is an ambitious target but achievable if the paper is well-executed.

### Journal 3: Environmental Pollution (Elsevier)
- **Impact Factor:** ~7.6 (2024)
- **Why it fits:** Radioactive contamination mapping falls under environmental pollution. ML-based spatial analysis is increasingly published here.
- **Typical review timeline:** 6-12 weeks.
- **Open access:** Optional gold OA.

---

## C.6 Strengths & Weaknesses

### Strengths
1. **Highest novelty of all three topics.** SOM-RF hybrid applied to combined Safecast + JAEA data for contamination zone mapping is genuinely new.
2. **Highest-impact journal potential.** Science of the Total Environment (IF ~8.2) is a realistic target. This is the only topic of the three with a plausible path to an IF > 5 journal.
3. **Compelling visual output.** Contamination maps of Fukushima are inherently striking and tell a powerful story. The geographic figures will be excellent.
4. **Strong real-world significance.** Post-nuclear-accident contamination mapping is one of the most important applications in radiation protection. The practical impact is clear.
5. **Citizen science angle.** The combination of citizen-science and official government data is a methodological contribution in itself and appeals to a broad audience.
6. **Temporal analysis potential.** Tracking contamination decay over 10+ years adds depth to the paper.

### Weaknesses
1. **Highest data preprocessing burden.** The Safecast dataset is enormous (150M+ points), requires geographic filtering, quality control, and spatial aggregation. The JAEA interface is complex. This is the biggest time risk.
2. **Spatial statistics complexity.** Spatial autocorrelation must be addressed in the validation approach. Standard random CV overstates model performance for spatial data. The student needs to implement spatial block CV (the `blockCV` package helps, but it is an additional learning curve).
3. **Data quality concerns.** Citizen-science data has inherent quality issues. Reviewers will ask about calibration, measurement protocols, and representativeness. The paper must include a thorough discussion of data limitations.
4. **R spatial packages learning curve.** Working with `sf`, `terra`, or `raster` packages requires spatial computing knowledge that may be new to the student.
5. **Risk of scope creep.** The dataset is so rich that it is easy to keep adding analyses (temporal trends, land use effects, decontamination tracking) and never finish the paper.

### Mitigation Strategies
- **For data size:** Immediately filter to a bounding box (e.g., 80km radius around the plant). Aggregate to 1km grid cells. This reduces millions of points to ~5,000 manageable rows.
- **For spatial CV:** Use the `blockCV` package, which automates spatial block cross-validation with minimal code.
- **For data quality:** Dedicate a subsection of the paper to "Data Quality Assessment." Compare Safecast means with co-located JAEA measurements to establish concordance. Report a correlation coefficient between the two data sources.
- **For spatial packages:** Use `sf` (Simple Features) which integrates well with `ggplot2` and `dplyr`. Avoid older packages like `sp`/`rgdal`.
- **For scope creep:** Strictly define the study scope in Week 1 (e.g., "Spatial analysis of a single time period, 2013-2014"). Do not add temporal analysis unless the spatial paper is complete ahead of schedule.

---

## C.7 Recommendation Score

| Criterion | Score (out of 10) | Notes |
|-----------|-------------------|-------|
| Publishability | 9 | High novelty + high impact journals = strong publication potential |
| Feasibility | 5 | Hardest preprocessing, spatial complexity, largest data |
| Novelty | 9 | Very few papers combine Safecast + JAEA + SOM + RF |
| Data Access | 7 | Safecast is open but huge; JAEA requires account and navigation |
| **Overall** | **7.5** | Highest ceiling but also highest risk. Best for ambitious students. |

---
---

# COMPARATIVE SUMMARY

## Side-by-Side Comparison

| Criterion | Topic A (Shielding) | Topic B (RadNet) | Topic C (Fukushima) |
|-----------|-------------------|-----------------|-------------------|
| Data access difficulty | Very Easy | Easy | Moderate |
| Preprocessing effort | Low | Moderate | High |
| Coding complexity | Low-Moderate | Moderate | Moderate-High |
| Paper writing effort | Moderate | Moderate | Moderate-High |
| Expected RF accuracy | R2 > 0.99 | Acc 85-92% | R2 0.85-0.95 |
| Novelty | Low | Moderate-High | High |
| Journal IF potential | ~2.8 | ~2.6-3.0 | ~2.6-8.2 |
| Risk of failure | Very Low | Low-Moderate | Moderate |
| Visual impact of paper | Moderate | High | Very High |
| **Overall score** | **7.5** | **8.0** | **7.5** |

## Final Recommendation

**For the safest, fastest path:** Choose **Topic A** (Gamma-Ray Shielding). You will almost certainly finish on time with strong numerical results (R2 > 0.99). However, reviewers may find the novelty insufficient without a unique material system.

**For the best balance of risk and reward:** Choose **Topic B** (EPA RadNet). It offers real-world data, strong novelty, reasonable preprocessing, and direct CSV access. The geographic visualization (SOM clusters on a US map) will be a standout figure. This is the recommended choice for a student who wants a publishable paper with manageable effort.

**For the highest publication ceiling:** Choose **Topic C** (Fukushima). If the student has strong R programming skills and is comfortable with spatial data, this has the highest novelty and the potential to target high-impact journals (IF > 5). However, it carries the highest risk of running behind schedule due to data preprocessing.

### If forced to choose one: **Topic B (EPA RadNet)** is the recommended choice.

It sits at the optimal point on the risk-reward curve: genuinely novel (few or no published SOM-RF studies on RadNet), high visual impact, manageable preprocessing, freely accessible CSV data with no registration, and strong fit for the Journal of Environmental Radioactivity. The 8-week timeline is realistic without requiring extraordinary effort or prior spatial computing experience.

---

*Report prepared for NE 493 -- Special Topics in Radiation Protection*
*King Abdulaziz University, Department of Nuclear Engineering*
*February 2026*

---

# NE 493 Research Project Evaluation Report

## AI in Radiation Protection — King Abdulaziz University

### Self-Organizing Maps (SOM) & Random Forest (RF) in R

**Prepared: 2026-02-16**

---

---

# TOPIC D: Radionuclide Identification from Gamma-Ray Spectra

---

## 1. Dataset Accessibility

### Primary Sources

**Source 1 — Kaggle Gamma-Ray Spectrum Datasets**

Several relevant datasets exist on Kaggle:

- **"Gamma Spectrum" / "Radioactive Decay Gamma Spectroscopy"** datasets uploaded by various contributors. Search Kaggle for "gamma spectrum," "gamma spectroscopy," or "radionuclide identification."
- A commonly referenced dataset is the **IAEA gamma-ray catalogue** spectra that have been reformatted into CSV files and uploaded to Kaggle.
- The **"MAGIC Gamma Telescope"** dataset (UCI/Kaggle) is sometimes confused with this topic but is NOT suitable — it classifies gamma vs. hadron events from Cherenkov telescopes, not radionuclide identification. Avoid this one.

**Source 2 — Synthetic / Simulated Spectra (Recommended Supplement)**

- The student can generate synthetic gamma spectra using **MCNP**, **Geant4**, or the freely available **GADRAS-DRF** (Gamma Detector Response and Analysis Software) from Sandia National Laboratories.
- GADRAS-DRF is available at: https://www.sandia.gov/research/homeland-security/ — requires a request form (US persons restriction may apply; alternatives below).
- The **InterSpec** tool from Sandia (open-source, GitHub: https://github.com/sandialabs/InterSpec) can also generate and read spectra.

**Source 3 — IAEA Nuclear Data Services**

- URL: https://www-nds.iaea.org/
- The IAEA provides reference gamma-ray energies and intensities for all radionuclides.
- The **IAEA LiveChart of Nuclides** (https://www-nds.iaea.org/relnsd/vcharthtml/VChartHTML.html) provides downloadable decay data.

**Source 4 — Idaho National Laboratory (INL) Gamma-Ray Spectra**

- The **Measured and Simulated Gamma-Ray Spectra** dataset has been published in conjunction with ML research papers.
- Some are available as supplementary data in published papers (e.g., the Kamuda & Sullivan 2019 dataset used in IEEE Transactions on Nuclear Science).

### Registration Requirements

- **Kaggle**: Free account required. Sign up at kaggle.com.
- **IAEA**: No registration needed for public nuclear data services.
- **GADRAS-DRF**: Request form required; may have export control restrictions for non-US persons. If inaccessible, use InterSpec (fully open-source, no restrictions).

### Data Format

- Kaggle datasets are typically in **CSV** format.
- Each row represents one spectrum measurement (or one energy bin).
- Common structure:
  - **Option A (Wide format):** Each row = one spectrum sample; columns = energy channel counts (e.g., 1024 or 4096 channels); last column = radionuclide label.
  - **Option B (Long format):** Columns include `spectrum_id`, `channel_number`, `counts`, `label`.
- IAEA data is available in CSV, ENDF, or HTML table formats.

### Approximate Size

- A typical gamma spectroscopy dataset for ML has:
  - **500 to 10,000 spectra** (rows in wide format)
  - **1024 to 4096 energy channels** (columns/features)
  - **5 to 20 radionuclide classes** (e.g., Co-60, Cs-137, Am-241, Ba-133, Na-22, K-40, Eu-152, etc.)
  - File size: **5 MB to 200 MB** depending on number of spectra and channels.
- If generating synthetic data, the student controls the size (recommend at least 500 spectra per class).

### Limitations and Restrictions

- Kaggle datasets may lack documentation on detector type, geometry, and acquisition conditions — this must be stated as a limitation.
- Synthetic data may not capture real-world noise, pile-up effects, and Compton continuum artifacts faithfully.
- Some high-quality measured datasets are behind export control or institutional access barriers.
- Mixed-source spectra (multiple radionuclides in one spectrum) are significantly harder; starting with single-source spectra is recommended.

### Preprocessing Pipeline for R

```
Step 1: Download CSV from Kaggle or generate spectra.
Step 2: Load into R with read.csv() or data.table::fread() for speed.
Step 3: Verify structure — each row should be one spectrum, columns = channel counts.
Step 4: Normalize spectra:
        - Divide each spectrum by its total counts (area normalization)
        - Or apply min-max scaling per spectrum
        - Or apply standard score (z-score) across channels
Step 5: Optionally rebin spectra (e.g., from 4096 to 1024 or 256 bins)
        to reduce dimensionality.
Step 6: Label encoding — convert radionuclide names to factors in R.
Step 7: Remove any spectra with zero total counts or obvious corruption.
Step 8: Split into training (70-80%) and testing (20-30%) sets.
```

---

## 2. Effort Estimate

### Week-by-Week Timeline

| Week | Task | Hours |
|------|------|-------|
| 1 | Literature review: gamma-ray spectrometry, ML for radionuclide ID. Identify and download 3-4 Kaggle datasets. Evaluate suitability. | 12-15 |
| 2 | Data cleaning and preprocessing. Normalize spectra. Exploratory visualization (plot sample spectra per isotope). Optionally generate synthetic spectra with InterSpec. | 12-15 |
| 3 | SOM implementation: build SOM on spectral data, vary grid sizes, generate clustering and component plane maps. | 12-15 |
| 4 | SOM analysis: interpret clusters, map radionuclide labels onto SOM, identify which spectral features drive clustering. | 10-12 |
| 5 | Random Forest: train RF classifier using raw channels and SOM-derived features. Hyperparameter tuning. | 12-15 |
| 6 | Model evaluation: confusion matrices, accuracy, F1 scores per class. Feature importance analysis. Compare RF alone vs. SOM+RF pipeline. | 10-12 |
| 7 | Paper writing: Introduction, Methods, Results sections. Generate all figures and tables. | 15-18 |
| 8 | Paper writing: Discussion, Conclusion. Formatting to journal style. Internal review and revision. | 12-15 |

**Total estimated effort: 95-117 hours over 8 weeks (~12-15 hrs/week)**

### Complexity Breakdown

- **Data collection effort:** Low to Moderate. If suitable Kaggle data is found, it takes 1-2 days. If synthetic data generation is needed, add 3-5 extra days.
- **Preprocessing complexity:** Moderate. Spectral normalization and rebinning require care. No missing values typically (counts are always recorded), but handling detector artifacts and background subtraction can be tricky.
- **Coding complexity in R:** Moderate to Hard. High-dimensional data (1024+ features) requires careful handling. SOM on 1024-dimensional input is computationally intensive. The `kohonen` package handles it well but visualization of component planes for 1024 channels requires dimensionality reduction strategies.
- **Paper writing effort:** Moderate. The topic is well-established in the nuclear security literature, so the introduction writes itself, but the novelty argument (SOM+RF combination) needs careful framing.

### Overall Difficulty Rating: **Hard**

The high dimensionality (1024+ channels) and the need for domain expertise in gamma spectroscopy make this challenging. However, the classification task itself is well-defined, which is an advantage.

---

## 3. ML Structure & Pipeline

### Data Preparation

**Feature Selection:**
- Raw features: 1024 (or 256 after rebinning) energy channel counts per spectrum.
- Derived features (optional): peak locations, peak heights, peak widths (FWHM), Compton-to-peak ratios, total counts, spectral centroid, spectral variance.
- Feature reduction: PCA can reduce 1024 channels to 20-50 principal components retaining >95% variance. SOM itself acts as a dimensionality reduction tool.

**Normalization:**
- Area normalization (divide by sum of all channel counts) is standard in gamma spectroscopy to remove count-rate dependence.
- After area normalization, apply min-max or z-score scaling for SOM input.

**Train/Test Split:**
- 70% training / 30% testing with stratified sampling to maintain class balance.
- Alternatively, 80/20 split with 5-fold cross-validation on the training set.

### SOM Implementation

**Input Matrix Structure:**
- Matrix dimensions: N_samples x N_features (e.g., 5000 x 256 after rebinning).
- Each row is one normalized gamma spectrum.
- Each column is the count rate in one energy bin.

**Grid Size Recommendations:**
- For 5000 samples with 10-15 classes: start with a 15x15 or 20x20 hexagonal grid.
- Rule of thumb: grid nodes ~ 5 * sqrt(N_samples). For N=5000, that is ~350 nodes, so approximately 19x19.
- Use hexagonal topology (default in `kohonen`) for better neighbor representation.

**What the SOM Reveals:**
- **Clustering of spectra by radionuclide:** Spectra from the same isotope should cluster together on the map. If Co-60 and Cs-137 spectra occupy distinct, well-separated regions, the SOM has learned the key spectral differences.
- **Component planes:** Show how each energy channel (or group of channels) varies across the map. The component plane for the 662 keV channel should highlight the region where Cs-137 spectra reside. The planes for the 1173 keV and 1332 keV channels should highlight the Co-60 region.
- **Transition zones:** Between clusters may reveal spectra that are ambiguous — low-count spectra or spectra with significant Compton overlap.
- **Outlier detection:** Spectra mapping to isolated nodes may be anomalous measurements.

**Visualization:**
- SOM U-matrix (distance matrix) showing cluster boundaries.
- Component plane maps for key energy channels (e.g., the channel around 662 keV, 1461 keV, etc.).
- Mapping of class labels onto the SOM grid (color-coded by radionuclide).
- Node count plots showing how many spectra map to each node.

**R Code Outline:**

```r
library(kohonen)
library(data.table)

# Load data
spectra <- fread("gamma_spectra.csv")
labels  <- spectra$radionuclide
spectra_mat <- as.matrix(spectra[, -"radionuclide"])

# Normalize (area normalization then scaling)
spectra_norm <- t(apply(spectra_mat, 1, function(x) x / sum(x)))
spectra_scaled <- scale(spectra_norm)

# Define SOM grid
som_grid <- somgrid(xdim = 20, ydim = 20, topo = "hexagonal")

# Train SOM
som_model <- som(spectra_scaled,
                 grid = som_grid,
                 rlen = 500,        # training iterations
                 alpha = c(0.05, 0.01),
                 keep.data = TRUE)

# Visualizations
plot(som_model, type = "changes")     # training progress
plot(som_model, type = "counts")      # node counts
plot(som_model, type = "dist.neighbours")  # U-matrix
plot(som_model, type = "codes")       # codebook vectors

# Component planes for specific channels
# (e.g., channel 662 corresponding to ~662 keV for Cs-137)
plot(som_model, type = "property",
     property = getCodes(som_model)[[1]][, 662],
     main = "Component Plane: Channel 662 (Cs-137 peak)")

# Map class labels onto SOM
som_cluster <- data.frame(node = som_model$unit.classif,
                          label = labels)
# Color-code SOM nodes by dominant class
```

### Random Forest Implementation

**Classification vs. Regression:**
- This is a **multi-class classification** problem. The target variable is the radionuclide identity (categorical: Cs-137, Co-60, Am-241, etc.).

**Target Variable:** `radionuclide` (factor with 5-20 levels depending on dataset).

**Predictor Variables:**
- **Approach 1 (Baseline):** All energy channel counts (256-1024 features) after normalization.
- **Approach 2 (SOM-enhanced):** SOM codebook vector coordinates (the BMU coordinates or the codebook vector of the BMU for each spectrum) concatenated with selected raw features.
- **Approach 3 (SOM cluster features):** SOM cluster membership as an additional feature in RF, or SOM node distances as new features.

**Hyperparameter Tuning:**
- `ntree`: Start with 500, test 100, 500, 1000, 2000. Use OOB error stabilization plot to determine sufficiency.
- `mtry`: Default for classification is sqrt(p). For p=256 channels, default mtry ~ 16. Search grid: 8, 16, 32, 64.
- `nodesize`: Default is 1 for classification. Test 1, 3, 5.
- Use the `caret` or `tuneRanger` package for automated grid/random search.

**Cross-Validation:**
- Primary: OOB (out-of-bag) error, which is built into Random Forest and provides an unbiased estimate.
- Secondary: 5-fold stratified cross-validation on the training set for more robust estimates.
- Final evaluation on the held-out 30% test set.

**Feature Importance:**
- Use Mean Decrease in Gini (MDG) and Mean Decrease in Accuracy (MDA/permutation importance).
- Plot top 20-30 most important channels. These should correspond to known photopeaks of the radionuclides in the dataset.
- This provides physical interpretability: if channel 662 (Cs-137 peak) ranks highest for Cs-137 classification, the model has learned physically meaningful features.

**Performance Metrics:**
- Overall accuracy (% correct across all classes).
- Per-class precision, recall, F1-score.
- Confusion matrix (critical for identifying which isotopes are confused with each other).
- Cohen's Kappa for multi-class agreement.
- Macro-averaged and weighted F1.

**R Code Outline:**

```r
library(randomForest)
library(caret)

# Prepare data
spectra_df <- data.frame(spectra_scaled)
spectra_df$label <- as.factor(labels)

# Train/test split (stratified)
set.seed(42)
train_idx <- createDataPartition(spectra_df$label, p = 0.7, list = FALSE)
train_data <- spectra_df[train_idx, ]
test_data  <- spectra_df[-train_idx, ]

# Train Random Forest
rf_model <- randomForest(label ~ .,
                         data = train_data,
                         ntree = 1000,
                         mtry = 16,
                         importance = TRUE)

# OOB error
print(rf_model)
plot(rf_model)  # error vs. ntree

# Predictions on test set
pred <- predict(rf_model, test_data)
confusionMatrix(pred, test_data$label)

# Feature importance
varImpPlot(rf_model, n.var = 30, main = "Top 30 Important Energy Channels")

# Hyperparameter tuning with caret
tune_grid <- expand.grid(mtry = c(8, 16, 32, 64))
ctrl <- trainControl(method = "cv", number = 5)
rf_tuned <- train(label ~ ., data = train_data,
                  method = "rf", tuneGrid = tune_grid,
                  trControl = ctrl, ntree = 500)
print(rf_tuned)
```

### How SOM and RF Work Together (Novelty Angle)

The novel integration strategy proceeds as follows:

1. **SOM as Unsupervised Pre-Analysis:** Train SOM on the full spectral dataset without labels. Identify natural clustering structure. This reveals whether the spectral data has inherent separability by radionuclide — a finding that stands on its own.

2. **SOM-Derived Features for RF:** Extract from the trained SOM:
   - BMU (Best Matching Unit) coordinates (x, y on the grid) for each spectrum — 2 new features.
   - Distance from each spectrum to its BMU (quantization error) — 1 new feature capturing how "typical" a spectrum is.
   - SOM cluster membership (after hierarchical clustering on the codebook vectors) — 1 categorical feature.
   - Distances to all SOM cluster centroids — K new features (where K = number of SOM clusters).

3. **RF with Augmented Features:** Train RF on original spectral channels + SOM-derived features. Compare performance to RF trained on raw channels alone.

4. **Interpretability Loop:** RF feature importance identifies which energy channels matter most. These can be mapped back onto the SOM component planes to show spatial co-variation of important features across the SOM grid.

5. **Novelty Argument:** Most published work uses either SOM alone (for visualization/clustering) or supervised classifiers alone (CNN, RF) for radionuclide ID. The combination of unsupervised topology learning (SOM) with supervised ensemble classification (RF), where SOM features augment the RF, is underexplored in gamma spectroscopy literature.

---

## 4. Expected Results & Figures

### Expected Figures and Tables

| Figure/Table | Description |
|---|---|
| Fig. 1 | Sample gamma-ray spectra for 5-8 different radionuclides (overlay plot or faceted) |
| Fig. 2 | SOM training progress (convergence curve) |
| Fig. 3 | SOM U-matrix showing cluster boundaries |
| Fig. 4 | SOM grid color-coded by dominant radionuclide class (the "map") |
| Fig. 5 | Component planes for 4-6 key energy channels (corresponding to known photopeaks) |
| Fig. 6 | RF OOB error convergence plot (error vs. number of trees) |
| Fig. 7 | Confusion matrix heatmap for RF classification on test set |
| Fig. 8 | Variable importance plot (top 30 energy channels by MDA and MDG) |
| Fig. 9 | Comparison bar chart: accuracy of RF-only vs. SOM+RF pipeline |
| Table 1 | Dataset summary (number of spectra per radionuclide, detector info) |
| Table 2 | SOM hyperparameters and configuration |
| Table 3 | RF hyperparameters (after tuning) |
| Table 4 | Per-class precision, recall, F1-score |
| Table 5 | Comparison of model performance (RF alone vs. SOM+RF) |

### Expected Results

- **SOM clustering:** Expect clear separation of isotopes with distinct spectral signatures (Cs-137, Co-60, Am-241). Isotopes with overlapping energies (e.g., Ba-133 and Eu-152, which share some gamma lines) will cluster closer together or overlap on the SOM.
- **RF classification accuracy:** For clean, single-source spectra with high counts: **95-99% accuracy**. For low-count or noisy spectra: **80-92% accuracy**. For mixed-source spectra (if attempted): **65-85% accuracy**.
- **Most important channels** will align with known photopeak energies, providing physical validation of the model.
- **SOM+RF improvement** over RF alone is expected to be modest (1-3% accuracy gain) but meaningful in terms of interpretability and handling of edge cases.

### Interpretation and Discussion Points

- Confusion matrix will show which isotope pairs are most commonly misclassified — these should correspond to isotopes with overlapping gamma energies.
- Feature importance analysis connects ML results to nuclear physics (photopeak energies), strengthening the paper's contribution.
- SOM visualization provides an intuitive "map" of the spectral landscape that reviewers and practitioners find valuable.

---

## 5. Target Journals

### Journal 1: Nuclear Instruments and Methods in Physics Research Section A (NIM-A)

- **Impact Factor:** ~1.7-1.9
- **Why it fits:** NIM-A is the premier journal for detector instrumentation and methods, including spectral analysis and ML-based nuclear detection. Radionuclide identification is a core topic.
- **Review timeline:** 2-4 months typically.
- **Open access:** Hybrid OA available through Elsevier; ~$2,500-3,000 APC.

### Journal 2: Applied Radiation and Isotopes

- **Impact Factor:** ~1.6-1.8
- **Why it fits:** Covers applications of radiation measurement including spectroscopy, detection, and identification. ML applications in radiation measurement are increasingly published here.
- **Review timeline:** 2-3 months.
- **Open access:** Hybrid OA through Elsevier; similar APC.

### Journal 3: Journal of Radioanalytical and Nuclear Chemistry (JRNC)

- **Impact Factor:** ~1.5-1.7
- **Why it fits:** Publishes nuclear analytical methods broadly. ML-enhanced spectroscopy falls within scope. Slightly easier acceptance threshold than NIM-A.
- **Review timeline:** 1-3 months (generally faster).
- **Open access:** Hybrid OA through Springer; ~$3,000 APC.

---

## 6. Strengths & Weaknesses

### Strengths

- **Directly relevant** to nuclear engineering and radiation protection — strong alignment with the NE 493 course.
- **Well-defined classification task** — clear target variable, clear success metrics.
- **Physical interpretability:** Feature importance maps directly to gamma-ray photopeak energies, providing a satisfying connection between ML and physics.
- **Strong novelty angle:** SOM+RF combination for radionuclide identification is underexplored in the literature.
- **Practical impact:** Radionuclide identification is important for homeland security, nuclear safeguards, environmental monitoring, and emergency response.
- **Rich visualization potential:** Gamma spectra, SOM maps, component planes, and confusion matrices all make for a visually compelling paper.

### Weaknesses / Risks

- **Data access uncertainty:** The student may not find a high-quality, well-documented Kaggle dataset. Many uploaded spectral datasets lack metadata about detector type, calibration, and measurement conditions.
- **High dimensionality:** 1024+ features can be computationally expensive for SOM training and may require rebinning or PCA preprocessing.
- **Domain expertise required:** Understanding gamma-ray interactions (photoelectric, Compton, pair production), energy calibration, and detector response functions is necessary to write a credible paper.
- **Synthetic data concerns:** If the student must generate synthetic data, reviewers may question real-world applicability.

### Mitigation Strategies

- **Data access:** Start searching for datasets in Week 1. If no suitable Kaggle dataset is found by end of Week 1, pivot to synthetic data generation using InterSpec (free, no restrictions). Clearly acknowledge synthetic data limitations in the paper.
- **High dimensionality:** Rebin spectra from 1024 to 256 channels early. This is physically justified (many detector systems have limited resolution anyway) and dramatically reduces computation.
- **Domain expertise:** The student should read 3-5 key papers in the first week. Recommended: Kamuda & Sullivan (2019, IEEE TNS), Medhat (2012, Annals of Nuclear Energy), and Yoshida et al. (2002, JRNC).

---

## 7. Recommendation Score

| Criterion | Score (out of 10) | Justification |
|---|---|---|
| **Publishability** | 8 | Strong fit for NIM-A or JRNC. SOM+RF combination is novel in this domain. |
| **Feasibility** | 6 | High dimensionality and possible data access issues add risk. Requires nuclear spectroscopy knowledge. |
| **Novelty** | 8 | SOM+RF for radionuclide ID is not common in the literature. Component plane interpretation adds value. |
| **Data Access** | 5 | Uncertain. Kaggle datasets may be poorly documented. May require synthetic data generation as fallback. |
| **Overall** | **6.5** | A rewarding but challenging project. Best for students with some gamma spectroscopy background. |

---

---

# TOPIC E: Indoor Radon Concentration Prediction

## (US EPA Radon Data + USGS Geology Data)

---

## 1. Dataset Accessibility

### Primary Source 1 — US EPA Radon Data

**EPA Map of Radon Zones:**
- URL: https://www.epa.gov/radon/epa-map-radon-zones
- This provides **county-level radon zone classifications** (Zone 1: highest potential, Zone 2: moderate, Zone 3: low) for the entire United States.
- Data is available as downloadable files and interactive maps.
- The EPA also provides **state-level radon data summaries** with average indoor radon concentrations by county.

**EPA/State Radon Survey Data:**
- The **EPA National Residential Radon Survey (1991)** and subsequent updates provide measured indoor radon concentrations.
- Supplemental state-by-state data is available from individual state radon programs (links from the EPA website).

**EPA SIRG (State Indoor Radon Grant) data:**
- Historical indoor radon test results aggregated by county and state.
- Available through EPA archives and state environmental agencies.

### Primary Source 2 — Lawrence Berkeley National Laboratory (LBNL) Radon Database

- URL: https://eetd.lbl.gov/ied/era/radon
- The LBNL radon database compiles indoor radon measurements from across the US.
- Published summary statistics (county-level means, geometric means, standard deviations) are available in academic papers by Price et al. and Nero et al.

### Primary Source 3 — USGS Geology Data

**USGS Geologic Maps and Databases:**
- URL: https://www.usgs.gov/programs/national-cooperative-geologic-mapping-program
- The **State Geologic Map Compilation (SGMC)** provides digitized geologic maps of the US.
- Download at: https://www.sciencebase.gov/catalog/item/5888bf4fe4b05ccb964bab9a
- Format: Shapefiles (GIS) that can be processed in R using the `sf` package.

**USGS Geochemistry of Soils:**
- URL: https://www.usgs.gov/programs/mineral-resources-program
- The **National Geochemical Database** provides soil uranium, radium, and thorium concentrations by location.
- Download: https://mrdata.usgs.gov/geochem/

**USGS Physiographic and Hydrologic Data:**
- Soil permeability, bedrock type, and related geological parameters are available through USGS databases.

### Primary Source 4 — US Census and Climate Data (Covariates)

- **US Census:** Housing characteristics (basement prevalence, building age, ventilation type) at county level from data.census.gov.
- **NOAA Climate Data:** Temperature, precipitation, wind speed at county level from https://www.ncdc.noaa.gov/ — relevant because radon entry is affected by temperature-driven pressure differentials.

### Registration Requirements

- **EPA:** No registration required. All data is public domain.
- **USGS:** No registration required. Public domain under US government open data policy.
- **US Census:** No registration for bulk downloads; API key (free) for programmatic access.
- **NOAA:** No registration for bulk downloads.

### Data Format

- EPA radon data: **HTML tables** on the EPA website (need web scraping or manual download) and some **PDF reports**. Some states provide CSV or Excel files.
- USGS geology: **Shapefiles** (.shp, .dbf, .prj, .shx) for GIS data; **CSV** for geochemistry.
- Census: **CSV** via API or bulk download.
- NOAA: **CSV** files.

### Approximate Size

After compilation, the master dataset will be at the **county level** for the US:
- **~3,143 counties** (rows)
- **20-40 predictor variables** (columns) covering:
  - Mean/median indoor radon concentration (target variable, in pCi/L or Bq/m³)
  - EPA radon zone (1, 2, 3)
  - Dominant geology type (categorical: granite, limestone, shale, etc.)
  - Soil uranium concentration (ppm)
  - Soil radium concentration
  - Soil permeability class
  - Mean annual temperature
  - Mean annual precipitation
  - Basement prevalence (% of homes with basements)
  - Population density
  - Median building age
  - Elevation
  - Latitude, longitude
- File size: **< 5 MB** after compilation (small, manageable dataset).

### Limitations and Restrictions

- **County-level aggregation:** Radon varies hugely within a county. County-level means smooth out local variation. This is a known limitation and must be acknowledged.
- **Data vintage:** The EPA National Residential Radon Survey is from 1991. State data varies in recency. Some county-level measurements may be from the 1990s or 2000s.
- **Self-selection bias:** Many radon measurements come from homeowners who suspected high radon (often tested during real estate transactions), potentially biasing county averages upward.
- **All data is US-only.** Generalization to other countries (like Saudi Arabia) requires discussion.
- **Assembly effort:** The main challenge is merging data from 4-5 different sources at the county FIPS code level. This is a significant data wrangling effort.

### Preprocessing Pipeline for R

```
Step 1:  Download EPA county-level radon data.
         May require scraping HTML tables using rvest package
         or finding pre-compiled CSV versions.
Step 2:  Download USGS geology shapefiles and geochemistry CSV.
Step 3:  Download climate data (county-level annual averages) from NOAA.
Step 4:  Download housing characteristics from Census Bureau.
Step 5:  Standardize all datasets to county FIPS codes.
Step 6:  Spatial join: assign dominant geology type to each county
         using sf package (point-in-polygon or area-weighted).
Step 7:  Merge all datasets by FIPS code using dplyr::left_join().
Step 8:  Handle missing values:
         - Counties with no radon measurements: remove or impute.
         - Missing geology/climate: impute from neighboring counties
           or use median imputation.
         - Expect 10-30% missingness in some variables.
Step 9:  Encode categorical variables (geology type) as factors.
Step 10: Log-transform radon concentration (radon is log-normally
         distributed — this is well-established in the literature).
Step 11: Scale/normalize continuous predictors for SOM input.
Step 12: Final dataset: ~2,500-3,000 counties with complete cases.
```

---

## 2. Effort Estimate

### Week-by-Week Timeline

| Week | Task | Hours |
|------|------|-------|
| 1 | Literature review on indoor radon prediction, geological controls on radon. Identify and begin downloading all data sources. | 12-15 |
| 2 | Data assembly: download EPA radon data, USGS geology data, climate and census data. Begin merging by FIPS code. | 15-18 |
| 3 | Complete data merging. Handle missing values. Exploratory data analysis (histograms, maps, correlations). Log-transform radon. | 12-15 |
| 4 | SOM implementation: train SOM on predictor variables. Generate maps, U-matrix, component planes. Interpret clusters geographically. | 12-15 |
| 5 | Random Forest: train RF regression model. Hyperparameter tuning. Feature importance. Cross-validation. | 12-15 |
| 6 | Advanced analysis: SOM+RF integration. Geographic mapping of predictions vs. actuals. Residual analysis. | 10-12 |
| 7 | Paper writing: Introduction, Methods, Results. Generate all figures including maps. | 15-18 |
| 8 | Paper writing: Discussion (connect to radon physics and geology), Conclusion. Formatting and revision. | 12-15 |

**Total estimated effort: 100-123 hours over 8 weeks (~13-15 hrs/week)**

### Complexity Breakdown

- **Data collection effort:** HIGH. This is the main bottleneck. Assembling data from 4-5 different federal sources and merging by county FIPS code is tedious and error-prone. Budget a full 1.5-2 weeks for this.
- **Preprocessing complexity:** Moderate to High. Missing value imputation, spatial joins (geology to counties), log-transformation, and ensuring all data aligns at the county level.
- **Coding complexity in R:** Moderate. The dataset itself is small (3000 rows x 30 columns) so computation is fast. The complexity is in data wrangling and spatial operations (sf package). SOM and RF are straightforward on this sized data.
- **Paper writing effort:** Moderate. Radon prediction is a well-studied field. The student must clearly articulate what the SOM+RF approach adds beyond existing EPA zone maps and regression models.

### Overall Difficulty Rating: **Moderate**

The ML part is straightforward (small, tabular dataset). The difficulty is almost entirely in the data assembly and the need to understand radon geophysics for a credible paper.

---

## 3. ML Structure & Pipeline

### Data Preparation

**Feature Selection (Predictor Variables):**

| Feature | Type | Source |
|---|---|---|
| Dominant geology type | Categorical | USGS SGMC |
| Soil uranium concentration (ppm) | Continuous | USGS Geochemistry |
| Soil permeability class | Ordinal | USGS / NRCS |
| Mean annual temperature (°C) | Continuous | NOAA |
| Mean annual precipitation (mm) | Continuous | NOAA |
| Elevation (m) | Continuous | USGS |
| Latitude | Continuous | Census |
| Longitude | Continuous | Census |
| Population density | Continuous | Census |
| % homes with basements | Continuous | Census AHS |
| Median year of building construction | Continuous | Census |
| Soil moisture index | Continuous | USGS/NOAA |

**Target Variable:** `log_radon_mean` — log-transformed mean indoor radon concentration (pCi/L) at the county level. Log-transformation is essential because radon concentrations follow a log-normal distribution.

**Normalization:**
- Continuous predictors: z-score standardization for SOM; RF does not require normalization but it does not hurt.
- Categorical predictors: one-hot encoding for SOM; RF handles factors natively.

**Train/Test Split:**
- 70/30 stratified by EPA radon zone (to ensure all zones are represented in both sets).
- Spatial cross-validation should be considered (blocking by state or region) to avoid spatial autocorrelation leakage.

### SOM Implementation

**Input Matrix Structure:**
- Matrix dimensions: ~2,500-3,000 rows (counties) x 12-20 columns (predictor features after encoding).
- Each row is one county's feature vector.
- Only predictor variables go into the SOM (target variable is mapped post-hoc).

**Grid Size Recommendations:**
- For ~3,000 samples: 5 * sqrt(3000) ~ 274 nodes. A 15x15 (225 nodes) or 17x17 (289 nodes) hexagonal grid is appropriate.
- Start with 12x12 and increase to 18x18, comparing topographic error and quantization error.

**What the SOM Reveals:**
- **Geographic/geologic clustering of counties:** Counties with similar geology, climate, and housing stock will cluster together on the SOM, regardless of their geographic proximity. For example, granite-bedrock counties in New England and the Rocky Mountains may cluster together.
- **Radon risk patterns:** When the target variable (mean radon) is mapped onto the SOM as a property overlay, the student can identify which clusters correspond to high-radon vs. low-radon counties.
- **Component planes:** The uranium component plane and the geology component plane should strongly correlate with the radon overlay, confirming known geophysical relationships.
- **Unexpected associations:** SOM may reveal that certain combinations of features (e.g., high soil permeability + moderate uranium + cold climate) produce the highest radon, providing nuanced insights beyond simple correlations.

**Visualization:**
- SOM U-matrix with cluster boundaries.
- Component planes for each predictor variable (12-15 small multiples).
- Radon concentration overlay on SOM grid (property map).
- SOM cluster assignments mapped back onto a US county map (geographic visualization of SOM clusters).

**R Code Outline:**

```r
library(kohonen)
library(sf)
library(dplyr)

# Load merged county-level dataset
county_data <- read.csv("county_radon_merged.csv")

# Prepare SOM input (predictors only, scaled)
predictor_cols <- c("uranium_ppm", "permeability", "mean_temp",
                    "precipitation", "elevation", "latitude",
                    "longitude", "pop_density", "pct_basement",
                    "median_bldg_year")
som_input <- scale(county_data[, predictor_cols])

# Handle categorical geology: one-hot encode
library(caret)
geo_dummies <- predict(dummyVars(~ geology_type, data = county_data),
                       newdata = county_data)
som_input <- cbind(som_input, scale(geo_dummies))

# Train SOM
som_grid <- somgrid(xdim = 15, ydim = 15, topo = "hexagonal")
som_model <- som(as.matrix(som_input),
                 grid = som_grid,
                 rlen = 500,
                 alpha = c(0.05, 0.01))

# Visualizations
plot(som_model, type = "changes")
plot(som_model, type = "dist.neighbours")  # U-matrix

# Component plane for uranium
plot(som_model, type = "property",
     property = getCodes(som_model)[[1]][, "uranium_ppm"],
     main = "Component Plane: Soil Uranium")

# Overlay radon concentration on SOM
radon_by_node <- tapply(county_data$log_radon_mean,
                        som_model$unit.classif, mean)
plot(som_model, type = "property",
     property = radon_by_node,
     main = "Mean Log-Radon by SOM Node")

# Hierarchical clustering on SOM codebook vectors
som_codes <- getCodes(som_model)[[1]]
hc <- hclust(dist(som_codes), method = "ward.D2")
som_clusters <- cutree(hc, k = 5)
```

### Random Forest Implementation

**Classification vs. Regression:**
- This is a **regression** problem. The target variable is log-transformed mean indoor radon concentration (continuous).
- An alternative formulation is classification into EPA zones (Zone 1/2/3), but regression provides more granular and useful predictions.

**Target Variable:** `log_radon_mean` (continuous, log pCi/L).

**Predictor Variables:** Same 12-20 features used for SOM, plus SOM-derived features (see integration below).

**Hyperparameter Tuning:**
- `ntree`: 500 to 2000. Plot OOB error vs. ntree to determine convergence.
- `mtry`: Default for regression is p/3. For p=15 predictors, default mtry=5. Search: 3, 5, 7, 10.
- `nodesize`: Default is 5 for regression. Test 3, 5, 10.
- Use `caret::train()` with method="rf" or `tuneRanger` for automated tuning.

**Cross-Validation:**
- 10-fold cross-validation (dataset is small enough).
- **Spatial cross-validation** (block by state or Census region) to test geographic generalization — this is important and publishable.
- OOB error as a quick internal validation.

**Feature Importance:**
- Permutation importance (%IncMSE) is preferred for regression.
- Plot importance of all features. Expected ranking: soil uranium > geology type > latitude > basement prevalence > soil permeability > temperature.
- Partial dependence plots for top 3-4 features to show nonlinear relationships.

**Performance Metrics:**
- R² (coefficient of determination) — expected 0.55-0.75 for county-level prediction.
- RMSE (root mean squared error) on log-radon scale.
- MAE (mean absolute error).
- Back-transform to original pCi/L scale for interpretable error reporting.
- Spatial residual map (plot prediction errors on a US county map to check for geographic bias).

**R Code Outline:**

```r
library(randomForest)
library(caret)

# Prepare data
model_data <- county_data[complete.cases(county_data), ]
model_data$geology_type <- as.factor(model_data$geology_type)

# Train/test split
set.seed(42)
train_idx <- createDataPartition(model_data$log_radon_mean,
                                  p = 0.7, list = FALSE)
train <- model_data[train_idx, ]
test  <- model_data[-train_idx, ]

# Train RF regression
rf_model <- randomForest(log_radon_mean ~ uranium_ppm + permeability +
                           geology_type + mean_temp + precipitation +
                           elevation + latitude + longitude +
                           pop_density + pct_basement + median_bldg_year,
                         data = train,
                         ntree = 1000,
                         mtry = 5,
                         importance = TRUE)

# Evaluate
print(rf_model)  # OOB R² and MSE
pred_test <- predict(rf_model, test)

# Metrics
R2 <- 1 - sum((test$log_radon_mean - pred_test)^2) /
          sum((test$log_radon_mean - mean(test$log_radon_mean))^2)
RMSE <- sqrt(mean((test$log_radon_mean - pred_test)^2))
MAE  <- mean(abs(test$log_radon_mean - pred_test))

cat("R²:", R2, "\nRMSE:", RMSE, "\nMAE:", MAE, "\n")

# Feature importance
varImpPlot(rf_model, main = "Feature Importance for Radon Prediction")

# Partial dependence plots
library(pdp)
partial(rf_model, pred.var = "uranium_ppm", plot = TRUE)
partial(rf_model, pred.var = "latitude", plot = TRUE)

# Hyperparameter tuning
tune_grid <- expand.grid(mtry = c(3, 5, 7, 10))
ctrl <- trainControl(method = "cv", number = 10)
rf_tuned <- train(log_radon_mean ~ ., data = train,
                  method = "rf", tuneGrid = tune_grid,
                  trControl = ctrl, ntree = 500)
```

### How SOM and RF Work Together (Novelty Angle)

1. **SOM for Exploratory Geospatial Typology:** The SOM creates a "typology" of US counties based on their geological, climatic, and housing characteristics. This unsupervised grouping may reveal natural "radon risk profiles" that are more nuanced than the EPA's 3-zone classification. For example, the SOM might identify 5-8 distinct county types.

2. **SOM Cluster as RF Feature:** The SOM cluster membership (e.g., cluster 1 through 5) is added as a new categorical predictor in the RF model. This encodes complex, nonlinear interactions among geological and environmental variables in a single feature.

3. **SOM Node Distances as RF Features:** The distance of each county to each SOM cluster centroid provides K continuous features that capture "similarity" to archetypical county profiles.

4. **Comparison Framework:**
   - Model A: RF with raw features only.
   - Model B: RF with raw features + SOM cluster membership.
   - Model C: RF with raw features + SOM node distances.
   - Compare R², RMSE across models A, B, C. Even a small improvement in Model B/C over Model A demonstrates the added value of unsupervised pre-processing.

5. **Interpretability Enhancement:** The SOM provides a visual "atlas" of county types. Policymakers can examine the SOM map to understand WHY a county has high predicted radon — not just from RF feature importance (which gives marginal effects) but from the holistic county profile (which SOM cluster it belongs to and what that cluster's characteristics are).

6. **Novelty Argument:** Previous radon prediction studies have used regression (linear, logistic), geostatistics (kriging), or single ML models (ANN, RF). The explicit SOM+RF pipeline — where SOM provides unsupervised county typology that feeds into RF — is novel in the radon literature. The geographic visualization of SOM clusters mapped onto the US is also a compelling and original contribution.

---

## 4. Expected Results & Figures

### Expected Figures and Tables

| Figure/Table | Description |
|---|---|
| Fig. 1 | US county map of mean indoor radon concentrations (choropleth) |
| Fig. 2 | Histogram of radon concentrations showing log-normal distribution, with log-transformed version |
| Fig. 3 | Correlation matrix heatmap of all predictor variables |
| Fig. 4 | SOM U-matrix showing county typology clusters |
| Fig. 5 | SOM component planes for 6-8 key variables (uranium, geology, temp, basement %, etc.) |
| Fig. 6 | SOM property map with mean radon overlay — showing which cluster types have highest radon |
| Fig. 7 | US county map color-coded by SOM cluster membership (geographic visualization) |
| Fig. 8 | RF feature importance plot (%IncMSE and IncNodePurity) |
| Fig. 9 | Partial dependence plots for top 4 features |
| Fig. 10 | Observed vs. predicted radon scatter plot (test set) with R² annotation |
| Fig. 11 | Spatial residual map (US counties colored by prediction error) |
| Fig. 12 | Model comparison bar chart (R², RMSE for Models A, B, C) |
| Table 1 | Dataset summary statistics (all variables) |
| Table 2 | SOM configuration and training parameters |
| Table 3 | SOM cluster profiles (mean value of each variable per cluster) |
| Table 4 | RF hyperparameters and tuning results |
| Table 5 | Model performance comparison (RF-only vs. SOM+RF variants) |

### Expected Results

- **SOM clusters:** Expect 4-7 meaningful clusters. Likely clusters include: (1) Northern granite/gneiss counties with high uranium and widespread basements (highest radon); (2) Midwestern glacial-till counties with moderate radon; (3) Southern/coastal counties with sandy soil, low uranium, few basements (lowest radon); (4) Rocky Mountain counties with high elevation and variable geology.
- **RF regression performance:**
  - R² = 0.55-0.75 for Model A (raw features)
  - R² = 0.58-0.78 for Model B (with SOM cluster feature)
  - RMSE on log-scale: 0.4-0.7 log(pCi/L)
- **Feature importance ranking:** Soil uranium > geology type > latitude > basement prevalence > soil permeability > temperature > elevation.
- **Model comparison:** SOM+RF should show a modest but statistically significant improvement over RF alone, particularly in reducing errors for counties in transition zones between EPA radon zones.

### Interpretation and Discussion Points

- The dominance of soil uranium in feature importance confirms the well-established geochemical control on radon.
- Basement prevalence as a top feature connects to radon entry mechanisms (soil-gas pressure differentials drive radon into basements).
- Latitude serving as a proxy for both climate (heating season length, which affects stack-effect radon entry) and geology (glacial vs. non-glacial regions).
- SOM clusters provide actionable risk profiles: public health officials can identify which "type" a county is and tailor mitigation recommendations.
- Spatial residuals should be examined for geographic clustering — if residuals cluster spatially, this indicates missing spatial predictors (possibly deep geological structures not captured in surface geology maps).

---

## 5. Target Journals

### Journal 1: Science of the Total Environment

- **Impact Factor:** ~8.2-9.8
- **Why it fits:** Publishes environmental monitoring, prediction, and public health studies extensively. Indoor radon prediction combining geological and environmental factors is squarely within scope. ML applications in environmental science are increasingly featured.
- **Review timeline:** 2-4 months (can be slow; high volume of submissions).
- **Open access:** Hybrid OA through Elsevier; ~$3,500 APC. Also publishes subscription articles.

### Journal 2: Journal of Environmental Radioactivity

- **Impact Factor:** ~2.5-3.0
- **Why it fits:** The most specialized journal for this exact topic. Regularly publishes indoor radon prediction studies, geogenic radon potential mapping, and ML applications for radon. Strong fit means higher acceptance probability.
- **Review timeline:** 2-3 months.
- **Open access:** Hybrid OA through Elsevier; ~$2,500-3,000 APC.

### Journal 3: International Journal of Environmental Research and Public Health (IJERPH)

- **Impact Factor:** ~3.4-4.6
- **Why it fits:** Open-access, covers environmental health including radon exposure. Accepts ML-based prediction studies. Faster review process.
- **Review timeline:** 1-2 months (MDPI journals are known for speed).
- **Open access:** Full OA (MDPI); ~$2,600 APC.

---

## 6. Strengths & Weaknesses

### Strengths

- **High public health relevance:** Radon is the second leading cause of lung cancer. This topic has real-world impact.
- **Excellent data availability:** All data is public domain, freely accessible, and well-documented by US federal agencies.
- **Small, manageable dataset:** ~3,000 rows x 20 columns. No computational bottlenecks. SOM and RF train in seconds.
- **Strong novelty for SOM+RF integration:** The county typology (SOM) feeding into prediction (RF) is a novel contribution in the radon literature. Most existing studies use either geostatistics or single ML models.
- **Rich visualization potential:** US county maps, SOM maps, component planes, partial dependence plots — the paper will be visually compelling.
- **Interdisciplinary appeal:** Bridges nuclear science, geology, public health, and machine learning. Appeals to multiple journal audiences.
- **Physical interpretability:** Every feature importance result can be connected to known radon geophysics.

### Weaknesses / Risks

- **Data assembly effort:** The biggest risk. Merging EPA, USGS, Census, and NOAA data by county FIPS code is tedious and error-prone. Mismatched FIPS codes, name variations, and missing counties are common.
- **County-level aggregation limits resolution:** Radon varies greatly within a county. County-level means may obscure important local variation. Reviewers may criticize this.
- **Potential for modest R² values:** County-level prediction of radon is inherently noisy. R² of 0.60-0.70 is realistic and respectable but may seem unimpressive to a student expecting >0.90.
- **US-specific data:** The study is entirely US-based, which limits direct applicability to Saudi Arabia (where the student is presumably based). This requires discussion in the paper.
- **Temporal mismatch:** Radon data may be from the 1990s-2000s, while geology and climate data are more recent. The assumption of temporal stability must be justified.

### Mitigation Strategies

- **Data assembly:** Look for pre-compiled radon datasets. The LBNL radon database and the dataset from Gelman & Hill's "Data Analysis Using Regression and Multilevel/Hierarchical Models" (which includes a famous radon dataset for Minnesota counties) can serve as starting points. The `rstanarm` R package includes a pre-cleaned version. Start with Minnesota as a proof of concept, then expand.
- **County-level limitation:** Frame this explicitly as a screening-level tool. Acknowledge that sub-county models would require more granular data. Cite existing county-level studies to establish precedent.
- **Modest R²:** Frame the contribution as the SOM typology and the interpretability, not just prediction accuracy. Compare favorably to existing EPA zone maps (which are just 3 categories).
- **US-specific:** Include a Discussion paragraph on transferability to other regions, particularly the Arabian Shield and its granite geology. Frame as a methodology that could be adapted with local data.

---

## 7. Recommendation Score

| Criterion | Score (out of 10) | Justification |
|---|---|---|
| **Publishability** | 8 | Strong fit for J. Environmental Radioactivity. Radon prediction is a popular topic. SOM+RF combination adds novelty. |
| **Feasibility** | 7 | Small dataset, straightforward ML. Main challenge is data assembly, which is tedious but not technically difficult. |
| **Novelty** | 7 | SOM county typology + RF prediction is new in radon literature. Not groundbreaking, but a solid incremental contribution. |
| **Data Access** | 8 | All data is public domain and freely available. Multiple sources must be merged, but each is individually accessible. |
| **Overall** | **7.5** | A well-balanced project with high publishability, manageable technical complexity, and strong public health relevance. Recommended for students who are comfortable with data wrangling. |

---

---

# Comparative Summary

| Criterion | Topic D (Radionuclide ID) | Topic E (Indoor Radon) |
|---|---|---|
| **Publishability** | 8/10 | 8/10 |
| **Feasibility** | 6/10 | 7/10 |
| **Novelty** | 8/10 | 7/10 |
| **Data Access** | 5/10 | 8/10 |
| **Overall** | **6.5/10** | **7.5/10** |
| **Best For** | Students with gamma spectroscopy background | Students comfortable with data wrangling |
| **Primary Risk** | Finding suitable spectral datasets | Assembling multi-source county dataset |
| **Difficulty** | Hard | Moderate |

### Final Guidance

**Topic E (Indoor Radon Prediction)** is the safer and more feasible choice for an undergraduate completing a project in 8 weeks. The data is guaranteed to be accessible (all public domain, US federal sources), the dataset is small and computationally manageable, and the public health angle makes for a compelling narrative. The data assembly effort is the main burden, but it is predictable and methodical — there are no surprises.

**Topic D (Radionuclide Identification)** has higher novelty potential and is more directly aligned with nuclear engineering, but carries significant risk around data access. If the student cannot find a suitable, well-documented Kaggle dataset within the first week, the project timeline compresses dangerously. The high dimensionality (1024+ spectral channels) also adds technical complexity. This topic is recommended only if the student has prior experience with gamma spectroscopy or if a suitable dataset is confirmed before committing.

For a student who must deliver a journal-quality paper within 8 weeks and wants to minimize risk while maximizing publishability, **Topic E is the recommended choice**.