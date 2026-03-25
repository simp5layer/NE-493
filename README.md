# Fukushima Contamination Regime Discovery

**Discovery of Behavioral Contamination Regimes in the Fukushima Exclusion Zone Using a Self-Organizing Map and Random Forest Pipeline**

A two-stage unsupervised-to-supervised ML pipeline (SOM followed by RF) that discovers distinct dose-rate decay regimes across the Fukushima exclusion zone without imposing prior categories, then identifies which environmental factors predict regime membership. This shifts the analytical framing from continuous regression to discrete behavioral classification.

## Project Phases

| Phase | Description | Status |
|-------|-------------|--------|
| 1. Data Acquisition | Download JAEA EMDB air dose rate datasets | Complete |
| 2. Data Filtration | Six-stage filter chain (distance, dose, temporal, etc.) | Complete |
| 3. Feature Engineering | Build temporal feature vectors per mesh cell | Complete |
| 4. SOM Clustering | Train 5x5/6x6 SOM on temporal dose-rate features | **Current** |
| 5. RF Classification | Random Forest with SOM cluster labels as targets | Upcoming |
| 6. SHAP Explainability | Feature importance and regime interpretation | Upcoming |

## Phase 1: Data Acquisition Pipeline

### Data Source

JAEA Environmental Radioactivity Monitoring Database (EMDB)
https://radioactivity.nra.go.jp/emdb/

The pipeline downloads all **air dose rate** measurement datasets (large classification), covering all 7 middle classifications:

| Code | Middle Classification | Description |
|------|----------------------|-------------|
| 02 | Drive Survey | KURAMA car-borne surveys |
| 03 | Walk Survey | Pedestrian dose-rate surveys |
| 04 | Survey Meter | Fixed-point ambient dose equivalent rate |
| 45 | Airborne Monitoring | Helicopter and drone surveys |
| 50 | Background | Background radiation maps |
| 60 | Monitoring Post | Fixed monitoring stations (real-time) |
| 78 | Other | Miscellaneous air dose rate data |

**Scope**: 70 measurement items x 14 fiscal years (2011-2024) = up to 980 ZIP files.

### Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Usage

```bash
# Preview what will be downloaded
python download_emdb.py --dry-run

# Run the full download (~35 minutes)
python download_emdb.py

# Run unattended (prevents Mac sleep)
caffeinate -i nohup python download_emdb.py > /dev/null 2>&1 &
tail -f download.log

# Resume after interruption (automatic)
python download_emdb.py

# Download a specific year range
python download_emdb.py --year-start 2020 --year-end 2024
```

### Dataset Structure

```
dataset/
├── manifest.json
├── 02_drive_survey/
│   └── item_087/
│       ├── 2011.zip
│       ├── 2012.zip
│       └── ...
├── 03_walk_survey/
├── 04_survey_meter/
├── 45_airborne_monitoring/
├── 50_background/
├── 60_monitoring_post/
└── 78_other/
```

### Features

- **Resumable**: State file tracks progress; re-running skips completed downloads
- **Polite**: 2-second delay between requests to respect the government server
- **Robust**: Exponential backoff retry on server errors and timeouts
- **Validated**: ZIP magic byte verification before saving files
- **Observable**: Dual logging (console + `download.log`) with progress counters

## Phase 2: Data Filtration Pipeline

**Status:** **Current**

**File:** `filter_emdb.py` (1559 lines, 52.7 KB)

### What it does

Processes all 463 ZIP files from JAEA EMDB, applying a six-stage filter chain to transform 14.5M raw air dose rate measurements into a SOM-ready dataset of 9.2M rows across 138K mesh cells.

### Filter Stages

0. **Preprocessing** — Drop non-air-dose rows, remove missing coordinates, deduplicate, normalize columns
1. **Distance Filter** — Keep measurements ≤ 80 km from FDNPP (literature standard: Andoh et al. 2020, Wainwright et al. 2018)
2. **Detection Limit** — Exclude records where value ≤ detection lower limit
3. **Dose Threshold** — Per-mesh filter: drop entire mesh cell if initial measurement (earliest date) ≤ 0.10 µSv/h
4. **Temporal Completeness** — Require ≥ 4 distinct measurement dates per mesh cell
5. **Measurement Type Consistency** — One type per mesh cell; majority wins (ties: Walk > Car > AirBorne > 定点)
6. **Height Flagging** — Non-destructive `height_anomalous` flag for measurements ≠ 100 cm

### Usage

```bash
# Full pipeline (default: 80 km, 0.10 µSv/h threshold, ≥4 dates)
python filter_emdb.py

# Preview file discovery without loading data
python filter_emdb.py --dry-run

# Also export CSV alongside Parquet
python filter_emdb.py --csv

# Sensitivity analysis at 0.10 and 0.15 µSv/h thresholds
python filter_emdb.py --sensitivity

# Custom parameters
python filter_emdb.py --distance-cutoff 50.0 --dose-threshold 0.15 --min-temporal-points 6
```

### Output Files

All output goes to the `output/` directory:

- **`filtered_emdb.parquet`** — Snappy-compressed Parquet, 9.2M rows (load with `pd.read_parquet()`)
- **`filtered_emdb.csv`** — UTF-8-BOM CSV (optional, via `--csv` flag)
- **`filter_summary.json`** — Per-stage row counts, per-item breakdowns, parameters used
- **`filter.log`** — Detailed processing log with per-file diagnostics

### Pipeline Results

```
Input:  14.5M rows from 463 ZIPs
Output:  9.2M rows across 138K mesh cells and 52 data sources

By measurement type:
  - AirBorne:  3.4M rows (37%)
  - Fixed (定点):  2.1M rows (23%)
  - Walk:  1.9M rows (21%)
  - Car:  1.7M rows (19%)

Data sources:
  - NRA (Nuclear Regulation Authority): 6.8M rows
  - Local municipalities: 2.3M rows
  - TEPCO: 400K rows

Geographic coverage:
  - Distance range: 0–80 km from FDNPP
  - Mesh cells: 138,768 (250m grid)
  - Dose rate range: 0–130 µSv/h
  - Year range: 2011–2025
```

### Design Rationale

**80 km boundary:** Matches JAEA EMDB standard, captures full NW contamination plume (extends ~50 km to Iitate), consistent monitoring density within zone (follows Andoh, Wainwright, MEXT).

**Mesh-level dose threshold:** Eliminates background-noise locations (~2.5x Japan's pre-accident natural background of 0.04 µSv/h) while preserving the full contamination signal.

**Parquet + CSV:** Fast columnar access and compression (Parquet) with transparency and external tool compatibility (CSV).

## Phase 3: Feature Engineering Pipeline

**Status:** **Complete**

**File:** `engineer_features.py` (1,444 lines, 52.1 KB)

### What it does

Transforms the 9.2M row filtered EMDB dataset into 138,768 mesh-cell temporal feature vectors suitable for SOM clustering. Aggregates measurements to daily medians per 250m mesh cell, then extracts 9 robust SOM features capturing dose-rate decay dynamics, temporal span, recontamination signals, and measurement consistency.

### Feature Set

The pipeline computes 9 features per mesh cell:

1. **total_decay_ratio** — Ratio of first to last daily median (captures initial deposition + total decay)
2. **log_linear_slope** — OLS slope of ln(dose-rate) vs time (exponential decay coefficient)
3. **log_linear_r²** — Goodness-of-fit for exponential model (temporal coherence)
4. **early_slope** — Slope in early period (2011-03-11 to 2013-03-11, Cs-134 half-life)
5. **late_slope** — Slope in late period (2013-03-11 onward, Cs-137 dominated)
6. **residual_cv** — Coefficient of variation of log-linear residuals (noise level)
7. **slope_ratio** — Ratio of early to late slope (regime transition indicator)
8. **n_increases** — Count of days with dose increase vs prior day (recontamination proxy)
9. **temporal_span_years** — Measurement interval in years (data coverage width)

All features are scaled using robust IQR-based normalization to handle outliers, then clipped to ±3σ for SOM input.

### Usage

```bash
# Full pipeline (default: output to output/)
python engineer_features.py

# Preview without processing
python engineer_features.py --dry-run

# Also export feature matrix as CSV
python engineer_features.py --csv

# Verbose logging
python engineer_features.py --verbose

# Custom input/output paths
python engineer_features.py --input output/filtered_emdb.parquet --output-dir output/

# Custom recontamination threshold (50% increase default)
python engineer_features.py --early-late-cutoff 2013-03-11
```

### Output Files

All output goes to the `output/` directory:

- **`feature_matrix.parquet`** — Snappy-compressed Parquet: 138,768 rows × 37 columns (9 SOM features + geometry + metadata)
- **`feature_matrix.csv`** — UTF-8 CSV (optional, via `--csv` flag)
- **`feature_summary.json`** — Per-feature statistics, scaling parameters, pipeline metadata
- **`feature_engineering.log`** — Detailed processing log with per-mesh diagnostics

### Pipeline Results

```
Input:  9.2M rows from 138K mesh cells
Output: 138,768 feature vectors (37 columns)

Feature statistics:
  - Decay ratio: median 2.4, range 0.1–180
  - Log-linear R²: median 0.91 (high temporal coherence)
  - Temporal span: median 6.2 years, range 0.1–14
  - Recontaminated cells: 12.3% flagged
  - Measurement consistency: 98.6% single-type cells

Data completeness:
  - Daily medians: 9.2M → 2.1M unique mesh-day combinations
  - Complete temporal series: 138,768 mesh cells
```

---

## Technical Details

For detailed specifications, rationale, and literature justification, see:
- `filtration-pipeline-spec.md` — Complete technical specification
- `data-filtration-phase.md` — Distance cutoff justification and literature review
