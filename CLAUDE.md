# Fukushima SOM-RF Pipeline

## Current Phase: 5 — RF Classification

### Active Context
- `som_clustering.R` — SOM training script and cluster discovery outputs
- `output/som_clusters.rds` — 6-cluster assignment for 138,768 mesh cells
- `output/cluster_profiles.csv` — Centroids and statistics for 6 contamination regimes
- `figures/` — 13 publication-ready manuscript figures

### Not Needed This Phase
- `docs/data-filtration-phase.md` — Phase 2 reference
- `docs/filtration-pipeline-spec.md` — Phase 2 reference
- `download_emdb.py`, `filter_emdb.py` — Phase 1-2 scripts

### Implementation Constraints
- SOM, RF, and SHAP must be implemented in R (kohonen, ranger, SHAP packages)
- SOM grid: 5x5 or 6x6, test 3-5 random seeds
- Validation: quantization error + topographic error
- No PCA on 9 features (destroys SHAP interpretability)
- Value is in interpretive framing, not chasing accuracy

### Phase Roadmap
| Phase | Status |
|-------|--------|
| 1. Data Acquisition | Complete |
| 2. Data Filtration | Complete |
| 3. Feature Engineering | Complete |
| 4. SOM Clustering | Complete |
| 5. RF Classification | **Active** |
| 6. SHAP Explainability | Upcoming |

*Note: Phases 5 & 6 may be deferred to second manuscript after Phase 4 publication.*
