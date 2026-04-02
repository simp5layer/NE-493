# Fukushima Contamination Regime Discovery — Project Context

This project develops a two-stage machine learning pipeline (SOM → RF) to discover and explain distinct dose-rate decay "regimes" across the Fukushima exclusion zone.

## 🎯 Research Objective
The goal is to move beyond continuous regression (predicting dose rates) to behavioral classification. We discover discrete regimes (e.g., rapid urban washoff vs. slow forest retention) using unsupervised learning and then explain them using environmental predictors.

## 🛠️ Project Structure & Technologies

### Architecture
1.  **Stage 1 (Unsupervised):** SOM trained on 9 temporal features to discover clusters.
2.  **Stage 2 (Supervised):** Random Forest classifier using SOM labels as targets and 12 environmental predictors (LULC, elevation, soil type, etc.) as features.
3.  **Explainability:** SHAP values to interpret the drivers of each regime.

### Tech Stack
-   **Data Engineering:** Python 3.10+ (Pandas, PyArrow, NumPy).
-   **Machine Learning:** R (Packages: `kohonen` for SOM, `ranger`/`randomForest` for RF, `SHAPforxgboost` or similar for XAI).
-   **Data Storage:** Parquet (for high-volume time series and feature matrices).

## 🏃 Building and Running

### Environment Setup
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Data Pipeline
1.  **Acquisition:** `python download_emdb.py`
2.  **Filtration:** `python filter_emdb.py` (Applies 80km cutoff, dose thresholds, and temporal completeness filters).
3.  **Feature Engineering:** `python engineer_features.py` (Extracts 9 temporal features per mesh cell).

### ML Pipeline (Active Phase)
-   **SOM Clustering:** [TODO: Implement in R]
-   **RF Classification:** [TODO: Implement in R]

## 📜 Development Conventions

### Data Constraints
-   **Radiation Data:** JAEA EMDB only. Safecast data is excluded due to systematic bias (overestimation at low dose, underestimation at high dose).
-   **Filtration:** Standard 80km distance cutoff from FDNPP.
-   **Features:** 9 temporal features (IQR-scaled and clipped to `[-3, 3]`).

### ML Implementation Constraints
-   **Language:** SOM, RF, and SHAP **must** be implemented in R.
-   **SOM Config:** 5x5 or 6x6 grid, hexagonal topology, 3-5 random seeds for stability testing.
-   **Feature Handling:** **No PCA** allowed on the 9 features (destroys SHAP interpretability).
-   **Value Proposition:** Focus on interpretive framing and "discovery" rather than chasing R² metrics.

## 📂 Key Files
-   `research-objective.md`: Master document for literature, datasets, and design.
-   `CLAUDE.md`: Active phase status and technical constraints.
-   `engineer_features.py`: Logic for the 9 temporal features.
-   `output/feature_matrix.parquet`: The 138k-row input for the SOM.
-   `docs/NE493_detailed_report.md`: Original project proposal and topic comparison.
