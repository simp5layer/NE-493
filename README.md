# Fukushima Contamination Regime Discovery

**Discovery of Behavioral Contamination Regimes in the Fukushima Exclusion Zone Using a Self-Organizing Map and Random Forest Pipeline**

A two-stage unsupervised-to-supervised ML pipeline (SOM followed by RF) that discovers distinct dose-rate decay regimes across the Fukushima exclusion zone without imposing prior categories, then identifies which environmental factors predict regime membership.

## Project Phases

| Phase | Description | Status |
|-------|-------------|--------|
| 1. Data Acquisition | Download JAEA EMDB air dose rate datasets | Complete |
| 2. Data Filtration | Six-stage filter chain (distance, dose, temporal, etc.) | Complete |
| 3. Feature Engineering | Build temporal feature vectors per mesh cell | Complete |
| 4. SOM Clustering | Train 5x5/6x6 SOM on temporal dose-rate features | **Complete** |
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
| `som_clustering.R` | 4 | Trains 5x5 SOM, discovers 6 contamination regimes via hierarchical clustering |

Run any script with `--help` for usage options, or `--dry-run` to preview without processing.

## Key Results

Phase 4 (SOM Clustering) identified 6 distinct contamination decay regimes across 138,768 mesh cells in the Fukushima exclusion zone via Ward's D2 hierarchical clustering:

1. **Slow Retention (9.9%)** — High dose retention in forested/mountainous terrain distant from FDNPP, characterized by slow exponential decay with half-life ~13.8 years
2. **Rapid Decay (6.2%)** — Biphasic decay pattern with fast initial washoff, half-life ~9.3 years
3. **Standard Decay (55.3%)** — Modal behavior across all temporal features, near-average characteristics across the exclusion zone
4. **Recontamination/Anomalous (7.3%)** — Frequent dose increases and reversals, cells closest to FDNPP with initial dose 2.57 µSv/h
5. **Consistent Exponential Decay (8.3%)** — Clean exponential fit (R²=0.84) with fewest reversals, half-life ~4.7 years
6. **High Variability (13.1%)** — Erratic behavior near reactor, highest initial dose (7.66 µSv/h), residual coefficient of variation 0.77

## Figures

| Figure | Description |
|--------|-------------|
| fig01_geographic_regime_map | Spatial distribution of 6 contamination regimes across exclusion zone |
| fig02_centroid_heatmap | Feature values at cluster centroids (9 features × 6 regimes) |
| fig03_parallel_coordinates | Temporal profile trajectories by regime |
| fig04_component_planes | Component plane visualizations from 5x5 SOM |
| fig05_umatrix | Unified distance matrix highlighting cluster boundaries |
| fig06_convergence | SOM training convergence across 5 random seeds |
| fig07_node_counts | Distribution of mesh cells across 25 SOM nodes |
| fig08_dendrogram | Ward's D2 hierarchical clustering tree (k=6 cutoff) |
| fig09_silhouette | Silhouette scores validating 6-cluster solution |
| fig10_optimal_k | Elbow plot and gap statistic for k selection |
| fig11_seed_stability | Cluster stability across 5 independent SOM seeds |
| fig12_cluster_sizes | Proportion of mesh cells in each of 6 regimes |
| fig13_cluster_boxplots | Temporal feature distributions by regime |

## Documentation

- `research-objective.md` -- Master document: literature landscape, datasets, pipeline design, expected results
- `docs/data-filtration-phase.md` -- 80 km distance cutoff justification and empirical analysis
- `docs/filtration-pipeline-spec.md` -- Technical specification for the filtration pipeline
- `docs/feasibility-and-novelty-check.md` -- Novelty assessment and Safecast exclusion rationale
- `docs/NE493_detailed_report.md` -- Comprehensive project report
