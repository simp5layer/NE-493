# SOM Clustering Pipeline -- Architecture Specification
## Fukushima Contamination Regime Discovery

**Version:** 1.0
**Date:** 2026-04-01
**Phase:** 4 of 6 (SOM Clustering)
**Target script:** `som_clustering.R`
**Upstream input:** `output/feature_matrix.parquet` (138,768 rows x 37 columns, of which 9 are IQR-scaled SOM features)

---

## Table of Contents

1. [Input Data Contract](#1-input-data-contract)
2. [R Package Stack](#2-r-package-stack)
3. [SOM Configuration and Training](#3-som-configuration-and-training)
4. [Quality Metrics](#4-quality-metrics)
5. [Topographic Error -- Custom Implementation](#5-topographic-error----custom-implementation)
6. [Seed Stability Analysis](#6-seed-stability-analysis)
7. [Model Selection Protocol](#7-model-selection-protocol)
8. [Post-SOM Hierarchical Clustering](#8-post-som-hierarchical-clustering)
9. [Back-Transformation to Original Units](#9-back-transformation-to-original-units)
10. [Auto-Labeling Regime Names](#10-auto-labeling-regime-names)
11. [Output File Specifications](#11-output-file-specifications)
12. [Figure Specifications](#12-figure-specifications)
13. [Figure Formatting Standards](#13-figure-formatting-standards)
14. [Physical Context for Interpretation](#14-physical-context-for-interpretation)
15. [Execution Order and Runtime Estimates](#15-execution-order-and-runtime-estimates)

---

## 1. Input Data Contract

### 1.1 Source File

```
output/feature_matrix.parquet
```

- **Rows:** 138,768 (one per 250m mesh cell)
- **Total columns:** 37 (metadata + raw features + z-scored features)
- **SOM input columns (9):** All suffixed with `_z`, IQR-scaled, clipped to [-3, 3]

### 1.2 SOM Feature Columns (Exact Names)

These are the ONLY columns fed to the SOM. Order matters -- use this order consistently in all code, figures, and tables.

| Position | Column Name | Human-Readable Label | Physical Meaning |
|----------|------------|---------------------|------------------|
| 1 | `total_decay_ratio_z` | Total Decay Ratio | Ratio of final to initial dose rate |
| 2 | `log_linear_slope_z` | Log-Linear Slope (day^-1) | Overall exponential decay rate |
| 3 | `log_linear_r2_z` | Log-Linear R^2 | Goodness-of-fit of exponential model |
| 4 | `early_slope_z` | Early Phase Slope (day^-1) | Decay rate in first 2 years (pre-2013-03-11) |
| 5 | `late_slope_z` | Late Phase Slope (day^-1) | Decay rate after 2 years |
| 6 | `residual_cv_z` | Residual CV | Coefficient of variation of residuals from log-linear fit |
| 7 | `slope_ratio_z` | Early/Late Slope Ratio | Ratio of early to late decay rate |
| 8 | `n_increases_z` | Number of Increases | Count of time periods where dose rate increased |
| 9 | `temporal_span_years_z` | Observation Span (years) | Duration of monitoring record |

### 1.3 Normalization Parameters

Stored in `output/feature_summary.json` under `normalization_params`. Each feature has:

```json
{
  "feature_name": {
    "median": <float>,
    "q25": <float>,
    "q75": <float>,
    "iqr": <float>
  }
}
```

The z-score transformation was: `z = (raw - median) / IQR`, then clipped to [-3, 3].

### 1.4 Loading in R

```r
library(arrow)
library(data.table)
library(jsonlite)

# Load full matrix (all 37 columns preserved for output)
dt <- as.data.table(read_parquet("output/feature_matrix.parquet"))

# Extract SOM input matrix (must be a plain numeric matrix, not data.frame)
som_features <- c(
  "total_decay_ratio_z", "log_linear_slope_z", "log_linear_r2_z",
  "early_slope_z", "late_slope_z", "residual_cv_z",
  "slope_ratio_z", "n_increases_z", "temporal_span_years_z"
)
som_matrix <- as.matrix(dt[, ..som_features])

# Verify dimensions
stopifnot(nrow(som_matrix) == 138768)
stopifnot(ncol(som_matrix) == 9)
stopifnot(all(som_matrix >= -3 & som_matrix <= 3))

# Load normalization params for back-transformation
params <- fromJSON("output/feature_summary.json")
norm_params <- params$normalization_params
```

### 1.5 Pre-Training Sanity Checks

Before training, verify:

1. No `NA` values in `som_matrix` (SOM cannot handle missing data)
2. No constant columns (std > 0 for all 9 features)
3. All values within [-3, 3] (the clipping bounds)
4. Print the correlation matrix of the 9 features to the console (for reference; do NOT decorrelate or apply PCA)

```r
stopifnot(!anyNA(som_matrix))
stopifnot(all(apply(som_matrix, 2, sd) > 0))
stopifnot(all(som_matrix >= -3 & som_matrix <= 3))
cat("Feature correlation matrix:\n")
print(round(cor(som_matrix), 3))
```

---

## 2. R Package Stack

### 2.1 Required Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `arrow` | >= 14.0 | Read/write Parquet files |
| `data.table` | >= 1.15 | Fast data manipulation |
| `kohonen` | >= 3.0.12 | SOM training (the `som()` function) |
| `cluster` | >= 2.1 | Silhouette widths, gap statistic |
| `mclust` | >= 6.0 | Adjusted Rand Index (`adjustedRandIndex()`) |
| `ggplot2` | >= 3.5 | All figures |
| `viridis` | >= 0.6 | Continuous color scales |
| `patchwork` | >= 1.2 | Multi-panel figure assembly |
| `jsonlite` | >= 1.8 | Read/write JSON |
| `ggspatial` | >= 0.4 | Scale bar, north arrow on geographic plots |
| `RColorBrewer` | >= 1.1 | Categorical cluster palettes |
| `scales` | >= 1.3 | Axis formatting helpers |

### 2.2 Installation

```r
install.packages(c(
  "arrow", "data.table", "kohonen", "cluster", "mclust",
  "ggplot2", "viridis", "patchwork", "jsonlite",
  "ggspatial", "RColorBrewer", "scales"
))
```

### 2.3 Package Loading Block

Place at the top of the script:

```r
suppressPackageStartupMessages({
  library(arrow)
  library(data.table)
  library(kohonen)
  library(cluster)
  library(mclust)
  library(ggplot2)
  library(viridis)
  library(patchwork)
  library(jsonlite)
  library(ggspatial)
  library(RColorBrewer)
  library(scales)
})
```

---

## 3. SOM Configuration and Training

### 3.1 Grid Specifications

Train SOMs on two grid sizes:

| Parameter | Value |
|-----------|-------|
| Topology | Hexagonal (`topo = "hexagonal"`) |
| Grid sizes | 5x5 (25 neurons) and 6x6 (36 neurons) |
| Neighborhood | Gaussian (default in kohonen) |

```r
grid_5x5 <- somgrid(xdim = 5, ydim = 5, topo = "hexagonal")
grid_6x6 <- somgrid(xdim = 6, ydim = 6, topo = "hexagonal")
```

### 3.2 Training Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `rlen` | 2000 | Number of training iterations; sufficient for 138k observations on a small grid |
| `alpha` | `c(0.05, 0.01)` | Learning rate decays linearly from 0.05 to 0.01 over training |
| `dist.fcts` | `"euclidean"` | Standard for IQR-scaled features with comparable ranges |
| `mode` | `"online"` | Default kohonen mode; presents one observation at a time |
| `keep.data` | `TRUE` | Required for post-hoc analysis |

### 3.3 Random Seeds

Five seeds per grid size, totaling 10 SOM trainings:

```r
seeds <- c(42L, 123L, 456L, 789L, 2024L)
grid_sizes <- list(
  "5x5" = somgrid(xdim = 5, ydim = 5, topo = "hexagonal"),
  "6x6" = somgrid(xdim = 6, ydim = 6, topo = "hexagonal")
)
```

### 3.4 Training Loop

```r
results <- list()

for (grid_name in names(grid_sizes)) {
  grid <- grid_sizes[[grid_name]]
  for (seed in seeds) {
    run_id <- paste0(grid_name, "_seed", seed)
    cat(sprintf("Training %s ...\n", run_id))

    set.seed(seed)
    som_model <- som(
      som_matrix,
      grid = grid,
      rlen = 2000,
      alpha = c(0.05, 0.01),
      dist.fcts = "euclidean",
      keep.data = TRUE
    )

    results[[run_id]] <- list(
      grid_name = grid_name,
      seed = seed,
      model = som_model
    )
  }
}
```

### 3.5 Convergence Monitoring

Each `som_model$changes` is a numeric vector of length `rlen` (2000). Each element is the mean distance from observations to their BMU at that iteration.

**Convergence criterion:** The changes vector should plateau (flatten) by approximately iteration 1500. Verify this by checking that the relative change between the mean of the last 100 iterations and the mean of iterations 1400-1500 is less than 1%.

```r
check_convergence <- function(model, run_id) {
  changes <- model$changes[, 1]
  late_mean <- mean(changes[1901:2000])
  mid_mean <- mean(changes[1400:1500])
  rel_change <- abs(late_mean - mid_mean) / mid_mean
  converged <- rel_change < 0.01

  cat(sprintf("  %s: final_change=%.6f, rel_delta=%.4f, converged=%s\n",
              run_id, late_mean, rel_change, converged))

  if (!converged) {
    warning(sprintf("Model %s may not have converged (rel_change=%.4f > 0.01)",
                    run_id, rel_change))
  }

  return(list(converged = converged, rel_change = rel_change,
              final_change = late_mean))
}
```

If any model fails convergence, log a warning but do NOT increase `rlen` automatically. Report it in the metrics output and let the analyst decide.

---

## 4. Quality Metrics

### 4.1 Quantization Error (QE)

**Definition:** Mean Euclidean distance from each observation to its Best Matching Unit (BMU) codebook vector.

**Computation:** kohonen stores these distances directly.

```r
compute_qe <- function(model) {
  mean(model$distances[, 1])
}
```

**Interpretation:** Lower QE = better representation of the data. QE is scale-dependent (depends on feature ranges), so compare across models trained on the same data only. There is no universal threshold; use for relative comparison.

### 4.2 Topographic Error (TE)

**Definition:** Proportion of observations whose first and second BMUs are NOT adjacent on the hexagonal grid.

**Target:** TE < 0.10 (i.e., at least 90% of observations have adjacent 1st and 2nd BMUs).

**Implementation:** See Section 5 (requires custom code; NOT built into the kohonen package).

### 4.3 Metric Summary Table

For each of the 10 trainings, record:

| Field | Type | Source |
|-------|------|--------|
| `grid_name` | string | "5x5" or "6x6" |
| `seed` | integer | The random seed used |
| `qe` | float | `compute_qe(model)` |
| `te` | float | `compute_te(model)` (Section 5) |
| `converged` | boolean | From convergence check |
| `rel_change` | float | From convergence check |

---

## 5. Topographic Error -- Custom Implementation

This is the most algorithmically complex component. The kohonen package does NOT provide topographic error. It must be implemented from scratch.

### 5.1 Algorithm Overview

For each of the 138,768 observations:
1. Compute the Euclidean distance from that observation to every codebook vector (25 for 5x5, 36 for 6x6)
2. Find the index of the 1st BMU (smallest distance) and 2nd BMU (second smallest distance)
3. Determine whether the 1st and 2nd BMU are adjacent on the hexagonal grid
4. TE = count(non-adjacent pairs) / 138,768

### 5.2 Hexagonal Adjacency Function

Hexagonal grid adjacency depends on the **row parity** of the neuron. In the kohonen package, grid coordinates are stored in `model$grid$pts` as a two-column matrix (columns named `x` and `y`). However, for hex adjacency calculation, convert to integer grid coordinates (column, row) starting from (1,1).

The kohonen package uses an "offset coordinates" hex layout. Adjacency offsets depend on whether the row index is even or odd:

**Odd rows (row = 1, 3, 5, ...):**

| Direction | (dcol, drow) |
|-----------|-------------|
| Right | (+1, 0) |
| Left | (-1, 0) |
| Upper-right | (0, -1) |
| Upper-left | (-1, -1) |
| Lower-right | (0, +1) |
| Lower-left | (-1, +1) |

**Even rows (row = 2, 4, 6, ...):**

| Direction | (dcol, drow) |
|-----------|-------------|
| Right | (+1, 0) |
| Left | (-1, 0) |
| Upper-right | (+1, -1) |
| Upper-left | (0, -1) |
| Lower-right | (+1, +1) |
| Lower-left | (0, +1) |

### 5.3 Implementation

```r
build_adjacency_set <- function(grid) {
  # Extract grid dimensions
  xdim <- grid$xdim
  ydim <- grid$ydim

  # Build integer grid coordinates (col, row) for each neuron
  # kohonen lays out neurons column-major: neuron 1 is (col=1, row=1),
  # neuron 2 is (col=1, row=2), etc.
  coords <- expand.grid(col = 1:xdim, row = 1:ydim)

  n_neurons <- nrow(coords)
  # Build an adjacency list: for each neuron index, store the set of
  # adjacent neuron indices
  adj <- vector("list", n_neurons)

  for (i in 1:n_neurons) {
    c_i <- coords$col[i]
    r_i <- coords$row[i]

    if (r_i %% 2 == 1) {
      # Odd row offsets
      offsets <- matrix(c(
         1,  0,
        -1,  0,
         0, -1,
        -1, -1,
         0,  1,
        -1,  1
      ), ncol = 2, byrow = TRUE)
    } else {
      # Even row offsets
      offsets <- matrix(c(
         1,  0,
        -1,  0,
         1, -1,
         0, -1,
         1,  1,
         0,  1
      ), ncol = 2, byrow = TRUE)
    }

    neighbors <- integer(0)
    for (j in 1:nrow(offsets)) {
      nc <- c_i + offsets[j, 1]
      nr <- r_i + offsets[j, 2]
      if (nc >= 1 && nc <= xdim && nr >= 1 && nr <= ydim) {
        # Convert (col, row) back to linear index
        # Linear index = (col - 1) * ydim + row  [column-major]
        neighbor_idx <- (nc - 1) * ydim + nr
        neighbors <- c(neighbors, neighbor_idx)
      }
    }
    adj[[i]] <- neighbors
  }

  return(adj)
}

compute_te <- function(model) {
  codebook <- model$codes[[1]]  # matrix: n_neurons x 9
  n_neurons <- nrow(codebook)
  data_mat <- model$data[[1]]   # matrix: 138768 x 9
  n_obs <- nrow(data_mat)

  # Build adjacency set
  adj <- build_adjacency_set(model$grid)

  # Compute distances from each observation to all codebook vectors
  # Use efficient matrix operation: for each obs, dist to all neurons
  # dist_matrix: n_obs x n_neurons
  dist_matrix <- matrix(0, nrow = n_obs, ncol = n_neurons)
  for (j in 1:n_neurons) {
    diff <- sweep(data_mat, 2, codebook[j, ])
    dist_matrix[, j] <- sqrt(rowSums(diff^2))
  }

  # Find 1st and 2nd BMU for each observation
  non_adjacent_count <- 0L
  for (i in 1:n_obs) {
    dists_i <- dist_matrix[i, ]
    sorted_idx <- order(dists_i)
    bmu1 <- sorted_idx[1]
    bmu2 <- sorted_idx[2]

    if (!(bmu2 %in% adj[[bmu1]])) {
      non_adjacent_count <- non_adjacent_count + 1L
    }
  }

  te <- non_adjacent_count / n_obs
  return(te)
}
```

### 5.4 Performance Note

The nested loop over 138,768 observations is computationally expensive in pure R. Two optimization strategies:

**Option A -- Vectorize the adjacency check (recommended):**

```r
compute_te_fast <- function(model) {
  codebook <- model$codes[[1]]
  n_neurons <- nrow(codebook)
  data_mat <- model$data[[1]]
  n_obs <- nrow(data_mat)

  adj <- build_adjacency_set(model$grid)

  # Vectorized distance computation
  # dist_matrix[i, j] = euclidean distance from obs i to neuron j
  dist_matrix <- as.matrix(pdist::pdist(data_mat, codebook))
  # If pdist is unavailable, use:
  # dist_matrix <- sqrt(
  #   outer(rowSums(data_mat^2), rep(1, n_neurons)) -
  #   2 * data_mat %*% t(codebook) +
  #   outer(rep(1, n_obs), rowSums(codebook^2))
  # )

  # Get 1st and 2nd BMU indices (vectorized)
  bmu1 <- apply(dist_matrix, 1, which.min)

  # Mask out 1st BMU to find 2nd BMU
  # Set 1st BMU distance to Inf, then find min again
  for (i in 1:n_obs) {
    dist_matrix[i, bmu1[i]] <- Inf
  }
  bmu2 <- apply(dist_matrix, 1, which.min)

  # Check adjacency (vectorized via sapply)
  is_adjacent <- sapply(1:n_obs, function(i) bmu2[i] %in% adj[[bmu1[i]]])

  te <- sum(!is_adjacent) / n_obs
  return(te)
}
```

**Option B -- Chunk processing:** If memory is a concern (dist_matrix is 138,768 x 36 = ~40MB for float64), process in chunks of 10,000 observations. This is unlikely to be necessary on 24GB RAM.

### 5.5 Adjacency Validation

Before using the adjacency function in TE computation, validate it against known cases:

```r
validate_adjacency <- function() {
  # For a 5x5 hex grid:
  grid <- somgrid(xdim = 5, ydim = 5, topo = "hexagonal")
  adj <- build_adjacency_set(grid)

  # Corner neuron (1,1) should have exactly 2 neighbors on a hex grid
  # Neuron 1 = (col=1, row=1), odd row
  # Neighbors: (2,1) = idx 6, (1,2) = idx 2 -- but check offset rules
  cat("Neuron 1 neighbors:", adj[[1]], "\n")
  # Should be 2 or 3 neighbors (corner)

  # Center neuron should have 6 neighbors
  center_idx <- 13  # (col=3, row=3) = (3-1)*5 + 3 = 13
  cat("Neuron 13 (center) neighbors:", adj[[center_idx]], "\n")
  stopifnot(length(adj[[center_idx]]) == 6)

  cat("Adjacency validation passed.\n")
}
```

Run this validation before any TE computation. If it fails, the column-major indexing assumption is wrong and must be corrected by inspecting `model$grid$pts`.

### 5.6 Critical Warning: kohonen Grid Layout

The kohonen package's internal neuron ordering may differ from the `expand.grid(col, row)` assumption above. Before trusting the adjacency function, verify the mapping:

```r
# After training a model, check:
grid <- model$grid
print(grid$pts)  # Shows the (x, y) coordinates of each neuron

# The pts matrix should reveal the layout order.
# If neuron 1 has x=1, y=1 and neuron 2 has x=1, y=2 (column-major),
# the expand.grid assumption is correct.
# If neuron 2 has x=2, y=1 (row-major), swap the expand.grid order.
```

**This verification is mandatory.** Print `grid$pts` for the first model and confirm the indexing before proceeding. If the layout is row-major, change `expand.grid` to `expand.grid(row = 1:ydim, col = 1:xdim)` and adjust the linear index formula accordingly.

---

## 6. Seed Stability Analysis

### 6.1 Purpose

Verify that SOM cluster assignments are not artifacts of random initialization. If different seeds produce substantially different clusterings, the SOM is unreliable.

### 6.2 Metric: Adjusted Rand Index (ARI)

ARI measures agreement between two clusterings, adjusted for chance. Range: -1 to 1. ARI = 1 means perfect agreement; ARI = 0 means chance-level agreement.

**Target:** Mean pairwise ARI > 0.80 within each grid size.

### 6.3 Computation

For each grid size, compute the pairwise ARI between all 5 seeds (10 unique pairs per grid size).

**Important:** ARI requires cluster labels, not raw BMU indices. At this stage, use the raw BMU assignment (`model$unit.classif`) as the clustering. Later, after hierarchical clustering (Section 8), recompute ARI using the final cluster labels.

```r
compute_seed_stability <- function(results, grid_name, seeds) {
  # Extract BMU assignments for each seed
  assignments <- list()
  for (seed in seeds) {
    run_id <- paste0(grid_name, "_seed", seed)
    assignments[[as.character(seed)]] <- results[[run_id]]$model$unit.classif
  }

  # Pairwise ARI matrix
  n_seeds <- length(seeds)
  ari_matrix <- matrix(1, nrow = n_seeds, ncol = n_seeds,
                       dimnames = list(seeds, seeds))

  for (i in 1:(n_seeds - 1)) {
    for (j in (i + 1):n_seeds) {
      ari_val <- adjustedRandIndex(
        assignments[[i]],
        assignments[[j]]
      )
      ari_matrix[i, j] <- ari_val
      ari_matrix[j, i] <- ari_val
    }
  }

  mean_ari <- mean(ari_matrix[upper.tri(ari_matrix)])
  min_ari <- min(ari_matrix[upper.tri(ari_matrix)])

  cat(sprintf("  %s: mean_ARI=%.4f, min_ARI=%.4f, target=0.80\n",
              grid_name, mean_ari, min_ari))

  return(list(
    ari_matrix = ari_matrix,
    mean_ari = mean_ari,
    min_ari = min_ari,
    pass = mean_ari > 0.80
  ))
}
```

### 6.4 Interpretation

- **mean ARI > 0.80:** Good stability. Proceed with model selection.
- **mean ARI 0.60--0.80:** Moderate stability. Report as a limitation. Consider whether one grid size is more stable than the other and prefer the more stable one.
- **mean ARI < 0.60:** Poor stability. The SOM grid may be too large for the data structure, or the data does not have strong cluster structure. Do NOT proceed without investigation.

---

## 7. Model Selection Protocol

### 7.1 Two-Stage Selection

**Stage 1 -- Select grid size:**

1. Compare mean QE across 5 seeds for each grid size
2. Compare mean TE across 5 seeds for each grid size
3. Compare mean ARI (seed stability) for each grid size
4. Selection rule: Prefer the grid size where **both** conditions hold:
   - Mean TE < 0.10
   - Mean ARI > 0.80
5. If both grid sizes satisfy the conditions, prefer the one with lower mean QE
6. If neither satisfies TE < 0.10, select the one with lower TE and report the failure

**Stage 2 -- Select seed within chosen grid size:**

1. Among the 5 seeds for the chosen grid size, select the seed with the lowest QE
2. This is the final SOM model used for all downstream analysis

### 7.2 Decision Logging

Print the decision to console and record in the metrics JSON:

```r
cat(sprintf("\n=== MODEL SELECTION ===\n"))
cat(sprintf("Grid selected: %s (QE=%.4f, TE=%.4f, ARI=%.4f)\n",
            selected_grid, selected_qe, selected_te, selected_ari))
cat(sprintf("Seed selected: %d (QE=%.4f)\n", selected_seed, best_qe))
```

---

## 8. Post-SOM Hierarchical Clustering

### 8.1 Purpose

The SOM produces 25 or 36 neuron-level prototypes. Hierarchical clustering groups these prototypes into k interpretable contamination regimes (target: 4--8 clusters).

### 8.2 Input

The codebook vectors from the selected SOM model:

```r
codebook <- selected_model$codes[[1]]  # matrix: n_neurons x 9
```

### 8.3 Ward's D2 Clustering

```r
# Euclidean distance matrix on codebook vectors
codebook_dist <- dist(codebook, method = "euclidean")

# Ward's D2 hierarchical clustering
hc <- hclust(codebook_dist, method = "ward.D2")
```

### 8.4 Optimal k Selection

Test k = 2 through k = 10. Use three methods:

**Primary: Average silhouette width**

```r
sil_widths <- sapply(2:10, function(k) {
  labels <- cutree(hc, k = k)
  sil <- silhouette(labels, codebook_dist)
  mean(sil[, "sil_width"])
})
names(sil_widths) <- 2:10
k_sil <- as.integer(names(which.max(sil_widths)))
```

**Secondary: Gap statistic (B = 100 bootstrap samples)**

```r
set.seed(42)
gap_stat <- clusGap(codebook, FUN = function(x, k) {
  list(cluster = cutree(hclust(dist(x), method = "ward.D2"), k = k))
}, K.max = 10, B = 100)
k_gap <- maxSE(gap_stat$Tab[, "gap"], gap_stat$Tab[, "SE.sim"],
                method = "Tibs2001SEmax")
```

**Visual: Elbow method on within-cluster sum of squares**

```r
wss <- sapply(2:10, function(k) {
  labels <- cutree(hc, k = k)
  sum(sapply(unique(labels), function(cl) {
    members <- codebook[labels == cl, , drop = FALSE]
    sum(scale(members, scale = FALSE)^2)
  }))
})
```

The elbow is determined visually from the plot (no automatic selection).

### 8.5 k Selection Decision Rule

1. If silhouette and gap agree, use that k
2. If they disagree, use the silhouette-optimal k (primary criterion)
3. Override: if the selected k produces any cluster with fewer than 1,388 observations (1% of 138,768), decrement k by 1 and re-check. Repeat until the minimum-size constraint is satisfied.
4. Final k must be in range [3, 8]. If silhouette selects k = 2, use k = 3 instead (too few regimes to be scientifically interesting). If silhouette selects k > 8, use k = 8 (too many regimes to interpret meaningfully).

### 8.6 Assigning Observations to Clusters

Each observation's BMU is known from the SOM. Each BMU (neuron) is assigned to a cluster by cutting the dendrogram. Map this through:

```r
# Cut dendrogram at selected k
neuron_clusters <- cutree(hc, k = selected_k)

# Map each observation's BMU to its cluster
bmu_assignments <- selected_model$unit.classif  # length 138768
obs_clusters <- neuron_clusters[bmu_assignments]  # length 138768
```

### 8.7 Minimum Cluster Size Check

```r
cluster_sizes <- table(obs_clusters)
min_size <- min(cluster_sizes)
min_pct <- 100 * min_size / length(obs_clusters)

cat(sprintf("Cluster sizes: %s\n", paste(cluster_sizes, collapse=", ")))
cat(sprintf("Smallest cluster: %d (%.2f%%)\n", min_size, min_pct))

if (min_size < 1388) {
  warning(sprintf(
    "Cluster %s has only %d observations (%.2f%%). Decrementing k.",
    names(which.min(cluster_sizes)), min_size, min_pct
  ))
  # Decrement and reassign (implement as a while loop in practice)
}
```

---

## 9. Back-Transformation to Original Units

### 9.1 Purpose

Z-scored centroids are uninterpretable to domain experts. Convert back to original units using the IQR scaling parameters from `feature_summary.json`.

### 9.2 Formula

```
original_value = z_score * IQR + median
```

Where `IQR` and `median` come from `normalization_params` for each feature.

### 9.3 Centroid Computation

Compute the mean z-score across all observations in each cluster, then back-transform:

```r
compute_cluster_centroids <- function(som_matrix, obs_clusters, norm_params,
                                       som_features) {
  k <- length(unique(obs_clusters))

  # Z-scored centroids
  centroids_z <- t(sapply(1:k, function(cl) {
    colMeans(som_matrix[obs_clusters == cl, , drop = FALSE])
  }))
  colnames(centroids_z) <- som_features
  rownames(centroids_z) <- paste0("Cluster_", 1:k)

  # Back-transform to original units
  centroids_orig <- centroids_z
  for (j in seq_along(som_features)) {
    feat_z <- som_features[j]
    feat_raw <- sub("_z$", "", feat_z)  # Remove _z suffix
    med <- norm_params[[feat_raw]]$median
    iqr <- norm_params[[feat_raw]]$iqr
    centroids_orig[, j] <- centroids_z[, j] * iqr + med
  }
  colnames(centroids_orig) <- sub("_z$", "", som_features)

  return(list(z_scored = centroids_z, original = centroids_orig))
}
```

### 9.4 Interpretation Table

For each cluster, produce a row with:

| Column | Content |
|--------|---------|
| `cluster_id` | Integer (1 to k) |
| `cluster_label` | Auto-generated name (Section 10) |
| `n_observations` | Count of mesh cells |
| `pct_of_total` | Percentage of 138,768 |
| `total_decay_ratio` | Back-transformed centroid value |
| `log_linear_slope` | Back-transformed centroid value (day^-1) |
| ... | (all 9 features in original units) |

This becomes Table 1 in the manuscript.

---

## 10. Auto-Labeling Regime Names

### 10.1 Purpose

Assign descriptive names to clusters based on their centroid signatures. These are PROPOSALS for the analyst to review and override if needed.

### 10.2 Labeling Rules

Apply the following rules in order of priority. Each rule examines the z-scored centroid values (not back-transformed). A cluster matches a rule if its centroid z-score for the relevant feature(s) is the most extreme among all clusters.

**Rule 1 -- Rapid Decay:**
The cluster with the highest-magnitude (most negative) `early_slope_z` AND the lowest `residual_cv_z`.

More precisely: rank clusters by `early_slope_z` (ascending, since more negative = faster early decay). The cluster ranked #1 gets this label IF it also has below-median `residual_cv_z`.

**Rule 2 -- Slow Retention:**
The cluster with the lowest-magnitude slopes (closest to zero for both `log_linear_slope_z` and `early_slope_z`) AND the highest `total_decay_ratio_z` (i.e., final dose rate is high relative to initial).

Operationalize: the cluster with the highest `total_decay_ratio_z` centroid, provided its `log_linear_slope_z` is also the closest to zero (least negative).

**Rule 3 -- Recontamination/Anomalous:**
The cluster with the highest `n_increases_z` centroid.

**Rule 4 -- Consistent Exponential Decay:**
The cluster with the highest `log_linear_r2_z` centroid (best fit to exponential model).

**Rule 5 -- Fallback for remaining clusters:**
For any cluster not yet labeled, identify its most extreme z-score deviation from zero (in absolute value) across all 9 features. Name it by that feature:

| Most Extreme Feature | Proposed Label |
|---------------------|---------------|
| `late_slope_z` (very negative) | "Accelerating Late Decay" |
| `late_slope_z` (near zero) | "Stagnant Late Phase" |
| `slope_ratio_z` (very high) | "Front-Loaded Decay" |
| `slope_ratio_z` (very low) | "Delayed Decay Onset" |
| `temporal_span_years_z` (very low) | "Short Monitoring Record" |
| Other | "Regime [cluster_id]" |

### 10.3 Implementation

```r
auto_label_clusters <- function(centroids_z) {
  k <- nrow(centroids_z)
  labels <- rep(NA_character_, k)
  used <- rep(FALSE, k)

  # Rule 1: Rapid Decay -- most negative early_slope + low residual_cv
  early_rank <- order(centroids_z[, "early_slope_z"])
  for (idx in early_rank) {
    if (!used[idx] &&
        centroids_z[idx, "residual_cv_z"] < median(centroids_z[, "residual_cv_z"])) {
      labels[idx] <- "Rapid Decay"
      used[idx] <- TRUE
      break
    }
  }

  # Rule 2: Slow Retention -- highest total_decay_ratio + flattest slope
  if (any(!used)) {
    tdr_rank <- order(centroids_z[, "total_decay_ratio_z"], decreasing = TRUE)
    for (idx in tdr_rank) {
      if (!used[idx]) {
        labels[idx] <- "Slow Retention"
        used[idx] <- TRUE
        break
      }
    }
  }

  # Rule 3: Recontamination -- highest n_increases
  if (any(!used)) {
    ninc_rank <- order(centroids_z[, "n_increases_z"], decreasing = TRUE)
    for (idx in ninc_rank) {
      if (!used[idx]) {
        labels[idx] <- "Recontamination/Anomalous"
        used[idx] <- TRUE
        break
      }
    }
  }

  # Rule 4: Consistent Exponential -- highest R2
  if (any(!used)) {
    r2_rank <- order(centroids_z[, "log_linear_r2_z"], decreasing = TRUE)
    for (idx in r2_rank) {
      if (!used[idx]) {
        labels[idx] <- "Consistent Exponential Decay"
        used[idx] <- TRUE
        break
      }
    }
  }

  # Rule 5: Fallback -- most extreme deviation
  if (any(!used)) {
    for (idx in which(!used)) {
      deviations <- abs(centroids_z[idx, ])
      max_feat <- names(which.max(deviations))
      max_val <- centroids_z[idx, max_feat]

      label <- switch(max_feat,
        "late_slope_z" = if (max_val < 0) "Accelerating Late Decay"
                         else "Stagnant Late Phase",
        "slope_ratio_z" = if (max_val > 0) "Front-Loaded Decay"
                          else "Delayed Decay Onset",
        "temporal_span_years_z" = if (max_val < 0) "Short Monitoring Record"
                                  else "Extended Monitoring",
        paste0("Regime ", idx)
      )
      labels[idx] <- label
    }
  }

  return(labels)
}
```

### 10.4 Manual Override

The auto-labels are stored in the output but should be reviewed. Print them with centroid values:

```r
cat("\n=== AUTO-GENERATED LABELS (review before manuscript) ===\n")
for (i in 1:k) {
  cat(sprintf("Cluster %d -> \"%s\" (n=%d, %.1f%%)\n",
              i, labels[i], cluster_sizes[i],
              100 * cluster_sizes[i] / sum(cluster_sizes)))
  cat(sprintf("  Key z-scores: early_slope=%.2f, total_decay=%.2f, ",
              centroids_z[i, "early_slope_z"],
              centroids_z[i, "total_decay_ratio_z"]))
  cat(sprintf("n_increases=%.2f, R2=%.2f\n",
              centroids_z[i, "n_increases_z"],
              centroids_z[i, "log_linear_r2_z"]))
}
```

---

## 11. Output File Specifications

### 11.1 File Manifest

| File | Format | Content |
|------|--------|---------|
| `output/som_model.rds` | R binary | Full kohonen object for the selected model |
| `output/som_clusters.parquet` | Parquet | All 37 original columns + 3 new columns |
| `output/som_metrics.json` | JSON | Complete metrics, configuration, centroids |
| `output/som_cluster_summary.csv` | CSV | Per-cluster stats in original units (Table 1) |
| `output/som_cluster_assignments.csv` | CSV | Minimal assignment table for supplementary |

### 11.2 som_model.rds

```r
saveRDS(selected_model, "output/som_model.rds")
```

Contains the full kohonen object including codebook vectors, training changes, grid specification, and data reference. This enables re-running post-hoc analyses without retraining.

### 11.3 som_clusters.parquet

The original 37-column parquet with three appended columns:

| New Column | Type | Description |
|------------|------|-------------|
| `som_bmu` | integer | BMU neuron index (1 to 25 or 36) |
| `som_cluster` | integer | Hierarchical cluster assignment (1 to k) |
| `som_cluster_label` | string | Human-readable regime name |

```r
dt[, som_bmu := selected_model$unit.classif]
dt[, som_cluster := obs_clusters]
dt[, som_cluster_label := cluster_labels[obs_clusters]]

write_parquet(dt, "output/som_clusters.parquet")
```

**Row order:** Must match the input parquet exactly (same mesh_id ordering). Do NOT sort.

### 11.4 som_metrics.json

Structure:

```json
{
  "generated_at": "ISO-8601 timestamp",
  "pipeline_phase": "4_som_clustering",
  "input_file": "output/feature_matrix.parquet",
  "n_observations": 138768,
  "n_features": 9,

  "training_config": {
    "rlen": 2000,
    "alpha": [0.05, 0.01],
    "dist_fcts": "euclidean",
    "mode": "online",
    "seeds": [42, 123, 456, 789, 2024],
    "grid_sizes_tested": ["5x5", "6x6"]
  },

  "all_runs": [
    {
      "grid": "5x5",
      "seed": 42,
      "qe": 0.0,
      "te": 0.0,
      "converged": true,
      "rel_change": 0.0
    }
  ],

  "grid_comparison": {
    "5x5": {
      "mean_qe": 0.0,
      "mean_te": 0.0,
      "mean_ari": 0.0,
      "min_ari": 0.0,
      "ari_matrix": [[]]
    },
    "6x6": { "..." : "..." }
  },

  "selected_model": {
    "grid": "5x5 or 6x6",
    "seed": 42,
    "qe": 0.0,
    "te": 0.0,
    "n_neurons": 25,
    "selection_reason": "Lowest QE among seeds with TE < 0.10"
  },

  "hierarchical_clustering": {
    "method": "ward.D2",
    "distance": "euclidean",
    "k_selected": 5,
    "k_silhouette": 5,
    "k_gap": 5,
    "silhouette_widths": { "2": 0.0, "3": 0.0 },
    "gap_values": { "2": 0.0, "3": 0.0 }
  },

  "clusters": [
    {
      "cluster_id": 1,
      "label": "Rapid Decay",
      "n_observations": 30000,
      "pct_of_total": 21.6,
      "centroid_z": {
        "total_decay_ratio_z": -1.2,
        "...": "..."
      },
      "centroid_original": {
        "total_decay_ratio": 0.08,
        "...": "..."
      },
      "neurons_in_cluster": [1, 2, 7, 8]
    }
  ],

  "figures_generated": [
    "figures/fig01_geographic_regime_map.png",
    "..."
  ]
}
```

### 11.5 som_cluster_summary.csv

This is the manuscript's Table 1. Columns:

```
cluster_id, cluster_label, n_observations, pct_of_total,
total_decay_ratio, log_linear_slope, log_linear_r2,
early_slope, late_slope, residual_cv,
slope_ratio, n_increases, temporal_span_years
```

All feature values are back-transformed original-unit centroid means. Format numeric values to 4 significant figures.

### 11.6 som_cluster_assignments.csv

Minimal table for supplementary materials:

```
mesh_id, latitude, longitude, som_cluster, som_cluster_label
```

Where `mesh_id`, `latitude`, `longitude` are extracted from the original 37 columns. Determine the correct column names by inspecting the parquet schema at runtime.

---

## 12. Figure Specifications

All figures are saved to the `figures/` directory (create if it does not exist). Each figure is saved as both 300 DPI PNG and PDF vector. Each figure has a companion `_caption.txt` file with a draft figure caption.

```r
dir.create("figures", showWarnings = FALSE)
```

### Figure 1: Geographic Regime Map (Hero Figure)

**File:** `figures/fig01_geographic_regime_map.{png,pdf}`

**Content:** Scatterplot of all 138,768 mesh cells at their (longitude, latitude) coordinates, colored by `som_cluster_label`.

**Requirements:**
- Point size: 0.1 (dense point cloud)
- Color: `RColorBrewer::brewer.pal(k, "Set2")`
- FDNPP marker: black triangle at (141.0328, 37.4211), size 3, with text annotation "FDNPP"
- Scale bar: bottom-left, 10 km (`ggspatial::annotation_scale()`)
- North arrow: top-right (`ggspatial::annotation_north_arrow()`)
- Legend: positioned outside right margin
- `coord_fixed()` to preserve geographic aspect ratio
- Title: "Contamination Regimes in the Fukushima Exclusion Zone"

```r
p1 <- ggplot(dt, aes(x = longitude, y = latitude, color = som_cluster_label)) +
  geom_point(size = 0.1, alpha = 0.6) +
  scale_color_brewer(palette = "Set2", name = "Regime") +
  annotate("point", x = 141.0328, y = 37.4211,
           shape = 17, size = 3, color = "black") +
  annotate("text", x = 141.0328, y = 37.43, label = "FDNPP",
           fontface = "bold", size = 3) +
  annotation_scale(location = "bl", width_hint = 0.2) +
  annotation_north_arrow(location = "tr", which_north = "true",
                         style = north_arrow_fancy_orienteering()) +
  coord_fixed() +
  labs(title = "Contamination Regimes in the Fukushima Exclusion Zone",
       x = "Longitude", y = "Latitude") +
  theme_minimal(base_size = 11)
```

**Note on column names:** The longitude and latitude columns in the parquet may be named differently (e.g., `lon`, `long`, `x`). Inspect the parquet schema and adapt accordingly.

### Figure 2: Cluster Centroids Heatmap

**File:** `figures/fig02_centroid_heatmap.{png,pdf}`

**Content:** Heatmap with k rows (clusters) and 9 columns (features). Values are z-scored centroid means. Color scale: diverging blue-white-red (`scale_fill_gradient2`).

**Requirements:**
- Rows labeled with cluster name and sample size: "Rapid Decay (n=30,000)"
- Columns labeled with human-readable feature names
- Color midpoint at 0
- Cell values printed inside tiles (2 decimal places)
- Use `geom_tile()` + `geom_text()`

### Figure 3: Parallel Coordinates Plot

**File:** `figures/fig03_parallel_coordinates.{png,pdf}`

**Content:** One line per cluster across 9 features (z-scored centroid values). X-axis: features (in the canonical order from Section 1.2). Y-axis: z-score.

**Requirements:**
- Line color: cluster palette (Set2)
- Line width: 1.2
- Points at each feature value: size 2
- Horizontal dashed line at y = 0 (population median reference)
- Human-readable x-axis labels, rotated 45 degrees
- Legend: cluster labels

### Figure 4: Component Planes

**File:** `figures/fig04_component_planes.{png,pdf}`

**Content:** 3x3 grid of hex maps, one per feature. Each hex map shows the SOM grid with neurons colored by that feature's codebook value.

**Requirements:**
- Use `viridis` color scale per panel
- Panel titles: human-readable feature names
- Hex topology must be rendered correctly (use kohonen's built-in `plot(model, type="property")` or replicate with ggplot using hexagonal coordinates from `model$grid$pts`)
- Assembled with `patchwork`

**Implementation note:** The kohonen package's `plot()` function with `type = "property"` renders component planes but produces base R graphics. To maintain ggplot consistency, either:
- Option A: Use `plot()` and save with `png()` / `pdf()` (simpler, acceptable)
- Option B: Extract hex coordinates from `model$grid$pts` and render with `ggplot2::geom_hex()` or custom polygon mapping (more work, consistent style)

Choose Option A for speed; Option B for style consistency. Document which was used.

### Figure 5: U-Matrix with Cluster Boundaries

**File:** `figures/fig05_umatrix.{png,pdf}`

**Content:** The U-matrix (unified distance matrix) showing inter-neuron distances on the SOM grid, with cluster boundaries overlaid.

**Requirements:**
- Color: viridis (dark = low distance/similar, bright = high distance/dissimilar)
- Cluster boundaries: thick black lines separating neurons of different clusters
- Cluster labels at cluster centroids on the grid

**Implementation:**
```r
# U-matrix: for each neuron, mean distance to its grid neighbors
# kohonen does not provide this directly; compute from codebook + adjacency
compute_umatrix <- function(model, adj) {
  codebook <- model$codes[[1]]
  n_neurons <- nrow(codebook)
  u_values <- numeric(n_neurons)

  for (i in 1:n_neurons) {
    neighbors <- adj[[i]]
    if (length(neighbors) > 0) {
      dists <- sapply(neighbors, function(j) {
        sqrt(sum((codebook[i, ] - codebook[j, ])^2))
      })
      u_values[i] <- mean(dists)
    }
  }
  return(u_values)
}
```

### Figure 6: Training Convergence (Supplementary)

**File:** `figures/fig06_convergence.{png,pdf}`

**Content:** Line plot of `som_model$changes` (y-axis: mean distance, x-axis: iteration 1 to 2000). Show all 10 trainings (5 seeds x 2 grids) as separate lines.

**Requirements:**
- Facet by grid size (2 panels)
- Color by seed
- Vertical dashed line at iteration 1500 (expected convergence)
- Log-scale y-axis if changes span multiple orders of magnitude

### Figure 7: Node Counts (Supplementary)

**File:** `figures/fig07_node_counts.{png,pdf}`

**Content:** Hex map of the selected SOM showing how many observations mapped to each neuron.

**Requirements:**
- Color: viridis, log-scaled if counts are highly skewed
- Number printed inside each hex cell
- Title: "Observation Count per SOM Neuron"

### Figure 8: Dendrogram (Supplementary)

**File:** `figures/fig08_dendrogram.{png,pdf}`

**Content:** Ward's D2 dendrogram of codebook vectors with branches colored by cluster assignment.

**Requirements:**
- Horizontal cut line at the selected k
- Branch colors match cluster palette (Set2)
- Leaf labels: neuron index
- Use base R `plot(hc)` + `rect.hclust()` or convert to ggplot with `ggdendro`

### Figure 9: Silhouette Plot (Supplementary)

**File:** `figures/fig09_silhouette.{png,pdf}`

**Content:** Silhouette plot for the selected k, at the observation level (not neuron level).

**Important:** Computing silhouette for 138,768 observations requires the full distance matrix, which is too large (138k x 138k). Instead, compute silhouette at the **neuron level** (25 or 36 neurons) using codebook distances, weighted by node counts.

```r
neuron_sil <- silhouette(neuron_clusters, codebook_dist)
plot(neuron_sil, col = brewer.pal(selected_k, "Set2"),
     main = sprintf("Silhouette Plot (k=%d, neuron-level)", selected_k))
```

### Figure 10: Optimal k Selection Panel (Supplementary)

**File:** `figures/fig10_optimal_k.{png,pdf}`

**Content:** 3-panel figure:
- Left: Silhouette width vs. k (2--10), with optimal k marked
- Center: Within-cluster sum of squares vs. k (elbow plot)
- Right: Gap statistic vs. k with error bars (SE), optimal k marked

**Requirements:**
- Assembled with `patchwork` into a single row
- Consistent x-axis (k = 2 to 10)
- Optimal k highlighted with a red vertical dashed line

### Figure 11: Seed Stability Heatmap (Supplementary)

**File:** `figures/fig11_seed_stability.{png,pdf}`

**Content:** Two heatmaps (one per grid size), each showing the 5x5 pairwise ARI matrix across seeds.

**Requirements:**
- Color: viridis
- Cell values printed (2 decimal places)
- Diagonal = 1.00 (by definition)
- Title includes mean ARI

### Figure 12: Cluster Size Bar Chart (Supplementary)

**File:** `figures/fig12_cluster_sizes.{png,pdf}`

**Content:** Bar chart with one bar per cluster, height = number of observations.

**Requirements:**
- Bar color: cluster palette (Set2)
- Bar labels: count and percentage
- X-axis: cluster labels
- Y-axis: observation count (comma-formatted)
- Horizontal dashed line at 1% threshold (1,388)

### Figure 13: Per-Cluster Boxplots (Supplementary)

**File:** `figures/fig13_cluster_boxplots.{png,pdf}`

**Content:** 9-panel figure (3x3 grid), one panel per feature. Each panel shows k side-by-side boxplots (one per cluster) of the z-scored feature values.

**Requirements:**
- Facet wrap by feature with human-readable titles
- Box fill color: cluster palette (Set2)
- Horizontal dashed line at y = 0
- `theme(legend.position = "bottom")`
- Outliers: small grey dots (size = 0.05) since n = 138k

---

## 13. Figure Formatting Standards

All figures must follow these conventions for visual consistency.

### 13.1 Theme

```r
theme_pub <- theme_minimal(base_size = 11) +
  theme(
    plot.title = element_text(face = "bold", size = 12),
    axis.title = element_text(size = 10),
    legend.title = element_text(face = "bold", size = 10),
    strip.text = element_text(face = "bold", size = 10)
  )
```

Apply `theme_pub` to all ggplot figures.

### 13.2 Color Palettes

- **Categorical clusters:** `RColorBrewer::brewer.pal(k, "Set2")` where k is the number of clusters. If k > 8, extend with `colorRampPalette(brewer.pal(8, "Set2"))(k)`.
- **Continuous scales:** `viridis::scale_fill_viridis()` or `viridis::scale_color_viridis()`
- **Diverging scales (heatmaps):** `scale_fill_gradient2(low = "#2166AC", mid = "white", high = "#B2182B", midpoint = 0)` (RdBu-inspired)

### 13.3 Saving

```r
save_figure <- function(plot_obj, filename, width = 8, height = 6) {
  ggsave(paste0("figures/", filename, ".png"), plot_obj,
         width = width, height = height, dpi = 300, bg = "white")
  ggsave(paste0("figures/", filename, ".pdf"), plot_obj,
         width = width, height = height, bg = "white")
}
```

For base R plots (component planes, dendrogram), use:

```r
save_base_figure <- function(plot_fn, filename, width = 8, height = 6) {
  png(paste0("figures/", filename, ".png"),
      width = width, height = height, units = "in", res = 300)
  plot_fn()
  dev.off()

  pdf(paste0("figures/", filename, ".pdf"),
      width = width, height = height)
  plot_fn()
  dev.off()
}
```

### 13.4 Captions

Each figure gets a companion `_caption.txt` file:

```r
save_caption <- function(filename, caption_text) {
  writeLines(caption_text, paste0("figures/", filename, "_caption.txt"))
}
```

Draft captions should follow journal style: "Figure N. [Brief descriptive title]. [Methodological detail]. [Key observation]."

### 13.5 Figure Dimensions

| Figure | Width (in) | Height (in) | Rationale |
|--------|-----------|-------------|-----------|
| Fig 1 (geographic map) | 8 | 10 | Portrait for geographic extent |
| Fig 2 (heatmap) | 10 | 5 | Wide for 9 feature columns |
| Fig 3 (parallel coords) | 10 | 5 | Wide for 9 features |
| Fig 4 (component planes) | 10 | 10 | Square 3x3 grid |
| Fig 5 (U-matrix) | 6 | 6 | Square |
| Fig 6 (convergence) | 10 | 4 | Wide time series |
| Fig 7 (node counts) | 6 | 6 | Square hex map |
| Fig 8 (dendrogram) | 8 | 5 | Wide for tree |
| Fig 9 (silhouette) | 6 | 5 | Standard |
| Fig 10 (optimal k) | 12 | 4 | Three-panel row |
| Fig 11 (ARI heatmap) | 10 | 4 | Two-panel row |
| Fig 12 (bar chart) | 8 | 5 | Standard |
| Fig 13 (boxplots) | 12 | 10 | 3x3 facet grid |

---

## 14. Physical Context for Interpretation

### 14.1 Expected Regime Structure

Based on Andoh et al. (2020) and the physics of Cs-137 environmental transport, the SOM should discover regimes corresponding to:

| Expected Regime | Physical Mechanism | Signature in Features |
|----------------|-------------------|----------------------|
| Rapid urban washoff | Rain-driven removal from impervious surfaces | Large negative early_slope, high slope_ratio, low total_decay_ratio, high R^2 |
| Moderate agricultural decay | Tillage mixes Cs into soil, moderate washoff | Moderate slopes, intermediate total_decay_ratio |
| Slow forest retention | Cs trapped in organic litter layer, recycled by trees | Slopes near zero, high total_decay_ratio, long temporal span |
| Anomalous/recontamination | Flood redeposition, resuspension, or decontamination reversal | High n_increases, low R^2, high residual_cv |

### 14.2 Half-Life Reference Values

From Andoh et al. (2020):
- Urban surfaces: ecological half-life approximately 0.5--1 year
- Agricultural land: approximately 2--3 years
- Forest floors: approximately 5+ years (some approaching physical Cs-137 half-life of 30.17 years)
- The log-linear slope relates to half-life as: `t_half = ln(2) / abs(slope)`

### 14.3 FDNPP Location

The Fukushima Daiichi Nuclear Power Station coordinates for all maps:
- **Latitude:** 37.4211
- **Longitude:** 141.0328

### 14.4 Interpretation Caveats

Include these caveats in any interpretation section:

1. The SOM discovers statistical structure, not physical causation. Regime labels are interpretive proposals, not measured land-use categories.
2. The 250m mesh resolution means each cell may contain mixed land-use types. A cell labeled "forest" by the SOM may contain a mix of forest and road surface.
3. The 9 temporal features were derived from dose-rate time series that include measurement uncertainty, spatial averaging within mesh cells, and seasonal variation. The SOM clusters reflect patterns in these processed features, not raw physical measurements.
4. The IQR scaling and [-3, 3] clipping preserves relative differences but attenuates extreme outliers. Any cluster driven by outlier behavior (e.g., extreme recontamination events) may be under-represented.

---

## 15. Execution Order and Runtime Estimates

### 15.1 Script Execution Order

The script should execute in this exact sequence:

```
1. Package loading and configuration
2. Data loading and sanity checks
3. SOM training loop (10 models)
4. Quality metric computation (QE for all 10)
5. Topographic error computation (all 10)  [SLOW]
6. Seed stability analysis (ARI matrices)
7. Model selection (grid + seed)
8. Hierarchical clustering on selected model
9. Optimal k selection
10. Cluster assignment and size validation
11. Back-transformation of centroids
12. Auto-labeling
13. Output file writing
14. Figure generation (all 13)
15. Final summary printout
```

### 15.2 Runtime Estimates (M4 MacBook Air, 24GB RAM)

| Step | Estimated Time | Notes |
|------|---------------|-------|
| Data loading | < 5 seconds | Parquet is fast |
| Each SOM training | 3--8 minutes | 138k obs x 9 features x 2000 iterations, online mode |
| 10 SOM trainings total | 30--80 minutes | The bottleneck |
| TE computation (each) | 1--3 minutes | Matrix distance + adjacency check |
| TE computation (10 total) | 10--30 minutes | |
| ARI computation | < 5 seconds | Fast pairwise comparison |
| Hierarchical clustering | < 1 second | Only 25 or 36 codebook vectors |
| Figure generation | 2--5 minutes | Geographic map is the slowest (138k points) |
| **Total estimated** | **45--120 minutes** | |

### 15.3 Progress Reporting

Print progress messages at each major step. Include elapsed time:

```r
t_start <- Sys.time()

log_progress <- function(msg) {
  elapsed <- round(difftime(Sys.time(), t_start, units = "mins"), 1)
  cat(sprintf("[%s min] %s\n", elapsed, msg))
}

log_progress("Starting SOM training loop...")
# ... training code ...
log_progress("SOM training complete.")
```

### 15.4 Checkpointing

After completing all 10 SOM trainings, save the intermediate results to avoid retraining if the script crashes during post-processing:

```r
saveRDS(results, "output/som_all_models_checkpoint.rds")
log_progress("Checkpoint saved: output/som_all_models_checkpoint.rds")
```

If the checkpoint file exists at script start, offer to reload instead of retraining:

```r
checkpoint_file <- "output/som_all_models_checkpoint.rds"
if (file.exists(checkpoint_file)) {
  cat("Checkpoint found. Loading previously trained models...\n")
  results <- readRDS(checkpoint_file)
  cat(sprintf("Loaded %d models from checkpoint.\n", length(results)))
} else {
  # Run training loop
}
```

---

## Appendix A: Feature Normalization Parameters

Extracted from `output/feature_summary.json` for reference. These are the exact values needed for back-transformation.

| Feature | Median | IQR | Q25 | Q75 |
|---------|--------|-----|-----|-----|
| total_decay_ratio | 0.20000 | 0.15927 | 0.13448 | 0.29375 |
| log_linear_slope | -0.000361 | 0.000147 | -0.000436 | -0.000289 |
| log_linear_r2 | 0.71262 | 0.10139 | 0.65559 | 0.75697 |
| early_slope | -0.000794 | 0.001003 | -0.001312 | -0.000309 |
| late_slope | -0.000253 | 0.000128 | -0.000320 | -0.000192 |
| residual_cv | 0.18224 | 0.22003 | 0.12514 | 0.34517 |
| slope_ratio | 3.23915 | 2.83662 | 2.02947 | 4.86608 |
| n_increases | 5.00000 | 3.00000 | 4.00000 | 7.00000 |
| temporal_span_years | 13.64271 | 0.62149 | 13.02122 | 13.64271 |

---

## Appendix B: Correlation Structure of SOM Features

Notable correlations from `feature_summary.json` (absolute r > 0.5):

| Feature Pair | r | Implication |
|-------------|---|-------------|
| log_linear_slope / late_slope | 0.886 | Overall slope dominated by late-phase behavior |
| log_linear_slope / residual_cv | -0.600 | Faster decay = lower residual variability |
| total_decay_ratio / early_slope | 0.594 | More early decay = lower final/initial ratio |
| total_decay_ratio / residual_cv | -0.504 | More decay = more consistent trajectories |
| total_decay_ratio / late_slope | 0.504 | Late-phase slope contributes to total decay |
| late_slope / residual_cv | -0.560 | Faster late decay = less residual noise |
| early_slope / slope_ratio | -0.753 | Mechanically linked (slope_ratio = early/late) |

These correlations are expected and should NOT be addressed by PCA or feature removal. The SOM handles correlated inputs natively, and preserving individual features is required for SHAP interpretability in Phase 6.

---

## Appendix C: Checklist for the Implementer

Before running the script:
- [ ] `output/feature_matrix.parquet` exists and has 138,768 rows
- [ ] `output/feature_summary.json` exists and is valid JSON
- [ ] All R packages from Section 2.1 are installed
- [ ] `figures/` directory is created (or script creates it)

After running the script:
- [ ] All 10 SOM models converged (warnings reported if not)
- [ ] TE < 0.10 for the selected model
- [ ] Mean ARI > 0.80 for the selected grid size
- [ ] All clusters have >= 1,388 observations
- [ ] 5 output files written (Section 11.1)
- [ ] 13 figures generated in PNG + PDF (Section 12)
- [ ] 13 caption files generated
- [ ] Auto-labels reviewed and overridden if needed
- [ ] `som_metrics.json` contains complete configuration and results
- [ ] Console output shows full decision trail (grid selection, seed selection, k selection)

---

*End of specification. This document is the authoritative reference for Phase 4 implementation. Any ambiguity should be resolved by consulting `research-objective.md` or asking the analyst.*
