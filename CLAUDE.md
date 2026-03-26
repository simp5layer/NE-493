# Fukushima SOM-RF Pipeline

## Current Phase: 4 — SOM Clustering

### Active Context
- `research-objective.md` — SOM approach (Stage 1 section), expected results
- `output/feature_matrix.parquet` — 138,768 rows x 9 SOM features (IQR-scaled, clipped [-3, 3])
- `output/feature_summary.json` — Per-feature statistics and scaling parameters

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
| 4. SOM Clustering | **Active** |
| 5. RF Classification | Upcoming |
| 6. SHAP Explainability | Upcoming |
