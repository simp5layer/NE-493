# Fukushima Contamination Regime Discovery

**Discovery of Behavioral Contamination Regimes in the Fukushima Exclusion Zone Using a Self-Organizing Map and Random Forest Pipeline**

A two-stage unsupervised-to-supervised ML pipeline (SOM followed by RF) that discovers distinct dose-rate decay regimes across the Fukushima exclusion zone without imposing prior categories, then identifies which environmental factors predict regime membership.

## Project Phases

| Phase | Description | Status |
|-------|-------------|--------|
| 1. Data Acquisition | Download JAEA EMDB air dose rate datasets | Complete |
| 2. Data Filtration | Six-stage filter chain (distance, dose, temporal, etc.) | Complete |
| 3. Feature Engineering | Build temporal feature vectors per mesh cell | Complete |
| 4. SOM Clustering | Train 5x5/6x6 SOM on temporal dose-rate features | In Progress |
| 5. RF Classification | Random Forest with SOM cluster labels as targets | Upcoming |
| 6. SHAP Explainability | Feature importance and regime interpretation | Upcoming |

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

## Pipeline Scripts

| Script | Phase | Description |
|--------|-------|-------------|
| `download_emdb.py` | 1 | Downloads JAEA EMDB air dose rate datasets |
| `filter_emdb.py` | 2 | Six-stage filtration: 14.5M rows to 9.2M rows (138K mesh cells) |
| `engineer_features.py` | 3 | Extracts 9 temporal features per mesh cell |

Run any script with `--help` for usage options, or `--dry-run` to preview without processing.

## Documentation

- `research-objective.md` -- Master document: literature landscape, datasets, pipeline design, expected results
- `docs/data-filtration-phase.md` -- 80 km distance cutoff justification and empirical analysis
- `docs/filtration-pipeline-spec.md` -- Technical specification for the filtration pipeline
- `docs/feasibility-and-novelty-check.md` -- Novelty assessment and Safecast exclusion rationale
- `docs/NE493_detailed_report.md` -- Comprehensive project report
