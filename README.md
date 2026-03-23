# Fukushima Contamination Regime Discovery

**Discovery of Behavioral Contamination Regimes in the Fukushima Exclusion Zone Using a Self-Organizing Map and Random Forest Pipeline**

A two-stage unsupervised-to-supervised ML pipeline (SOM followed by RF) that discovers distinct dose-rate decay regimes across the Fukushima exclusion zone without imposing prior categories, then identifies which environmental factors predict regime membership. This shifts the analytical framing from continuous regression to discrete behavioral classification.

## Project Phases

| Phase | Description | Status |
|-------|-------------|--------|
| 1. Data Acquisition | Download JAEA EMDB air dose rate datasets | **Current** |
| 2. Data Preprocessing | Clean, merge, and engineer temporal features | Upcoming |
| 3. SOM Clustering | Train 5x5/6x6 SOM on temporal dose-rate features | Upcoming |
| 4. RF Classification | Random Forest with SOM cluster labels as targets | Upcoming |
| 5. SHAP Explainability | Feature importance and regime interpretation | Upcoming |

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
